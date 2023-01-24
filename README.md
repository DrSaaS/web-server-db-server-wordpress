Our 3-Tier Setup
A Laptop or PC to serve as a client
An EC2 Linux Server as a web server (Here we shall install WordPress)
An EC2 Linux server as a database (DB) server
We shall use RedHat OS for this project

#I  launched 2 EC2 Instance (Red Hat Linux) - One for the webserver and the other for the database server.

# I launched 3 10Gib EBS Volumes (In the same AZ euwest-2b-) and attached them to each instance

Instance(Webserver)
VolWebserver1, VolWebserver2, VolWebserver3 

# I connected (SSH)into my webserver instance from the terminal (Windows Terminal) and from the local directory
in which the pem key was stored.

# Then I ran lsblk to check the available volumes

lsblk
# All 3 volumes attached.  xvdf,xvdg,xvdh

ls /dev/ to see all devicesattached

df -h To see all mounts and free space

# Next, we use gdisk utility to create a single partition on each of the 3 disk volumes by running:

sudo gdisk /dev/xvdf(or volume name) - Partition the 1st volume

# At the prompt, we enter p:

p

# Next we write/commit by entering w:

w


# Partition created

# We do the same for the other 2

sudo gdisk /dev/xvdg

sudo gdisk /dev/xvdh

----------------------------------
PROJECT 6 VIDEO METHOD IN BETWEEN DASHES
------------------------------------------
df -h    shows mount points available
1. To create partition on the physical disk
2. Then create a physical volume
3. Then a volume group
4. Create a logical volume


To create partition#
------------------
sudo gdisk /dev/xdvf   - first physical volume

Then click n for a new partition

partition number, we can select default 1
first sector: choose default Enter
Last sector: choose default   Enter
current type is Linux Filesystem but we want Linux LVM

# We use code 8e00 to change type to LVM

8e00

# Next, we enter p to print what we have done

p

# number start end .......
    1     2048
# Then we write by entering w:

w

# message operation completed successfully

# We repeated the steps for xvdg and xvdh

#partitions created were:

xvdf1
xvdg1
xvdh1


# Next, we installed LVM2 package: Logical Volume Manager 

sudo yum install lvm2

# Next we create physical volumes in the three partitions using pvcreate command:

sudo pvcreate /dev/xvdf1 /dev/xvdg1 /dev/xvdh1

# We ran sudo pvs to show physical volumes

sudo pvs

# Now we need to add them to a volume group (concatenates the physical volumes). 
# We called the volume group webdata-vg

sudo vgcreate webdata-vg /dev/xvdf1 /dev/xvdg1 /dev/xvdh1

# volume group webdata-vg was successfully created

# To see volume groups

sudo vgs   

# Next we create the logical volume which will be used. It will be 2. One for the web application (apps-lv)
# and other half called logs-lv  to store logs

sudo lvcreate -n apps-lv -L 14G vg-webdata       -n means name

sudo lvcreate -n logs-lv -L 14G vg-webdata

# Logical volumes created successfully

# I ran command sudo lvs to see the logical volumes created.

# with LVM we can increase out logical volume on the fly by creating another physical volume
# then we add to the volume group with vgextend, then extend the logical volume

# Next, we need to add a file system  ext4 is one

sudo mkfs.ext4 /dev/webdata-vg/apps-lv

sudo mkfs.ext4 /dev/webdata-vg/logs-lv

# Next we need a mount point for our devices

#Documentation wants us to 

# Create /var/www/html directory to store website files

sudo mkdir -p /var/www/html   -p means parent and creates www and html. 
# It will fail without -p if www doesn't exist

# Create /home/recovery/logs to store backup of log data

sudo mkdir -p /home/recovery/logs

# Next, we Mount /var/www/html on the apps-lv logical volume

sudo mount /dev/webdata-vg/apps-lv /var/www/html/  in reality, it is html being mounted

# - Always check(before mounting) if contentexisting(html) as mounting will wipe out existing data if present

ls -l /var/www/html

Use rsync utility to backup all the files in the log directory /var/log into /home/recovery/logs (This is required before mounting the file system)
sudo rsync -av /var/log/ /home/recovery/logs/
# We have to back up the log files before the mount operation
# we want to mount on /var/log/ but back up first in /home/recovery/logs
# We checked and verified that it was backed up
ls -l /home/recovery/logs/log
# Now we can mount before restoring the files
mount /var/log on logs-lv logical volume. 
# (Note that all the existing data on /var/log will be deleted. That is why we need to copy the files(back up) 
to home/recovery/logs, mount on /var/log and then restore files from recovery/logs to var/log

