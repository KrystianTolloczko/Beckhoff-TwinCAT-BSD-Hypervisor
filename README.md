# Beckhoff TwinCAT BSD Hypervisor Example Configs

Beckhoff github: https://github.com/Beckhoff/TCBSD_Hypervisor_Samples

In the following sample Debian is to be installed in a virtual machine under TwinCAT/BSD. The shell scripts
from the GitHub repository https://github.com/Beckhoff/TCBSD_Hypervisor_Samples/tree/main/vm_autostart
can be used as a template.


```
vm_autostart
├── Makefile
├── rc.d
│ └── vmubuntu
└── vmubuntu
```

The bhyve call implies the following configuration steps:
1. the creation of virtual hard disks
2. the use of ISO images
3. booting a UEFI-based virtual machine
4. VNC based interaction with the virtual machine and
5. the configuration of a bridged network
Proceed as follows for the installation:
1. As described in the chapter ZFS data sets as storage location for virtual machines [} 56], a ZFS data set
is first created for the virtual machine.
`doas zfs create -p -o mountpoint=/vms/vmubuntu zroot/vms/vmubuntu`
2. For the installation of the operating system the Debian "network install" CD-ISO should be used. The ISO
file can be downloaded with fetch(8), as described in the chapter Use of installation media (ISO
images) [} 58], and later passed to the bhyve call as a disk of an ahci-hd device:

```
doas fetch -o /vms/debian-installer.iso https://cdimage.debian.org/debian-cd/current/amd64/isocd/
debian-11.5.0-amd64-netinst.iso
```

3. To be able to install Debian on a virtual hard disk, the following command creates a ZFS volume which is
used as backend for the emulated nvme device in the upper bhyve call:
`doas zfs create -V 20G zroot/vms/vmubuntu/disk0`
4. Debian uses EFI variables to store information about bootable disks. Therefore, a copy of the /usr/
local/share/uefi-firmware/BHYVE_BHF_UEFI_VARS.fd file should be placed on the ZFS data
set for the virtual machine:
`doas cp /usr/local/share/uefi-firmware/BHYVE_BHF_UEFI_VARS.fd /vms/vmubuntu/EFI_VARS.fd`
5. For the Debian installation, the virtual machine requires an Internet connection. For this purpose, a
bridged network [} 60] is created on the TwinCAT/BSD host to which the virtual machine is connected
via a virtio-net based network interface and the tap0 instance.
6. In addition, the virtual machine should be able to be operated via a VNC connection in order to be able to
use the graphical installation of the Debian installer at the first start. The vmubuntu shell script therefore
configures the packet filter rules to allow incoming TCP connections on port 5900 of the TwinCAT/BSD
host.
7. With the customized bhyve(8) call within the vmubuntu shell script, the virtual machine can be started
as follows:
`doas sh vmubuntu start` 
or 
`doas service vmubuntu enable`
and 
`doas service vmubuntu start`
8. The started bhyve(8) process then generates the following output on the command line:
fbuf frame buffer base: 0x881e00000 [sz 16777216]
9. Now a VNC client, such as Ultra-VNC can be used to connect to the virtual machine

Once the installation of Debian is complete, the virtual machine is restarted. This will terminate the VNC
connection. After reconnecting, the Debian operating system can be used in the virtual machine.

# **Autostart script -rename 'vmubuntu'**

