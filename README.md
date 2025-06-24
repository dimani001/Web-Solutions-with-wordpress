**Project Title: Web Solutions with WordPress (RHEL 9 + LVM + EBS + Apache + MySQL)**

###  Prerequisites
* AWS account
* SSH Key Pair
* Basic knowledge of EC2, volumes, and Linux commands


##  Step One: Launch EC2 Instances

###  1.1 Server Instance (WordPress Server)

* **AMI**: Red Hat Enterprise Linux 9
* **Instance Type**: t3.micro
* **Storage**: Keep default 10 GB root volume
* **Tags**: Name it `Wordpress`
* **Security Group**:

  * Allow SSH (port 22)
  * Allow HTTP (port 80)
  * Allow HTTPS (port 443)

###  1.2 Database Instance (MySQL DB Server)

* Repeat the above steps to create a second RHEL 9 instance
* **Tag**: Name it `Database`
* **Security Group**:

  * Allow SSH (port 22)
  * Allow MySQL/Aurora (port 3306) from **WordPress server's private IP only or 0.0.0.0/0**
    
![instances](https://github.com/user-attachments/assets/8ed4077c-d596-4a87-b2b3-23f593bc1c17)


##  Step Two: Create & Attach EBS Volumes

###  2.1 Create Volumes (3 for each instance)

* Go to **Elastic Block Store > Volumes**
* Create **6 volumes**, each **10 GB**, type: `gp2`
* Availability Zone must match EC2 instances
![EBS Volumes](https://github.com/user-attachments/assets/3289bc8c-e8b4-42a0-b50a-d5cf1d6c5898)

###  2.2 Attach Volumes

#### To WordPress Server:

* Attach 3 volumes:

  * `/dev/nvme1n1`, `/dev/nvme2n1`, `/dev/nvme3n1`

#### To Database Server:

* Attach 3 volumes:

  * `/dev/nvme1n1`, `/dev/nvme2n1`, `/dev/nvme3n1`


##  Step Three: Setup LVM on WordPress Server

###  Connect via SSH:

```bash
ssh -i your-key.pem ec2-user@<wordpress-public-ip>
```

###  Prepare Volumes:

- List attached volumes:

```bash
lsblk
```
- Use **df -h ** to see all mounted drives and free spaces
![list of all disks](https://github.com/user-attachments/assets/9ab10e26-d721-4fb2-baab-9f9b8cc18e0e)

- To create a single partition on each of the 3 disks (/dev/nvme1n1, /dev/nvme2n1, /dev/nvme3n1) using gdisk, follow the interactive steps for each disk.

Repeat these steps for all three disks:

```bash
sudo gdisk /dev/nvme1n1
sudo gdisk /dev/nvme2n1
sudo gdisk /dev/nvme3n1
```
Type "n"  for new partition,<enter>x4 to accept default partition number,default first sector, default last sector (use entire disk), and default type (Linux filesystem). Then,type "w" to Write changes to disk and lastly, type "y" to confirm write.
![disk partitioning](https://github.com/user-attachments/assets/d45d22db-c77a-403e-b80c-c412650a7bb5)

- Use the lsblk utility to see all newly created partitions for the 3 volumnes

###  Setup LVM:

```bash
sudo yum install -y lvm2
```
- This installs Logical Volume Manager tools used for creating Volume Groups (VGs), Logical Volumes (LVs), and managing Physical Volumes (PVs).

- Use the pvcreate utility tool to mark each of the volumes as LVM Physical Volumes

```bash
sudo pvcreate /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1
```
Once done, you can verify the physical volumes with:

```bash
sudo pvs
```
![pvs created](https://github.com/user-attachments/assets/e5de43c0-7fbc-4b9d-a389-2471947a0df8)

- Create a Volume Group called webdata-vg using the three physical volumes.

```bash
sudo vgcreate webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1
```
Verify the Volume Group:

```bash
sudo vgs
```
![vgs created](https://github.com/user-attachments/assets/93d7f9ec-bfb7-4759-b882-dfe310210654)

- Create two logical volumes of 14G each, assuming your total volume group (webdata-vg) size is around 28G.

```bash
sudo lvcreate -n app-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg
```
- Verify the logical volumes:

```bash
sudo lvs
```
- Verify the entire setup to be sure all has been configured properly

```bash
sudo vgdisplay -v #view complete setup - VG, PV, and LV sudo lsblk 
```

### Format & Mount:

- format the logical volumes using ext4 filesystems
```bash
sudo mkfs.ext4 /dev/webdata-vg/app-lv
sudo mkfs.ext4 /dev/webdata-vg/logs-lv
```
- Create a directory to store website file

```bash
sudo mkdir -p /var/www/html /home/recovery/logs
```
- Create another directory for the log files

```bash
sudo mkdir -p /home/recovery/logs
```
- Mount the newly created directory for website files on tyhe app logical volume we earlier created

```bash
sudo mount /dev/webdata-vg/app-lv /var/www/html
```
- Back up all the files on the logs logical volume before mounting, this is done using rsync utility

```bash
sudo rsync -av /var/log/. /home/recovery/logs/
```
- Mount the .var/logs on the log-lv

```bash
sudo mount /dev/webdata-vg/logs-lv /var/log
```
- Restore the log files back into /var/log/ directory

```bash
sudo rsync -av /home/recovery/logs/. /var/log
```


###  Persist on Reboot:
Ensure that the mount configurations persist after server restart, this can be done by updating the UUID of the /etc/fstab

```bash
sudo blkid
sudo vi /etc/fstab
# Add the UUID for the log-lv with the one you copied , save and exit. then test the configuration
sudo mount -a
```
- Reload the daemom

```bash
sudo systemctl reload daemon
```
- Verify the setup

```bash
df -h
```
![LVM volumes mounted](https://github.com/user-attachments/assets/c65c8c16-2993-4907-a151-3f909bb33308)

##  Step Four: Setup LVM on DB Server (Repeat Step Three)

We can now proceed to installing and configuring MYSQL server that will serve as the database for our website on the server instance, to do this, we can follow same process as we did in the server instance to create ec2 instance, create and attach the 3 ebs volumes, ssh into your nstance and create partitions
Use the same procedure with:

* Volume Group: `mysqldata-vg`
* Logical Volumes: `db-lv`, `logs-lv`
* Mount Points: `/db`, `/var/log`


##  Step Five: Install Apache, PHP & WordPress

###  On WordPress Server:

- update the instance

```bash
sudo yum -y update
```
```bash
sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
```
- Verify
```bash
httpd -v           # Check Apache version
php -v             # Check PHP version
```

- Enable, and start apache
```bash
sudo systemctl enable httpd
sudo systemctl start httpd
```
![apache HTTP server running](https://github.com/user-attachments/assets/5b5267d3-b5f1-4830-a8b7-e5d0a3ff637c)

###  Install PHP:

```bash
sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install -y yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum module reset php -y
sudo yum module enable php:remi-7.4 -y
sudo yum install -y php php-opcache php-gd php-curl php-mysqlnd php-json php-fpm php-mbstring php-xml
```

###  Enable PHP Permissions:

```bash
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo setsebool -P httpd_execmem 1
sudo setsebool -P httpd_can_network_connect_db 1
sudo setsebool -P httpd_can_network_connect 1
```

- Restart Apache
```bash
sudo systemctl restart httpd
```

###  Test PHP:

- Create an info.php page to test if your configuration is correct
```bash
sudo vi /var/www/html/info.php
```
- write the following code to check php config

```bash
echo "<?php phpinfo(); ?>" 
sudo chown apache:apache /var/www/html/info.php
```

Visit: `http://<wordpress-public-ip>/info.php`
![infio php web browser](https://github.com/user-attachments/assets/fd2631e3-a9ef-485f-9cb4-00beba54cf8c)


###  Install WordPress:
- Download and Copy wordpress to the /var/www/html directory
```bash
cd /var/www/html
sudo wget https://wordpress.org/latest.tar.gz
sudo tar -xzf latest.tar.gz
sudo cp -r wordpress/* .
sudo rm -rf wordpress latest.tar.gz
sudo chown -R apache:apache /var/www/html
```


##  Step Six: Install & Configure MySQL on DB Server

```bash
sudo yum -y update
sudo yum install -y mysql-server
sudo systemctl start mysqld
sudo systemctl enable mysqld
```

###  Create DB & User:
- First , log in as root user

```bash
sudo mysql
```
- Then create a new db and create a user that can access the db from the webserver

```bash
CREATE DATABASE wordpress_db;
CREATE USER 'wpuser'@'<wordpress-private-ip>' IDENTIFIED BY 'strongpassword';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wpuser'@'<wordpress-private-ip>';
FLUSH PRIVILEGES;
```
- Confirm DB was created

```bash
 SHOW DATABASES;
 ```
![wordpress_db created](https://github.com/user-attachments/assets/3ee532aa-8623-41bb-8839-9306ff85f139)


### Troubleshooting Tips:

* Ensure MySQL is **listening** on `0.0.0.0:3306`
* Open port 3306 in **DB security group**, limited to WordPress server private IP or 0.0.0.0/0
* Ensure user is created with exact IP (e.g., `'wpuser'@'172.31.45.123'`)
* If access is denied, re-run:

```sql
ALTER USER 'wpuser'@'172.31.45.123' IDENTIFIED BY 'strongpassword';
FLUSH PRIVILEGES;
```
- Test your db connection by logging in to your db from your webserver

- Then access your webserver instance and also install mysql client
```bash
sudo yum install -y mysql-client
```
- Log in to the mysql server remotely from your webserver
```bash
sudo mysql -u myuser -p -h (your database server ip address)
```
![mysql client connect](https://github.com/user-attachments/assets/f5b8c8dc-1460-4d8a-aa7f-bb5755cbd9aa)

Now that you have successfully setup and configured mysql and connected to it remotely from your webserver, it is essential we set up wordpress to do the same.
- visit your-ip-address/wordpress in your web browser and you should get the same result as below;



##  Step Seven: Connect WordPress to MySQL

###  Update wp-config:

```bash
cd /var/www/html
sudo cp wp-config-sample.php wp-config.php
sudo vi wp-config.php
# Replace values:
define( 'DB_NAME', 'wordpress_db' );
define( 'DB_USER', 'wpuser' );
define( 'DB_PASSWORD', 'strongpassword' );
define( 'DB_HOST', '<database-private-ip>' );
```

###  Clean Up Permissions:

```bash
sudo chown apache:apache /var/www/html/wp-config.php
```


##  Step Eight: Final WordPress Setup

* Visit: `http://<wordpress-public-ip>`
* Follow the UI to configure site title, admin login, password, etc.
![wordpress login](https://github.com/user-attachments/assets/5893e0b1-5f32-4797-ac32-d687c8feb139)


## Additional Notes:

* If HTTP times out, check:

  * Apache status: `sudo systemctl status httpd`
  * Security Group allows port 80
  * SELinux is not blocking content
* If PHP doesn't load, ensure `/etc/httpd/conf.d/php.conf` includes:

```apache
<FilesMatch \.php$>
    SetHandler application/x-httpd-php
</FilesMatch>
```


Your RHEL-based WordPress server with MySQL backend and LVM-backed storage is now up and running.

Customize your new site — maybe start with a stunning landing page for **Hairs by Dimani** 
![Screenshot (271)](https://github.com/user-attachments/assets/295b6a44-17e1-4574-b569-38736ae581f4)
