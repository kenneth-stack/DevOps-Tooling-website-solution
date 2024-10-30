# Step 1: Prepare NFS Server
 __1.__  Create an t3. micro EC2 instance, make use of REdhat OS

__2.__  create 3 or 4 ebs volumes and attach to your instance

 __3.__ Access your instance via ssh

```bash
ssh -i <keypair.pem> ec2-user@<ipaddress>
```

 __4.__ Based on your LVM experience from Project 6, Configure LVM on the Server.
- Instead of formating the disks as ext4 you will have to format them as xfs

- Ensure there are 3 Logical Volumes. lv-opt lv-apps, and lv-logs

- Create mount points on /mnt directory for the logical volumes as follow:

Mount lv-apps on /mnt/apps – To be used by webservers Mount lv-logs on /mnt/logs – To be used by webserver logs Mount lv-opt on /mnt/opt – To be used by Jenkins server in Project 8

 __5.__ Use ``lsblk`` command to inspect what block devices are attached to the server.
 ```bash
lsblk
 ```
 ![block_devices](/images/Screenshot%20(546).png)

  __6.__ Use`` df -h`` command to see all mounts and free space on your server.
```bash
df -h
```
![](/images/Screenshot%20(547).png)

 __7.__ Use `gdisk` to create a single partition on each of the 3 disks

```bash
sudo gdisk /dev/xvdb
```
**NOTE:** To create a partition, follow these steps within gdisk:

- Type n to create a new partition.
- Press Enter to accept the default partition number.
- Press Enter to accept the default first sector.
- Press Enter to accept the default last sector, using the entire disk.
- Type w to write the changes and exit.
- 
![partition1](/images/Screenshot%20(548).png)

```bash
sudo gdisk /dev/xvdc
```

![partition2](/images/Screenshot%20(549).png)

```bash
sudo gdisk /dev/xvdd
```
![](/images/Screenshot%20(550).png)

 __8.__ Use `lsblk` utility to view the newly configured partition on each of the 3 disks.

```bash
lsblk
```
![](/images/Screenshot%20(551).png)

 __9.__   Install `lvm2` package. Lvm2 is used for managing disk drives and other storage devices

```bash
sudo yum install lvm2
```
![](/images/Screenshot%20(552).png)

__10.__ Use the `pvcreate` utility tool to mark each of the volumes as physical volumes to be used by LVM

```bash
sudo pvcreate /dev/xvdb1 /dev/xvdc1 /dev/xvdd1
```
![](/images/Screenshot%20(553).png)

__11.__ Verify that the physical volume has been created by running 
```bash
sudo pvs
```
![sudopvs](/images/Screenshot%20(554).png)

__12.__ Use `vgcreate` utility to add all 3 PVs to a volume group. Name the VG `webdata-vg`
```bash
sudo vgcreate webdata-vg /dev/xvdb1 /dev/xvdc1 /dev/xvdd1
```
![vgcreate](/images/Screenshot%20(555).png)


__13.__ Create 3 logical volumes, name one opt-lv ,app-lv and the other logs-lv. For app-lv, use half of the disk size, then use the remaining part fpor the logs-lv
```bash
sudo lvcreate -n opt-lv -L 8G webdata-vg
sudo lvcreate -n apps-lv -L 8G webdata-vg
sudo lvcreate -n logs-lv -L 8G webdata-vg
```

![volume_created](/images/Screenshot%20(556).png)

__14.__ Verify that the logical volumes has been created
```bash
sudo lvs
```
![](/images/Screenshot%20(557).png)

__15.__ Format the logical volumes using xfs
```bash
    sudo mkfs -t xfs /dev/webdata-vg/apps-lv
    sudo mkfs -t xfs /dev/webdata-vg/opt-lv
    sudo mkfs -t xfs /dev/webdata-vg/logs-lv
 ```
 ![](/images/Screenshot%20(558).png)

 __15.__ Create `/mnt` directory to store website files
 
