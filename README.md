## Juniper ZTP upgrade

### Objective
This script will automate the upgrade of a Juniper device to a final Junos version using
different Junos firmwares. So far tested only on a few QFX (QFX10008/16, QFX5200) switches
running Junos 15.1 and automatically upgraded to 17.4->18.4->19.4->20.2 without human 
interaction

The idea is to use the Juniper ZTP feature that is enabled by default in most of the switches.
The Juniper device will boot and receive via DHCP a configuration file with a pointer to a
SLAX script. The script will be in charge of upgrading the device to the desired Junos version

### Benefits
Just power on your switch, connect the management interface and the switch will upgrade itself.
A syslog message will be sent once the process is over.

On devices supporting ZTP where this feature is not enabled by default one needs to enable ZTP
first via the following command:
```
{master:0}[edit]
vagrant@virtual-QFX-1# set chassis auto-image-upgrade
vagrant@virtual-QFX-1# commit
configuration check succeeds
commit complete
```

### Requirements
This solution requires the following:
1. One Juniper device to be upgraded
2. DHCP server configured with a pool of ip address and the Juniper ZTP extensions
3. Junos initial configuration file (`network-base-dhcp.conf`) provided by the DHCP server
4. Web server providing:
   - All Junos firmware files to be used (ie: `jinstall*`)
   - ASCII file called `firmwares` containing the list of Junos files to use
   - The script `ztp-upgrade.slax`

### Minimal requirements
In case you want to avoid the DHCP configuration and give it a try you still need:
1. One Juniper device to be upgraded
2. ~~DHCP server configured with a pool of ip address and the Juniper ZTP extensions~~
3. ~~Junos initial configuration file (`network-base-dhcp.conf`) provided by the DHCP server~~
4. Web server providing:
   - All Junos firmware files to be used (ie: `jinstall*`)
   - ASCII file called `firmwares` containing the list of Junos files to use
   - The script `ztp-upgrade.slax`

Then log into the Junos device to be upgraded and run the following commands (assuming 10.1.1.1 is the ip address of the web server):
```
root@qfx5200> configure
root@qfx5200# set event-options generate-event ztp-upgrade time-interval 60
root@qfx5200# set event-options policy ztp-upgrade events ztp-upgrade
root@qfx5200# set event-options policy ztp-upgrade then execute-commands commands "op url http://10.1.1.1/ztp-upgrade.slax output syslog server 10.1.1.1"
root@qfx5200# commit
```

The Web server should have the following files available:
```
[root@server html]# pwd
/var/www/html
[root@server html]# tree
.
├── firmwares
├── ztp-upgrade.slax
└── images
    ├── jinstall-host-qfx-5e-flex-x86-64-17.4R3.16-secure-signed.tgz
    ├── jinstall-host-qfx-5e-flex-x86-64-18.4R1.8-secure-signed.tgz
    ├── jinstall-host-qfx-5e-flex-x86-64-18.4R3.3-secure-signed.tgz
    ├── jinstall-host-qfx-5e-flex-x86-64-19.3R2.9-secure-signed.tgz
    ├── jinstall-host-qfx-5e-flex-x86-64-19.4R1.10-secure-signed.tgz
    ├── jinstall-host-qfx-5e-flex-x86-64-19.4R2.6-secure-signed.tgz
    ├── jinstall-host-qfx-5e-flex-x86-64-20.1R1.11-secure-signed.tgz
    └── jinstall-host-qfx-5e-flex-x86-64-20.2R1.10-secure-signed.tgz
```
You need to create a `firmwares` file containing the list of junos firmwares to be used per model
```
cat /var/www/html/firmwares
# No whitespaces between comma separated items
# You can specify multiple entries, so different upgrades will be performed
#
# NOTE: Only one firmware version per major junos release will be used
# In case of more than one version per major junos release, only the first
# will be used. In the example below version 18.4R3.3 will be skipped
# for a QFX10016 chassis

# <Model,junos_firmware>
# Model is given by the command "show version | match Model"

# QFX5200
# 15.1 => 17.4 => 18.4 => 19.4 => 20.2
qfx5200-32c-32q,jinstall-host-qfx-5e-flex-x86-64-17.4R3.16-secure-signed.tgz
qfx5200-32c-32q,jinstall-host-qfx-5e-flex-x86-64-18.4R3.3-secure-signed.tgz
qfx5200-32c-32q,jinstall-host-qfx-5e-flex-x86-64-18.4R1.8-secure-signed.tgz
qfx5200-32c-32q,jinstall-host-qfx-5e-flex-x86-64-18.4R2.7-secure-signed.tgz
qfx5200-32c-32q,jinstall-host-qfx-5e-flex-x86-64-18.3R1.9-secure-signed.tgz
qfx5200-32c-32q,jinstall-host-qfx-5e-flex-x86-64-19.4R2.6-secure-signed.tgz
qfx5200-32c-32q,jinstall-host-qfx-5e-flex-x86-64-19.3R1.8-secure-signed.tgz
qfx5200-32c-32q,jinstall-host-qfx-5e-flex-x86-64-20.2R1.10-secure-signed.tgz
```


