# Provisioning Windows With Foreman (without WDS)

These templates are consumed by a boot.wim created using [ForemanWimScripts](https://github.com/LiamLeane/ForemanWimScripts). The templates are acquired when needed rather than stored as part of the wim so changes to your provisioning templates are immediately reflected in the build.

Task list (it's easy to miss a step):

* Create install repo directory structure
* Create Samba/Windows share for repo
* Build wim files
* Acquire boot files from ADK & wimboot
* Add Windows Media & OSes to Foreman
* Add Windows provisioning scripts to Foreman (or use [Foreman Templates](https://github.com/theforeman/foreman_templates) to directly synchronize this repo)
* Fin

## Building the WIM's
See [ForemanWimScripts](https://github.com/LiamLeane/ForemanWimScripts) for handy scripts which will build the WIMs for you.

The boot.wim is based on PE10 (2016) but supports installing 2012 & 2012R2 images too. It should also work for client editions but this is untested.

NB - The only WIM that needs modifying for this process to work is the boot.wim file, the others are highly recommended (Drivers & Updates) but the process will also work with any RTM install.wim from a Windows ISO.

### Setting up an Install Repo

As with [Foreman-ESXi](https://github.com/LiamLeane/Foreman-ESXi) we need a repo to host our install packages. While Windows does not (currently) work well with HTTP sources the scripts transform the HTTP media path in to a UNC path for acquisition so its strongly recommended to dump your WIM files on a web server so it’s easy to check your paths.

Repository layout should look like this;

* Windows
    * 6.2
    * 6.3
    * 10.0
    * boot
        * boot
		
![image](https://cloud.githubusercontent.com/assets/490726/15801956/e881c1ea-2a72-11e6-98c0-6712fa2d5701.png)


The version specific folders should contain the install.wim for their respective release, boot.wim goes in the boot folder. Version numbers should be used rather than release names due to how puppet will report the host to Foreman when facter runs.

You will need a Samba/Windows share for the install script to access the image. When Windows is installed the HTTP path set on OS media will have the HTTP path appended to the deploymentShare parameter (EG if the path to the 6.2 install.wim is http://server/Windows/6.2/install.wim and deploymentShare is set to "\\server\sources" then the share path to Windows 6.2 would be \\server\sources\Windows\6.2\install.wim. Due to security changes in PE10 this share must have a username/password set on it with read access.

### boot folder

This requires two files;
* bootmgr - You can find it on a machine with ADK installed at "C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment\amd64\Media"
* wimboot - Use version 1.0.5 https://git.ipxe.org/release/wimboot/wimboot-1.0.5.zip

More recent versions of wimboot do not work reliably with BIOS boots.

### boot/boot folder

Copy everything from "C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment\amd64\Media\Boot" on a machine with ADK installed to your boot/boot folder. Since its a headless install you can delete all the language folders other then en-us if you want to save some space.

## Configuring Foreman

### Install Media

Create a new installation medium pointing to the share you created. Your URL should look something like this http://server/sources/Microsoft/Windows/$version/

### Create Global Properties

The installer script expects two properties for the username/password for mounting the install share to be set;

* choco-repo - **Optional** If set will change the choco repo to a new value (EG artifactory server)
* deploymentShare - Path to the Windows install share 
* deploymentShareUser - Username of an account with access to the install share
* deploymentSharePassword - Plaintext password of the account
* ntp-server - **Optional** If set will update the NTP service to use the specified NTP server, otherwise uses NTP pool

![image](https://cloud.githubusercontent.com/assets/490726/15801868/94bc9bae-2a70-11e6-8c7d-d216e9c5157e.png)
 
### Create OSes

Each OS corresponds to a single Windows Version & Edition (EG to have Standard & Datacenter versions of 2012R2 you would have two entries), set the following entries for each OS you create;

| Property  | Value  | Notes |
|---|---|---|
| Name  | windows  |   |
| Major version  |   | Windows major version number, probably 6 or 10 |
| Minor version  |   | Windows minor version number. 2016:0 2012R2: 3, 2012: 2  |
| Description  |   | Whatever you want  |
| Family  | Windows  |   |
| Password Hash  | Base64  | Better then nothing I guess  |

In addition each OS should have two properties set;

* windowsEdition - The name of the image to apply from the install.wim (see bellow)
* windowsLicenseKey - Key to use to license Windows

![image](https://cloud.githubusercontent.com/assets/490726/15801884/e89060c6-2a70-11e6-94a8-a6fba9bc72b2.png)

#### Windows Editions

##### 2012

* Windows Server 2012 SERVERSTANDARDCORE
* Windows Server 2012 SERVERSTANDARD
* Windows Server 2012 SERVERDATACENTERCORE
* Windows Server 2012 SERVERDATACENTER

##### 2012R2

* Windows Server 2012R2 SERVERSTANDARDCORE
* Windows Server 2012R2 SERVERSTANDARD
* Windows Server 2012R2 SERVERDATACENTERCORE
* Windows Server 2012R2 SERVERDATACENTER

##### 2016

* Windows Server 2016 Technical Preview 5 SERVERSTANDARDCORE
* Windows Server 2016 Technical Preview 5 SERVERSTANDARD
* Windows Server 2016 Technical Preview 5 SERVERDATACENTERCORE
* Windows Server 2016 Technical Preview 5 SERVERDATACENTER
 
## Windows Finish Script

While the unattend & install scripts are basic and simply give you a standard windows install the finish script includes some steps you may want to rework for your environment;

* All static interfaces defined in Foreman (IE where the subnet is set to static) will have their address, netmask, gateway, DNS servers and DNS suffix set based on the subnet & domain details from Foreman.
* NIC's found on the machine but not defined in Foreman will be disabled and renamed with the prefix "NOID_" followed by a string of random chars
* If ntp-server is set as a global parameter then time will be immediately synchronized with that, otherwise one of the NTP pool servers will be used instead.
* chocolatey.org is installed for package management. 
* The latest version of puppet agent is installed. It sets 30 minutes for run interval, sets the service to automatic and starts it.
* Powershell execution policy is set to unrestricted
* WinRM is enabled supporting basic auth without encryption
