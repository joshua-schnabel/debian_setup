# Debian Server setup for netcup root server

## Image

netcup: Debian (10) Buster

## Basic configuration

### Update Software

Login as root via Webconsole

```
apt-get update
apt-get upgrade
apt-get install curl pydf unzip git apparmor-utils
```

### Edit Hostname

`sudo hostnamectl set-hostname {domain}`

Edit /etc/hosts:

```
127.0.0.1       localhost
127.0.1.1       s4.1js.de s4

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
185.228.139.217 s4.1js.de s4
2a03:4000:23:41e::2 s4.1js.de s4
```
https://www.cyberciti.biz/faq/how-to-change-hostname-on-debian-10-linux/

### Enable ipv6

/etc/network/interfaces.d/50-cloud-init.cfg or /etc/network/interfaces
```
iface eth0 inet6 static
    address 2a03:4000:23:41e::2/64 <- your ipv6
    gateway fe80::1
```

https://www.snel.com/support/how-to-add-additional-ipv6-on-debian/

### Create User

Non guessable username + password, username should be unique per host

```
adduser username
usermod -aG sudo username
groupadd sshuser
usermod -aG sshuser username
```

Generate ssh key on client via putty gen with 8192

### SSHD Config

Example configuration for sshd. Sets the ssh-port to 2345:

```
Port 2345
Protocol 2

AllowGroups sshuser
PasswordAuthentication no
AuthenticationMethods publickey
PubkeyAuthentication yes
PermitRootLogin no
PermitEmptyPasswords no
UsePAM no
ChallengeResponseAuthentication no
RekeyLimit 64M #nach 64 MByte Traffic ein neuer symmetrischer

LoginGraceTime 30s
StrictModes yes
MaxAuthTries 2
MaxSessions 4
ClientAliveInterval 300
ClientAliveCountMax 0

AuthorizedKeysFile     .ssh/authorized_keys .ssh/authorized_keys2

X11Forwarding yes
PrintMotd yes
AcceptEnv LANG LC_*
Subsystem       sftp    /usr/lib/openssh/sftp-server
```

### Coloring console output (for putty users)

* bash:
```
sudo nano ~/.bashrc
```
remove comment in front of `$force_color`

* nano
```
sudo mkdir /usr/share/nanorc
cd /usr/share/nanorc
sudo git clone https://github.com/scopatz/nanorc.git .
```
add include `/usr/share/nanorc/*.nanorc` to `/etc/nanorc`


### Change root password

```
sudo passwd root
```


### Automatic notification of package updates with Apticron

```
sudo apt install apticron
sudo dpkg-reconfigure apticron
sudo cp /usr/lib/apticron/apticron.conf ./apticron.conf
```

### Mails

Needed for Apticron or any other local email sender, e.g. for own cron jobs.

Problem in instructions: No `"` in /etc/mail.rc


https://decatec.de/linux/linux-einfach-e-mails-versenden-mit-msmtp/

### Rootkit-Detection with rkhunter

```
sudo apt install rkhunter
```

Edit `/etc/rkhunter.conf`:

```
ALLOW_SSH_PROT_V1=0
SCRIPTWHITELIST=/usr/bin/egrep
SCRIPTWHITELIST=/usr/bin/fgrep
SCRIPTWHITELIST=/usr/bin/which
PKGMGR=DPKG
MAIL-ON-WARNING
```

### Configure iptables  

Enable persistent iptables:

```
apt-get install iptables-persistent
```

Simple start-configuration for ipv4 aswell as ipv6:

https://gist.github.com/jirutka/3742890

**Addition**: Switch ssh-Port to 2345 (or another port, as specified in the ssh config).


## Installation and configuration of docker

### Docker

[Basic installation steps](https://docs.docker.com/engine/install/debian/)
[Post installation steps](https://docs.docker.com/engine/install/linux-postinstall/)

Enable ipv6 in `/etc/docker/daemon.json`:

```
{
  "ipv6": true,
  "fixed-cidr-v6": "fd00:dead:beeb::/48",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```
[source](https://docs.docker.com/compose/install/)

### Generate https-cert with ACME.sh manually:

```
acme.sh --issue -d s4.1js.de -d xmpp.joshua-schnabel.de.de -w /media/data/docker/volumes/nginx/webroot --keylength ec-384 --cert-file /media/data/certs/cert.pem --key-file /media/data/certs/key.pem  --fullchain-file /media/data/certs/fullchain.pem --reloadcmd "/media/data/certs/certHook.sh"
```

### Use automatic nginx + letsencrypt https with docker:

* [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy)
* [letsencrypt companion](https://github.com/nginx-proxy/docker-letsencrypt-nginx-proxy-companion)


### Cool Things

openssl passwd -6 -salt dddddddd