(important)
sudo mount /dev/webdata-vg/logs-lv /var/log

# check mounts by using mount command:

mount | grep webdata

# Restore log files back into /var/log directory

sudo rsync -av /home/recovery/logs/log/ /var/log  dot(.) used in docs, logs used in vid

ls -l /var/log

# Files successfully recovered

# The logical mounts are still temporary and will be lost if the instance restarts

#Update /etc/fstab file so that the mount configuration will persist after restart of the server.


The UUID of the device will be used to update the /etc/fstab file;

sudo blkid
# or to be specific

sudo blkid /dev/webdata-vg/apps-lv
#extracted the UUID and included the path:



UUID=8fe0e9a5-f5b0-45e6-a9be-8387eb0c15fa /var/www/html      
UUID=8d1deec7-896b-4cfc-bfb0-e8fb61f1e08f  /var/log      






sudo vi /etc/fstab

Update /etc/fstab in this format using your own UUID and rememeber to remove the leading and ending quotes.


#Test the configuration and reload the daemon
#sudo mount -a will output errors if not mounted properly

sudo mount -a

# Next, we reload the daemon

sudo systemctl daemon-reload

# We verify our setup by running df -h 
df -h


---------------------------

Step 2 — Prepare the Database Server
Launch a second RedHat EC2 instance that will have a role – ‘DB Server’
Repeat the same steps as for the Web Server, but instead of apps-lv create db-lv and mount it
 to /db directory instead of /var/www/html/.

To create partition#
------------------
sudo gdisk /dev/xdvb   - first physical volume

Then click n for a new partition

partition number, we can select default 1
first sector: choose default Enter
Last sector: choose default   Enter
current type is Linux Filesystem but we want Linux LVM

# We use code 8e00 to change type to LVM

8e00

# Next, we enter p to print what we have done

p

# number start end .......
    1     2048
# Then we write by entering w:

w

# message operation completed successfully

# We repeated the steps for xvdc 


# Next, we installed LVM2 package:

sudo yum install lvm2

# Next we create physical volumes in the three partitions using pvcreate command:

sudo pvcreate /dev/xvdb1 /dev/xvdc1 

# We ran sudo pvs to show physical volumes

sudo pvs

# Now we need to add them to a volume group (concatenates the physical volumes). 
# We called the volume group vg-webdata

sudo vgcreate db-vg /dev/xvdb1 /dev/xvdc1 

# volume group db-vg was successfully created

# Next we create the logical volume (db-lv) which will be mounted to /db

sudo lvcreate -n db-lv -L 20G db-vg       -n means name

sudo lvcreate -n logs-lv -L 20G db-vg
# Logical volume created successfully

# Next we need to create the db folder in root

sudo mkdir /db


# with LVM we can increase out logical volume on the fly by creating another physical volume
# then we add to the volume group with vgextend, then extend the logical volume

# Next, we need to add a file system  ext4 

sudo mkfs.ext4 /dev/database-vg/db-lv
sudo mkfs.ext4 /dev/database-vg/logs-lv


# we create /home/recovery/log and Back up var/log files
 sudo mkdir -p /home/recovery/log && sudo
 rsync -av /var/log /home/recovery/log

# WE CAN SKIP THE BACK UP STAGE AS WE HAVE ONLY JUST CREATED /db

#Now we can mount on /db

sudo mount /dev/database-vg/db-lv /db
sudo mount /dev/database-vg/logs-lv /var/log


# We check by running df -h
df -h  

 sudo rsync -av /home/recovery/log /var/log 

# The logical mounts are still temporary and will be lost if the instance restarts

#Update /etc/fstab file so that the mount configuration will persist after restart of the server.


The UUID of the device will be used to update the /etc/fstab file;

sudo blkid




sudo vi /etc/fstab

Update /etc/fstab in this format using your own UUID and rememeber to remove the leading and ending quotes.


#Test the configuration and reload the daemon
#sudo mount -a will output errors if not mounted properly

sudo mount -a

# Next, we reload the daemon

sudo systemctl daemon-reload

# We verify our setup by running df -h 
df -h




------------------------------------------------
Step 3 — Install WordPress on our Web Server EC2
Update the repository

sudo yum -y update

# Install wget, Apache and it’s dependencies

sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json

Start Apache

sudo systemctl enable httpd
sudo systemctl start httpd
To install PHP and it’s dependencies

sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum module list php    lists available php versions
sudo yum module reset php
sudo yum module enable php:remi-7.4
sudo yum install php php-opcache php-gd php-curl php-mysqlnd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo setsebool -P httpd_execmem 1     - To instruct SELinux to allow Apache to execute PHP code via PHP-FPM run
Restart Apache
#check if httpd installed 
sudo systemctl status httpd
# start it if dead
sudo systemctl start httpd

#check if httpd installed again if was dead before
sudo systemctrl status httpd

#Lastly we checked the webserver's public IP address on the web browser.
#Apache page was displayed. Apache was successfully installed and configured.

visit the database server through its public ip on the web. it should show apache installed Test Page
-----------------------------------------

# 
------------------------------------------

Download wordpress and copy wordpress to var/www/html

  mkdir wordpress
  cd   wordpress
# We shall do all the extraction in the wordpress folder then copy it to our /var/www/html folder
# Let us download it
  sudo wget http://wordpress.org/latest.tar.gz
# next we extract it
  sudo tar -xvzf latest.tar.gz
  sudo rm -rf latest.tar.gz

# we need to create new wordpress config file first
# We navigated to the inner wordpress(/..wordpress/wordpress) folder containing wp-config-sample.php and copied into into a new file wp-config.php
  sudo cp -R wp-config-sample.php wp-config.php
-----------------------------------------------CONTINUE FROM HERE 17/04/2022 1:33:18

sudo cp -R wordpress/. /var/www/html/
 
Configure SELinux Policies

  sudo chown -R apache:apache /var/www/html/wordpress
  sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
  sudo setsebool -P httpd_can_network_connect=1

Step 4 — Install MySQL on your client server(red hat doesn't have mysql client) and DB Server EC2
sudo yum update
sudo yum install mysql-server
Verify that the service is up and running by using sudo systemctl status mysqld, if it is not running, restart the service and enable it so it will be running even after reboot:

sudo systemctl start mysqld (Start service on webserver (client))
sudo systemctl enable mysqld
sudo mysqlctl status mysqld (Ensure that it is running and active)

Step 5 — Configure DB to work with WordPress
sudo systemctl start mysqld (Start service on databaseserver (server))
sudo systemctl enable mysqld
sudo mysqlctl status mysqld (Ensure that it is running and active)

# Now we have mysqld enabled and running on both client and server
# We run mysql_secure_installation to enforce strong password

sudo mysql_secure_installation

sudo mysql

sudo mysql -u root -p
#Enter mysql password

CREATE DATABASE wordpress;
CREATE USER `admin`@`172.31.46.29` IDENTIFIED WITH mysql_native_password BY 'Olupero2023%mysql';    # private ip address of webserver
GRANT ALL ON wordpress.* TO 'admin'@'172.31.46.29' WITH GRANT OPTION;
FLUSH PRIVILEGES;
SHOW DATABASES;

select user,host from mysql.user  # just to test

# We need to set the bind address configure /etc
sudo vi /etc/my.cnf  (for red hat)
#add
[mysqld]
bind-address=172.31.46.29
# Restart mysql as config file modified
sudo systemctl restart mysqld


Step 6 — Configure WordPress to connect to remote database.
Hint: Do not forget to open MySQL port 3306 on DB Server EC2. For extra security, you shall allow access to the DB server ONLY from your Web Server’s IP address, so in the Inbound Rule configuration specify source as /32



Install MySQL client and test that you can connect from your Web Server to your DB server by using mysql-client
sudo yum install mysql
sudo mysql -u admin -p -h 172.31.45.77
Verify if you can successfully execute SHOW DATABASES; command and see a list of existing databases.

# sudo vi wp-config.php   with db name , user name and hoast name(dbservers IP)
# move default apache to different location  sudo mv /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/welcome.conf_backup
Change permissions and configuration so Apache could use WordPress:
On Webserver: 
sudo chown -R apache:apache /var/www/html/
Configure SELinux Policies

  sudo chown -R apache:apache /var/www/html/
  sudo chcon -t httpd_sys_rw_content_t /var/www/html/ -R
  sudo setsebool -P httpd_can_network_connect=1
  sudo setsebool -P httpd_can_network_connect_db 1

Enable TCP port 80 in Inbound Rules configuration for your Web Server EC2 (enable from everywhere 0.0.0.0/0 or from your workstation’s IP)

# We accessed WordPress http://<Web-Server-Public-IP-Address>



# Wordpress successfully installed




















