# ansible_desktop
Workstation configuration

Uses a script like this, run using @reboot cronjob:

```
#!/bin/bash
#
# Workstation install bootstrap script
#
# Process:
# 1. curl inf-d-tools-admin-1/bquayle/ws_init.sh|sudo /bin/bash (this script)
# 2. ^ this does premiminary configuration work (unique hostname, etc...), then
#      installs ansible and git and does the ansible-pull
# 3. ^ this installs workstation config, which includes:
#    - cronjob to run the ansible-pull
#    - developer tools (compiler, VS Code, python3, etc...)
#    - system utilities (xfce, netstat, tmux, etc...)
#    - office tools (libreoffice, etc...)
PFILE=/home/supervisor/pfile.log

if [[ -f $PFILE ]]
then
  phase=$(tail -1 $PFILE)
  echo $phase |grep -E -q '^[0-5]'
  if [[ $? -ne 0 ]]
  then
    PHASE=0
  else
    PHASE=$phase
  fi
  echo "PHASE set to $PHASE"
else
  PHASE=0
fi

case $PHASE in
0)
  echo "Beginning O'Reilly Auto standard desktop deployment at $(date)"
  HN="L$(hostid)"
  if (( $? != 0 ))
  then
    NIC=$(ls /sys/class/net|grep -v lo|head -1)
    if [[ -f /sys/class/net/${NIC}/address ]]
    then
      HN="L$(cat /sys/class/net/${NIC}/address|sed -i 's/://g')"
    else
      echo "ERROR: Can't find a place to set hostname."
      exit 1
    fi
  fi
  hostnamectl set-hostname ${HN}.oreillyauto.com

  echo "Hostname set to ${HN}.oreillyauto.com"

  echo "Upgrading OS"
  apt clean all
  add-apt-repository multiverse
  add-apt-repository universe
  apt update
  apt -y upgrade
  echo "OS upgrade complete."
  echo "Setting multi-user target until imaging completes"
  systemctl set-default multi-user.target
  echo "1" |tee -a $PFILE
  reboot
;;
1)

  #
  # Install git and ansible
  #
  echo "Installing ansible and git"
  apt -y install software-properties-common
  apt-add-repository ppa:ansible/ansible
  apt update
  apt -y install ansible git
  echo "2" |tee -a $PFILE
  reboot
;;
2)
  nc -w 1 github.com 443 1>/dev/null
  if [[ $? -eq 0 ]]
  then
    echo "Running ansible-pull to bring in workstation configuration."
    ansible-pull -U https://github.com/billq/ansible_bootstrap.git
  else
    echo "Connection to github failed.  Run ansible-pull manually."
  fi
  if [[ $(dmidecode -s system-product-name) = VirtualBox ]]
  then
    echo "3" |tee -a $PFILE
  else
    echo "4" |tee -a $PFILE
  fi
  reboot
;;
3)
  #
  # Install VB guest image
  #
  echo "Adding VB guest extensions"
  apt -y install virtualbox-guest-dkms virtualbox-guest-x11
  echo "4" |tee -a $PFILE
  reboot
;;
4)
  echo "Removing orphans"
  apt-get -y autoremove
  apt-get -y autoclean
  echo "Setting up unattended- upgrades"
  apt-get install -y unattended-upgrades
  echo "Removing bootstrap cronjob"
  crontab -u supervisor -l > ~supervisor/bootcron
  crontab -u supervisor -r
  passwd -l supervisor
  echo "Setting graphical target for user"
  systemctl set-default graphical.target

  echo "Desktop initialization complete"
  echo "5" |tee -a $PFILE
  systemctl poweroff
;;
5)
  exit
;;
esac
```

Stick the above into a file that you can pull down using curl from a webserver, then execute that file.  Read the header comment
