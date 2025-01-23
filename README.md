WORK IN PROGRESS NOT COMPLETE (not code just documentation and a howto with references to source material and adapted complete instructions from various sources) 
# moosefssinglesys
The goal here is a COMPLETE howto & explainer (as well as notes of my build I'm making public for others to reference for their own setups) on how to build moosefs for a single system setup 
(not recommended or officially supported for any kind of corporate/production setup, but extremely viable for homelabs) The goal here is to setup a single system as a "clustered" setup for the 
purposes of HA and monitoring. While there are other excellent tools out there like MergerFS which can create a unionFS type setup in a JBOD, MooseFS 
provides some resiliency ala RAID without the issues RAID is known for. Additionally the additional element of SnapRAID that MergerFS would require 
for similar resiliency (which I would still recommend. In my case, I am planning a future expansion into a storage cluster, so this is the first step 
towards that setup. Currently I only have one machine with 62 HDDs attached to storage expanders and controllers. This particular setup has worked for years for me. And given the 
mix of drive sizes, I needed a solution that will scale well AS WELL as self monitor/fix MooseFS provides that as well as scaling to other machines. 

The big PRO and CON here as noted above, the huge PRO for MooseFS is that it works well, it has various monitoring tools and incredibly scalable
The big CON here is setup info is complicated/setup itself being complicated, community support is somewhat hard to find with good community documentation, I'm HOPING with this documentation I can alleviate that. 
With that I used some documentation from an existing fork called LizardFS (which instructions & scripts below will work for as much of it is sourced from
LizardFS as well and worked for me)

I wrote these instructions as I set up and fixed it up upon any errors and such. 

Skip this part if you already know the difference between a stacked file system and Union FS
Differences between MooseFS and a Union setup like MergerFS. MHDDS or UnionFS
MooseFS is stacked file system. Basically a file system over an existing filesystem (like ext4, xfs, btrfs, etc) MooseFS exists over that layer converting
into it's own format. 
more details here tho regarding forensics and data recovery if you want to deep dive https://www.sciencedirect.com/science/article/pii/S266628172300197X
The MAIN difference here is that with Moosefs one can't simply pull a drive and access all the data on another machine unless the metadata is accessible
Union FS setups one can easily do that. So this is an important distinction one has to take note of. 

For those using existing data on drives (usually not recommended to do, but a thing those of us migrating from another solution will run into), prepare a seperate folder in each drive/chunkserver
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

EASIEST WAY: Now that the install is done, use this script to setup the chunkservers OR skip to manual steps below.
What the script does, looks for the mounted drives (and gets your local IP for the config)(ideally empty but will work AND WON'T delete existing data, 
just requires a manual move of data into the mount), this does NOT start the mfsmaster, mfsmetalogger or mfscgiserv services. 
DO NOT CREATE THE MOUNT FOLDER BEFORE USING THIS SCRIPT! Do it AFTER this is all setup otherwise the folder will be added as
a drive (which we don't want). Otherwise one can ALSO create the mount elsewhere outside of /mnt. In my case I used /run/media/$urusernamehere/mfsmount 
AFTER running the script skip over all the steps near the bottom and create the systemd units (if using initd, do the corresponding service creation for that). 
This script will get you about 90 percent of the way there. After that it's fine tuning it tho the defaults work great for me. 

```
#!/bin/bash

IPADDR=$(ip route get 1.1.1.1 | awk '/1.1.1.1/ {print $7}')
PORT=0

for DISKDIR in $(ls /mnt/); do

  if mountpoint /mnt/$DISKDIR; then

      mkdir /var/lib/mfs/chunkserver-$DISKDIR;
      rm /mnt/$DISKDIR/.lock;

      chown -R mfs:mfs /mnt/$DISKDIR;
      chown -R mfs:mfs /var/lib/mfs/chunkserver-$DISKDIR;

      echo "MASTER_HOST =" $IPADDR > /etc/mfs/mfschunkserver-$DISKDIR.cfg;
      echo "SYSLOG_IDENT = chunkserver-$DISKDIR" >> /etc/mfs/mfschunkserver-$DISKDIR.cfg;
      echo "CSSERV_LISTEN_PORT = $(( 9500 + $PORT))" >> /etc/mfs/mfschunkserver-$DISKDIR.cfg;
      echo "DATA_PATH = /var/lib/mfs/chunkserver-$DISKDIR" >> /etc/mfs/mfschunkserver-$DISKDIR.cfg;
      echo "HDD_CONF_FILENAME = /etc/mfs/mfshdd-$DISKDIR.cfg" >> /etc/mfs/mfschunkserver-$DISKDIR.cfg;
      echo "HDD_LEAVE_SPACE_DEFAULT = 50GiB" >> /etc/mfs/mfschunkserver-$DISKDIR.cfg;

      echo "/mnt/$DISKDIR" > /etc/mfs/mfshdd-$DISKDIR.cfg;

      chown mfs:mfs /etc/mfs/mfschunkserver-$DISKDIR.cfg;
      chown mfs:mfs /etc/mfs/mfshdd-$DISKDIR.cfg;
      echo "Starting chunkserver $PORT"

      (( PORT++ ))

      mfschunkserver -c /etc/mfs/mfschunkserver-$DISKDIR.cfg; echo "Started $DISKDIR: $?";

  fi
done
```
Note the script gave me the BEST results in setting up the drives-as-chunkservers up quickly, the steps below I used for the rest minus the chunkserver setup
(I may remove that but leaving it as a detail for anyone who wants to go at it manually as I did initially). 

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
4. Set `CSSERV_LISTEN_PORT` to non-default, unused port (like 9522) in /etc/mfs/chunkserver2.cfg (already set to that port by default needs to be uncommented) recommended to set each
   subsequent file with an incremental port, 9523, 9524, etc
5. Run the second chunkserver with mfschunkserver -c /etc/chunkserver2.cfg
6. Repeat if you need even more chunkservers on the same machine (the goal here)
7. In the chunkserver.cfg (and numbered) files (or mfschunkserver.cfg if you opted to use the default naming) at the end of the file add the mount point
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
on staying in a single machine setup then change the * to the machine IP,
but honestly default as is can work.

critically important is making sure the default names are in the /etc/hosts file
add mfsmaster

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


Additional notes:
creating a systemd unit for each of the server portions and the chunkservers especially, 
Each chunkserver aka mount for single server will require it's own unit. 
`cd /etc/systemd/system`


 mfscgiserv.service

```
 [Unit]
 Description=MooseFS CGI server
 After=syslog.target network.target ypbind.service mfsmaster.service

 [Service]
 Type=forking
 TimeoutSec=600
 ExecStart=/usr/sbin/mfscgiserv start
 ExecStop=/usr/sbin/mfscgiserv stop

 [Install]
 WantedBy=multi-user.target
```

mfschunkserver.service

```

[Unit]
Description=MooseFS chunkserver
After=syslog.target network.target ypbind.service

[Service]
Type=forking
TimeoutSec=600
ExecStart=/usr/sbin/mfschunkserver -c /etc/mfs/chunkserver.cfg start
ExecStop=/usr/sbin/mfschunkserver -c /etc/mfs/chunkserver.cfg stop

[Install]
WantedBy=multi-user.target
```
Increment for each config/chunkserver you have locally

mfschunkserver1.service (given the script above daemonizes the service 

```
[Unit]
Description=MooseFS chunkserver
After=syslog.target network.target ypbind.service

[Service]
Type=forking
TimeoutSec=600
ExecStart=/usr/sbin/mfschunkserver -c /etc/mfs/chunkserver1.cfg start
ExecStop=/usr/sbin/mfschunkserver -c /etc/mfs/chunkserver1.cfg stop

[Install]
WantedBy=multi-user.target
```

mfsmaster.service

```
[Unit]
Description=MooseFS master node
After=syslog.target network.target ypbind.service

[Service]
Type=forking
TimeoutSec=600
ExecStart=/usr/sbin/mfsmaster start
ExecStop=/usr/sbin/mfsmaster stop

[Install]
WantedBy=multi-user.target
```

mfsmetalogger.service
```

[Unit]
Description=MooseFS metalogger
After=syslog.target network.target ypbind.service

[Service]
Type=forking
TimeoutSec=600
ExecStart=/usr/sbin/mfsmetalogger start
ExecStop=/usr/sbin/mfsmetalogger stop

[Install]
WantedBy=multi-user.target
```

mfsmount.service
```

[Unit]
Description=MooseFS mounts
After=syslog.target network.target ypbind.service mfsmaster.service

[Service]
Type=forking
TimeoutSec=600
ExecStart=/usr/bin/mfsmount /fileserver2 -H 192.168.0.240
ExecStop=/usr/bin/umount /fileserver2

[Install]
WantedBy=multi-user.target
```
After setting those unit files, run to enable (copy paste as one command)
`systemctl enable mfscgiserv && systemctl enable chunkserve* && systemctl enable mfsmaster && systemctl enable mfsmetalogger`
Then start the services. chunkserve* is that way in case you have mutiple local chunkservers as one would expect if this guide was followed
`systemctl start mfscgiserv && systemctl start chunkserve* && systemctl start mfsmaster && systemctl start mfsmetalogger`
-Deep dive into forensics of stacked file systems like MooseFS https://www.sciencedirect.com/science/article/pii/S266628172300197X
-Single server Instructions adapted from MooseFS fork LizardFS with additional information for more complete instructions for anyone coming in new (like I did)
https://lizardfs.com/tabs/running-multiple-chunkservers-on-the-same-machine
-full Setup instructions reference adapted for this document
https://github.com/moosefs/moosefs/tree/master
-setup script source https://github.com/lizardfs/lizardfs/issues/764