```bash
sudo mkdir /mnt/apps /mnt/logs /mnt/opt
```

 __16.__ Mount `/lv-apps` on `/mnt/apps`logical volume
```bash
sudo mount /dev/webdata-vg/apps-lv /mnt/apps
sudo mount /dev/webdata-vg/opt-lv /mnt/opt
sudo mount /dev/webdata-vg/logs-lv /mnt/logs
```
![](/images/Screenshot%20(559).png)

 __17.__ Update `/etc/fstab` file so that the mount configuration will persist after restart of the server. The UUID of the device will be used to update the `/etc/fstab` file.

```bash
sudo blkid
```
![](/images/Screenshot%20(560).png)

```bash
sudo vi /etc/fstab
```
Update /etc/fstab in this format using your own UUID and remember to remove the leading and ending quotes.

![](/images/Screenshot%20(561).png)


__18.__ Test the configuration and reload the daemon
```bash
sudo mount -a
sudo systemctl daemon-reload
```
__19.__ Verify your setup by running `df h`, output must look like this:
```bash
df -h
```
![](/images/Screenshot%20(562).png)

__20.__ Install NFS server and configure it to start on reboot.

```bash
  sudo yum -y update
  sudo yum install nfs-utils -y
  sudo systemctl start nfs-server.service
  sudo systemctl enable nfs-server.service
  sudo systemctl status nfs-server.service
```

![nfs_installation](/images/Screenshot%20(563).png)

Export the mounts for webservers’ subnet cidr to connect as clients. For simplicity, you will install your all three Web Servers inside the same subnet, but in production set up you would probably want to separate each tier inside its own subnet for higher level of security. To check your subnet cidr – open your EC2 details in AWS web console and locate ‘Networking’ tab and open a Subnet link:
Make sure we set up permission that will allow our Web servers to read, write and execute files on NFS:

```bash
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt

sudo systemctl restart nfs-server.service
```
![](/images/Screenshot%20(564).png)

__21.__ Configure access to NFS for clients within the same subnet (example of Subnet CIDR – 172.31.32.0/20 ):
```bash
sudo vi /etc/exports

/mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/opt <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)

Esc + :wq!

sudo exportfs -arv
```
![](/images/Screenshot%20(565).png)
![](/images/Screenshot%20(566).png)

__22.__ Check which port is used by NFS and open it using Security Groups (add new Inbound Rule)
```bash
rpcinfo -p | grep nfs
```
![](/images/Screenshot%20(567).png)

**Important note: In order for NFS server to be accessible from your client, you must also open following ports: TCP 111, UDP 111, UDP 2049**

![](/images/Screenshot%20(568).png)

### STEP TWO: CONFIGURE THE DATABASE SERVER (MYSQL)

__1.__ Launch 1 Ec2 instance on ubuntu OS

- ssh into it

__2.__ Install mysql server

```bash
sudo apt install -y mysql-server
```
![mysql_installation](/images/Screenshot%20(570).png)

-  Create a database and name it tooling

- Create a database user and name it webaccess

- Grant permission to webaccess user on tooling database to do anything only from the webservers subnet cidr
  
```bash
sudo mysql
CREATE DATABASE tooling;
CREATE USER 'webaccess'@'172.31.0.0/20' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON tooling.* TO 'webaccess'@'172.31.16.0/20';
FLUSH PRIVILEGES;
exit
```
![sudo_mysql](/images/Screenshot%20(571).png)

- Edit the MySQL configuration file to bind it to all IP addresses, (0.0.0.0) Open the MySQL configuration file, which is located at /etc/mysql/mysql.conf.d/mysqld.cnf:

- Go to your ec2 security group in bound rules and add the port 3306 (default mysql port) and allow access from your subnet cidr

## Configuring Web Servers
We will be configuring the three web servers for nfs server connection and installing both apache and php on them.

### Installing NFS Client

1. Install the nfs-client and apache on all the three webservers:

