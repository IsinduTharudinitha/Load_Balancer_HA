This setup describes the configuration of a web cluster with load balancing using Squid, Apache, and Keepalived on Ubuntu. Below are the detailed steps, including setting up network interfaces, static IPs, SSH access, web server configuration, and Squid for load balancing.

### Step-by-Step Guide:

---

### **Step 1: Attach Network Interfaces**

1. Attach two network adapters to the virtual machine:

   - **NAT Adapter**: Provides internet access to the guest VM.
   - **Host-Only Adapter**: Enables SSH from the host and access to web servers on the guest.

2. Go to the VM's network settings and add the two network adapters:
   - NAT for internet.
   - Host-only for internal SSH and web access.

---

### **Step 2: Assign Static IP (Host-Only Adapter)**

1. Identify the name of the Host-Only adapter, likely `enp0s8`.

2. Use `systemd-networkd` to configure the static IP on Debian systems. (note: for ubuntu this can done using network managers nmtui-edit command or using ubuntus netplan configs):

   ```bash
   sudo cp /etc/systemd/network/20-wired.network /etc/systemd/network/40-wired.network
   sudo nano /etc/systemd/network/40-wired.network
   ```

3. Edit `/etc/systemd/network/40-wired.network` to assign a static IP:

   ```ini
   [Match]
   Name=enp0s8

   [Network]
   Address=192.168.56.10/24
   ```

4. Restart the network service to apply changes:
   ```bash
   sudo systemctl restart systemd-networkd
   ```

---

### **Step 3: Connect to Guest VM via SSH**

1. Install OpenSSH Server on the guest VM:

   ```bash
   sudo apt install openssh-server
   ```

2. Connect from the host to the guest via SSH:
   ```bash
   ssh debian@192.168.56.10
   ```

---

### **Step 4: Install Web Server and Required Packages**

Install Apache, Squid, OpenSSL, and Keepalived:

```bash
sudo apt install apache2 squid openssl keepalived psmisc ssl-cert
```

---

### **Step 5: Configure Apache Web Server**

1. Change Apache ports:

   - Edit `/etc/apache2/ports.conf` to change ports 80 to 8080 and 443 to 8443.
   - Edit the virtual host files to reflect the same changes.

2. **VirtualHost configuration for HTTP (`/etc/apache2/sites-available/000-default.conf`):**

   ```ini
   <VirtualHost *:8080>
       ServerAdmin webmaster@localhost
       DocumentRoot /var/www/html/webcluster-http
       ErrorLog ${APACHE_LOG_DIR}/error.log
       CustomLog ${APACHE_LOG_DIR}/access.log combined
   </VirtualHost>
   ```

3. **VirtualHost configuration for HTTPS (`/etc/apache2/sites-available/default-ssl.conf`):**

   ```ini
   <VirtualHost *:8443>
       ServerAdmin webmaster@localhost
       DocumentRoot /var/www/html/webcluster-https
       SSLEngine on
       SSLCertificateFile /etc/ssl/certs/ssl-cert-snakeoil.pem
       SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key
       ErrorLog ${APACHE_LOG_DIR}/error.log
       CustomLog ${APACHE_LOG_DIR}/access.log combined
   </VirtualHost>
   ```

4. If the `ssl-cert` package is not installed, create a self-signed certificate and add the path of the certificate file to apache default-ssl.conf properly:
   ```bash
   sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
       -keyout /etc/ssl/private/apache-selfsigned.key \
       -out /etc/ssl/certs/apache-selfsigned.crt
   ```

---

### **Step 6: Create Web Roots for HTTP and HTTPS Sites**

1. Create directories for web files:

   ```bash
   sudo mkdir /var/www/html/webcluster-http
   sudo mkdir /var/www/html/webcluster-https
   ```

2. Set permissions and ownership:

   ```bash
   sudo chown debian:debian /var/www/html/webcluster-http
   sudo chmod -R 755 /var/www/html/webcluster-http
   sudo chown debian:debian /var/www/html/webcluster-https
   sudo chmod -R 755 /var/www/html/webcluster-https
   ```

3. Add sample `index.html` for both web roots:
   ```html
   <!DOCTYPE html>
   <html lang="en">
   <head>
       <meta charset="UTF-8">
       <meta name="viewport" content="width=device-width, initial-scale=1.0">
       <title>Document</title>
   </head>
   <body>
       <h1>WebCluster: <Cluster Number> Protocol: <Protocol> version</h1>
   </body>
   </html>
   ```
4. Enable SSLmodule and SSL site for apache
   ```bash
   sudo a2enmod ssl
   sudo a2ensite default-ssl.conf
   sudo systemctl restart apache2
   ```

---

### **Step 7: Configure Squid with Load Balancing**

1. Edit the Squid configuration file:

   ```bash
   sudo nano /etc/squid/squid.conf
   ```

