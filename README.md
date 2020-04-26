# Setting up the enironment for the projects

- DLL Sideloading: VM Windows config
- Symlink attack: VM Windows config
- XSS: WebServer config + ZAP/KALI config
- Iptables config: N/A


## VirtualBox Windows config
- Download and install VirtualBox
- Download an image of Windows 10. We'll need to install the PRO version
- In VirtualBox, click New and create a Windows 10 64-bit machine with default settings
- In Settings for the newly created VM, mount the Windows iso and start the VM
- Perform a default installation of Windows 10 Pro. When prompted, select an offline account for logging in. We will want to have an admin account and a user/guest account
- Once installed, shutdown the VM, unmount the ISO, create a snapshot and restart the VM
- 
- 
- 

## WebServer config
- For this, we are going to use this [repository](https://github.com/MecatronicaUncu/Red-Social-Asociacion). Clone it and follow the installation instructions on it.

## KALI config
- Just download a VM image for VirtualBox from [Offensive-Security](https://www.offensive-security.com/kali-linux-vm-vmware-virtualbox-image-download/), and open it in VirtualBox

## ZAP config
- TBD