```bash
sudo dnf install nfs-utils nfs4-acl-tools -y
```
![](images/Screenshot%20(572).png)

2. Install Apache (httpd) and git:

Our applicaiton requires a web server to handle HTTP requests, and **Apache** is the most commonly used web server for WordPress installations. 

1. Install **Apache** using the `dnf` package manager:
```bash
   sudo dnf install httpd git
```
>NB: **git** is needed to clone the github repo on the local machine (web servers)

2. Start and enable Apache to ensure it runs on boot:
```bash
   sudo systemctl start httpd
   sudo systemctl enable httpd
 ```

3. Check that Apache is running:
```bash
   sudo systemctl status httpd
```
3. Access the web browser:
   `http://your-server-ip`

   If everything is configured correctly, you should see the default redhat page.

![Default redhat page](images/Screenshot%20(577).png)

### Mounting NFS Shares

1. Mount **/var/www/** and target the NFS server:

```bash
sudo mount -t nfs -o rw,nosuid 172.31.6.37:/mnt/apps /var/www
```
![](images/Screenshot%20(573).png)

> The command **sudo mount -t nfs -o rw,nosuid 172.31.6.37:/mnt/apps /var/www** is used to mount a remote NFS (Network File System) directory on your local system. Let's break it down:
> - **mount:** The command to mount a file system.
>- **t nfs:** Specifies the type of file system to mount, which in this case is NFS.
>- **o rw,nosuid:** These are options passed to the mount command:
>     - **rw:** Mounts the file system with read-write access.
>     - **nosuid:** Prevents the execution of "set-user-identifier" or "set-group-identifier" (suid/sgid) programs from the mounted file system. This increases security by preventing potential privilege escalation from the remote system.
>- **172.31.6.37:/mnt/apps**: The NFS server and exported directory.
>     - **72.31.6.37**: is the IP address of the NFS server.
>     - **/mnt/apps**: is the directory being shared by the NFS server that you're mounting.
>     - **/var/www**: The local directory where the NFS share will be mounted. After the command is run, the contents of the remote /mnt/apps directory will be accessible locally under /var/www.


2. Verify the NFS is mount successfully with the command below:
```bash
df -h
```

3. Persistent Mounts: To ensure the volumes are automatically mounted at boot, updated /etc/fstab:

```bash
sudo vi /etc/fstab
```
Paste the code below:

```yml
# nfs mount location 
172.31.6.37:/mnt/apps /var/www nfs defaults 0 0
```

```bash
sudo systemctl daemon-reload

sudo mount -a
```
### Installing PHP and extensions

1. Verify RHEL Version

Before starting, ensure you are running **RHEL 9.4**. This can be done with the following command:

```bash
cat /etc/redhat-release
```


Enable Necessary Repositories

To install the latest PHP version, we need to enable additional repositories, which are not enabled by default on **RHEL 9**.


#### Install the EPEL Repository

**EPEL (Extra Packages for Enterprise Linux)** is a repository that contains additional software packages that are not provided in the default RHEL repository but are often needed for full functionality.

```bash
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
```
![](/images/Screenshot%20(578).png)

#### Install the Remi Repository

**Remi’s repository** is required to install **PHP 8.3**, as RHEL’s default repositories only provide PHP versions up to 8.2.

```bash
sudo dnf install https://rpms.remirepo.net/enterprise/remi-release-9.rpm
```

### Step 4: Install PHP 8.3 and Extensions

1. enable the module stream for **PHP 8.3**:
```bash
   sudo dnf module switch-to php:remi-8.3
 ```
2. install the module stream for **PHP 8.3** with default extension:
```bash
   sudo dnf module install php:remi-8.3
```

3. Install **PHP 8.3** and the necessary extensions for php-based application:
```bash
   sudo dnf install php php-opcache php-gd php-curl php-mysqlnd php-xml php-json php-mbstring php-intl php-soap php-zip
 ```


