### Step 1: Create the First VM

Create a new VM. 

Change its Hostname, IP Address, MAC Address, and Product UUID to desired than make clone for new VM.
________________________________________

### Step 2: Set Unique Hostname, IP Address, MAC Address, and Product UUID for Each VM

- **Set Unique Hostnames**
Edit the `/etc/hostname` file and change the hostname:
```bash
sudo nano /etc/hostname
```
5.	sudo nano /etc/hosts
Change the entry that references the hostname, for example:
127.0.1.1    vm1.local    vm1
6.	Reboot the VM to apply the changes:
7.	sudo reboot
2. Set Unique IP Addresses
For static IP addressing, you will need to manually assign unique IP addresses to each VM. This depends on your network setup (static or DHCP).
For static IP addressing:
1.	Edit the network configuration file. For Ubuntu, this is typically /etc/netplan/00-installer-config.yaml (for newer versions) or /etc/network/interfaces (older versions).
2.	For netplan (Ubuntu 18.04 and newer), the file will look like this:
3.	sudo nano /etc/netplan/00-installer-config.yaml
Example configuration for a static IP setup:
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      dhcp4: no
      addresses:
       - 192.168.1.101/24   # Use unique IP addresses for each VM (e.g., 192.168.1.102, 192.168.1.103, etc.)
      gateway4: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
4.	Apply the changes:
5.	sudo netplan apply
Repeat these steps for each VM, ensuring they have unique IP addresses.
3. Set Unique MAC Addresses
Proxmox automatically generates unique MAC addresses for each VM when they are cloned. However, you can manually specify a MAC address if necessary.
1.	In the Proxmox Web Interface:
o	Go to the VM and select the Hardware tab.
o	Click on Network device (usually named net0).
o	Click Edit and change the MAC address field (Proxmox auto-generates a unique one for you).
2.	Alternatively, you can also configure the MAC address from the CLI of Proxmox by editing the VM configuration file /etc/pve/qemu-server/<VMID>.conf. Add a unique MAC address for each VM like:
3.	net0: virtio=XX:XX:XX:XX:XX:XX,bridge=vmbr0
Ensure the MAC address is unique for each VM.
4. Set Unique Product UUID
The product UUID in Proxmox is typically tied to the machine’s identity, and clones will often retain the same UUID. However, you can regenerate it to make it unique.
1.	Regenerate the machine-id to ensure the product UUID is unique:
o	SSH into the VM or access the console.
o	Delete the existing machine-id: 
o	sudo rm /etc/machine-id
o	Re-generate the machine-id: 
o	sudo systemd-machine-id-setup
o	You can check if the UUID changed by running: 
o	cat /etc/machine-id
2.	For Proxmox, check the UUID by using:
3.	sudo dmidecode -t system
Ensure that each VM has a unique product UUID by checking the output of the above command.
________________________________________
Step 3: Clone the VM to Create Additional VMs
1.	After setting up the first VM with the desired hostname, IP address, MAC address, and product UUID, you can clone it:
o	In Proxmox, select the VM in the GUI and click on Clone.
o	Choose the option to create a Full Clone (not linked).
o	After cloning, repeat the steps to ensure that each VM has: 
	A unique hostname.
	A unique static IP (if you are using static IP addresses).
	A unique MAC address (Proxmox generates this automatically unless you modify it).
	A unique product UUID (use the machine-id regeneration method above).
2.	Apply the changes and reboot each VM after cloning to ensure that all configurations are applied properly.
________________________________________
Step 4: Verify Configurations
After creating and configuring each VM, verify the following:
1.	Unique Hostname:
o	Run hostname on each VM to verify they have unique names.
2.	Unique IP Address:
o	Verify the IP address on each VM using ip a or ifconfig.
3.	Unique MAC Address:
o	Run ip a or ifconfig inside the VM to verify the MAC address is unique.
4.	Unique Product UUID:
o	Verify the product UUID using sudo dmidecode -t system on each VM.
________________________________________
