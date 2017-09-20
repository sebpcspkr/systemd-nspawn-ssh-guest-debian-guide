# Topic
Basically, just a guide for a basic systemd-nspawn use case: how to create a debian 9 systemd-nspawn container with ssh daemon  (guest) , on a debian 9 (host). Some steps. Tested on Debian 9 aka Stretch.



# Step 1 
Get the package for systemd-nspawn container on the host
```
# apt-get update
# apt-get install systemd-container
```
# Step 2
Enable systemd-networkd on the host
```
# systemctl enable --now systemd-networkd systemd-resolved
```
Share the resolv.conf (or not)
```
# ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf 
```
# Step 3
Create a basic image with debootstrap on the host
```
# cd /var/lib/machines
# debootstrap --include=ssh,locales,dbus,net-tools --components=main,contrib,non-free --arch amd64 stretch stretch.debootstrap http://ftp.de.debian.org/debian
```
# Step 4
Backup the primary image on the host (or not :) )
```
# tar --numeric-owner -jcvf stretch.debootstrap.tar.bz2 stretch.debootstrap
# mv stretch.debootstrap.tar.bz2 /var/tmp
```

# Step 5
Prepare your container on the host
```
# mv stretch.debootstrap stretch.container001
```
# Step 6
Set the root password of the container from the host
```
# systemd-nspawn --machine stretch.container001 
Spawning container stretch.container001 on /var/lib/machines/stretch.container001.
Press ^] three times within 1s to kill container.
root@stretch:~# passwd
Enter new UNIX password: 
Retype new UNIX password: 
passwd: password updated successfully
root@stretch:~# logout
Container stretch.container001 exited successfully.
```
# Interlude 1
At this moment, the hostname of your guest is "stretch"
And if you take a look, 'ip addr show' shows your usual interfaces


# Step 7
Start your container on the host
```
# machinectl start stretch.container001 
```

# Interlude 2
'ip addr show' shows another one interface, probably something like "ve-stretch.con@ifXX"

# Step 8
Enable systemd-networkd on the guest (host alrealdy done upper) and prepare your ssh first user
```
# machinectl login stretch.container001 
Connected to machine stretch.container001. Press ^] three times within 1s to exit session.

Debian GNU/Linux 9 <HOST-HOSTNAME> pts/0

xxxxx login: root
Password: 
Linux xxxx 4.9.0-3-amd64 #1 SMP Debian 4.9.30-2+deb9u3 (2017-08-06) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
root@<HOST-HOSTNAME>:~# systemctl enable --now  systemd-resolved
Created symlink /etc/systemd/system/multi-user.target.wants/systemd-resolved.service → /lib/systemd/system/systemd-resolved.service.
root@<HOST-HOSTNAME>:~# systemctl enable --now systemd-networkd
root@<HOST-HOSTNAME>:~# adduser guest
Adding user `guest' ...
Adding new group `guest' (1000) ...
Adding new user `guest' (1000) with group `guest' ...
Creating home directory `/home/guest' ...
Copying files from `/etc/skel' ...
Enter new UNIX password: 
Retype new UNIX password: 
passwd: password updated successfully
Changing the user information for guest
Enter the new value, or press ENTER for the default
	Full Name []: Guest
	Room Number []: 001
	Work Phone []: 001
	Home Phone []: 001
	Other []: 
Is the information correct? [Y/n] Y

root@<HOST-HOSTNAME>:~# logout
*Now you can Press ^] three times within 1s to exit session.
```

# Step 9
Connect from host to guest with ssh
```
# ssh guest@stretch.container001
The authenticity of host 'stretch.container001 (xxxxxxxxxxxxxxxxxxxxxx%ve-stretch.con)' can't be established.
ECDSA key fingerprint is SHA256:xxxxxxxxxxxxxxxxxxxxxxxx
Are you sure you want to continue connecting (yes/no)?
guest@stretch.container001's password: 
Linux xxxxx 4.9.0-3-amd64 #1 SMP Debian 4.9.30-2+deb9u3 (2017-08-06) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
guest@<HOST-HOSTNAME>:~$ exit
logout
Connection to stretch.container001 closed.
```



# Interlude 3
Take a look at nspawn process
```
# ps auxwww | grep nspawn
root     30302  0.0  0.1  52864  4460 ?        Ss   23:05   0:00 /usr/bin/systemd-nspawn --quiet --keep-unit --boot --link-journal=try-guest --network-veth -U --settings=override --machine=stretch.container001
```
Or at machinectl
```
#  machinectl --all
MACHINE              CLASS     SERVICE        OS     VERSION ADDRESSES
.host                host      -              debian 9       192.168.1.39...
stretch.container001 container systemd-nspawn debian 9       10.0.0.10...

2 machines listed.
```
# Furthermore
## Automatically start containers at host startup
```
# systemctl enable machines.target
# systemctl enable systemd-nspawn@stretch.container001.service
Created symlink /etc/systemd/system/machines.target.wants/systemd-nspawn@stretch.container001.service → /lib/systemd/system/systemd-nspawn@.service.
```
## Prevent conflict on hostname 
To avoid 'Hostname conflict, changing published hostname from 'xxxx to 'xxxx3'.Stop the guest then:
```
echo container001 > /var/lib/machines/stretch.container001/etc/hostname
```
Then startup again





