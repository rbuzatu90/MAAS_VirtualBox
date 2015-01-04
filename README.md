MAAS_VirtualBox
===============

This project offers a set of basic extensions for MAAS to integrate VirtualBox VMs. It has been tested with VirtualBox 4.3.20r96996 for OS X (Mavericks 10.9.5), hence different versions of VirtualBox and/or VirtualBox running on other host machines may have some issues.

These extensions work well for testing, demoing and development, but they are not recommended for production use.


Features
--------

The VirtualBox extensions for MAAS offer these features:
- VirtualBox VMs can be autodiscovered, they can be started, commissioned and acquired.
- Multiple VirtualBox host machines are allowed, provided they are accessible from the MAAS server and the MAAS server can offer PXE Boot, DHCP and DNS services.


Current Limitations
-------------------

At the moment, the VirtualBox extensions for MAAS have these limitations:
- VirtualBox VMs cannot be checked, i.e. administrators using MAAS cannot check from MAAS if the node is ON or OFF by using the "Check power state" button in the "Edit Node" page.
- VirtualBox VMs cannot be stopped from MAAS.
- The Wake-on-LAN option is no longer present in MAAS
- The description of the physical zones in MAAS is no longer freely usable (it is reserved for the extensions).
- A power request is sent to all the available VirtualBox host machines, i.e. at the moment the scripts do not check which VirtualBox machine hosts the VM.
- The error status of many commands in the powering process is not checked.
- The testing is limited to Ubuntu VMs on VirtualBox host machines in OS X Mavericks 10.9.5.


Basic Concepts
--------------

The VirtualBox extensions for MAAS are used to control VirtualBox VMs from the MAAS server. At the moment, only the "Start Node" (or Power ON) action has been implemented, but there are already scripts that can be used to integrate "Stop Node" (or Power DOWN) and "Check power state" (or Power CHECK).

The extensions rely on these basic concepts:
- The MAAS server and the VirtualBox host machines are connected to the same network: this is also essential to allow PXE boot of the VMs, the use of OS images hosted by the MAAS server and the use of DHCP and DNS services, also offered by the MASS Server.
- The MAAS server must send ssh commands to the VirtualBox host machines.
- The VirtualBox extensions replace the Wake-on-LAN option in MAAS: this means that the parameters used to identify a VirtualBox VM must be added to a node by using the "Power Type" options for Wake-on-LAN. It also means that the standard Wake-on-LAN power option is no longer available on the MAAS server.
- The VirtualBox host machines are identified by MAAS physical zones.
- A MAAS physical zone can refer only to one VirtualBox host machine.
- Many MAAS physical zones can refer to the same VirtualBox host machine, although this is not advisable: the best approach is to identify one physical zone in MAAS for one VirtualBox host machine.
- The maas user on the maas server must be modified from its default configuration in order to execute the extensions scripts correctly.


Extensions Architecture
-----------------------

Once installed, the extensions are placed in:
- _The power template folder for MAAS_: this is usually /etc/maas/templates/power.
- _The VBox_extensions folder in the MAAS default user_: this is usually maas, therefore some scripts are placed in /home/maas/VBox_extensions or /var/lib/maas/VBox_extensions. 
- _The Vbox_host_extensions folder in the VirtualBox default user_: this is usually a standard user, for example IvanZoratti, therefore some scripts are placed in OS X in /Users/IvanZoratti/Vbox_host_extensions and in Ubuntu in /home/IvanZoratti/Vbox_host_extensions.

Each request (a Power ON request) follows this flow:
1. The request is activated by an admn action, for example "Commission Node"
2. MAAS calls the script generated by a power template, in this case "/etc/maas/templates/power/ether_wake.template"
3. The power template calls a script in the VBox_extension folder, in this case "power_on"
4. The power script checks the incoming request and sends a request to all the available VirtualBox hosts machines via SSH. The request calls a script in the VBox_host_extension folder, in this case "startvm"


