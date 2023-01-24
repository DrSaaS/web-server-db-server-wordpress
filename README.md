Task
----
The goal is to deploy a Wordpress Application in a 3-Tier Setup on EC2 Instances

- Presentation Layer  This is the user interface - client server or browser on laptop.
- Business Layer : Application or Webserver - An EC2 Linux Server as a web server (Here we shall install WordPress)
- Data Access or Management Layer : - An EC2 Linux server as a database (DB) server 
- Launch, attach and configure EBS Volumes
-  We shall use RedHat OS for this project

---

I  launched an EC2 Instance (Red Hat Linux) - to act as the web server.
\
\
![Instance One Webserver](https://github.com/deleonab/web-server-db-server-wordpress/blob/master/instanceWebServer.JPG?raw=true)

I launched 3 10Gib EBS Volumes (In the same AZ euwest-2b-) and attached them to the webserver instance
I used Gdisk to partition each disk 

```
sudo gdisk /dev/xvdf

 GPT fdisk (gdisk) version 1.0.3

Partition table scan:
  MBR: not present
  BSD: not present
  APM: not present
  GPT: not present

Creating new GPT entries.

Command (? for help branch segun-edits: p
Disk /dev/xvdf: 20971520 sectors, 10.0 GiB
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): D936A35E-CE80-41A1-B87E-54D2044D160B
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 20971486
Partitions will be aligned on 2048-sector boundaries
Total free space is 2014 sectors (1007.0 KiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048        20971486   10.0 GiB    8E00  Linux LVM

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): yes
OK; writing new GUID partition table (GPT) to /dev/xvdf.
The operation has completed successfully.
Now,  your changes has been configured succesfuly, exit out of the gdisk console and do the same for the remaining disks.
```
\
\
I then used the pvcreate utility to mark each as physical volumes (after installing lvm2)
```
sudo pvcreate /dev/xvdf1
sudo pvcreate /dev/xvdg1
sudo pvcreate /dev/xvdh1
```

Use the vgcreate utility to add all 3 PVs to a volume group (VG). I named the VG webdata-vg
```
sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1
```
Then I created the logical volumes
```
sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg
```

The next step is to use mkfs.ext4 to format the logical volumes with ext4 filesystem
```
sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```
Now we should prepare the file structure for our website
I will create the /var/www/html directory to store the website files and  /home/recovery/logs to store a backup of log data
```
sudo mkdir -p /var/www/html

sudo mkdir -p /home/recovery/logs
```
Finally, I will mount /var/www/html on apps-lv logical volume
```
sudo mount /dev/webdata-vg/apps-lv /var/www/html/
```

I will use the rsync utitilty to back up all the files in the log directory /var/log into /home/recovery/logs as all data on /var/log will be wiped off.
```
sudo rsync -av /var/log/. /home/recovery/logs/
```
Mount /var/log on logs-lv logical volume
```
sudo mount /dev/webdata-vg/logs-lv /var/log
```
Restore log files back into /var/log directory
```
sudo rsync -av /home/recovery/logs/. /var/log
```
We will update the /etc/fstab file so that the mount configuration will persist after restart of the server.

#####  PREPARE DATABASE SERVER

I launched a second RedHat EC2 instance as the DB Server
I repeated the same steps as for the Web Server, but instead of apps-lv create db-lv and mounted it to /db directory instead of /var/www/html/.

##### installWordpress
I Installed WordPress on the Web Server EC2 Instance
```
sudo yum -y update
```
Install wget, Apache and it’s dependencies



```
sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
```
![Install Apache](https://github.com/deleonab/web-server-db-server-wordpress/blob/master/apachesuccessfullyinstalled.JPG?raw=true)

Start Apache
```
sudo systemctl enable httpd
sudo systemctl start httpd
```
![Start Apache](https://github.com/deleonab/web-server-db-server-wordpress/blob/master/httpdstart.JPG?raw=true)

To install PHP and it’s dependencies
```
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum module list php
sudo yum module reset php
sudo yum module enable php:remi-7.4
sudo yum install php php-opcache php-gd php-curl php-mysqlnd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
setsebool -P httpd_execmem 1
```
Restart Apache
```
sudo systemctl restart httpd
```
Download wordpress and copy wordpress to var/www/html
```
  mkdir wordpress
  cd   wordpress
  sudo wget http://wordpress.org/latest.tar.gz
  sudo tar xzvf latest.tar.gz
  sudo rm -rf latest.tar.gz
  cp wordpress/wp-config-sample.php wordpress/wp-config.php
  cp -R wordpress /var/www/html/
  ```
Configure SELinux Policies
```
  sudo chown -R apache:apache /var/www/html/wordpress
  sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
  sudo setsebool -P httpd_can_network_connect=1
 ````
#####  Install MySQL on our DB Server EC2
```
sudo yum update
sudo yum install mysql-server
```
Let us verify that the service is up and running by using sudo systemctl status mysqld
```
sudo systemctl restart mysqld
sudo systemctl enable mysqld
```
##### — I will configure the DB to work with WordPress
```
sudo mysql
CREATE DATABASE wordpress;
CREATE USER `myuser`@`<Web-Server-Private-IP-Address>` IDENTIFIED BY 'mypass';
GRANT ALL ON wordpress.* TO 'myuser'@'<Web-Server-Private-IP-Address>';
FLUSH PRIVILEGES;
SHOW DATABASES;
exit
```

I will open MySQL port 3306 on DB Server EC2 and allow access to the DB server ONLY from our Web Server’s IP address so in the Inbound Rule configuration specify source as /32

I will Install MySQL client and test that I can connect from the Web Server toyour DB server by using mysql-client
```
sudo yum install mysql
sudo mysql -u admin -p -h <DB-Server-Private-IP-address>

```
Verify the successful connection
```
SHOW DATABASES;
```
command and see a list of existing databases.

I will now enable TCP port 80 in the Inbound Rules configuration for the Web Server EC2 (enable from everywhere 0.0.0.0/0)

I will now access from my browser the link to your WordPress http://<Web-Server-Public-IP-Address>/wordpress/
 
![Wordpress Installed](https://github.com/deleonab/web-server-db-server-wordpress/blob/master/wordpressinstallpage.JPG?raw=true)
 
 
 \
 \
 ![Wordpress logged in](https://github.com/deleonab/web-server-db-server-wordpress/blob/master/wordpress-installed-loggedin.JPG?raw=true)
