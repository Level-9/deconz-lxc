# deconz-lxc
## Code and documentation to run the deCONZ software in a LXC-container.

### Overview ###
This is the documentation for how to run the [deCONZ-software](https://www.dresden-elektronik.com/wireless/software/deconz.htm) as provided by dresden elektronik and [Phoscon](phoscon.de), as a daemon, in an unprivilleged LXC-container. [LXC](https://linuxcontainers.org) is an alternative to the popular Docker platform. Both technologies has their pros and cons, therefore - do your own due dilligence before choosing plattform.

A lot of Linux distributions support LXD and LXC but this documentation focus on the required steps for Ubuntu Server using snap. The instructions will be similar for other distributions that support snapd.

### Prerequisites ###
A zigbee dongle, i.e. a ConBee USB-gateway is required to run the software.

Make sure that you are running a distribution that supports snaps. Most Ubuntu flavors supports snaps out of the box but if you are running a different Linux flavor then please check out the documentation for [installing](https://snapcraft.io/docs/installing-snapd) snapd.

When snapd is installed the installation of LXD is as simple as: `sudo snap install lxd`.

After that you need to configure the LXD-daemon. This is done by entering this command: `sudo lxd init`, or for a slightly simpler setup: `sudo lxd init --minimal`. It is beyond the scope of this documentation to explain all the configuration options, please see the official sites documentation for that. The defaults will most likely be fine.

When this is done enter the command `lxc list` to make sure everything is working. An empty list should be returned, since you don't have any running containers or virtual machines yet.

    +-------------+---------+------------------------+------+-----------------+-----------+
    |    NAME     |  STATE  |          IPV4          | IPV6 |      TYPE       | SNAPSHOTS |
    +-------------+---------+------------------------+------+-----------------+-----------+

### Editing the script ### 
The first part of the script sets some variables that define how the software and daemon sould be run.

- `DEVICE` (The path of your Conbee Zigbee gateway. E.g. "/dev/ttyUSB0").  
- `NAME` (The desired name of the container).
- `USERNAME` (The none-privileged user running the deCONZ-daemon).
- `PASSWD` (The password of the user. The password must be changed or the script will fail).
- `TIMEZONE` (Set it to your location. For a list of possible values enter `timedatectl list-timezones`).
- `IMAGE` (The Ubuntu image to use).
- `POOL` (The storage pool to use. The default option "default" should be fine for most users).
- `VOLUME` (The name of the persistent disk where the deCONZ configuration will be stored).
- `OPTIONS` (The options used to start the daemon).

### Running the script ###
After editing, simply enter `./deconz-lxc` and the script should create the specified container and install the deCONZ software. If you chosed to use a cloud-init aware image then the script will wait for cloud-init to complete. If you didn't then ignore the message that states that cloud-init was not found. If you enter `lxc list` again then the container sould be listed.

    +-------------+---------+------------------------+------+-----------------+-----------+
    |    NAME     |  STATE  |          IPV4          | IPV6 |      TYPE       | SNAPSHOTS |
    +-------------+---------+------------------------+------+-----------------+-----------+
    | deCONZ      | RUNNING | 172.16.11.50 (eth0)    |      | CONTAINER       | 0         |
    +-------------+---------+------------------------+------+-----------------+-----------+

By default he LXC-containers might not be accessible from outside the connected network. Please review the documentation for more information about this. Your setup and your security requirements are unknown to me and therefore I cannot guide you any further on this topic.

### Accessing the Phoscon Web-App ###
Depending on your home-network setup simply point the browser of your choice to the ip-adress of your container as listed earlier or to the name of the container if you have configured proper name resolution.

### Accessing the deCONZ Gui application ###
LXC-container behaives slightly different than Docker containers. LXC-container's are more similar to standard vitual machines in that sense that standard support for running daemons (i.e. systemd) is present and therefore it is really no problem to install VNC in the container afterwards, if you require it. If you chose to use SSH to access the container, then `ssh -X yourcontainername` could be an alternative if you have sufficient network throughput.

### So where is my data? ###
If you let all storage option remain the default values, then the configuration is stored on the default pool under the name "deCONZ-data". To list the storage pools enter: `lxc storage ls`.

    +---------+--------+--------------------------------------------+-------------+---------+
    |  NAME   | DRIVER |                   SOURCE                   | DESCRIPTION | USED BY |
    +---------+--------+--------------------------------------------+-------------+---------+
    | default | zfs    | /var/snap/lxd/common/lxd/disks/default.img |             | 2       |
    +---------+--------+--------------------------------------------+-------------+---------+

Then to list the content of the default pool enter: `lxc storage volume ls default`

    +----------------------------+------------------------------------------------------------------+-------------+--------------+---------+
    |            TYPE            |                               NAME                               | DESCRIPTION | CONTENT-TYPE | USED BY |
    +----------------------------+------------------------------------------------------------------+-------------+--------------+---------+
    | container                  | deCONZ                                                           |             | filesystem   | 1       |
    +----------------------------+------------------------------------------------------------------+-------------+--------------+---------+
    | custom                     | deCONZ-DATA                                                      |             | filesystem   | 1       |
    +----------------------------+------------------------------------------------------------------+-------------+--------------+---------+

Note that the deCONZ-DATA field "TYPE" is "custom" for your persistent data.

If you would like to access the files without having to go through the container the can be found in your hosts file system while the container is running. However it is only accessible from the correct namespace as root. Enter the correct namespace as root by enter: `sudo nsenter -t $(cat /var/snap/lxd/common/lxd.pid) -m` 

Then you can list the files.

`ls /var/snap/lxd/common/lxd/storage-pools/default/custom/default_deCONZ-DATA`

    root@yourcomputer:/var/snap/lxd/common/lxd/storage-pools/default/custom/default_deCONZ-DATA# ls
    config.ini  devices zcldb.txt  zll.db

For more info about how to access container files please visit https://blog.simos.info/how-to-view-the-files-of-your-lxd-container-from-the-host/

### Upgrading ###
Normally you could and should upgrade and refresh your installation by entering `sudo apt-get update && sudo apt-get uppgrade`. Since the deCONZ repository is added the deCONZ software updates also.

**You must of course always make a backup using the Phoscon app before making any changes** but it should also be safe to simply delete the container and rerun the deconz-lxc script. The volume deCONZ-DATA should be re-attached and all configuration intact.

### Other notes ###
The deCONZ update service is not run under your choosen user. It is still being run by root.

