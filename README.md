# Docker Media Computer 2018

## Ubuntu

### Install Ubuntu server

* Download the 16.04 Server ISO from https://www.ubuntu.com/download
* Disconnect your media drives
* Boot into installer
* Chose: Install Ubuntu Server
* Step through the language settings pages
* Enter server name. (This is what your server will be called on the network)
* Enter user information (**Remember it!!**)
    * Do not encrypt the home directory
* Set your timezone
* Partition disks
    * Chose Guided - use entire disk (**Data disks should be disconnected**)
    * Check the output and choose yes to write changes to the disk.
* Configure the package manager.
    * Just hit enter
* Configuring taskel
    * Select the default to auto update
* Software selection - Select the following packages
    * Samba file server
    * Standard system utilities
    * OpenSSH server

### Log into server

* Log into the server to get the IP address. Type `ip a` for your servers ip address. Or check your router to see what ip it was given.

* You can now loginto your server via SSH using Bash in windows 10 `ssh username@ip`

```bash
ssh username@192.168.1.40 - example
```

* Update and install upgrades

```bash
sudo apt update
sudo apt upgrade
```

## Setting up BTRFS

* Shutdown the server and power off using the command `sudo poweroff`
* Install the data drives into the server and power up.
* Check if the disks are now visible ot the system by issuing `lsblk`. You shoudl see the disks. ex.
  * sdb
  * sdc
  * sdc
  * and so forth
* Create the mount point for the file system to live. In this tutorial I will use BigPurple as the name of the directory. Use what you wish to call it.

```bash
sudo mkdir /mnt/BigPurple
```

* Create the first BTRFS file structure

    ```bash
    sudo mkfs.btrfs -L “BigPurple” -f /dev/sdb
    ```

* Mount the drive. We will use the first available drive for the mount point. In my example it is sdb.

    ```bash
    sudo mount /dev/sdb /mnt/BigPurple
    ```

* Create the subvolume.
    * Change into the directory of the mount point.

    ```bash
    cd /mnt/BigPurple
    ```

    * Create the subvolume.

    ```bash
    sudo btrfs subvolume create BigPurple
    ```

    * Get out of the BigPurple directory by typing ``cd`` and hitting enter.

    * Unmount the drive.

    ```bash
    sudo umount /mnt/BigPurple
    ```

    * Remount the subvolum with these options.

    ```bash
    sudo mount -o compress=zlib,subvol=BigPurple,autodefrag /dev/sdb /mnt/BigPurple
    ```

* Adding other drives to the BTRFS filesystem.

    * Add all the remaing drives one by one. Do this for each drive untill they are all a part of the BigPurple filesystem

    ```bash
    sudo btrfs device add -f /dev/sdc /mnt/BigPurple
    ```

    * Check to see if all devices are listed.

    ```bash
    sudo btrfs filesystem show /mnt/BigPurple
    ```

* Convert the BTRFS to a Raid1 - Raid 1 will split your total storage by half but give you better redundancy.

    ```bash
    sudo btrfs filesystem balance start -dconvert=raid1 -mconvert=raid1 /mnt/BigPurple
    ```

* Setting up the FSTAB
    * We need to make sure the server mounts the drives on reboot.
    * Get the subvolume ID

    ```bash
    sudo btrfs subvolume list /mnt/BigPurple
    ```

    * Edit the /etc/fstab file

    ```bash
    sudo nano /etc/fstab
    ```

    * Add this line to the bottom of the fstab. Use the same mnt point we used before and make sure to use the correct subvol name. (That is all one line)

    ```
    /dev/sdb /mnt/BigPurple    btrfs  defaults,compress=zlib,subvolid=257,autodefrag 0 0
    ```

    * Save by hitting ctrl-x, say Y and enter
    * Reboot

    ```bash
    sudo reboot
    ```

    * Log back in and check that the volumes mounted correctly

    ```bash
    df -h
    sudo btrfs filesystem show /mnt/BigPurple
    ```

## Installing Docker

* Follow the instructions here:

    * https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-16-04

* Install docker-compose

    ```bash
    sudo apt install docker-compose
    ```

* Set the environment up

    ```bash
    sudo pico /etc/environment
    ```

* Paste these lines into the bottom of the file.

    ```
    PUID=1000
    PGID=1000
    ```

* Set user and permissions for the server

    ```bash
    sudo chmod 777 -R /mnt/BigPurple
    ```

## Installing the Server using Docker Compose

* Check out this repository to your /opt directory. Be sure to have the period/full stop, at the end of the git clone line.

    ```bash
    cd /opt
    git clone https://github.com/NoobTaco/MediaServer2018.git .
    ```

* Make edits to the docker-compose.yml file.
    * Change directory names to point to your directory structure. Again in this example I am using BigPurple.

## Adding SSL Proxy settings

_Optional_

If you wish to use ssl proxy domain names for your server then uncoment the following containers:
* ngnix
* nginx-gen
* letsencrypt-nginx-proxy-companion

Then add or uncomment the following blocks of environment settings for each container you wish to have outward exposure.

```
    environment:
      - VIRTUAL_HOST=something.something.com
      - VIRTUAL_NETWORK=nginx-proxy
      - VIRTUAL_PORT=2368
      - LETSENCRYPT_HOST=something.something.com
      - LETSENCRYPT_EMAIL=some@email.com
```

For example. To expose the Ghost blog with proxy to blog.something.com simply change the VIRTUAL_HOST to blog.something.com as well as the LETSENCRYPT_HOST.  The port is the container port, and not the port you expose.

After you bring the server up, letsencrypt will check to see if it has a valid SSL cert. Will grab a new one if it does not and will forward all incoming requests to https://blog.something.com  

Your MUST have port 443 and 80 open on your router as well as have your domain register set to send the subdomains to your server.

## Starting the Server

Navigate to your /opt directory and issue the following command to start downloading and starting all the servers.

```bash
sudo docker-compose -f /opt/docker-compose.yml up -d
```

If you are using the SSL proxy please be patient as it can take a while to generate the certs. 

You can check the status of your servers by issuing `docker ps` to list the running containers.