WORK IN PROGRESS NOT COMPLETE (not code just documentation and a howto with references to source material)
# moosefssinglesys
This is a howto & explainer (as well as notes of my build I'm making public for others to reference for their own setups) on how to build moosefs for a single system setup 
(not recommended for any kind of corporate/production setup, but viable for homelabs) The goal here is to setup a single system as a "clustered" setup for the 
purposes of HA and monitoring. While there are other excellent tools out there like MergerFS which can create a unionFS type setup in a JBOD, MooseFS 
provides some resiliency ala RAID without the issues RAID can be known for. Additionally the additional element of SnapRAID that MergerFS would require 
for similar resiliency. In my case, I am planning a future expansion intoa storage cluster, so this is the first step towards that setup. Currently
I only have one machine with 67 HDDs attached to storage expanders and controllers. This particular setup has worked for years for me. And given the 
mix of drive sizes, I needed a solution that will scale well AS WELL as self monitor/fix MooseFS provides that as well as scaling to other machines .

Skip this part if you already know the difference between a stacked file system and Union FS
Differences between MooseFS and a Union setup like MergerFS. MHDDS or UnionFS
MooseFS is stacked file system. Basically a file system over an existing filesystem (like ext4, xfs, btrfs, etc) MooseFS exists over that layer converting
into it's own format. 
more details here tho regarding forensics and data recovery if you want to deep dive https://www.sciencedirect.com/science/article/pii/S266628172300197X
The MAIN difference here is that with Moosefs one can't simply pull a drive and access all the data on another machine unless the metadata is accessible
Union FS setups one can easily do that. So this is an important distinction one has to take note of. 

For those using existing data on drives (usually not recommended to do), prepare a seperate folder in each drive/chunkserver
mfs or moosefs or whatever you choose, that will be where your data goes. Alternately one can also can migrate the data
off the first chunkserver intended drive, then configure it as a chunkserver and move the data back to the mfs


I setup on Arch via AUR so it was just a matter of installing only moosefs which includes everything needed, 
for other distros but for the purposes of simplicity I'll cover Arch based distros and Debian based ones 
(one can replace the package manager commands with the ones required as the package names will be similar 
across the board for the others minus arch which installs based off the one command)
For Arch/Manjaro,
from AUR if you're not installing from there I recommend installing yay first

 `pacman -S yay`

Then 

 `yay -S moosefs`

For Ubuntu and derivatives

 `sudo su
 apt update
 apt upgrade -y`

Download and add repository key:

 `wget -O - https://ppa.moosefs.com/moosefs.key | apt-key add -`

Add an appropriate repository entry in /etc/apt/sources.list.d/moosefs.list:nano

` echo "deb http://ppa.moosefs.com/moosefs-3/apt/$(awk -F= '$1=="ID" { print $2 ;}' /etc/os-release)/$(lsb_release -sc) $(lsb_release -sc) main" > /etc/apt/sources.list.d/moosefs.list`

And run:

 `apt update`

Install MooseFS Master

And now you can install the Master with CGI using the following command:

 `apt install moosefs-master moosefs-cgi moosefs-cgiserv moosefs-cli`

Now that the install is done

Some prep work to set up the configs for configging
Drive count (if you have a large number of drives this step is optional)

 `lsblk | grep disk | wc -l`

(this will tell you how many physical drives you have. so subtract 1 from tha total since the boot/root drive won't be a chunkserver)

Setting up chunkserver configs

1. Prepare another mfschunkserver.cfg file for the new chunkserver in a different path, let’s call it /etc/mfs/chunkserver2.cfg
 1a. Since I have 61 physical drives connected to a single machine I had to make multiple configs so I ran this to create the multiple copies of the sample file to save time on copying each one individually.
Adjust accordingly. If you have 10 drives then the number where mine is 61 in the line below would be 10 for you. 

`cd /etc/mfs`
` for i in chunkserver{1..61} ; do cp mfschunkserver.cfg.sample "$i".cfg ; done`

2. Prepare another mfshdd.cfg file for the new chunkserver in a different path, let’s call it /etc/mfs/mfshdd2.cfg
 2a. Same thing as above but with this cfg file
`cd /etc/mfs`
` for i in mfshdd{1..61} ; do cp mfshdd.cfg.sample "$i".cfg ; done`

3. Set HDD_CONF_FILENAME in chunkserver.cfg files to the path of newly prepared mfshdd.cfg file, in this example /etc/mfs/mfshdd.cfg, /etc/mfshdd1.cfg and so forth. For obvious
   reasons a best practice would be to use each chunkserver.cfg with the corresponding numbered mfshdd.cfg
   `HDD_CONF_FILENAME = /etc/mfs/mfshdd1.cfg` would go in the chunkserver1.cfg and so forth.  
5. Set CSSERV_LISTEN_PORT to non-default, unused port (like 9522) in /etc/mfs/chunkserver2.cfg (already set to that port by default needs to be uncommented) 
6. Run the second chunkserver with mfschunkserver -c /etc/chunkserver2.cfg
7. Repeat if you need even more chunkservers on the same machine (the goal here)
8. In the chunkserver.cfg (and numbered) files (or mfschunkserver.cfg if you opted to use the default naming) at the end of the file add the mount point
for each drive mount you're using. (also noted below in the next section)

Minimal setup using steps above to get started 

Just three steps to have MooseFS up and running:
1. Install at least one Master Server

    Prepare default config (as root):

`cd /etc/mfs
cp mfsmaster.cfg.sample mfsmaster.cfg
cp mfsexports.cfg.sample mfsexports.cfg`

The mfsexports.cfg is set to allow any client in the network
to access the server, this may need to be adjusted based on 
your own security profile in your home network. If you plan 
on staying in a single machine setup then change the * to 
127.0.0.1 or localhost, but honestly default as is can work.

  Prepare the metadata file (as root):

`cd /var/lib/mfs
cp metadata.mfs.empty metadata.mfs
chown mfs:mfs metadata.mfs
rm metadata.mfs.empty`

Run Master Server (as root): mfsmaster start
Make this machine visible under mfsmaster name, e.g. by adding a DNS entry (recommended) or by adding it in /etc/hosts on all servers that run any of MooseFS components.

2. Install at least two Chunkservers

    Prepare default config (as root):

`cd /etc/mfs
cp mfschunkserver.cfg.sample mfschunkserver.cfg
cp mfshdd.cfg.sample mfshdd.cfg`

At the end of mfshdd.cfg file make one or more entries containing paths to HDDs / partitions designated for storing chunks, e.g.:

`/mnt/chunkserver1 (in my case I used my drive's UUIDs as the mount which for me works but you may have a different way of going about it)
/mnt/chunkserver2
/mnt/chunkserver3`

It is recommended to use XFS as an underlying filesystem for disks designated to store chunks. More than two Chunkservers are strongly recommended.
In my case I have existing data on the drives and am using BTRFS, this for me adds a little extra resiliency and recovery at the FS level, since
I have had this setup for some years now, I'm doing manual data migration into MooseFS by migrating the data from the various drives lower FS (BTRFS) into
stacked MooseFS setup

Change the ownership and permissions to mfs:mfs to above mentioned locations (if you have multiple chunkservers as denoted above this is the command
with wildcard to get them all in one command):

`chown mfs:mfs /mnt/chunks*
chmod 770 /mnt/chunks*`

Start the Chunkserver: 
`mfschunkserver start`

Repeat steps above for second (third, ...) Chunkserver.
3. Client side: mount MooseFS filesystem

Mount MooseFS (as root):

`mkdir /mnt/mfs
mount -t moosefs mfsmaster: /mnt/mfs`

or: `mfsmount -H mfsmaster /mnt/mfs` if the above method is not supported by your system

You can also add an /etc/fstab entry to mount MooseFS during the system boot:

`mfsmaster:    /mnt/mfs    moosefs    defaults,mfsdelayedinit    0 0`

Once that's done if you did the fstab approach type 
`mount -a`
and if no errors, you're ready to roll. 


-Deep dive into forensics of stacked file systems like MooseFS https://www.sciencedirect.com/science/article/pii/S266628172300197X
-Single server Instructions adapted from MooseFS fork LizardFS with additional information for more complete instructions for anyone coming in new (like I did)
https://lizardfs.com/tabs/running-multiple-chunkservers-on-the-same-machine
-full Setup instructions reference adapted for this document
https://github.com/moosefs/moosefs/tree/master
