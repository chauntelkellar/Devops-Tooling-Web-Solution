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

6. Whitelist known IP submet range to allow traffic, allow HTTP, and allow HTTPS, allow SQL connection from a known submet range, and enable firewall

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
192.168.1.1178:/mnt/html on /var/www type nfs4 (rw,relatime,vers=4.2,rsize=262144,wsize=262144,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.1.121,local_lock=none,addr=192.168.1.178)
```

6. Enable mount to be persistent

```bash
vi /etc/fstab
```

```bash
#pasted the following value in the file
echo "192.168.1.178:/mnt/html                  /var/www	          nfs     defaults        0 0" >> /etc/fstab
```

