WORK IN PROGRESS NOT COMPLETE
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

For those using existing data, prepare a mountpoint


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

Some prep work to set up the system
Drive count (if you have a large number of drives this step is optional)

 `lsblk | grep disk | wc -l`

(this will tell you how many physical drives you have. so subtract 1 from tha total since the boot drive won't be a chunkserver)

Setting up chunkserver configs

1. Prepare another mfschunkserver.cfg file for the new chunkserver in a different path, let’s call it /etc/mfs/chunkserver2.cfg
 1a. Since I have 67 physical drives connected to a single machine I had to make multiple configs so I ran this to create the multiple copies of the sample file to save time on copying each one individually.
Adjust accordingly. If you have 10 drives then the number where mine is 67 in the line below would be 10 for you. 

` for i in chunkserver{1..67} ; do cp mfschunkserver.cfg.sample "$i".cfg ; done`

2. Prepare another mfshdd.cfg file for the new chunkserver in a different path, let’s call it /etc/mfs/mfshdd2.cfg
 2a. Same thing as above but with the filename being different 

` for i in mfshdd{1..67} ; do cp mfshdd.cfg.sample "$i".cfg ; done`

3. Set HDD_CONF_FILENAME in chunkserver.cfg files to the path of newly prepared mfshdd.cfg file, in this example /etc/mfs/mfshdd.cfg and so forth. For obvious
   reasons a best practice would be to use each chunkserver.cfg with the corresponding numbered mfshdd.cfg
5. Set CSSERV_LISTEN_PORT to non-default, unused port (like 9522) in /etc/mfs/chunkserver2.cfg (already set to that port by default needs to be uncommented) 
6. Run the second chunkserver with mfschunkserver -c /etc/chunkserver2.cfg
7. Repeat if you need even more chunkservers on the same machine (the goal here)






Single server Instructions adapted from MooseFS fork LizardFS with additional information for more complete instructions for anyone coming in new (like I did)
https://lizardfs.com/tabs/running-multiple-chunkservers-on-the-same-machine
