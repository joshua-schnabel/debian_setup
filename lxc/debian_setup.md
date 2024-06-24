## Setup Debian LXC Image

### Install Tools

````bash
apt update
apt upgrade
apt install curl pydf git sudo figlet resolvconf lsb-release
````

### Fix Resolver

````bash
rm /etc/resolv.conf && touch /etc/.pve-ignore.resolv.conf
sudo ln -s /run/resolvconf/resolv.conf /etc/resolv.conf
echo 2 > /proc/sys/net/ipv6/conf/eth0/accept_ra
````
Dadurch wird das Überschreiben der resolv.conf durch Proxmox verhindert und router advertisement enforced.

### SSH Konfigurieren

````bash
groupadd sshuser
````

````bash
nano /etc/ssh/sshd_config
````

````
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

Ciphers aes256-ctr,aes192-ctr,aes128-ctr
KexAlgorithms diffie-hellman-group-exchange-sha256
MACs hmac-sha2-512,hmac-sha2-256

AuthorizedKeysFile     .ssh/authorized_keys .ssh/authorized_keys2

X11Forwarding yes
PrintMotd yes
AcceptEnv LANG LC_*
Subsystem       sftp    /usr/lib/openssh/sftp-server
````

### Bash

````bash
nano /root/.bashrc
````

````bash
# ~/.bashrc: executed by bash(1) for non-login shells.
# see /usr/share/doc/bash/examples/startup-files (in the package bash-doc)
# for examples

# If not running interactively, don't do anything
case $- in
    *i*) ;;
      *) return;;
esac

# don't put duplicate lines or lines starting with space in the history.
# See bash(1) for more options
HISTCONTROL=ignoreboth

# append to the history file, don't overwrite it
shopt -s histappend

# for setting history length see HISTSIZE and HISTFILESIZE in bash(1)
HISTSIZE=1000
HISTFILESIZE=2000

# check the window size after each command and, if necessary,
# update the values of LINES and COLUMNS.
shopt -s checkwinsize

# If set, the pattern "**" used in a pathname expansion context will
# match all files and zero or more directories and subdirectories.
#shopt -s globstar

# make less more friendly for non-text input files, see lesspipe(1)
#[ -x /usr/bin/lesspipe ] && eval "$(SHELL=/bin/sh lesspipe)"

# set variable identifying the chroot you work in (used in the prompt below)
if [ -z "${debian_chroot:-}" ] && [ -r /etc/debian_chroot ]; then
    debian_chroot=$(cat /etc/debian_chroot)
fi

# set a fancy prompt (non-color, unless we know we "want" color)
case "$TERM" in
    xterm-color|*-256color) color_prompt=yes;;
esac

# uncomment for a colored prompt, if the terminal has the capability; turned
# off by default to not distract the user: the focus in a terminal window
# should be on the output of commands, not on the prompt
force_color_prompt=yes

if [ -n "$force_color_prompt" ]; then
    if [ -x /usr/bin/tput ] && tput setaf 1 >&/dev/null; then
        # We have color support; assume it's compliant with Ecma-48
        # (ISO/IEC-6429). (Lack of such support is extremely rare, and such
        # a case would tend to support setf rather than setaf.)
        color_prompt=yes
    else
        color_prompt=
    fi
fi

if [ "$color_prompt" = yes ]; then
    PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
else
    PS1='${debian_chroot:+($debian_chroot)}\u@\h:\w\$ '
fi
unset color_prompt force_color_prompt

# If this is an xterm set the title to user@host:dir
case "$TERM" in
xterm*|rxvt*)
    PS1="\[\e]0;${debian_chroot:+($debian_chroot)}\u@\h: \w\a\]$PS1"
    ;;
*)
    ;;
esac

# enable color support of ls and also add handy aliases
if [ -x /usr/bin/dircolors ]; then
    test -r ~/.dircolors && eval "$(dircolors -b ~/.dircolors)" || eval "$(dircolors -b)"
    alias ls='ls --color=auto'
    #alias dir='dir --color=auto'
    #alias vdir='vdir --color=auto'

    #alias grep='grep --color=auto'
    #alias fgrep='fgrep --color=auto'
    #alias egrep='egrep --color=auto'
fi

# colored GCC warnings and errors
#export GCC_COLORS='error=01;31:warning=01;35:note=01;36:caret=01;32:locus=01:quote=01'

# some more ls aliases
#alias ll='ls -l'
#alias la='ls -A'
#alias l='ls -CF'

# Alias definitions.
# You may want to put all your additions into a separate file like
# ~/.bash_aliases, instead of adding them here directly.
# See /usr/share/doc/bash-doc/examples in the bash-doc package.

if [ -f ~/.bash_aliases ]; then
    . ~/.bash_aliases
fi

