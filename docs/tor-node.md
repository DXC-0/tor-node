## Introduction
<br/>
As a reminder, tor is a project allowing everyone to obtain a satisfactory level of anonymity and replacing the usual VPNs. The project is not linked to a private company and the relays are all operated by volunteers or third party groups. The fact that the set is semi-decentralized, forcing you to pass several rebounds, allows you to cover your location on the internet. This is useful in highly censored, dictatorial countries or in regimes where freedom of expression begins to be undermined. It also serves poor people, without money, anonymity should be a right and not a luxury, which is why many actors help the tor project to continue to exist and provide broadband for the network. Here we will see how to deploy a secure server running a rebound for the tor network.
<br/>

---

## Server Deployment

#### Chose a solid distribution

First chose a correct distribution. I love minimalist rolling distributions with arch, but most cloud providers do not offer this option.

* Archlinux
* Debian / Ubuntu
* Red Hat Bases (Fedora, Alma, Rocky ...)

#### Connect with root and create an user

The first step is to connect with the root account provided by the cloud provider.
Create a new user and add it to the wheel sudo group, if it is not present, install it.

```
useradd srv-admin
passwd srv-admin
usermod -aG wheel srv-admin
```

Well, you now have a clean user, so we'll already be able to secure a little ssh and prevent connections to the root and old cloud provider password.

* Here we have changed the default port on 5588 (this avoids most bots that scan on 22 defaut ssh)
* We forbid root connection
* We limit sessions to one
* Also, we block attempts to up to three
* We remove X11 and all horrible stuff

Go to the directory /etc/ssh and open sshd_config and modify with :

```
Protocol 2
Port 5588
PermitRootLogin no
MaxAuthTries 3
MaxSessions 1
AllowUsers "srv-admin"
DenyUsers root
AuthorizedKeysFile	.ssh/authorized_keys
KbdInteractiveAuthentication no
UsePAM yes
AllowTcpForwarding no
AllowAgentForwarding no
PermitTunnel no
LoginGraceTime 300
X11Forwarding no
ClientAliveInterval 0
PermitTTY no
PrintMotd no
Banner /etc/banner.exemple
Subsystem	sftp	/usr/lib/ssh/sftp-server
LogLevel VERBOSE
```

<br/>

---

## Firewalld configuration

Here we add firewalld and reject all connections except those on the custom ssh port (5588). We also put the port chosen for tor (here 9696)

* Install firewalld for network security
```
yay -S firewalld
sudo systemctl enable --now firewalld.service
sudo firewall-cmd --set-default-zone=block
sudo firewall-cmd --add-port=5588/tcp --zone=block --permanent
sudo firewall-cmd --add-port=9696/tcp --zone=block --permanent
```

Reload and restart the service : 

```
sudo firewall-cmd --reload
sudo systemctl restart firewalld.service
```
<br/>

---

## Hardening

Modify the /etc/passwd file and change root line with : 

```
root:x:0:0:root:/root:/sbin/nologin
```

Disable TTY access :

```
sudo rm -rf /etc/securetty
sudo touch /etc/securetty
sudo chmod 600 /etc/securetty
```

Enforce the PAM configuration : 

```
sudo nano /etc/pam.d/login
```
Edit the file with theses lines : 
```
auth    required       pam_listfile.so \
        onerr=succeed  item=user  sense=deny  file=/etc/ssh/deniedusers
```
Restric file modification : 

```
chmod 600 /etc/ssh/deniedusers
```

Edit the file ``` sudo nano /etc/ssh/deniedusers ``` and add "root"

Disable root password and groups :

```	
sudo passwd -l root
sudo usermod -L root
```

Install and enable fail2ban : 

```
yay -S fail2ban
sudo systemctl enable --now fail2ban.service
```

Enable SSH jail : 

```
sudo nano /etc/fail2ban/jail.local
```


```
[sshd]

enabled = true
port = 5588
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 1w
```

Reload and restart fail2ban

```
sudo systemctl reload fail2ban.service && sudo systemctl restart fail2ban.service
```

Install AppArmor (MAC module): 

```
yay -S apparmor
```

Edit kernel parameters : 

```

sudo nano /etc/boot/XXXXXlinux.conf

```

Add this line : 

```
lsm=landlock,lockdown,yama,integrity,apparmor,bpf
```

Reboot the server and check if apparmor is active : 

```
aa-status
```

Enforce defaults profiles : 


```
aa-enforce /etc/apparmor.d/*
```

<br/>

---

## Tor installation (Archlinux)

Install tor on the concerned server :

```
yay -S tor
```

Modify the Torrc file with your custom configuration : 

```
sudo nano /etc/tor/torrc
```

Paste the configuration :

```
User tor #user for daemon work
Log notice syslog
DataDirectory /var/lib/tor
ORPort 9696 #External port recheable for tor
ORPort 443 NoListen 
ORPort 127.0.0.1:9090 NoAdvertise
Nickname TorIsMyFriend #RelayName (visible on tor metrics)
ContactInfo DXC-0 #Optional (visible on tor metrics)
MyFamily 8764E9BE25CE405C76E1DD0F0937D9BF6EA16A0C #IMPORTANT !!! Paste all of your relays hashes
ExitPolicy reject *:* # no exits allowed
ExitRelay 0 #prohibit exit
SocksPort 0
```

Start the service : 

```
sudo systemctl enable --now tor.service
```

<br/>

---

## Tor installation (Debian)

Install apt-transport-https :

```
sudo apt install apt-transport-https
```

Create source list : 
```
sudo nano /etc/apt/sources.list.d/tor.list
```

Detect the codename : 

```
lsb_release -c
```

Add these lanes to the file : 

```
deb     [signed-by=/usr/share/keyrings/deb.torproject.org-keyring.gpg] https://deb.torproject.org/torproject.org <CODENAME> main
deb-src [signed-by=/usr/share/keyrings/deb.torproject.org-keyring.gpg] https://deb.torproject.org/torproject.org <CODENAME> main
```

Install gnupg :

```
sudo apt install gnupg
```

Add GPG keys : 
```
wget -qO- https://deb.torproject.org/torproject.org/A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89.asc | gpg --dearmor | tee /usr/share/keyrings/deb.torproject.org-keyring.gpg >/dev/null
```

Update and install tor : 

```
apt update
apt install tor deb.torproject.org-keyring
```

Modify the Torrc file with your custom configuration : 

```
sudo nano /etc/tor/torrc
```

Paste the configuration :

```
User debian-tor #user for daemon work
Log notice syslog
DataDirectory /var/lib/tor
ORPort 9696 #External port recheable for tor
ORPort 443 NoListen
ORPort 127.0.0.1:9090 NoAdvertise
Nickname TorIsMyFriend #RelayName (visible on tor metrics)
ContactInfo DXC-0 #Optional (visible on tor metrics)
MyFamily 8764E9BE25CE405C76E1DD0F0937D9BF6EA16A0C #IMPORTANT !!! Paste all of your relays hashes
ExitPolicy reject *:* # no exits allowed # no exits allowed
ExitRelay 0 #prohibit exit
SocksPort 0
```

Start the service : 

```
sudo systemctl enable --now tor@default.service
```