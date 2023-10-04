```
adduser jsklio
usermod -a -G docker jsklio
usermod -a -G sudo jsklio
usermod -a -G sshuser jsklio
```


```
sudo nano ~/.bashrc
```
remove comment in front of $force_color

```
curl https://raw.githubusercontent.com/scopatz/nanorc/master/install.sh | sh
```
