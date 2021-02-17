# Devops Tooling Website Solution

Here is how I set up NFS shared folders for 3 Apache Web Servers (with the goal of understanding how NFS servers can help stor logs and other shared folders in one location):

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
    /mnt 172.31.6.178/24 (rw,sync,no_root_squash)
    /mnt/opt 172.31.6.178/24 (rw,sync,no_root_squash)
    /mnt/html 172.31.6.178/24 (rw,sync,no_root_squash)
    /mnt/logs 172.31.6.178/24 (rw,sync,no_root_squash)
    ```
    
10. To export the file system, I ran the exportfs command with the -arv flag.  

    > -a flag means export or unexport all directories, -r means reexport all directories, synchronizing /var/lib/nfs/etab with /etc/exports and files under /etc/exports.d, and -v enables verbose output.
    
    ```
    Output:
    exporting 172.31.6.178/24:/mnt/logs
    exporting 172.31.6.178/24:/mnt/html
    exporting 172.31.6.178/24:/mnt/opt
    exporting 172.31.6.178/24:/mnt
    ```
    
11. Install firewalld and allow traffic to the necessary NFS services (mountd, nfs, rpc-bind) via firewall, then reload to apply the changes:

    ```
    sudo yum install firewalld
    sudo yum enable firewalld
    sudo yum start firewalld
    
    sudo firewall-cmd --permanent --add-service=nfs
    sudo firewall-cmd --permanent --add-service=rpc-bind
    sudo firewall-cmd --permanent --add-service=mountd
    sudo firewall-cmd --permanent --add-source=172.31.6.178/24
    sudo firewall-cmd --reload
    ```
   
