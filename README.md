# Web Solution with Wordpress
This is a 3 tier achitecture including presentation tier, application tier, and database tier. These 3 tier are summarized below:
1. A Laptop or PC to serve as a client
2. An EC2 Linux Server as a web server (This is where you will install WordPress)
3. An EC2 Linux server as a database (DB) server

## Project Setup

### Prepare the Web Server
- Launch a Redhat EC2 instance for Web-Server, create 3 volumes each of 10GiB
<img width="1491" height="289" alt="Screenshot 2025-09-18 at 21 39 21" src="https://github.com/user-attachments/assets/f23f56d6-c393-4455-862c-93ebd46e82a1" />

- Attach each volume to the Web-Server EC2 instance one at a time
<img width="1498" height="282" alt="Screenshot 2025-09-18 at 21 48 09" src="https://github.com/user-attachments/assets/72d031f7-9077-4d36-9f77-e70e985e2e85" />

- Open up linux terminal for configuration
- confirm that all block device are attached to server, use `lsblk` namely; xvdf, xvdg, xvdh.
- Free up space in server `df -h`
- Install gdisk

```sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm```

```sudo dnf install gdisk```
- Using gdisk, On each of the 3 disk create a seperate single partition

``` sudo gdisk /dev/xvdf```

confirm that all 3 disk partition have been configured, use `lsblk`

<img width="903" height="943" alt="Screenshot 2025-09-19 at 21 42 18" src="https://github.com/user-attachments/assets/a3ae7643-ef18-43f6-9e6d-84e3d95e1591" />

- Install lvm2 package `sudo yum install lvm2`, then check available partion `lvmdiskscan`
- Mark all 3 disk as PVs to be used by LVM

```sudo pvcreate /dev/xvdf1```

```sudo pvcreate /dev/xvdg1```

```sudo pvcreate /dev/xvdh1```

- Confirm each PVs was created successfully with `sudo pvs`

- Add each PVs to VG, Name the VG webdata-vg

```sudo vgcreate webdata-vg /dev/xvdf1 /dev/xvdg1 /dev/xvdh1```

- Create 2 LVs namely; apps-lv(to store data for website) and logs-lv(to store data for logs), split the available PV size for both

```sudo lvcreate -n apps-lv -L 14G webdata-vg```

```sudo lvcreate -n logs-lv -L 14G webdata-vg```

- Confirm LVs was created successfully with `sudo lvs`
- Verify the entire set up

```sudo vgdisplay```
```sudo lsblk```

- Format LVs with ext4 filesystem

```sudo mkfs -t ext4 /dev/webdata-vg/apps-lv```

```sudo mkfs -t ext4 /dev/webdata-vg/logs-lv```

- Create /var/www/html directory to store website files `sudo mkdir -p /var/www/html`
- Create /home/recovery/logs to store backup of log data `sudo mkdir -p /home/recovery/logs`
- Mount /var/www/html on apps-iv logical volume

```sudo mount / dev/webdata-vg/apps-lv /var/www/html/```

⁠- Backup all the files in the log directory /var/log into /home/recovery/logs, Use rsync utility

```sudo rsync -av /var/1og/```

- Mount /var/log on logs-lv

```sudo mount /dev/webdata-vg/logs-lv/ /var/log```

- Restore log file back to /var/log directory

```sudo rsync -av /home/recovery/logs/ /var/log```

- Update etc/fstab file with UUID of device

```sudo blkid```

```sudo vi /etc/fstab```

- Test configuration and reload daemon

```sudo mount -a```
```sudo systemctl daemon-reload```

<img width="1356" height="480" alt="Screenshot 2025-09-21 at 20 44 43" src="https://github.com/user-attachments/assets/3ae77ac2-a218-49b5-ae50-4ae0a59f463e" />

### Prepare the Database Server

Launch a Redhat EC2 instance for DB-Server, repeat same steps in preparing Web-Server but this time creating just one volume of 20GiB, when creating your LVs, replace `apps-lv` with `db-lv` and mount is on `/db` directory and not `/var/www/html/`

### Install Wordpress on Web Server EC2

- Update the repository `sudo yum -y update`

- Install wget, Apache and it's dependencies `sudo yum -y wget httpd php php-mysqlnd php-fpm php-son`

- Start Apache

```sudo sytemctl enable httpd```

```sudo sytemctl start httpd```

- isntall PHP and it's dependencies

```sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm```

```sudo dnf install https://rpms.remirepo.net/enterprise/remi-release-10.rpm```

```sudo yum module list php```

```sudo yum module reset php```

```sudo yum module enable php:remi-10```

```sudo yum install php php-opcache php-gd php-curl php-msqlnd```

```sudo sytemctl start php-fpm```

```sudo sytemctl enable php-fpm```

```setsebool -P httpd_execmem 1```

- Restart Apache

```sudo sytemctl restart httpd```

- Download Wordpress and copy Wordpress to /var/www/html/

```mkdir wordpress```

```cd wordpress```

```sudo wget http://wordpress.org/latest.tar.gz```

```sudo tar -xzvf atest.tar.gz```

```sudo rm -rf atest.tar.gz```

```cp wordpress/wp-config-sample.php wordpress/wp-config.php```

```cp -R wordpress /var/www/html/```

- Configure SElinux policies

```sudo chown -R apache:apache /var/www/html/wordpress```

```sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R```

```sudo setsebool -P httpd_can_network_content=1```

### Install MYSQL on DB Server EC2

```sudo yum update```

```sudo yum install mysql-server```

Confirm that it is running `sudo systemctl status mysqld` if not restart and enable

```sudo systemctl restart mysqld```

```sudo systemctl enable mysqld```

### Configure DB to work with Wordpress

```sudo mysql```

```CREATE DATABASE wordpress;```

```CREATE USER myuser @ <Web-Server-Private-IP-Address> IDENTIFIED BY 'mypass' ;```

```GRANT ALL ON wordpress.* TO 'myuser'@'<Web-Server-Private-IP-Address>' ;```

```FLUSH PRIVILEGES;```

```SHOW DATABASES;```

```exit```

<img width="830" height="730" alt="Screenshot 2025-09-27 at 11 10 58" src="https://github.com/user-attachments/assets/b10e5910-586d-41a9-987d-be912566c74a" />

### Configure Wordpress to connect to remote Database

- Make sure the remote database allows connections from Web server IP address.
- Install MySQL and allow it to connect from Web server to DB server
```sudo yum install mysql```
```sudo mysql -u admin -p -h <DB-server-Private-IP-address```

```CREATE USER 'Webuser' IDENTIFIED BY 'MyPassword1!';GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'%';FLUSH PRIVILEGES;```

- Access browser link to wordpress `http://<web-server-public-IP-address>/wordpress/`

<img width="1226" height="890" alt="Screenshot 2025-09-27 at 11 57 26" src="https://github.com/user-attachments/assets/328f687f-95b1-41bf-9002-2e118750f993" />

<img width="1493" height="907" alt="Screenshot 2025-09-27 at 12 03 02" src="https://github.com/user-attachments/assets/af270b8c-783f-4bfe-b775-ce466f7fbe22" />












