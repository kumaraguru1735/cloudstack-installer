# CloudStack 4.20 Installation Guide (Ubuntu Server 22.04)

This guide provides step-by-step instructions to install **CloudStack Management** and **CloudStack Agent with KVM** (version 4.20) on **Ubuntu Server 22.04**.

***

## Prerequisites

- Download and install Ubuntu Server 22.04.5 [ubuntu-22.04.5-live-server-amd64.iso](https://releases.ubuntu.com/jammy/ubuntu-22.04.5-live-server-amd64.iso)).
- All commands require root or sudo privileges.

***

## 1. CloudStack Management Server Installation

### 1.1. Date & Time Zone Setup

```sh
sudo timedatectl set-timezone Asia/Kolkata
```

***

### 1.2. Network Setup

## 0. (Optional, Recommended) Use Old Network Interface Names (eth0, eth1)

**Disabling Predictable Network Interface Names ensures your system uses traditional names like eth0.**  
Add these steps **before your Network Setup section**.

```sh
sudo nano /etc/default/grub
```

Find the line:

```sh
GRUB_CMDLINE_LINUX=""
```

Change it to:

```sh
GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"
```

Save and exit the editor.

Update GRUB and reboot:

```sh
sudo update-grub
sudo reboot
```

After reboot, your network interfaces will show as **eth0**, **eth1**, etc.

***

Disable cloud-init network configuration:

```sh
sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
# Add:
network: {config: disabled}
```

Configure Netplan (update `$ADAPTER`, `$IP`, `$GATEWAY`):

```sh
sudo nano /etc/netplan/50-cloud-init.yaml
# Example configuration:
network:
    version: 2
    renderer: networkd
    ethernets:
        $ADAPTER:
            dhcp4: no
            dhcp6: no
    bridges:
        br0:
            interfaces: [$ADAPTER]
            dhcp4: no
            dhcp6: no
            addresses: [$IP/24]
            gateway4: $GATEWAY
            nameservers:
                addresses: [8.8.8.8, 8.8.4.4]
```

Apply changes:

```sh
sudo netplan apply
sudo systemctl restart NetworkManager
```

***

### 1.3. Hostname & Hosts File

Set the hostname:

```sh
sudo hostnamectl set-hostname node.domain.com
```

Edit `/etc/hosts`:

```
127.0.0.1 localhost
$IP node.domain.com
```

***

### 1.4. System Package Installation

```sh
sudo apt update && sudo apt upgrade -y
sudo apt-get install -y openntpd openssh-server vim htop tar intel-microcode bridge-utils mysql-server
```

***

### 1.5. Add CloudStack Repository

```sh
echo "deb [arch=amd64] http://download.cloudstack.org/ubuntu jammy 4.20" | sudo tee /etc/apt/sources.list.d/cloudstack.list
wget -O - http://download.cloudstack.org/release.asc | gpg --dearmor > cloudstack-archive-keyring.gpg
sudo mv cloudstack-archive-keyring.gpg /etc/apt/trusted.gpg.d/
sudo apt update && sudo apt upgrade -y
sudo apt-get install -y cloudstack-management cloudstack-usage
```

***

### 1.6. Database and Management Setup (After Installing CloudStack Management)

1. **Configure MySQL for CloudStack**

   Append optimized settings to your MySQL configuration:

   ```sh
   echo -e "\nserver_id = 1\nsql-mode=\"STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,ERROR_FOR_DIVISION_BY_ZERO,NO_ZERO_DATE,NO_ZERO_IN_DATE,NO_ENGINE_SUBSTITUTION\"\ninnodb_rollback_on_timeout=1\ninnodb_lock_wait_timeout=600\nmax_connections=1000\nlog-bin=mysql-bin\nbinlog-format = 'ROW'" | sudo tee -a /etc/mysql/mysql.conf.d/mysqld.cnf

   echo -e "[mysqld]" | sudo tee /etc/mysql/mysql.conf.d/cloudstack.cnf

   sudo systemctl restart mysql
   ```

2. **Fix MySQL Root Authentication**

   > ```
   > ###################################################################################
   > # In the next command if it will ask for password just press enter and do nothing #
   > ###################################################################################
   > ```

   Run the following command to set the root password and enable the `mysql_native_password` plugin:

   ```sh
   mysql -u root -p -e "
   SELECT user,authentication_string,plugin,host FROM mysql.user;
   ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'vexora';
   UPDATE user SET plugin='mysql_native_password' WHERE User='root';
   FLUSH PRIVILEGES;
   "
   ```

3. **Set Up the CloudStack Database**

   ```sh
   cloudstack-setup-databases cloud:vexora@localhost --deploy-as=root:vexora
   ```

4. **Initialize the CloudStack Management Server**

   ```sh
   cloudstack-setup-management
   ```

✅ If everything is correct, ```tail -f /var/log/cloudstack/management/management-server.log``` should show DB connection success and CloudStack startup.
***

### 1.7. Firewall

```sh
sudo ufw allow 8080,8250,8443,9090/tcp
sudo ufw reload
```

***

### 1.8. Java Debug Option

Add to `/etc/default/cloudstack-management`:

```sh
echo 'JAVA_DEBUG="-Xrunjdwp:transport=dt_socket,address=8000,server=y,suspend=n"' | sudo tee -a /etc/default/cloudstack-management
```

***

### 1.9. Agent and KVM Configuration

Installing Cloud Stack Agent

```sh
sudo apt install -y cloudstack-agent nfs-kernel-server
```

***

### 2.5. Firewall for KVM Node

```sh
sudo ufw allow proto tcp from any to any port 22,1798,16514,5900:6100,49152:49216
sudo ufw reload
```

***

### 2.6. Configure Libvirt

- **/etc/libvirt/libvirtd.conf**:

  ```
  listen_tls = 0
  listen_tcp = 0
  tls_port = "16514"
  tcp_port = "16509"
  auth_tcp = "none"
  mdns_adv = 0
  ```

- **/etc/default/libvirtd**:

  ```
  # Uncomment the following line:
  LIBVIRTD_ARGS="--listen"
  ```

- **/etc/libvirt/libvirt.conf**:

  ```
  remote_mode="legacy"
  ```

- **For Ubuntu 24.04 or newer**: Set libvirtd to traditional mode:
  ```sh
  sudo systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
  ```

Restart libvirt:

```sh
sudo systemctl restart libvirtd
```

***

### 2.7. Configure CloudStack Agent

- Set `guid` and `host` in `/etc/cloudstack/agent/agent.properties`.

Restart agent:

```sh
sudo systemctl restart cloudstack-agent
```

***

### 2.8. NFS Storage Preparation

```sh
sudo mkdir -p /export/primary
sudo mkdir -p /export/secondary
echo "/export *(rw,async,no_root_squash,no_subtree_check)" | sudo tee -a /etc/exports
sudo apt install nfs-kernel-server
sudo service nfs-kernel-server restart

sudo mkdir -p /mnt/primary
sudo mkdir -p /mnt/secondary
sudo mount -t nfs localhost:/export/primary /mnt/primary
sudo mount -t nfs localhost:/export/secondary /mnt/secondary
```
***
