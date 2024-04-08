# Home PC/Server Setup.
Random useful things I've added to my Ubuntu machine for QOL.

My setup (at time of writing):

```
OS: Ubuntu 22.04.4 LTS
KERNEL: 6.5.0-26-generic
CPU: AMD Ryzen 5 3600 6-Core
GPU: NVIDIA GeForce RTX 3060
GPU DRIVER: NVIDIA 535.161.07
RAM: 16 GB
```

## Steam / Playing games
On Ubuntu install, I selected install third party drivers and everything just worked out of the box. A really great site to check compatibility for linux is [protondb](https://www.protondb.com/). 

Steam/Valve has been doing fantastic work for supporting linux gaming, to improve UX on the steam deck.

Some notes though; install steam using apt, or with the `.deb` from the [steam download page](https://store.steampowered.com/about/), avoid the snap store.

Check out the page for the game you want to run on protondb, if needed, you can force run a game with proton by going to `Library` -> game you want to run -> right-click -> properties, then:

<img title="Steam" src="img/Screenshot from 2024-04-08 19-01-15.png">

**NOTE**: try running the game as-is _before_ you run it with proton. Some games work better natively.

### Adding an extra drive for steam games;
First check out the disks section below on how to mount a drive easily, but Steam now automagically finds these for you. Navigate to `Settings` -> `Storage` -> `Add Drive` and drives added to /mnt appear. 

<img title="Steam" src="img/Screenshot from 2024-04-08 22-49-47.png">

Then on installing a new game just choose the one you want.

## VPN, Password Management
I've been using [Proton VPN](https://protonvpn.com/) and [ProtonPass](https://proton.me/pass). These work well on mobile as well and you get both when you get the paid version of Proton VPN. They both also have ubuntu apps. The company is very active on reddit, so if you have concerns/questions I'd start there. 

## Backup, disks digression
I saw [this](https://www.youtube.com/watch?v=W30wzKVwCHo) great video on getting set up with an easy set and forget local backup solution using Pika. For me, I have a 2TB SSD connected to my PC, so just use that as the backup location.

The main features that I liked was that I could roll back a folder to a different version. I've been using linux for gaming as well lately and it helps to be able to roll back to a version of your system that works if you break anything.

I have set a daily schedule. But to do this, I've just formatted my SSD as ext4, which you can do in the disks utility, like so:

<img title="Disks util" src="img/Screenshot from 2024-04-08 17-52-12.png">

<img title="Disks util" src="img/Screenshot from 2024-04-08 17-52-25.png">

If you want backups to a spare drive to continue, its a good idea to make your drive mount on startup.

First make a folder, I prefer in `/mnt`, for the drive of interest:

```
sudo mkdir /mnt/big-chungus
# Need to update permissions
sudo chmod -R 777 big-chungus/
```

I suggest you rename the drives you want to use in the Disks utility, so they're easier to find. You can also change _all_ of your fstab settings in Disks. Just go to:

<img title="Disks util" src="img/Screenshot from 2024-04-08 22-25-09.png">

Here you can also select how you want to identify the disk,
and it will automagically update your fstab.

Update the `Mount Point` field, for me I want this 2TB disk to be accessible at `/mnt/big-chungus`. 

Rather than rebooting to check if your new fstab worked, just click the triangle or the square to mount and unmount the specific drive you've selected. <span style="color:gold"> Fear fstab and boot lockups no longer! </span>

Backing up back to backups, we can now use this big disk for our system backups. The linked YouTube video already covered this really well, so have a look there.

## Setting up a drive as a NAS

_Why?_
- Raspberry Pi SD cards are usually small. You can mount a network drive to a pi for dockering, or for saving data if you have a rpi zero or other IoT things.
- I wanted to see if I could.

I have a third entry which is a 500Gig SSD, which I want to set up as a home NAS.
I suggest first giving your PC a static IP before doing any further network dependent setup.

For my setup, I have an ASUS TUF-AX3000 router, which surprisingly has a ton of nice features, including letting me download
the .cfg file for backup and easily setting up a static IP.

For the latter, you can check your own IP in wonderful 
<span style="color:red">R</span> 
<span style="color:lime">G</span>
<span style="color:blue">B</span>
with:

```
ip -c -h address
```

And then do the needful. If you don't have admin access or for whatever other reason, see [here](https://pimylifeup.com/ubuntu-static-ip-netplan/) for other options.

From here, mount the drive you want to use, using Disks, and set the mount point to:

```
/mnt/ubunas
```

After you've created the folder and updated permissions as above:

```
sudo chmod -R 777 /mnt/ubunas/
```


*Side note; you might want a different file system if you want windows to see this drive too. Ext4 is fine for linux only setups. Choose NFS for windows inclusivity. Remember to also change your fstab entry to have ntfs in stead of ext4 if you do this.*

```
sudo apt install samba samba-common-bin
```

Now, remember to take a leap of faith and restart to make your `fstab` changes take place. If you run into the dreaded boot error, you can always just boot into root from your boot manager, then remove the last few entries in your fstab and try again. Godspeed!

Once booted with nicely mapped drives (try a quick `ls /mnt/`), we can add our desired drive to our samba config:

```
sudo vim /etc/samba/smb.conf
```

Which is just adding:

```
[ubunas]
path="/mnt/ubunas"
writeable=yes
create mask=0777
directory mask=0777
public=no
```

To the bottom. See, `example_samba.conf`

Then you can make a user with;

```
sudo adduser --no-create-home --disabled-password --disabled-login <USERNAME>
```

Where, for my raspberry pi, I've done:

```
sudo adduser --no-create-home --disabled-password --disabled-login shaunberrypi
```

Then do:

```
sudo smbpasswd -a shaunberrypi
```

We need to let this new user have access to the smb folder, with:

```
sudo chmod a+rwx /mnt/ubunas/
```

Remember to restart the samba service and check it with:

```
systemctl restart smbd

systemctl status smbd
```

And make sure the firewall doesn't murder it:
```
sudo ufw allow samba
```

Then you can ssh to the pi and check if you can connect.

```
ssh raspi@IP
```
Make a mount point on your pi with:

```
sudo mkdir /mnt/smb_share

sudo chmod -R 777 /mnt/smb_share
```

Install cifs which we will use for mounting samba:

```
# Should be installed already on raspbian
sudo apt install cifs-utils
```

Make a credentials file with the samba login for the raspberry pi:

```
vim ~/.credentials

# Which should contain:
username=target_user_name
password=target_user_password

# Update security of creds:
chmod 0600 ~/.credentials
```

Then you can:

```
vim /etc/fstab

# Add below line to the bottom of the file:
//<IP>/ubunas /mnt/smb_share cifs credentials=/home/shaun/.credentials 0 0
```

And do a `sudo mount -a` to refresh fstab.

Test it out, by making a new file on your pi with:

```
touch /mnt/smb_share/my_first_file.txt
```