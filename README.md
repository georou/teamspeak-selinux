# teamspeak-selinux

This module is designed for the server daemon VOIP program, [Teamspeak](https://www.teamspeak.com/en/). It is written to be run on CentOS 7 and RHEL 7 with systemd.

Since you need to install the daemon from src as there is no rpm, the placement of files is important in relation to SELinux. All file locations can be referenced in the .fc (File Context) file.

Important to note - there could be stray fringe permissions that I haven't been able to trigger. It should be noted that you actively monitor for AVC alerts.

There is also a man page (teamspeakd-selinux.8) included if desired. `gzip teamspeakd-selinux.8 && cp teamspeakd-selinux.8.gz /usr/share/man/man8`

### Untested:
* Using MySQL as a database instead of sqlite
* tsdns - this shouldn't be used anyway since it is legacy and SRV DNS records has replaced it. See below how-to add SRV record

## Pre-Installation of teamspeak:
```
# Download and untar the teamspeak src from their website
https://www.teamspeak.com/en/downloads.html#server

# Add the teamspeak user and directories
useradd -Mrs /sbin/nologin -c "Teamspeak VOIP daemon" teamspeak && install -d -m 0750 -o teamspeakd -g teamspeakd /opt/teamspeak

# Copy github files to required locations
cp -a /dir/to/teamspeaksource/* /opt/teamspeak
cp teamspeak.service /etc/systemd/system
cp ts3server.ini /opt/teamspeak
cp teamspeak.xml /etc/firewalld/service

# Reload required services
firewall-cmd --reload
systemctl daemon-reload

# Edit the ts3server.ini file to reflect the settings you want, before starting the daemon

# For future refernce, add TS DNS SRV and A record to your DNS how-to:
_ts3._udp.example.com. IN SRV 0 10 9987 ts1.example.com.
ts1.example.com. IN A ip.add.ress.1

```

## Policy Installation:
```sh
# Clone the repo
git clone https://github.com/georou/teamspeak-selinux.git

# Optional - Copy relevant .if interface file to /usr/share/selinux/devel/include to expose them when building and for future modules
install -Dp -m 0664 -o root -g root teamspeakd.if /usr/share/selinux/devel/include/myapplications/teamspeakd.if

# Compile the selinux module (see below)

# Install the SELinux policy module. Compile it before hand to ensure proper compatibility (see below)
semodule -i teamspeakd.pp

# Add teamspeak ports that are added by default
semanage port -a -t teamspeakd_port_t -p udp 9987
semanage port -a -t teamspeakd_port_t -p tcp 30033
semanage port -a -t teamspeakd_port_t -p tcp 10011
semanage port -a -t teamspeakd_port_t -p udp 2011-2110

# Edit the firewalld xml to include the ports you have selected
# Adding the ports that are commented are optional and I wouldn't recommend it unless you need to use those features. See the teamspeak docs for what the ports do: https://support.teamspeakusa.com/index.php?/Knowledgebase/Article/View/44/16/which-ports-does-the-teamspeak-3-server-use
firewall-cmd --add-service=teamspeak.xml

# Restore all the correct context labels
fixfiles -vF /opt/teamspeak /var/log/teamspeak /etc/systemd/system/teamspeak.service /opt/teamspeak/ts3server.sqlitedb

# Start teamspeakd
systemctl start teamspeak.service

# Ensure it's working in the proper confinement
ps -eZ | grep teamspeak
```

## How To Compile The Module Locally (Needed before installing)
Ensure you have the `selinux-policy-devel` package installed.
```sh
# Ensure you have the devel packages
yum install selinux-policy-devel setools-console
# Change to the directory containing the .if, .fc & .te files that you cloned from git
cd teamspeakd-selinux
make -f /usr/share/selinux/devel/Makefile teamspeakd.pp
semodule -i teamspeakd.pp
```

## Debugging and Troubleshooting

* If you're getting permission errors, uncomment permissive in the .te file and try again. Re-check logs for any issues. Or `semanage permissive -a murmurd_t`
* Easy way to add in allow rules is the below command, then copy or redirect into the .te module. Rebuild and re-install:
* Don't forget to actually look at what is suggested. audit2allow will most likely go for a coarse grained permission!

```sh
ausearch -m avc,user_avc,selinux_err -ts recent | audit2allow -R
```
If you get a could not open interface info [/var/lib/sepolgen/interface_info] error. 
Ensure policycoreutils-devel is installed and/or run: `sepolgen-ifgen`


## Compatibility Notes
Built on CentOS 7.4 at the time with:
```
selinux-policy-targeted-3.13.1-166.el7_4.7.noarch
selinux-policy-3.13.1-166.el7_4.7.noarch
selinux-policy-devel-3.13.1-166.el7_4.7.noarch
```