2. Add the following configuration for load balancing:

   ```ini
    acl all src all
    acl backend dst 192.168.56.10 192.168.56.20 192.168.56.30
    acl redir proto HTTP
    http_access allow backend

    cache_peer 192.168.56.10 parent 8080 0 proxy-only no-query no-digest originserver \
    login=PASS sourcehash name=backend1
    cache_peer_access backend1 allow backend redir

    cache_peer 192.168.56.10 parent 8443 0 proxy-only no-query no-digest originserver \
    ssl sslflags=DONT_VERIFY_PEER login=PASS sourcehash name=backend1_ssl
    cache_peer_access backend1_ssl allow backend !redir

    cache_peer 192.168.56.20 parent 8080 0 proxy-only no-query no-digest originserver \
    login=PASS name=backend2
    cache_peer_access backend2 allow backend redir

    cache_peer 192.168.56.20 parent 8443 0 proxy-only no-query no-digest originserver \
    ssl sslflags=DONT_VERIFY_PEER login=PASS name=backend2_ssl
    cache_peer_access backend2_ssl allow backend !redir

    http_access deny all

    #disable caching any request
    cache deny all

    #disable memory pools
    memory_pools off

    http_port 80 accel defaultsite=192.168.56.10
    # Please, note the following should be in a single line
    https_port 443 accel cert=/etc/ssl/certs/ssl-cert-snakeoil.pem key=/etc/ssl/private/ssl-cert-snakeoil.key defaultsite=192.168.56.10

   ```

3. Reload Squid to apply the changes:
   ```bash
   sudo systemctl reload squid
   ```

### Step 08: Configure Keepalived for High Availability

1. Create the Keepalived configuration file:
   ```bash
   sudo nano /etc/keepalived/keepalived.conf
   ```
2. Add the following full configuration:

   ```bash
   vrrp_script chk_squid {
       script "/usr/bin/killall -0 squid"
       interval 5
       fall 4
       rise 1
   }

   vrrp_instance VI_1 {
       interface enp0s8
       state MASTER
       virtual_router_id 51
       priority 150
       virtual_ipaddress {
           192.168.56.30/24
       }
       track_script {
           chk_squid
       }
   }
   ```

3. Start and enable Keepalived:
   ```bash
   sudo systemctl start keepalived
   sudo systemctl enable keepalived
   ```

---

### Step 09: Clone the VM and SSH into the New One

1. Power off the current VM.
2. Clone the VM using the **Full Clone** option, and ensure that "Generate new MAC addresses for all network adapters" is enabled.
3. Power on the cloned VM and SSH into it:
   ```bash
   ssh debian@192.168.56.10
   ```

---

### Step 10: Change the Static IP of the Second Server

1. Edit the network configuration file:
   ```bash
   sudo nano /etc/systemd/network/40-wired.network
   ```
2. Update the file with the following static IP configuration:

   ```
   [Match]
   Name=enp0s8

   [Network]
   Address=192.168.56.20/24
   ```

3. Restart the systemd network service and reconnect using the new IP:
   ```bash
   sudo systemctl restart systemd-network
   ssh debian@192.168.56.20
   ```

---

### Step 11: Change Hostname and Update `/etc/hosts`

1. Change the hostname to **WebCluster2**:
   ```bash
   sudo hostnamectl set-hostname WebCluster2
   ```
2. Edit the `/etc/hosts` file to update the hostname from old one to new **WebCluster2**:
   ```bash
   sudo nano /etc/hosts
   ```
3. Log out of the SSH session and reconnect to see the new hostname.

---

### Step 12: Update Web Server Pages

1. Modify the index pages to reflect **Web Cluster 2** for easy identification:
   ```bash
   sudo nano /var/www/html/webcluster-http/index.html
   sudo nano /var/www/html/webcluster-https/index.html
   ```
2. Include text indicating **Web Cluster 2** in both HTTP and HTTPS index pages.

---

### Step 13: Lower Keepalived Priority for Backup Server

1. Edit the Keepalived configuration on the backup server:
   ```bash
   sudo nano /etc/keepalived/keepalived.conf
   ```
2. Modify the configuration by setting the priority to 100:

   ```bash
   vrrp_script chk_squid {
       script "/usr/bin/killall -0 squid"
       interval 5
       fall 4
       rise 1
   }

   vrrp_instance VI_1 {
       interface enp0s8
       state MASTER
       virtual_router_id 51
       priority 100
       virtual_ipaddress {
           192.168.56.30/24
       }
       track_script {
           chk_squid
       }
   }
   ```

3. Reload Keepalived to apply the changes:
   ```bash
   sudo systemctl reload keepalived
   ```

---

### Step 14: Test the High Availability and Load Balancing

1. Power on the old VM and check if the load balancing and failover setup works as expected.
2. Ensure the system transitions between MASTER and BACKUP when necessary, using Squid and Apache.

---

### Step 15: Troubleshooting Commands

Use the following commands to troubleshoot any issues with the Keepalived and Squid setup:

- Check Keepalived status:
  ```bash
  sudo systemctl status keepalived
  ```
- View assigned IP addresses:
  ```bash
  ip a
  ```
- View open ports and associated processes:
  ```bash
  sudo ss -tupln
  ```
- Test Apache configuration:
  ```bash
  sudo apachectl configtest
  ```
- Test Squid configuration:
  ```bash
  sudo squid -k parse
  ```
- Test Keepalived configuration:
  ```bash
  sudo keepalived -t
  ```
- Monitor Squid access logs:
  ```bash
  sudo tail -f /var/log/squid/access.log
  ```

---

This configuration guide sets up load balancing, high availability with Keepalived
