# DevOps_Tooling_Website_Solution

#### In this project you will implement a solution that consists of following components:
- AWS Webserver: Red Hat Enterprise Linux 8.
- Database Server: Ubuntu 24.04 + MySQL.
- Storage Server: Red Hat Enterprise Linux 8 + NFS Server.
- Programming Language: PHP.
- Code Repository: GitHub.

On the diagram below we can see a common pattern where several stateless Web Servers share a common database and also access the same files using Network File Sytem (NFS) as a shared file storage. Even though the NFS server might be located on a completely separate hardware - for Web Servers it look like a local file system from where they can serve the same files.

![image](https://github.com/user-attachments/assets/ccb21a71-6a39-4b37-bcd4-9f6305e16811)

It is important to know what storage solution is suitable for what use cases, for this - we need to answer following questions: what data will be stored, in what format, how this data will be accessed, by whom, from where, how frequently, etc. Base on this you will be able to choose the right storage system for your solution.


## Step 1 - Prepare NFS Server
1. Spin up a new EC2 instance with RHEL Linux 9 Operating System.
![image](https://github.com/user-attachments/assets/2c4ef0c0-aff2-44a9-aedf-fa3868e130fd)

![image](https://github.com/user-attachments/assets/59951865-2712-46c5-8750-6f559e1f5141)


3. Based on LVM experience from Project 6, Configure LVM on the Server.

- Based on your LVM experience from Project 6, Configure LVM on the Server. Create 3 volumes in the same AZ as your Web Server EC2, each of 20 GiB.
![image](https://github.com/user-attachments/assets/80194ff5-faff-4673-8561-7f48c7217fb9)

- Updating the Machine.

````
sudo yum update -y
````

List the disk
```
lsblk
```
![image](https://github.com/user-attachments/assets/c733fbb8-7726-4463-8a02-9d8535771f78)

Use df -h command to see all mounts and free space

```
df -h
```
![image](https://github.com/user-attachments/assets/0d55ad7a-ec24-4ae8-8b3f-8c57f1f5b373)

Use gdisk utility to create a single partition on each of the 3 disks:
```
sudo gdisk /dev/xvdbb
```
![image](https://github.com/user-attachments/assets/35b6c0f0-5992-4e23-8c72-f5f81d43bf5c)
we follow the same steps for remaining two and create partision:

Use lsblk utility to view the newly configured partition on each of the 3 disks.
![image](https://github.com/user-attachments/assets/7cd71b03-c86e-4d22-b915-cc8be4bb6473)


Install lvm2 package: creating logical volumes, which can be resized or moved without needing to unmount file systems.
```
sudo yum install lvm2 -y
```

Check for available partitions.
```
sudo lvmdiskscan 
```
![image](https://github.com/user-attachments/assets/a529d40f-96be-4667-9d53-4a12107300af)

Create Physical Volumes Use pvcreate utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM
```
sudo pvcreate /dev/xvdbb1 /dev/xvdbc1 /dev/xvdbd1
```
![image](https://github.com/user-attachments/assets/fe65966b-0a85-4d26-9d70-92c8437d1672)

Verify that your Physical volume has been created successfully
```
sudo pvs
```
![image](https://github.com/user-attachments/assets/22480d3a-daa8-45a7-b874-e52379efefa7)

Use vgcreate utility to add all 3 PVs to a volume group (VG) Name the VG webdata-vg
```
sudo vgcreate webdata-vg /dev/xvdbb1 /dev/xvdbc1 /dev/xvdbd1
```

Verify that your VG has been created successfully
```
sudo vgs
```
![image](https://github.com/user-attachments/assets/26356b1d-7512-4261-84eb-f5e96d036d7d)

**Create Logical Volumes Use lvcreate utility to create logical volumes
```
sudo lvcreate -L 14G -n lv-apps webdata-vg
sudo lvcreate -L 14G -n lv-logs webdata-vg
sudo lvcreate -L 14G -n lv-opt  webdata-vg
```

Verify that our Logical Volume has been created successfully
```
sudo lvs
```

Verify the entire setup #view complete setup - VG , PV, and LV
```
sudo vgdisplay -v
```

![image](https://github.com/user-attachments/assets/e862aa8b-c776-4e28-9037-35fe4e68b973)


2. Instead of formatting the disks as ext4 you will have to format them as xfs.
- Ensure there are 3 Logical Volumes lv-opt lv-apps, and lv-logs
```
sudo lsblk
```
![image](https://github.com/user-attachments/assets/b435eec9-f269-41cb-8b42-72f14847beaa)

Format the Logical Volumes as XFS:
```
sudo mkfs.xfs /dev/webdata-vg/lv-apps
sudo mkfs.xfs /dev/webdata-vg/lv-logs
sudo mkfs.xfs /dev/webdata-vg/lv-opt
```
![image](https://github.com/user-attachments/assets/e97a0840-7bb4-46d3-8569-c3aa3c647253)


3. Create mount points on /mnt directory for the logical volumes as follows: Mount lv-apps on /mnt/apps - To be used by webservers Mount lv-logs on /mnt/logs - To be used by webserver logs Mount lv-opt on /mnt/opt - To be used by Jenkins server in Project 8.

![image](https://github.com/user-attachments/assets/912cd0aa-ef96-400d-ba9b-1aa888c5ac08)

Mount Logical Volumes
```
sudo mount /dev/webdata-vg/lv-apps /mnt/apps
sudo mount /dev/webdata-vg/lv-logs /mnt/logs
sudo mount /dev/webdata-vg/lv-opt /mnt/opt
```
Add Mount Points to /etc/fstab
```
sudo vi /etc/fstab
```
Add the following lines:

```
/dev/webdata-vg/lv-apps /mnt/apps xfs defaults 0 0
/dev/webdata-vg/lv-logs /mnt/logs xfs defaults 0 0
/dev/webdata-vg/lv-opt /mnt/opt xfs defaults 0 0
```

Verify Mounts:
```
sudo mount -a
```

3. Install NFS server, configure it to start on reboot and make sure it is u and running.
Update the System and Install NFS Utilities:

```
sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
```

4.Export the NFS Mounts
Use subnet cidr to connect as clients. For simplicity, we will install our all three Web Servers inside the same subnet, but in production set up would probably want to separate each tier inside its own subnet for higher level of security. To check our subnet cidr - open our EC2 details in AWS web console and locate Networking tab and open a Subnet link:

Set Permissions on Mount Points.

```
sudo chown -R nobody:nobody /mnt/apps
sudo chown -R nobody:nobody /mnt/logs
sudo chown -R nobody:nobody /mnt/opt
sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt
```

Restart NFS Server

```
sudo systemctl restart nfs-server.service
```

Configure access to NFS for clients within the same subnet (example of Subnet CIDR - 172.31.32.0/20 ):

```
sudo vi /etc/exports
```

Add the following lines:
```
/mnt/apps 172.31.32.0/20(rw,sync,no_all_squash,no_root_squash)
/mnt/logs 172.31.32.0/20(rw,sync,no_all_squash,no_root_squash)
/mnt/opt 172.31.32.0/20(rw,sync,no_all_squash,no_root_squash)
```

![image](https://github.com/user-attachments/assets/40138b8c-271d-4a05-bbf6-8d8e1227356b)
save and exit from the editor by Esc + :wq!

Export the NFS Shares:

```
sudo exportfs -arv
```

5. Check which port is used by NFS and open necessary ports in Security Groups (add new Inbound Rule)

Check NFS Ports:

```
rpcinfo -p | grep nfs
```
##### Important note: In order for NFS server to be accessible from our client,we open following ports:
- TCP 111
- UDP 111
- UDP 2049

![image](https://github.com/user-attachments/assets/d140b957-7893-4063-8e45-7572d138edd6)


## Step 2 - Configure the database server

Log to aws account console and create EC2 instance of t2.micro type with Ubuntu Server launch in the default region ap-south-1. name instance MySQL server.

![image](https://github.com/user-attachments/assets/e75db174-6c68-4255-9d42-bdb7bb379ea4)

```
sudo apt update -y
```
1. Install MySQL server:

```
sudo apt install mysql-server
```

2. Check status:

```
sudo systemctl status mysql
```
3. Enable MySQL to start on boot:

```
sudo systemctl enable mysql
```
![image](https://github.com/user-attachments/assets/86547ed4-1d8a-4bdd-8464-3a558a938b84)

4. Create a database Tooling and User name it webaccess.

Login to Database:

```
sudo mysql
CREATE DATABASE tooling;
CREATE USER 'webaccess'@'%' IDENTIFIED BY 'mypass';
GRANT ALL PRIVILEGES ON tooling. * TO 'webaccess'@'%' WITH GRANT OPTION;
exit
```
![image](https://github.com/user-attachments/assets/6577ce42-48dc-45a0-96d2-f4a9901bd9ab)

## Step 3 - Prepare the Web Servers
In this step we will do the following:
- Configure NFS client (this step must be done on all three servers).
- Deploy a Tooling application to our Web Servers into a shared NFS folder.
- Configure the Web Servers to work with a single MySQL database For server One.

1. Launch a new EC2 instance with REDHAT Operating System.

![image](https://github.com/user-attachments/assets/36ca9a1a-714b-4fcf-94c1-bb9ab16b88de)


Updating machine...

```
sudo yum update -y
```

Installing NFS CLIENT:

```
sudo yum install nfs-utils nfs4-acl-tools -y
```

2. Mount /var/www/ and target the NFS server's export for apps

```
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/apps /var/www
sudo mount -t nfs -o rw,nosuid 172.31.33.253:/mnt/apps /var/www/

```

3. 