Logging
-------

A basic logging of all the operations is kept in the /tmp/VBox.log file. The file is updated by the template and the scripts in the MAAS server. The file is removed when the MAAS server reboots.


MAAS Preparation
----------------

These actions must be performed in order to prepare the MAAS server:
1. Check the status of the maas user, i.e. home directory and shell. Retrieve the status of the user with:
  * grep "maas" /etc/passwd
2. Make the maas user interactive and sudoer: from a sudoer user in the MAAS server:
  * sudo -u maas chsh -s /bin/bash
  * sudo passwd maas
    * ...Type you favourite password for the maas user
  * sudo adduser maas sudo
3. Login to maas using the new password
4. In the maas home folder, copy the VBox_extensions folder and files.
5. It is advisable, for testing and demoing, to set the accept-all option for all the nodes registered on the MAAS server. This action is not mandatory (see https://maas.ubuntu.com/docs/nodes.html)


Installation
------------

Login to the maas user and run the installation script:
* cd VBox_extensions
* ./install_extensions user@vbox_host_address, where <vbox_host_address> is the IP address of the VirtualBox host machine to add to MAAS and <user> is a standard user on the VirtualBox host machine. For example, a valid parameter is IvanZoratti@192.168.56.1
* Run the install_extensions script for all the VirtualBox host machines you want to control from the MAAS server.

This is in brief what the script does:
1. Check the presence of the user@vbox_host_address paramter
2. Retrieve and save the MAAS API Key
  * The script requires the maas password to sudo
3. Copy the new power template in the MAAS server power template folder
4. Generate a new public key for the maas user
  * If the key already exists, the script may require to overwrite the old key
5. Copy the maas user public key to the VirtualBox host machine
  * The password for the designated user on the VirtualBox host machine is required
6. Copy the extension scripts on the VirtualBox host machine


How to use the extensions
-------------------------

Here is a simple HOW TO for these extensions.

On the VirtualBox host machine:
* Create a VirtualBox VM using the standard CLI (VBoxManage) or using the VirtualBox GUI. Remember to set up PXE Boot in order to manage the VM from MAAS.
* Start the VM for the first time. The machine should require a PXE Boot and the MAAS server should offer an IP address and a OS image. 
* Once the VM has been started for the first time, if the MAAS server has been set to accept all nodes, it should appear in the list of nodes in the MAAS console.
* Run the CLI command: 
```
    VBoxManage list vms
    "UbuDsk14.10.1 - 141109.01" {8ef63638-7e48-4499-b78f-2ac63469a58e}
    "DevStack 141110.01" {0b72ee52-fb67-4703-9ea0-2246f10926f1}
    "UbuSrv 141116.01" {f6125f5f-50ae-4bc6-aa78-89cdc48f226d}
    ...
    "Net Testing" {51d35b5c-a3bf-4036-9320-ee67e1f3b27f}
```
    __NOTE__: The code in brackets is the unique ID of the VM.
* Select the last 12 digits of the unique ID. For example, the last 12 digits of the VM called `Net Testing` are `ee67e1f3b27f`.

On the MAAS server, using the MAAS web console:
* Create a physical zone that identifies a VirtulaBox host machine. For example:
  * Create a new physical zone with name "MyMacBooPro"
  * Add a description that contains __only__ the user and the IP address of the VirtualBox host machine. In this case: IvanZoratti@192.168.56.1 
* Edit the new node on the MAAS server console.
  * Change the name of the new node with a more adequate name
  * Select Power Type "Wake-on-LAN"
  * Insert the selected 12 digits "ee67e1f3b27f" with the format ee:67:e1:f3:b2:7f
  * Select the Physical zone related to the VirutalBox host machine, for example "MyMacBookPro"
* Save the changes

In order to test that everything works fine, you can now commission the new node. You should see the VM starting automatically on the VirtualBox host machine and starting the commissioning. You can check in the VBox.log file if the commands have been executed correctly.