4. Start and enable **PHP-FPM**:
```bash
   sudo systemctl start php-fpm
   sudo systemctl enable php-fpm
```

6.  Configure SELinux (Security-Enhanced Linux)

> **SELinux** is a security module that enforces strict access control policies on your system, especially important for enterprise environments like Red Hat. It helps limit the damage that could be caused by compromised services, including the web server and PHP. By default, **SELinux** is set to **enforcing** mode on **RHEL**. This mode restricts many actions that Apache and PHP-FPM might need to function correctly.

- Check SELinux Status

Verify that SELinux is enabled and in **enforcing** mode:
```bash
sestatus
```
![](/images/Screenshot%20(579).png)


- Configure SELinux for PHP and Apache

To allow **Apache** and **PHP-FPM** to run without issues, you need to allow specific actions that would otherwise be restricted by SELinux.

1. Allow **Apache** to execute memory operations (needed by PHP’s OpCache):
   ```bash
   sudo setsebool -P httpd_execmem 1
   ```

2. Allow **Apache** to make network connections (required for external HTTP requests, for example, for connecting to APIs or downloading plugins/themes):
   ```bash
   sudo setsebool -P httpd_can_network_connect 1
   ```

3. Allow **Apache** to connect via a NFS server:

```bash
sudo setsebool -P httpd_use_nfs on
```


4. Verify the Change: After enabling the boolean, you can verify that it is set correctly:

```bash
getsebool httpd_use_nfs
getsebool httpd_execmem
getsebool httpd_can_network_connect
```
![](/images/Screenshot%20(580).png)


7. Verify PHP Installation

After installation, check that PHP 8.3 is correctly installed and running.

- Check the **PHP version**:

```bash
   php --version
```

- List the installed **PHP modules**:
   ```bash
   php -m
   ```

   Ensure all necessary extensions for WordPress (like `curl`, `gd`, `mbstring`, etc.) are listed, see recommended list from [Wordpress](https://make.wordpress.org/hosting/handbook/server-environment/).
![](/images/Screenshot%20(581).png)


8. Test PHP Functionality

Create a **PHP info page** to verify that PHP is correctly served through Apache:

- Create a test PHP file:
```bash
   sudo nano /var/www/html/info.php
```

- Add the following code:
```php
   <?php
   phpinfo();
   ?>
```
- Restart Apache to apply the changes:
```bash
   sudo systemctl restart httpd
```

- Access this file via your web browser:
   `http://your-server-ip/info.php`

   If everything is configured correctly, a page displaying detailed PHP information should appear. kindly delete the info.php once testing is done.

![PHP info page](images/Screenshot%20(582).png)


## Deploying Tooling Website

- clone the PHP-based application from github

```bash
sudo git clone https://github.com/fmanimashaun/tooling.git
```

- import database:

```bash
cd tooling
sudo mysql -h 172.31.14.66 -u webaccess -p tooling < tooling-db.sql 
```

- Add a new admin user called myuser into the tooling database:

```bash
sudo mysql -h 172.31.14.66 -u webaccess -p tooling
```

```sql
INSERT INTO `users` (`username`, `password`, `email`, `user_type`, `status`)
VALUES ('myuser', '4e2fcc3b5aa765d61d7649deb882cc32', 'user@mail.com', 'admin', '1');
```

- update the database connection info in the function.php file

```bash
sudo cp -R html/ /var/www/   # assuming you are still inside the tooling directory
sudo nano /var/www/html/function.php
```

```php
$db = mysqli_connect('172.31.3.89', 'webaccess', 'Password.1', 'tooling');
```


- Restart Apache: Once you’ve made the change, restart the Apache service to apply the new settings:

```bash
sudo systemctl restart httpd
```

- Visit the browser to view the application:
```bash
http://<public ip of any of the web servers>/index.php
```
> login using username: **myuser** and password: **password**

![](images/Screenshot%20(583).png)
![](images/Screenshot%20(584).jpg)


