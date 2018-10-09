# Raspberry Pi as a TorrentBox

Instead of running your computer day and night to download GBs of torrents, we are going to use Raspberry Pi as a always on TorrentBox. 

## Step 1: OS installation

We are going to use **Ubuntu Mate** on our Pi. You can download your Pi image from [https://ubuntu-mate.org/raspberry-pi/](https://ubuntu-mate.org/raspberry-pi/) .

After the installation we'll proceed to our next step.

## Step 2: Set Static IP

We'll give a static IP to our Pi. ie, 192.168.1.150 in my case.

First check the list of all the devices, copy the name of whatever device you are using.
`nmcli con show`
or
`ifconfig`

If you are using wifi then check if your wifi reachable to Pi.
`nmcli device wifi rescan `
`nmcli device wifi list`

**Disable NetworkManager** because it's absolutely crap when it comes to static IP.

`sudo systemctl disable NetworkManager.service`
`sudo systemctl stop NetworkManager.service`

### For WIFI:
`sudo vim /etc/network/interfaces`

make new entries similar to this:

> auto wlxc4e984141270 iface wlxc4e984141270 inet static
>         address 192.168.1.150
>         netmask 255.255.255.0
>         network 192.168.1.0
>         broadcast 192.168.1.255
>         gateway 192.168.1.1
>         dns-nameservers 8.8.8.8, 8.8.4.4
>         wpa-ssid Nitish_Home_Wifi
>         wpa-psk 25a1b8455f312e2e5d76139553fb10472fbe2b9efbcb487c957ca13efd8e5ae5d

Replace "*wlxc4e984141270*" with your own network interface name, which you got earlier using nmcli.

To generate wpa-psk hash, visit [http://jorisvr.nl/wpapsk.html](http://jorisvr.nl/wpapsk.html) and generate the hash using your SSID and password.

### For Ethernet:

> auto enxb827eba27ec7 iface enxb827eba27ec7 inet static
>         address 192.168.1.150
>         netmask 255.255.255.0
>         network 192.168.1.0
>         broadcast 192.168.1.255
>         gateway 192.168.1.1
>         dns-nameservers 8.8.8.8, 8.8.4.4

Replace "*enxb827eba27ec7*" with your own network interface name, which you got earlier using nmcli.

### Restart network:

`sudo systemctl restart networking.service`

`sudo ifdown wlxc4e984141270`
`sudo ifup wlxc4e984141270`

check if internet is working fine.
`ping 8.8.8.8`

Just to make sure that everything is perfect even after a restart.
`sudo reboot`

## Step 3: Setup pendrive/HDD

Attach your pendrive/HDD in your Pi. I'll be using a 1TB HDD.

Check **device name** of your HDD:
`sudo lsblk`

Generally pendrive is automatically mounted when inserted so we'll unmount it first.
`mount `

Check where your HDD is mounted and unmount it.
`sudo umount /media/nitish/newHDD`

We'll mount our HDD on */mnt/hdd* 

`sudo mkdir /mnt/hdd`

I had been having various permission related issues while mounting HDD so I'll use `chown` to avoid any further problems regarding permission.

`sudo chown -R nitish:nitish /mnt/`

Use your raspberry pi username instead of "*nitish*"

> Now, this is most important part of the tutorial, if you mess up this step you might not be able to boot your pi. 

We'll mount our HDD using `/etc/fstab` file. But before that let's grab our HDD's UUID.

`sudo blkid`

copy your pendrive's UUID.

`sudo vim /etc/fstab`

At the end of fstab file add the following line.

`UUID=5161561B41165	/mnt/hdd	ntfs	auto,rw,umask=0000	0 0`


> Note: Each entry is separated by tab. UUID is unique to every device
> so replace it with whatever UUID you got from blkid step.  Your
> pendrive/HDD might have difference file format so like vfat, ext4 so
> replace ntfs with suitable format.

Mount the HDD now.

`sudo mount -a`

There shouldn't be any errors.
Run `sudo mount ` to check if your HDD is mounted correctly or not.

Move to our HDD and try to create a file.

`cd /mnt/hdd`

`touch hello`

`ls`

if you can see the hello file listed then congratulations our most difficult step is over. Reset of the tutorial is just a piece of cake. (Read: copy paste.)

## Step 4: Installing torrent client

We'll be using deluge as our torrent client.

`sudo apt-get install deluged -y`
`sudo apt-get install deluged-console -y`

`start deluge`

`sudo deluged`

Since all the config files are created, now kill deluge and edit the config files.

`sudo pkill deluged`

Backup config file
`cp ~/.config/deluge/auth ~/.config/deluge/auth.old`

`vim ~/.config/deluge/auth`

Add this entry to the file

>nitish:password:10

Replace the following entry with yours.
*nitish*: your pi username
*password*: your password

Now we'll install deluge web client so that we can access our torrent client from anywhere in the network.

`sudo apt-get install python-mako`

`sudo apt-get install deluge-web`

`sudo deluge-web`

`sudo pkill deluge-web`

`vim ~/.config/deluge/web.conf`

You can edit the port number you want your deluge client to be running on. 
I'll leave it default at 8112.

## Step 5: Create deluge service

By create deluge service, you don't have to again and again login and restart deluge when power goes off.

`sudo vim /etc/systemd/system/deluge-web.service`

Make following entries.
>[Unit]
>Description=Deluge Bittorrent Client Web Interface
>Documentation=man:deluge-web
>After=network-online.target
>#Wants=deluged.service
>[Service]
>Type=simple
>User=nitish
>Group=nitish
>UMask=000
 >#This 5 second delay is necessary on some systems to ensure deluged has been fully started
>ExecStartPre=/bin/sleep 5
>ExecStart=/usr/bin/deluge-web
>Restart=on-failure
>[Install]
>WantedBy=multi-user.target


Enable the service
`sudo systemctl enable /etc/systemd/system/deluged.service`
`sudo systemctl start deluged`


Check if service is working or not.
`sudo systemctl status deluged`

Do similar steps for deluge-web service
`sudo vim /etc/systemd/system/deluge-web.service`

>Entries:
>[Unit]
>Description=Deluge Bittorrent Client Web Interface
>Documentation=man:deluge-web
>After=network-online.target
>#Wants=deluged.service
>[Service]
>Type=simple
>User=nitish
>Group=nitish
>UMask=000
>
> #This 5 second delay is necessary on some systems to ensure deluged has been fully started
>#ExecStartPre=/bin/sleep 5
>ExecStart=/usr/bin/deluge-web
>Restart=on-failure
>[Install]
>WantedBy=multi-user.target


`sudo systemctl enable /etc/systemd/system/deluge-web.service`
`sudo systemctl start deluge-web`
`sudo systemctl status deluge-web`


Reboot and from another machine check if WebUI is available or not.

[http://192.168.1.150:8112](http://192.168.1.150:8112)

## Step 5: File transfer

Accessing downloaded files in Rasp from Windows

For this we need to install samba server.

`sudo apt-get install samba`
`sudo apt-get install samba-common-bin`

Back up config files
 `sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.old`

`sudo vim /etc/samba/smb.conf`

Append this config file and add the following lines:

>[hdd]
>comment = 1TB shared HDD
>path = /mnt/hdd
>writeable = Yes
>browsable = Yes
>public = Yes
>force user = nitish


**Note: Using above settings, anyone in your network can access your Raspberry Pi download folder.**

Now go to your windows machine and create a shortcut and add the following as the target.

`\\192.168.1.150\hdd`

Now you can access your HDD and copy all the downloaded content.

## Step 6: Configure Web-UI
[http://192.168.1.150:8112](http://192.168.1.150:8112)

Access the WebUI using the password we set in *step 4* while installing deluge-web.

Go to preferences on the upper right portion. You can set directory to which you want to download your torrent to.

set your hdd as your download location `/mnt/hdd`

## Step 7: Single user mode

We don't need Ubuntu Mate GUI anymore. Since everything is accessible through SSH and WebUI, we'll disable GUI to reduce overall memory usage and system load.

`sudo systemctl enable ssh`

`sudo systemctl start ssh`

`sudo systemctl enable multi-user.target`

`sudo systemctl set-default multi-user.target`

`sudo reboot`

Now after the reboot you'll see a black screen which asks you to login, but you don't need to login anymore. Disconnect keyboard, mouse and HDMI but keep Ethernet/Wifi adapter and power intact. 

And that's it! In 7 simple steps we can setup a Always on TorrentBox.
I've tried and tested this method and it works flawlessly. 

I've always had problems maintaining ratios on private tracker, raspberry helped me a lot in maintaining ratios and saving power.

If unfortunately there happens to be a power cut, the raspberry resumes it's work when electricity is back.



