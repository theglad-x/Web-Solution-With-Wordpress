# Web-Solution-With-Wordpress

# Step 1: Prerequisities
1. Launch a Redhat EC2 instance on AWS for webserver.
2. Configure security group, open port 80(http), 22(ssh), and 3306(mysql).
3. Create 3 volumes in the same AZ as the Web Server EC2, each of 10GB.
4. Attach the volumes to the server instance using device names /dev/sdc and /dev/sdd.
![volumes](https://github.com/user-attachments/assets/22461ae0-e1fd-4b39-a4bc-67503c5c8570)
![Attach volume](https://github.com/user-attachments/assets/b536b444-c118-4297-96d5-a3788f72bd31)


# Step 2: 
1. Connect to the web server instance
```bash
 ssh -i "keypair" ec2-user@ec2-34-224-95-90.compute-1.amazonaws.com
```
![ssh](https://github.com/user-attachments/assets/25697f2c-b352-46f4-b8f6-9606944f603f)
2. Inspect block devices attached to the server
```bash
lsblk
```
![lsblk](https://github.com/user-attachments/assets/8677cc60-a211-4e24-8ad2-e6f88628d1ca)

3. See all mounts and free space on your server
```bash
df -h
```
![mount and free space](https://github.com/user-attachments/assets/61bf2a81-8947-4b60-8f32-608005668422)
4. Create a single partition on each of the 3 disks
```bash
sudo gdisk /xvd3
```
![gdisk](https://github.com/user-attachments/assets/2406ed2a-ba96-4ffa-a6fb-10a0128f81cd)
![gdisk-data](https://github.com/user-attachments/assets/d8263dbe-f57d-4b13-9e5e-7d200c1868ad)
5. Use ```lsblk``` utility to view the newly onfigured partition on each of the 3 disks
![lsblk-verify](https://github.com/user-attachments/assets/4d78eeaa-fca5-4d76-b004-adfbf9ceb8f0)
6.  Install ```lvm2``` package and use ```sudo lvmdiskscan``` to check for a vailabe partitions.
```bash
sudo yum install lvm2
```
![lvm2](https://github.com/user-attachments/assets/3c288a29-321a-412f-a0da-8e908a15d02d)

7. Mark each 3 disks as physical volumes (pvs) to be used by LVM
``` bash
  sudo pvcreate /dev/xvdb
  sudo pvcreate /dev/xvdc
  sudo pvcreate /dev/xvdd
```
![pvcreate](https://github.com/user-attachments/assets/a304b4ed-3026-416e-bc67-2c09c1c32e8b)
8. Verify that the physical volume has been created successfully
```bash
sudo pvs
```
![pvs](https://github.com/user-attachments/assets/d3acbb8d-af56-4dfc-997a-8d768e333c26)
9. Use ```vgcreate``` utility to add all 3 PVs to a volume group (VG). Name the VG ```webdata-vg```
```bash
sudo vgcreate webdata-vg /dev/xvdb /dev/xvdc /dev/xvdd
```
verify the VG is created and running
![vgcreate](https://github.com/user-attachments/assets/950b1669-da5c-4c03-b1ad-ea1d0576da9e)
10. Create 2 logical volumes. ```apps-lv``` to be used to store data for the website while, ```logs-lv``` will be used to store data for logs.
```bash
sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg
```
![create-volumes](https://github.com/user-attachments/assets/52d3bb0f-b5c6-4c88-a827-6b830ee4474b)
11. Verify the 2 logical volumes has been created successfully
```bash
sudo lvs
```
![linked-volumes](https://github.com/user-attachments/assets/7a8d5689-4fe1-45d4-8c97-cbb6cc2e18a4)
12. Verify the entire setup
```bash
sudo vgdisplay -v
```
![vgdisplay](https://github.com/user-attachments/assets/898c8584-fcb0-4fd5-b065-af34f2910c7b)
13. Format the logical volumes with ```ext4``` filesystem
```bash
sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```
![vol-format](https://github.com/user-attachments/assets/a82a77de-0d56-4f5d-b721-0e45f3352d65)

14. Create ```/var/www/html``` directory to store website files
```bash
sudo mkdir -p /var/www/html
```
![website](https://github.com/user-attachments/assets/5ad16282-9bef-41c9-8fe0-837bbc9aef1f)
15.  Create ```/home/recovery/logs``` directory to store backup of log data
```bash
sudo mkdir -p /home/recovery/logs
```
16. Mount ```/var/www/html``` on ```apps-lv```logical volume
```bash
sudo mount /dev/webdata-vg/apps-lv /var/www/html/
```
17. Use ```rsync``` utility to backup all the files in the log directory ```/var/log``` into ```/home/recovery/logs```
```bash
sudo rsync -av /var/log/ /home/recovery/logs/
```
18. Mount ```/var/log``` on ```log-lv```
```bash
sudo mount /dev/webdata-vg/logs-lv   /var/log/
```
19. Restore log files back into ```/var/log``` directory
```bash
sudo rsync av /home/recovery/logs/ /var/log
```
![rsync](https://github.com/user-attachments/assets/7c055d1c-b81d-4359-a8a2-4456871e21b3)
20. Update ```/etc/fstab``` file so that the mount configuration will persist after restart of the server. 
The UUID of the device will be used to update the ```/etc/fstab``` file.
```bash
sudo blkid
```
![blkid](https://github.com/user-attachments/assets/4eb39f72-6fe9-43da-adaf-ea9f891bcc46)
```bash
sudo vi /etc/fstab
```
Update /etc/fstab in this format using your own UUID and remember to remove the leading and ending quotes.
![UUID](https://github.com/user-attachments/assets/ad493486-72fe-4156-90b6-5a283b2583b4)
21. Test the configuration and reload the daemon
```bash
sudo mount -a
sudo systemctl daemon-reload
```
22. Verify the setup by running ```df -h```
![UUID2](https://github.com/user-attachments/assets/ad94c488-1fef-42a4-852f-0d46be39c7b8)
# Step 2: Prepare the Database Server
1. Launch a second RedHat EC2 instance for DB Server
2. Repeat the same steps as for Web Server, but instead of ```apps-lv``` create ```db-lv``` and mount it to ```/db``` directory instead of ```/var/www/html/```
# Step 3: Install WordPress on your Web Server EC2
1. Update the repositiory
```bash
sudo yum -y update
```
![update](https://github.com/user-attachments/assets/dd69c100-76cf-4204-b218-0b46b1393d2f)
2. Install wget, Apache and it's dependencies
```bash
sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
```
![wget-apache-php](https://github.com/user-attachments/assets/365ed96b-a9f4-478d-a8d2-aeef45bc34b6)
3. Enable and start Apache
```bash
sudo systemctl enable httpd 
sudo systemctl start httpd
```
4. Install PHP and it's dependencies.
```bash
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum module list php
sudo yum module reset php
sudo yum module enable php:remi-7.4
sudo yum install php php-opcache php-gd php-curl php-mysqlnd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm setsebool -P httpd_execmem 1
```
![php](https://github.com/user-attachments/assets/0c9799af-f06a-46ae-9815-1635cd90ad1a)
5. Restart Apache
```bash
sudo systemctl restart httpd
```
6. Download and Copy wordpress to ```/var/www/html``` directory
```bash
mkdir wordpress
cd wordpress
sudo wget http://wordpress.org/latest.tar.gz
sudo tar xzvf latest.tar.gz
sudo rm -rf latest.tar.gz
sudo cp wordpress/wp-config-sample.php wordpress/wp-config.php
cp -R wordpress /var/www/html
```
9. Configure SElinux policies
```bash
sudo chown -R apache:apache /var/www/html/wordpress
sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
sudo setsebool -P httpd_can_network_connect=1
```
# Step 4: Install  MySQL and configure ```mysql-server``` on the DB Server EC2 
1. Update the respository
```bash
sudo yum -y update
```
2. Install mysql-server
```bash
sudo yum install mysql-server
```
![mysql](https://github.com/user-attachments/assets/6345ba70-39a9-4770-a2f2-b236f1590c52)
3. Verify that the service is up and running by using ```sudo systemctl status mysqld```
```bash
sudo systemctl status mysqld
sudo systemctl restart mysqld
sudo systemctl enable mysqld
```
![status-mysql](https://github.com/user-attachments/assets/c9a8c02b-33af-40f7-ba2e-aecd0dc7937c)
# Step 5: Configure the DB to work with wordpress
First, log in as root user
```bash
sudo mysql
```
Create a new database user
```bash
CREATE DATABASE wordpress;
CREATE USER 'myuser'@'web-server-private-IP-address' IDENTIFIED BY 'password';
GRANT ALL ON wordpress.* TO 'myuser'@'web-server-private-IP-address';
FLUSH PRIVILEGES;
SHOW DATABASES;
```
# Step 6: Configure WordPress to connect to remote database
1. Access webserver instance and install mysql client
```bash
sudo yum install -y mysql-client
```
2. Log in to the mysql server remotely from webserver.
```bash
sudo mysql -u myuser -p -h (your mysql server ip address)
```
![connect](https://github.com/user-attachments/assets/b774bb33-67b0-496a-a7a4-b6e4507fc84f)

# Step 7: 
1. Log into your webserver ec2 instance to access your wordpress directory.
```bash
sudo cd /var/www/html/wordpress
```
2. Access the ```wp-config``` file to Configure wordpress to use your mysql database created earlier
```bash
sudo vi wp-config.php
```
Replace the variables of DB user, DB host, DB password .
![config-db](https://github.com/user-attachments/assets/b9d6a928-1b62-49c5-8e1d-55264efa1e93)
- DB host - mysql server ip address.
- DB password - database password.
- DB user - database user we created earlier
- Save and exit
- Visit your webserver IP
![wordpress](https://github.com/user-attachments/assets/5b72cdbc-7254-49c7-9f02-696641d60ef1)
If this message is seen - it means wordpress hass successfully connected to remote MySQL 
![wordpress-install](https://github.com/user-attachments/assets/6c89dfa3-11e8-4a3f-a84d-886f3180db4b)