# enable programmable completion features (you don't need to enable
# this, if it's already enabled in /etc/bash.bashrc and /etc/profile
# sources /etc/bash.bashrc).
if ! shopt -oq posix; then
  if [ -f /usr/share/bash-completion/bash_completion ]; then
    . /usr/share/bash-completion/bash_completion
  elif [ -f /etc/bash_completion ]; then
    . /etc/bash_completion
  fi
fi
````

### MOTD

````bash
nano /etc/update-motd.d/10-banner
````

````bash
#!/bin/bash

distributor_id=$(lsb_release -a 2>/dev/null | grep 'Distributor ID:' | awk -F: '{print $2}' | xargs)
codename=$(lsb_release -a 2>/dev/null | grep 'Codename:' | awk -F: '{print $2}' | xargs)
version=$(cat /etc/debian_version)

figlet -f slant $distributor_id
echo "$codename $version"
echo ""
````

````bash
nano /etc/update-motd.d/20-sysinfo
````

````bash
#!/bin/bash

# Define colors
LABEL_COLOR="\e[31m"    # Red
VALUE_COLOR="\e[39m"    # Default
NUMBERS_COLOR="\e[33m"  # Yellow
RESET_COLOR="\e[0m"     # Reset

# Define color functions for percentages
percent_color() {
    local percent=$1
    if (( $(awk 'BEGIN {print ('"$percent"' <= 70)}') )); then
        echo -e "\e[32m"  # Green
    elif (( $(awk 'BEGIN {print ('"$percent"' <= 80)}') )); then
        echo -e "\e[33m"  # Yellow
    elif (( $(awk 'BEGIN {print ('"$percent"' <= 90)}') )); then
        echo -e "\e[31m"  # Red
    else
        echo -e "\e[35m"  # Purple
    fi
}

# Function to print labeled value
print_value() {
    local label=$1
    local value=$2
    printf "${LABEL_COLOR}%-25s${RESET_COLOR}${VALUE_COLOR}%s${RESET_COLOR}\n" "$label" "$value"
}

# Function to print labeled value with numbers colored
print_value_with_numbers() {
    local label=$1
    local value=$2
    printf "${LABEL_COLOR}%-25s${RESET_COLOR}${NUMBERS_COLOR}%s${RESET_COLOR}\n" "$label" "$value"
}

# Function to print category header
print_category() {
    local category=$1
    echo -e "\n${LABEL_COLOR}== $category ==${RESET_COLOR}"
}

# Function to convert load average to percentage
load_to_percent() {
    local load=$1
    local cores=$(nproc)
    awk -v l="$load" -v c="$cores" 'BEGIN { printf "%.2f", (l/c)*100 }'
}

# Function to count online CPUs
count_online_cpus() {
    local online_list=$1
    echo $online_list | tr ',' '\n' | wc -l
}

# Gather additional welcome info
CURRENT_TIME=$(date +"%Y-%m-%d %H:%M:%S")
UPTIME_INFO=$(uptime -p)
LAST_LOGIN=$(last -n 1 -R -F -w $USER | head -n 1)

# Welcome message with username
echo -e "${LABEL_COLOR}==================================${RESET_COLOR}"
echo -e "${LABEL_COLOR}Welcome${RESET_COLOR} ${VALUE_COLOR}$(whoami)${RESET_COLOR}"
echo -e "${LABEL_COLOR}Current Time:${RESET_COLOR} ${VALUE_COLOR}$CURRENT_TIME${RESET_COLOR}"
echo -e "${LABEL_COLOR}System Uptime:${RESET_COLOR} ${VALUE_COLOR}$UPTIME_INFO${RESET_COLOR}"
echo -e "${LABEL_COLOR}Last Login:${RESET_COLOR} ${VALUE_COLOR}$LAST_LOGIN${RESET_COLOR}"
echo -e "${LABEL_COLOR}==================================${RESET_COLOR}"


# OS Info
print_category "OS Info"
print_value "OS Version:" "$(lsb_release -d | cut -f2-)"
print_value "Kernel:" "$(uname -r)"
print_value "Hostname:" "$(hostname)"

# Hardware Info
print_category "Hardware Info"
CPU_INFO=$(lscpu | grep 'Model name' | awk -F: '{print $2}' | xargs)
CPU_SOCKETS=$(lscpu | grep 'Socket(s)' | awk '{print $2}')
CPU_CORES=$(lscpu | grep 'Core(s) per socket' | awk '{print $4}')
CPU_THREADS=$(lscpu | grep 'Thread(s) per core' | awk '{print $4}')
CPU_ONLINE_LIST=$(lscpu | grep 'On-line CPU(s) list' | awk -F: '{print $2}' | xargs)
CPU_ONLINE=$(count_online_cpus "$CPU_ONLINE_LIST")
print_value "CPU Info:" "$CPU_INFO, Sockets: $CPU_SOCKETS, Cores: $CPU_CORES, Threads: $CPU_THREADS, Online Cores: $CPU_ONLINE"