```
#!/bin/sh
# SPDX-License-Identifier: 0BSD
# Copyright (C) 2022 Beckhoff Automation GmbH & Co. KG

set -e
set -u

usage() {
	cat >&2 << EOF
USAGE: ${0##*/} [COMMAND]
Control the virtual machine environment named '${vm_name}'
COMMANDS:
    start       Start the VM
    stop        Shutdown the running VM
    help        Print this help message
EXAMPLES
# run the virual machine
${0##*/} start
# shutdown the virtual machine
${0##*/} stop
EOF
}

# Cleanup any resources used by the VM after shutdown
cleanup() {
	# do not trap during cleanup procedure
	trap '' EXIT
	set +e
	set +u

	pfctl -a "bhf/bhyve/${vm_name}" -f /dev/null > /dev/null 2>&1
	if test -e "${temp_firewall_rules}"; then
		rm "${temp_firewall_rules}"
	fi
}

prepare_host() {
	# Ensure that kernel modul vmm.ko is loaded
	kldload -n vmm.ko

	# accept incoming VNC connections
	temp_firewall_rules="/etc/pf.conf.d/${vm_name}"
	printf "pass in quick proto tcp to port %s\n" "${vm_vnc_port}" > "${temp_firewall_rules}"
	pfctl -a bhf/bhyve/"${vm_name}" -f "${temp_firewall_rules}"
}

# Setups the VM environment and actually executes the bhyve process
run_vm() {
	trap 'cleanup' EXIT

	prepare_host

	while true; do

		if test -e "/dev/vmm/${vm_name}"; then
			bhyvectl --vm="${vm_name}" --destroy
		fi

		_bhyve_rc=0
		bhyve \
			-c sockets=1,cores=2,threads=2 \
			-m 4G \
			-l bootrom,/usr/local/share/uefi-firmware/BHYVE_BHF_UEFI.fd,/vms/vmubuntu/EFI_VARS.fd \
			-s 0,hostbridge \
			-s 2,fbuf,rfb=0.0.0.0:"${vm_vnc_port}",w=1280,h=1024 \
			-s 3,xhci,tablet \
			-s 10,nvme,/dev/zvol/zroot/vms/vmubuntu/disk0 \
			-s 20,virtio-net,tap0 \
			-s 31,lpc \
			-A -H -P -w \
			"${vm_name}" || _bhyve_rc=$?

# Ubuntu Installer (add after -s10)
# -s 15,ahci-cd,/vms/ubuntu-installer.iso,ro \

		if test "${_bhyve_rc}" -ne 0; then
			break
		fi

	done
}

shutdown_vm() {
	# do not trap on exit during shutdown command
	trap '' EXIT
	set +e
	set +u

	kill "${_pid}"
}

get_bhyve_pid() {
	printf "%s" "$(pgrep -f "bhyve: ${vm_name}")"
}

# Execution of virtual machines requires root previleges
if test "$(id -u)" -ne 0; then
	printf "%s must be run as root\n" "${0##*/}"
	exit 1
fi

# Default values for VM configuration
readonly vm_name="vmubuntu"
readonly vm_vnc_port="5900"
readonly _cmd="${1?Error: No COMMAND specified$(usage)}"

_pid="$(get_bhyve_pid)"
case "${_cmd}" in
	start)
		if test -z "${_pid:+x}"; then
			run_vm
		else
			printf "%s is already running with pid: %s\n" "${vm_name}" "${_pid}"
			exit 1
		fi
		;;

	stop)
		if ! test -z "${_pid:+x}"; then
			shutdown_vm
		fi
		;;

	status)
		if ! test -z "${_pid:+x}"; then
			printf "'%s' is running with pid: %s\n" "${vm_name}" "${_pid}"
		else
			printf "'%s' is not running.\n" "${vm_name}"
		fi
		;;

	-h | --help | help)
		usage
		;;

	*)
		usage
		printf "Unknown COMMAND: '%s'\n" "${_cmd}"
		exit 1
		;;
esac
```


you need also makefile and rc.d file to start the script as system service. see both scripts below:

# **Makefile-rename**
```
PREFIX?=/usr/local
BINDIR=$(DESTDIR)$(PREFIX)/bin
RCDIR=$(DESTDIR)$(PREFIX)/etc/rc.d

INSTALL=/usr/bin/install
MKDIR=/bin/mkdir

SCRIPT=vmubuntu

install:
	$(MKDIR) -p $(BINDIR)
	$(INSTALL) -m 544 $(SCRIPT) $(BINDIR)/

	$(MKDIR) -p $(RCDIR)
	$(INSTALL) -m 555 rc.d/* $(RCDIR)/
```

# **Rc.d/vmubuntu-rename**

```
#!/bin/sh
# SPDX-License-Identifier: 0BSD
# Copyright (C) 2019 - 2022 Beckhoff Automation GmbH & Co. KG

# PROVIDE: vmubuntu
# REQUIRE: DAEMON NETWORKING dmesg
# BEFORE: LOGIN TcSystemService
# KEYWORD: nojail shutdown

. /etc/rc.subr

name="vmubuntu"
rcvar="${name}_enable"

command="/usr/local/bin/${name}"

export PATH="${PATH}:/usr/local/bin:/usr/local/sbin"

start_cmd="${command} start > /dev/null 2>&1 &"
stop_cmd="${command} stop > /dev/null 2>&1 &"
status_cmd="${command} status"

load_rc_config "${name}"
run_rc_command "$1"
```

quick test of running vms:  `doas service vmubuntu status` or `ps -a | grep bhyve`

