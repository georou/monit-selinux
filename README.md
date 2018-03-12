# monit-selinux
Monit 5.25.1 SELinux policy module for CentOS 7 &amp; RHEL 7 systems with systemd. This policy now follows the monit rpm package provided in the EPEL repo.

Forked from [Tresys' Ref Policy](https://github.com/TresysTechnology/refpolicy-contrib/blob/aede270ab97e863cbe2b8a1459b8c72ae5786356/monit.te) and tweaked to work with Red Hat's policy & systemd.

Since RH's policy is slight different, changes to port, kernel_read_system_state and fs needed to be made. Check the original commit in this repo for the exact changes.

If you need to use monit instead of systemd to start/stop services - you'll need to add in the interface where commented in this policy to allow that functionality + activate the boolean.


### Untested / Notes:
* The example service file in this git uses Restart=on-failure. This is to try and ensure monit comes back up if it crashes - if you `kill -9` it, it will come back up. Use systemctl stop monit
* SSL certificate functionality for local web server (monit's web server) not tested

## Installation
```sh
# Clone the repo
git clone https://github.com/georou/monit-selinux.git

# Optional - Copy relevant .if interface file to /usr/share/selinux/devel/include to expose them when building and for future modules
install -Dp -m 0664 -o root -g root monit.if /usr/share/selinux/devel/include/myapplications/monit.if

# Optional - Upload the .service file
cp -v monit.service /etc/systemd/system && systemctl daemon-reload

# Install the SELinux policy module. Compile it before hand to ensure proper compatibility (see below)
semodule -i monit.pp

# Restore all the correct context labels
restorecon -v /etc/systemd/system/monit.service
restorecon -Rv /var/lib/monit
restorecon -v /etc/monitrc
restorecon -Rv /etc/monit.d
restorecon -v /etc/rc.d/init.d/monit
restorecon -v /var/log/monit.log
restorecon -v /usr/bin/monit

# Add the port label to SELinux (OPTIONAL - Only if you want to use monit's web monitoring GUI) (the port can be different, change it in the /etc/monitrc conf file)
semanage port -a -t monit_port_t -p tcp 2812

# Start monit
systemctl enable monit.service
systemctl start monit.service

# Ensure it's working
monit status && ps -eZ | grep monit
```

## How To Compile The Module Locally(Recommended before installing)
Ensure you have the `selinux-policy-devel` package installed.
```sh
# Ensure you have the devel packages
yum install selinux-policy-devel
# Change to the directory containing the .if, .fc & .te files
cd monit-selinux
make -f /usr/share/selinux/devel/Makefile monit.pp
semodule -i monit.pp
```

## Debugging and Troubleshooting

* If you're getting permission errors, uncomment permissive in the .te file and try again. Re-check logs for any issues. Or `semanage permissive -a prometheusd_t`
* Easy way to add in allow rules is the below command, then copy or redirect into the .te module. Rebuild and re-install:
* Don't forget to actually look at what is suggested. audit2allow will most likely go for a coarse grained permission!

```sh
ausearch -m avc,user_avc,selinux_err -ts recent | audit2allow -R
```
If you get a could not open interface info [/var/lib/sepolgen/interface_info] error. 
Ensure policycoreutils-devel is installed and/or run: `sepolgen-ifgen`