CPU_LOAD=$(uptime | awk -F'load average:' '{ print $2 }' | xargs)
LOAD_1MIN=$(echo $CPU_LOAD | cut -d, -f1 | xargs)
LOAD_5MIN=$(echo $CPU_LOAD | cut -d, -f2 | xargs)
LOAD_15MIN=$(echo $CPU_LOAD | cut -d, -f3 | xargs)
LOAD_1MIN_PERCENT=$(load_to_percent $LOAD_1MIN)
LOAD_5MIN_PERCENT=$(load_to_percent $LOAD_5MIN)
LOAD_15MIN_PERCENT=$(load_to_percent $LOAD_15MIN)
printf "${LABEL_COLOR}%-25s${RESET_COLOR}1 min: $(percent_color $LOAD_1MIN_PERCENT)%s%%${RESET_COLOR}, 5 min: $(percent_color $LOAD_5MIN_PERCENT)%s%%${RESET_COLOR}, 15 min: $(percent_color $LOAD_15MIN_PERCENT)%s%%${RESET_COLOR}\n" "CPU Load:" "$LOAD_1MIN_PERCENT" "$LOAD_5MIN_PERCENT" "$LOAD_15MIN_PERCENT"

MEM_TOTAL=$(free -m | awk '/^Mem:/ {print $2}')
MEM_USED=$(free -m | awk '/^Mem:/ {print $3}')
MEM_PERCENT=$((100 * MEM_USED / MEM_TOTAL))
printf "${LABEL_COLOR}%-25s${RESET_COLOR}${NUMBERS_COLOR}%sMB${RESET_COLOR} used of ${NUMBERS_COLOR}%sMB${RESET_COLOR} ( $(percent_color $MEM_PERCENT)%d%%${RESET_COLOR} )\n" "RAM:" "$MEM_USED" "$MEM_TOTAL" "$MEM_PERCENT"

SWAP_TOTAL=$(free -m | awk '/^Swap:/ {print $2}')
SWAP_USED=$(free -m | awk '/^Swap:/ {print $3}')
if (( SWAP_TOTAL > 0 )); then
    SWAP_PERCENT=$((100 * SWAP_USED / SWAP_TOTAL))
    printf "${LABEL_COLOR}%-25s${RESET_COLOR}${NUMBERS_COLOR}%sMB${RESET_COLOR} used of ${NUMBERS_COLOR}%sMB${RESET_COLOR} ( $(percent_color $SWAP_PERCENT)%d%%${RESET_COLOR} )\n" "Swap:" "$SWAP_USED" "$SWAP_TOTAL" "$SWAP_PERCENT"
else
    print_value "Swap:" "Deactivated"
fi

print_category "Disk Usage"
df -h --output=source,size,used,pcent | grep '^/dev/' | while read line; do
    DISK_NAME=$(echo $line | awk '{print $1}')
    DISK_SIZE=$(echo $line | awk '{print $2}')
    DISK_USED=$(echo $line | awk '{print $3}')
    DISK_PERC=$(echo $line | awk '{print $4}' | tr -d '%')
    printf "  ${LABEL_COLOR}%-23s${RESET_COLOR}${NUMBERS_COLOR}%s${RESET_COLOR} used of ${NUMBERS_COLOR}%s${RESET_COLOR} ( $(percent_color $DISK_PERC)%s%%${RESET_COLOR} )\n" "$DISK_NAME:" "$DISK_USED" "$DISK_SIZE" "$DISK_PERC"
done

# Network Info
print_category "Network Info"
ip -o -4 addr show | awk '{print $2}' | sort -u | while read -r iface; do
    ips_v4=$(ip -o -4 addr show dev "$iface" | awk '{print $4}' | tr '\n' ' ')
    ips_v6=$(ip -o -6 addr show dev "$iface" | awk '{print $4}' | tr '\n' ' ')
    print_value "Interface:" "$iface"
    print_value "  IPv4:" "$ips_v4"
    print_value "  IPv6:" "$ips_v6"
done

# System Info
print_category "System Info"
PKG_INSTALLED=$(dpkg -l | wc -l)
PKG_UPDATES=$(apt list --upgradable 2>/dev/null | wc -l)
PKG_SECURITY=$(apt list --upgradable 2>/dev/null | grep -i security | wc -l)
print_value "Packages:" "Installed: $PKG_INSTALLED, Updates: $PKG_UPDATES, Security: $PKG_SECURITY"
print_value "Reboot Necessary:" "$(if [ -f /var/run/reboot-required ]; then echo Yes; else echo No; fi)"
````

````
chmod +x /etc/update-motd.d/10-banner
chmod +x /etc/update-motd.d/20-sysinfo
````
