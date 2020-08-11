## Juniper ZTP upgrade

### Objective
This script will automate the upgrade of a Juniper device to a final Junos version using
different Junos firmwares. So far tested only on a QFX switch running Junos 15.1 and
automatically upgraded to 17.2->18.3->19.4->20.1->20.2 without human interaction

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



