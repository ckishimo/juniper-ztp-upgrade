# Juniper ZTP upgrade

# Objective
This script will automate the upgrade of a Juniper device to a final Junos version using
different Junos firmwares. So far tested only on a QFX switch running Junos 15.1 and
automatically upgraded to 17.2->18.3->19.4->20.1->20.2 without human interaction

The idea is to use the Juniper ZTP feature that is enabled by default in most of the switches.
The Juniper device will boot and receive via DHCP a configuration file with a pointer to a
SLAX script. The script will be in charge of upgrading the device to the desired Junos version

