# Devops Tooling Website Solution

Here is how I set up NFS shared folders for 3 Apache Web Servers (with the goal of understanding how NFS servers can help store logs and other shared folders in one location).

# Setup NFS Server

1. I deployed a node with an LVM setup similar to the one I created in [Web-Solution-Wordpress](https://github.com/chauntelkellar/Web-Solution-Wordpress/blob/main/Project-Notes.md). <br> Instead of formating the disks as `ext4` they will now be formated as `xfs`

2. Installed NFS Server

   `dnf install nfs-utils`
   
3. Created mount points on /mnt directory for the logical volumes and set permissions
   
   ```
   sudo mkdir /mnt/hmtl    #To be used by webservers
   sudo mkdir /mnt/logs    #To be used by webserver logs
   sudo mkdir /mnt/opt     #To be used by Jenkins in future project for CI pipeline
   
   sudo chmod a+rwx /mnt/html/
   sudo chmod a+rwx /mnt/logs/
   sudo chmod a+rwx /mnt/opt/
   ```
 4. Used `mkfs.xfs` to format the logical volumes with xfs filesystem
 
    ```
    sudo mkfs.xfs /dev/nfs-vg/lv-logs
    sudo mkfs.xfs /dev/nfs-vg/lv-apps
    sudo mkfs.xfs /dev/nfs-vg/lv-opt
    ```
 
 5. Mount LVs to the mount points
    
    ```
    mount /dev/nfs-vg/lv-apps /mnt/html
    mount /dev/nfs-vg/lv-opt /mnt/opt
    mount /dev/nfs-vg/lv-logs /mnt/logs
    ```
    
 6.  Adding mount info to fstab file to allow persistence on system reboot
     
     ```
     /dev/nfs-vg/lv-opt    /mnt/opt                xfs     defaults        0 0
     /dev/nfs-vg/lv-apps   /mnt/html               xfs     defaults        0 0
     /dev/nfs-vg/lv-logs   /mnt/logs               xfs     defaults        0 0
     ```
     
7. Checking the mount

   ```
   df -h 
   
   Output:
   
   Filesystem                    Size  Used Avail Use% Mounted on
   devtmpfs                      372M     0  372M   0% /dev
   tmpfs                         404M     0  404M   0% /dev/shm
   tmpfs                         404M   11M  393M   3% /run
   tmpfs                         404M     0  404M   0% /sys/fs/cgroup
   /dev/xvda2                     10G  1.9G  8.2G  19% /
   tmpfs                          81M     0   81M   0% /run/user/1000
   /dev/mapper/nfs--vg-lv--apps  5.0G   68M  5.0G   2% /mnt/html
   /dev/mapper/nfs--vg-lv--opt   5.0G   68M  5.0G   2% /mnt/opt
   /dev/mapper/nfs--vg-lv--logs  4.9G   68M  4.9G   2% /mnt/logs

8. Starting and enabling nfs service

   ```
   sudo systemctl start nfs-server.service
   sudo systemctl enable nfs-server.service
   ```
   
 9. Edited the /etc/exports file to allow the NFS for remote server
 
    ```
    sudo vi /etc/exports
    #add the below text and save :wq
    /mnt 192.168.1.178/24 (rw,sync,no_root_squash)
    /mnt/opt 192.168.1.178/24 (rw,sync,no_root_squash)
    /mnt/html 192.168.1.178/24 (rw,sync,no_root_squash)
    /mnt/logs 192.168.1.178/24 (rw,sync,no_root_squash)
    ```
    
10. To export the file system, I ran the exportfs command with the -arv flag.  

    > -a flag means export or unexport all directories, -r means reexport all directories, synchronizing /var/lib/nfs/etab with /etc/exports and files under /etc/exports.d, and -v enables verbose output.
    
    ```
    Output:
    exporting 192.168.1.178/24:/mnt/logs
    exporting 192.168.1.178/24:/mnt/html
    exporting 192.168.1.178/24:/mnt/opt
    exporting 192.168.1.178/24:/mnt
    ```
    
11. Install firewalld and allow traffic to the necessary NFS services (mountd, nfs, rpc-bind) via firewall, then reload to apply the changes:

    ```
    sudo yum install firewalld
    sudo yum enable firewalld
    sudo yum start firewalld
    
    sudo firewall-cmd --permanent --add-service=nfs
    sudo firewall-cmd --permanent --add-service=rpc-bind
    sudo firewall-cmd --permanent --add-service=mountd
    sudo firewall-cmd --permanent --add-source=192.168.1.178/24
    sudo firewall-cmd --reload
    ```
   
# Configure the database server

1. Install Mariadb and login ass root and accept all the default values

```bash
sudo apt update
sudo apt install mariadb-server -y
sudo mysql_secure_installation
mariadb
```

2. Create databse, user account, grant permission to user on tooling database to only be accessible from webservers subnet, and reload 

```bash
CREATE DATABASE tooling;
CREATE USER 'webaccess'@'192.168.1.%' IDENTIFIED BY '2020Over';
GRANT ALL PRIVILEGES ON tooling.* TO 'webaccess'@'192.168.1.%' IDENTIFIED BY '2020Over' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

3. Validate database was created

```bash
SELECT User, Host FROM mysql.user WHERE Host <> 'localhost';
Output:

+-----------+-------------+
| User      | Host        |
+-----------+-------------+
| webaccess | 192.168.1.% |
+-----------+-------------+
1 row in set (0.000 sec)
```

4. Amend the bind config 

```bash
vi /etc/mysql/mariadb.conf.d/50-server.cnf
bind-address         = 0.0.0.0
```

5.  Enable and start mariadb

```bash
sudo systemctl enable mariadb.service
systemctl start mariadb.service
```

6. Whitelist known IP subnet range to allow traffic, allow HTTP, and allow HTTPS, allow SQL connection from a known submet range, and enable firewall

```bash
sudo ufw allow from 192.168.1.0/24
sudo ufw allow http
sudo ufw allow https
sudo ufw allow from 192.168.1.0/24 to any port 3306
sudo ufw enable
```

7. Install git to get sql dump file to inmport

```bash
sudo apt install git -y
```

8. Clone repo for Tooling to the /tmp folder

```bash
git clone https://github.com/dinulhaque/tooling.git /tmp/tooling

```

9. Import sql dump file

```bash
mysql -u webaccess  -h 192.168.1.120 -p tooling  < /tmp/tooling/tooling-db.sql
```

# Prepare the webservers

1. Install NFS

```bash
dnf install nfs-utils nfs4-acl-tools -y
```

2. Whitelist NFS and MariaDB server IP and reload firewall

```bash
firewall-cmd --permanent --add-source=192.168.1.0/24
sudo firewall-cmd --reload
```

3. Use showmount command to show mount information for the NFS Server

```bash
showmount -e 192.168.1.178

#output:

/mnt/logs (everyone)
/mnt/html (everyone)
/mnt/opt  (everyone)
/mnt      (everyone)
```

4. Mount /var/www/ and target the NFS server's export for apps and provide read/write permissions

```bash
mkdir -p /var/www
chmod a+rwx /var/www/
mount -t nfs 192.168.1.178:/mnt/html /var/www

```

5. confirm the remote file system was mounted use 

```bash
mount | grep nfs
```

Output:

```bash
sunrpc on /var/lib/nfs/rpc_pipefs type rpc_pipefs (rw,relatime)
192.168.1.178:/mnt/html on /var/www type nfs4 (rw,relatime,vers=4.2,rsize=262144,wsize=262144,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.1.178,local_lock=none,addr=192.168.1.178)
```

6. Enable mount to be persistent

```bash
vi /etc/fstab
```

```bash
#pasted the following value in the file
echo "192.168.1.178:/mnt/html                  /var/www	          nfs     defaults        0 0" >> /etc/fstab
```

6. Install pache, php, and mariadb

```bash
sudo dnf install httpd php php-mysqlnd
sudo dnf install mariadb-server -y
sudo systemctl start mariadb 
```
7. Configure Apache to serve contents over HTTP/S and mysql and reload the firewall to apply changes

```bash
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=mysql
sudo firewall-cmd --reload
```

8. Start Apache webserver

```bash
sudo systemctl start httpd
sudo systemctl status httpd
```

9. check mount on webserver 

```bash
df -h
```

```bash
[root@osboxes html]# df -h
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 887M     0  887M   0% /dev
tmpfs                    914M     0  914M   0% /dev/shm
tmpfs                    914M  9.6M  904M   2% /run
tmpfs                    914M     0  914M   0% /sys/fs/cgroup
/dev/sda2                236G  6.3G  230G   3% /
/dev/sda5                254G  2.0G  252G   1% /home
/dev/sda1                976M  263M  646M  29% /boot
tmpfs                    183M  1.2M  182M   1% /run/user/42
tmpfs                    183M  4.6M  179M   3% /run/user/1000
/dev/sr0                  59M   59M     0 100% /run/media/osboxes/VBox_GAs_6.1.17
192.168.1.178:/mnt/html 1014M   40M  975M   4% /var/www
```

10. Mount httpd log to /mnt/logs

```bash
mount -t nfs 192.168.1.178:/mnt/logs /var/log/httpd
```

11. Install git and clone repo for tooling to the /tmp folder

```bash
sudo dnf install git -y
git clone https://github.com/chauntelkellar/tooling.git /tmp/tooling
```

12. Copy /tmp/tooling/html folder contents to /var/www/html

```bash
cp -R /tmp/tooling/html /var/www/html
```

 Showing /var/www/html folder contents:

```bash
ls -ltr /var/www/html/
total 40
-rwxr-xr-x. 1 apache apache 2964 Mar 08 11:54 admin_tooling.php
-rwxr-xr-x. 1 apache apache 1531 Mar 08 11:54 create_user.php
-rwxr-xr-x. 1 apache apache 4353 Mar 08 11:54 functions.php
drwxr-xr-x. 2 apache apache  183 Mar 08 11:54 img
-rwxr-xr-x. 1 apache apache 3217 Mar 08 11:54 index.php
-rwxr-xr-x. 1 apache apache  781 Mar 08 11:54 login.php
-rwxr-xr-x. 1 apache apache   19 Mar 08 11:54 README.md
-rwxr-xr-x. 1 apache apache 1097 Mar 08 11:54 register.php
-rwxr-xr-x. 1 apache apache 1704 Mar 08 11:54 style.css
-rwxr-xr-x. 1 apache apache 1027 Mar 08 11:54 tooling_stylesheets.css
```

13. Change permissions

Changing folder permission and chmod permission

```bash
sudo chown -R apache:apache /var/www/
chmod 770 -R /var/www
```

14. Amend httpd.conf file to reference index.php file instead of default index.html

```bash
sudo vi /etc/httpd/conf/httpd.conf
#Added in index.php before index.html

<IfModule dir_module>
    DirectoryIndex index.php index.html
</IfModule>

```

15. Updated db setting in functions.php file

```bash
sudo vi /var/www/html/functions.php

#added in file
$db = mysqli_connect('192.168.1.120', 'webaccess', '2020Over', 'tooling');

#save and exit
:wq!
```

16. Mount logs  

```bash
mount -t nfs 192.168.1.178:/mnt/logs /var/log/httpd/
```

17. Make persistent

```bash
sudo vi /etc/fstab

#add below to EOF
192.168.1.178:/mnt/logs                  /var/log/httpd    nfs            defaults        0 0
```

19. Restarted mariadb and httpd and connected to remote sql

```bash
mysql -u webaccess -h 192.168.1.120 -p '2020Over'
```

20. Testing access via browser on http://192.168.1.121, success!
![screenshot/toolingwebsite.png](/screenshot/toolingwebsite.png)

References

[How to Set Up NFS Server and Client on CentOS 8](https://www.tecmint.com/install-nfs-server-on-centos-8/)

[Configuring MariaDB for Remote Client Access](https://mariadb.com/kb/en/configuring-mariadb-for-remote-client-access/)

