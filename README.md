## Juniper ZTP upgrade

- [Objective](#Objective)
- [Benefits](#Benefits)
- [Requirements](#Requirements)
   - [Minimal requirements](#minimal-requirements)
- [DHCP](#DHCP)
- [Examples](#Examples)
   - [Upgrade QFX5200 from Junos 20.1 to 20.2](#Upgrade-QFX5200-from-Junos-201-to-202)

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
1. Update the server ip address in the `ztp-upgrade.slax` script (variable `$SERVER`)
2. One Juniper device to be upgraded
3. DHCP server configured with a pool of ip address and the Juniper ZTP extensions
4. Junos initial configuration file (`network-base-dhcp.conf`) provided by the DHCP server
5. Web server providing:
   - All Junos firmware files to be used (ie: `jinstall*`)
   - ASCII file called `firmwares` containing the list of Junos files to use
   - The script `ztp-upgrade.slax`

### Minimal requirements
In case you want to avoid the DHCP configuration and give it a try you still need:
1. Update the server ip address in the `ztp-upgrade.slax` script (variable `$SERVER`)
2. One Juniper device to be upgraded
3. ~~DHCP server configured with a pool of ip address and the Juniper ZTP extensions~~
4. ~~Junos initial configuration file (`network-base-dhcp.conf`) provided by the DHCP server~~
5. Web server providing:
   - All Junos firmware files to be used (ie: `jinstall*`)
   - ASCII file called `firmwares` containing the list of Junos files to use
   - The script `ztp-upgrade.slax`

Then log into the Junos device to be upgraded and run the following commands (assuming 10.5.5.1 is the ip address of the web server):
```
root@qfx5200> configure
root@qfx5200# set event-options generate-event ztp-upgrade time-interval 60
root@qfx5200# set event-options policy ztp-upgrade events ztp-upgrade
root@qfx5200# set event-options policy ztp-upgrade then execute-commands commands "op url http://10.5.5.1/ztp-upgrade.slax output syslog"
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
    ├── jinstall-host-qfx-5e-flex-x86-64-19.4R2.6-secure-signed.tgz
    ├── jinstall-host-qfx-5e-flex-x86-64-19.3R2.9-secure-signed.tgz
    ├── jinstall-host-qfx-5e-flex-x86-64-20.1R1.11-secure-signed.tgz
    └── jinstall-host-qfx-5e-flex-x86-64-20.2R1.10-secure-signed.tgz
```
You need to create a `firmwares` file containing the list of junos firmwares to be used per model. The following is an example for a QFX5200
```
cat /var/www/html/firmwares
# No whitespaces between comma separated items
# You can specify multiple entries, so different upgrades will be performed
#
# NOTE: Only one firmware version per major junos release will be used
# In case of more than one version per major junos release, only the first
# will be used. In the example below version 18.4R1.8 will be skipped
# for a QFX5200 switch

# <Model,junos_firmware>
# Model is given by the command "show version | match Model"

# QFX5200
# 15.1 => 17.4 => 18.4 => 19.4 => 20.2
qfx5200-32c-32q,jinstall-host-qfx-5e-flex-x86-64-17.4R3.16-secure-signed.tgz
qfx5200-32c-32q,jinstall-host-qfx-5e-flex-x86-64-18.4R3.3-secure-signed.tgz
qfx5200-32c-32q,jinstall-host-qfx-5e-flex-x86-64-18.4R1.8-secure-signed.tgz
qfx5200-32c-32q,jinstall-host-qfx-5e-flex-x86-64-19.4R2.6-secure-signed.tgz
qfx5200-32c-32q,jinstall-host-qfx-5e-flex-x86-64-20.2R1.10-secure-signed.tgz
```

### DHCP
- In case you have a DHCP server, the minimal configuration is a pool with the Juniper ZTP options, something like
```
$ cat dhcpd.conf
# dhcpd.conf
#

default-lease-time 600;
max-lease-time 7200;
#authoritative;
log-facility local7;

option ztp-file-server code 150 = { ip-address };
option space ztp-ops;
option ztp-ops.image-file-name code 0 = text;
option ztp-ops.config-file-name code 1 = text;
option ztp-ops.image-file-type code 2 = text;
option ztp-ops.transfer-mode code 3 = text;
option ztp-ops-encap code 43 = encapsulate ztp-ops;

# This is a very basic subnet declaration.
subnet 10.5.5.0 netmask 255.255.255.0 {
  range 10.5.5.100 10.5.5.200;
  option domain-name "internal.example.org";
  option routers 10.5.5.1;
  option broadcast-address 10.5.5.255;
  default-lease-time 600;
  max-lease-time 7200;

  host juniper {
     hardware ethernet ec:38:73:56:83:61;
     fixed-address 10.5.5.100;
     option ztp-file-server 10.5.5.1;
     option host-name "ztp";
     option ztp-ops.config-file-name "ztp-upgrade.slax";
     # firmware option is not used
     # option ztp-ops.image-file-name "/dist/images/jinstall-host-qfx-5e-flex-x86-64-19.1R1.6-secure-signed.tgz";
     option ztp-ops.transfer-mode "http";
  }
}
```

- Start the DHCP server like:
```
# -f: foreground
# -d: debugging
# -cf: configuration file
$ ifconfig enp1s0f1 10.5.5.1 netmask 255.255.255.0 up
$ dhcpd -f -d -cf /etc/dhcpd.conf enp1s0f1
```

- The Juniper device should get an ip address and the SLAX script. The logs should be something like:
```
root@myrouter>
Auto Image Upgrade: Active on INET client interface : vme.0

Auto Image Upgrade: Interface::   "vme"

Auto Image Upgrade: Server::      "10.5.5.1"

Auto Image Upgrade: Image File::  "NOT SPECIFIED"

Auto Image Upgrade: Config File:: "ztp-upgrade.slax"

Auto Image Upgrade: Gateway::     "10.5.5.1"

Auto Image Upgrade: Protocol::    "http"

Auto Image Upgrade: Start fetching ztp-upgrade.slax file from server 10.5.5.1 t
hrough vme using http

Auto Image Upgrade: File ztp-upgrade.slax fetched from server 10.5.5.1 through
vme

Auto Image Upgrade: Executing script ztp-upgrade.slax

{master:0}
root@Unregistered>
```
- After that the device will start the upgrade process in about 60 secs

### Examples

#### Upgrade QFX5200 from Junos 20.1 to 20.2
- Note the device `hostname` will change a few times during the upgrade process

```
# Initial Junos version
root@qfx5200> show version
localre:
--------------------------------------------------------------------------
Hostname: qfx5200
Model: qfx5200-32c-32q
Junos: 20.1R1.11 flex

# Start the execution of the script in 1 minute
# Note the system hostname cannot contain the string "Juniper"
root@qfx5200> configure
root@qfx5200# set event-options generate-event ztp-upgrade time-interval 60
root@qfx5200# set event-options policy ztp-upgrade events ztp-upgrade
root@qfx5200# set event-options policy ztp-upgrade then execute-commands commands "op url http://10.5.5.1/ztp-upgrade.slax output syslog server 10.5.5.1"
root@qfx5200# commit and-quit

# Show the log messages
{master:0}
root@qfx5200> show log messages | match "Upgrade Juniper ZTP"
Aug 12 14:34:13  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): New device model: qfx5200-32c-32q, serial: WH9999999999, junos: 20.1R1.11, boot: 2020-08-12 14:13:01 UTC, ip: 10.1.1.2
Aug 12 14:34:14  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Retrieving the list of Junos firmwares from: http://10.5.5.1/firmwares
Aug 12 14:34:15  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Skip Junos file 17.4R3.16-secure-signed.tgz
Aug 12 14:34:16  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Skip Junos file 18.4R3.3-secure-signed.tgz
Aug 12 14:34:17  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Skip Junos file 18.4R1.8-secure-signed.tgz
Aug 12 14:34:20  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Skip Junos file 19.4R2.6-secure-signed.tgz
Aug 12 14:34:21  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Skip Junos file 20.2R1.10-secure-signed.tgz
Aug 12 14:34:22  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Target Junos version file for model qfx5200-32c-32q is: jinstall-host-qfx-5e-flex-x86-64-20.2R1.10-secure-signed.tgz
Aug 12 14:34:23  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Cleaning up some storage space
Aug 12 14:34:35  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Upgrading to jinstall-host-qfx-5e-flex-x86-64-20.2R1.10-secure-signed.tgz. Download in progress. Be patient
Aug 12 14:41:51  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): File from: http://10.5.5.1/images/jinstall-host-qfx-5e-flex-x86-64-20.2R1.10-secure-signed.tgz copiey
Aug 12 14:41:52  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Installing file /tmp/jinstall-host-qfx-5e-flex-x86-64-20.2R1.10-secure-signed.tgz with options: unlink
Aug 12 14:48:47  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Junos jinstall-host-qfx-5e-flex-x86-64-20.2R1.10-secure-signed.tgz was just installed. Output:  Verified...
Aug 12 14:48:48  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Rebooting in 1 minute... (to cancel type: clear system reboot)

(...The device is being upgraded to Junos 20.2, a couple of reboots later...)

# Once the device is back up again, the script will run again...
Aug 12 15:07:20  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): New device model: qfx5200-32c-32q, serial: WH9999999999, junos: 20.2R1.10, boot: 2020-08-12 15:05:13 UTC, ip: 10.1.1.2
Aug 12 15:07:21  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Retrieving the list of Junos firmwares from: http://10.5.5.1/firmwares
Aug 12 15:07:22  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Skip Junos file 17.4R3.16-secure-signed.tgz
Aug 12 15:07:23  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Skip Junos file 18.4R3.3-secure-signed.tgz
Aug 12 15:07:24  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Skip Junos file 18.4R1.8-secure-signed.tgz
Aug 12 15:07:27  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Skip Junos file 19.4R2.6-secure-signed.tgz
Aug 12 15:07:28  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Skip Junos file 20.2R1.10-secure-signed.tgz
Aug 12 15:07:29  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Target Junos version file for model qfx5200-32c-32q is: jinstall-host-qfx-5e-flex-x86-64-20.2R1.10-secure-signed.tgz
Aug 12 15:07:30  Registering cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Already running version 20.2R1.10. No need to upgrade
Aug 12 15:07:45  Upgraded cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Trying default configuration file http://10.5.5.1/default.conf
Aug 12 15:07:46  Upgraded cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Warning: Not able to find default configuration file http://10.5.5.1/default.conf
Aug 12 15:07:47  Upgraded cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Removing script from event-options
Aug 12 15:08:02  Juniper-WH9999999999 cscript: Upgrade Juniper ZTP (s/n: WH9999999999): Upgrade completed: qfx5200-32c-32q, serial: WH9999999999, junos: 20.2R1.10, boot: 2020-08-12 15:05:13 UTC, ip: 10.1.1.2
```
