# monit-selinux
Monit SELinux policy module for CentOS 7 &amp; RHEL 7 systems with systemd

Forked from [Tresys' Ref Policy](https://github.com/TresysTechnology/refpolicy-contrib/blob/aede270ab97e863cbe2b8a1459b8c72ae5786356/monit.te) and tweaked to work with Red Hat's policy & systemd.

Since RH's policy is slight different, changes to port, kernel_read_system_state and fs needed to be made. Check the original commit in this repo for the exact changes.

If you need to use monit instead of systemd to start/stop services - you'll need to add in the interface where commented in this policy to allow that functionality + activate the boolean.


### Important to note:
This policy uses the rpm package compiled from monit's source file which has an srpm file in it. Thus some of the file locations will be different or addition steps need to be made:

1. You need to manually add logrotate if you decide to use /var/log over syslog - I've attached a sample to use with systemd
2. monit doesn't automatically create the /var/lib/monit folder for it's state and id files
3. The service file uses Restart = on-failure. This is to try and ensure monit comes back up if it crashes - if you kill -9 it, it will come back up. Use systemctl
4. Default bin path for monit is /usr/local/bin - changed to /usr/bin here
5. SSL certificate functionality not tested


## Installation
```sh
# Clone the repo
git clone https://github.com/georou/monit-selinux.git

# Create the directory for it's files
mkdir -v /var/lib/monit

# Upload the .service file
cp -v monit.service /etc/systemd/system
systemctl daemon-reload

# Install the SELinux policy module. Compile it before hand to ensure proper compatibility (see below)
semodule -i monit.pp

# Restore all the correct context labels
restorecon -v /etc/systemd/system/monit.service
restorecon -Rv /var/lib/monit
restorecon -v /etc/monitrc
restorecon -v /etc/rc.d/init.d/monit
restorecon -v /var/log/monit.log
restorecon -v /usr/bin/monit

# Add the port to SELinux (the port can be different, change it in the /etc/monitrc conf file)
semanage port -a -t monit_port_t -p tcp 2812

# Start monit
systemctl enable monit.service
systemctl start monit.service

# Ensure it's working
monit status
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

* If you're getting permission errors, uncomment permissive in the .te file and try again. Re-check logs for any issues.

* Ease way to add in allow rules is the below command, then copy or redirect into the .te module. Rebuild and re-install:
* Don't forget to actually look at what is suggested. audit2allow will most likely go for a coarse grained permission!

```sh
ausearch -m avc -ts recent | audit2allow -R
# If you get a could not open interface info [/var/lib/sepolgen/interface_info] error, install:
yum install policycoreutils-devel
```


## Compatibility Notes
Built on CentOS 7.3 at the time with:
```
policycoreutils-python-2.5-11.el7_3.x86_64
selinux-policy-3.13.1-102.el7_3.16.noarch
selinux-policy-targeted-3.13.1-102.el7_3.16.noarch
policycoreutils-2.5-11.el7_3.x86_64
selinux-policy-devel-3.13.1-102.el7_3.16.noarch
```