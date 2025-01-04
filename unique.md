## Change Hostname

- Change hostname with hostnamectl:
```bash
sudo hostnamectl set-hostname <new-hostname>
```

- Edit /etc/hostname:
``` bash
sudo nano /etc/hostname
```
Replace the existing hostname with the new one.


- Edit /etc/hosts:
```bash
sudo nano /etc/hosts
```
Update the line with 127.0.1.1 to reflect the new hostname.


- Restart the hostname service:
```bash
sudo systemctl restart systemd-hostnamed
```

Verify the new hostname:
```bash
hostname
```

## Change IP Address

- First, find the name of the network interface (e.g., eth0, enp0s3, etc.):
```bash
ip a
```

- Ubuntu and some other distributions use netplan for network configuration.   
- Edit the netplan configuration file (usually located in /etc/netplan/):   
```bash
ls/etc/netplan/
```

```bash
sudo nano /etc/netplan/01-netcfg.yaml
```

- Update the configuration to set a static IP. It should look like this:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.0.2/24
      gateway4: 192.168.0.1
      nameservers:
        addresses:
          - 8.8.8.8
```

- Apply the configuration:
```bash
sudo netplan apply
```

- Verify the change:
```bash
ip a
```
