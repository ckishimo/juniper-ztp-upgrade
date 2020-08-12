## Juniper ZTP upgrade

### Objective
This script will automate the upgrade of a Juniper device to a final Junos version using
different Junos firmwares. So far tested only on a few QFX (QFX10008/16, QFX5200) switches
running Junos 15.1 and automatically upgraded to 17.2->18.3->19.4->20.1->20.2 without human 
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

Then you can trigger the script via the following commands (assuming 10.1.1.1 is the ip address of the web server):
```
root@qfx5200> configure
root@qfx5200# set event-options generate-event ztp-upgrade time-interval 60
root@qfx5200# set event-options policy ztp-upgrade events ztp-upgrade
root@qfx5200# set event-options policy ztp-upgrade then execute-commands commands "op url http://10.1.1.1/ztp-upgrade.slax output syslog server 10.1.1.1"
root@qfx5200# commit
```





