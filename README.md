# DEVOPS TOOLING WEBSITE SOLUTION

##  NFS Server

* Network File System (NFS) is a distributed file system protocol originally developed by Sun Microsystems (Sun) in 1984, allowing a user on a client computer to access files over a computer network much like local storage is accessed. NFS, like many other protocols, builds on the Open Network Computing Remote Procedure Call (ONC RPC) system. NFS is an open standard defined in a Request for Comments (RFC), allowing anyone to implement the protocol.

* NFS is a very useful tool but, historically, it has suffered from many limitations, most of which have been addressed with version 4 of the protocol. The downside is that the latest version of NFS is harder to configure when you want to make use of basic security features such as authentication and encryption since it relies on Kerberos for those parts. And without those, the NFS protocol must be restricted to a trusted local network since data goes over the network unencrypted (a sniffer can intercept it) and access rights are granted based on the client's IP address (which can be spoofed).

*As a member of a DevOps team, We will implement a tooling website solution which makes access to DevOps tools within the corporate infrastructure easily accessible*


![tool1](https://user-images.githubusercontent.com/85270361/153243141-d22931b1-7f87-4102-b149-3fca416c7235.PNG)



### Step 1 — Prepare the NFS Server


* We need to create a single partition on each of the 3 disks we added to our NFS Server using the command


```
sudo gdisk /dev/xvdf
```

![partition](https://user-images.githubusercontent.com/117458922/223452852-b0cc6998-7060-40fc-b9f8-64577ab075cc.png)


then `/dev/xvdg` and `/dev/xvdh` respectively

* We need to mark the disks as LVM physical volumes with the command `sudo pvcreate /dev/xvdf1 /dev/xvdg1 /dev/xvdh1` and then confirm with `sudo pvs`

![physical volumes](https://user-images.githubusercontent.com/117458922/223454862-fbe80015-bc39-47f3-9028-6696397b9fe9.png)

* We will add the PVs to a volume group. We will Name the VG webdata-vg with the and command `sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1` then confirm with `sudo vgs`

![Volume group](https://user-images.githubusercontent.com/117458922/223455720-06fb4193-2402-4097-aa0b-b3b2556d5715.png)

![vgs](https://user-images.githubusercontent.com/117458922/223455764-a7daf001-8ff6-4f3a-a8b1-dc73c406c02d.png)

* We will create 3 logical volumes lv-opt , lv-apps and lv-logs with the command;

```
sudo lvcreate -n lv-apps -L 9G webdata-vg
sudo lvcreate -n lv-logs -L 9G webdata-vg
sudo lvcreate -n lv-opt -L 9G webdata-vg
```

![Logical volumes](https://user-images.githubusercontent.com/117458922/223457266-33f5507a-377c-4fe7-b719-659a5d6e16d1.png)

* We need to format the logical volume with XFS filesystem using the command 

```
sudo mkfs -t xfs /dev/webdata-vg/lv-apps
sudo mkfs -t xfs /dev/webdata-vg/lv-logs
sudo mkfs -t xfs /dev/webdata-vg/lv-opt
```

![format](https://user-images.githubusercontent.com/117458922/223457990-b9aa5cff-1447-4a9f-8e41-85c6cb435dc2.png)

* We need to create mount points on /mnt directory for the logical volumes: Mount `lv-apps on /mnt/apps` `lv-opt on /mnt/opt` `lv-logs on /mnt/logs`

```
sudo mkdir /mnt/apps
sudo mkdir /mnt/logs
sudo mkdir /mnt/opt
```

![directories](https://user-images.githubusercontent.com/117458922/223460370-2fad979f-bbe1-4fbe-ad16-34ca25bd6b33.png)

* Now, we will Mount them accordingly using the command below;

```
sudo mount /dev/webdata-vg/lv-apps /mnt/apps
sudo mount /dev/webdata-vg/lv-logs /mnt/logs
sudo mount /dev/webdata-vg/lv-opt /mnt/opt
```

![mount](https://user-images.githubusercontent.com/117458922/223461648-6986dcf2-4f43-460b-b658-f3e935f5cce6.png)

* We will now Install NFS server, configure it to start on reboot and make sure it is up and running by following the commands below;

```
sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
```

![nfs status](https://user-images.githubusercontent.com/117458922/223462693-0ce2d891-4276-4e77-b8e2-8b19ed8c8794.png)

* We will make sure we set up permission that will allow our Web servers to `read, write and execute` files on NFS:

```
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt

sudo systemctl restart nfs-server.service
```

![permissions](https://user-images.githubusercontent.com/117458922/223464722-75c871b5-f037-45f4-9bb1-d5f01073f878.png)


* We will Export the mounts for webservers’ subnet cidr to connect as clients. For simplicity, we will instaly the Web Servers inside the same subnet, but in production set up we would probably want to separate each tier inside its own subnet for higher level of security.

*To check our subnet cidr – open the EC2 details in AWS web console and locate ‘Networking’ tab and open a Subnet link:*

* We will now configure access to NFS for clients within the same subnet (example of Subnet CIDR – 172.31.43.0/20 ):

```
sudo vi /etc/exports

/mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/opt <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)

Esc + :wq!

sudo exportfs -arv
```

![exports](https://user-images.githubusercontent.com/117458922/223464893-5854b5e6-4adb-4b6f-a978-b3b906dca0f4.png)

* Check which port is used by NFS and open it using Security Groups on the EC2 Instance by adding new Inbound Rule

```
rpcinfo -p | grep nfs
```

![ports check](https://user-images.githubusercontent.com/117458922/223466399-2f421582-e908-4222-9a6f-d35545cc12aa.png)

*Note: In order for NFS server to be accessible from your client, you must also open following ports: TCP 111, UDP 111, UDP 2049*

![sgs](https://user-images.githubusercontent.com/117458922/223465998-4da3cdb3-e9cb-44ac-a7d0-8c3dee325a89.png)


### Step 2 — Configure the database server

* We are going to Install MySQL with the command**

```
sudo apt-get update
```

```
sudo apt -y install mysql-server
```

* We are going to configure MySQL with the command**


```
sudo mysql_secure_installation
```


- **We need to create a database and name it tooling**


```
mysql> CREATE database tooling;
```

* We need to create a database user 'webaccess' and grant permission to the user on tooling database to do anything only from the webservers subnet cidr

```
mysql> CREATE user 'webaccess'@'172.31.48.0/20' IDENTIFIED WITH mysql_native_password by 'password';
```
```
mysql> GRANT ALL PRIVILEGES ON tooling.* TO 'webaccess'@'172.31.48.0/20';
```

```
mysql> FLUSH PRIVILEGES;
```

![db](https://user-images.githubusercontent.com/117458922/223470454-6f62a618-dfd2-4cf9-a6f3-11b35bca9740.png)



### Step 3 — Prepare the Web Servers

Launch a new EC2 instance with RHEL Operating System

The Webserver will use the NFS as its backend storage. Hence, it is important to configure the web servers as NFS client. We will follow the below steps to 
prepare each servers.

* Install NFS client

```
sudo yum install nfs-utils nfs4-acl-tools -y
```
* Mount /var/www/ and target the NFS server’s export for apps

```
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/apps /var/www
```

![mount app on web](https://user-images.githubusercontent.com/117458922/223473371-f38a24a7-e923-471f-b0cd-bd8ddabe266d.png)

* Verify that NFS was mounted successfully by running df -h. Make sure that the changes will persist on Web Server after reboot:

![df -h](https://user-images.githubusercontent.com/117458922/223475404-5acfd83b-0f07-49e7-b7d0-311a51cea302.png)

```
sudo vi /etc/fstab
```

add following line

```
<NFS-Server-Private-IP-Address>:/mnt/apps /var/www nfs defaults 0 0
```

![persist](https://user-images.githubusercontent.com/117458922/223473755-5825ff5d-4c80-46f6-a72f-aa1ed9dfdd21.png)

* Install Remi’s repository, Apache and PHP

```
sudo yum install httpd -y

sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm

sudo dnf module reset php

sudo dnf module enable php:remi-7.4

sudo dnf install php php-opcache php-gd php-curl php-mysqlnd

sudo systemctl start php-fpm

sudo systemctl enable php-fpm

sudo setsebool -P httpd_execmem 1
```

* We will verify that Apache files and directories are available on the Web Server in `/var/www` and also on the NFS server in `/mnt/apps`. If we see the same files – it means NFS is mounted correctly. We will try to create a new file touch test.txt from one server and check if the same file is accessible from other Web Servers.

![touch](https://user-images.githubusercontent.com/117458922/223475549-c1b23449-89f4-43ba-bdb6-5ee52ae91fda.png)

![touches](https://user-images.githubusercontent.com/117458922/223475670-6e0a91a6-1c44-4adf-8f92-2867e029bd1c.png)

* We will locate the log folder for Apache on the Web Server and mount it to NFS server’s export for logs. Repeat step №4 to make sure the mount point will persist after reboot.

```
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/logs /var/www

sudo vi /etc/fstab
```

add following line

```
<NFS-Server-Private-IP-Address>:/mnt/logs /var/log/httpd nfs defaults 0 0
```

![mount logs](https://user-images.githubusercontent.com/117458922/223651148-77dc5192-3ba4-4067-99ae-e702f2c34bda.png)


* Fork the tooling source code from [Darey.io Github Account](https://github.com/darey-io/tooling) to your Github account. (Learn how and why to fork a repo [here](https://www.youtube.com/watch?v=f5grYMXbAV0))

Follow these commands;

```
sudo yum install git -y

git init

git clone https://github.com/darey-io/tooling.git
```

![git repo fork](https://user-images.githubusercontent.com/117458922/223650223-d8d33ae1-38c6-49c6-af65-8c9506032531.png)


* We need to install mysql client so as to be able to access our website can access our database server remotely.**

```
sudo yum install mysql
```

* We will deploy the tooling website’s code to the Webserver and ensure that the `html` folder from the repository is deployed to `/var/www/html`
 
 change directory to tooling
```
cd tooling
sudo cp -R html/. /var/www/html
```

![deploy html](https://user-images.githubusercontent.com/117458922/223651448-eeb1621e-df00-4ae9-b521-d8749cc1ec39.png)

*Note 1: Do not forget to open TCP port 80 on the Web Server.

Note 2: If you encounter 403 Error – check permissions to your /var/www/html folder and also disable SELinux sudo setenforce 0 To make this change permanent – open following config file sudo vi /etc/sysconfig/selinux and set SELINUX=disabledthen restrt httpd.*

* Update the website’s configuration to connect to the database in `/var/www/html/functions.php` file. Apply `tooling-db.sql` script to your database using this command 

```
mysql -h -u -p < tooling-db.sql
```

![connect](https://user-images.githubusercontent.com/117458922/223653633-c4f80dd9-35d0-4d07-a0d5-45209ef25a1e.png)

* Open the website in your browser http:///index.php and login with `admin` `admin`

Note: If the page you see is the defualt welcome page, use the command below to move the default welcome page

```
cd tooling

ls /etc/httpd/conf.d/welcome.conf

sudo mv /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/welcome.backup

sudo systemctl restart httpd
```

* Reload the page, we should see the following pages if everything is done correctly

![login page](https://user-images.githubusercontent.com/117458922/223655666-2c9c4745-c207-47d4-9326-f86f456c9e12.png)

![logged in](https://user-images.githubusercontent.com/117458922/223655759-1dff0563-47fc-4f84-a5d9-8bba9242d166.png)

*Congratulations!*
