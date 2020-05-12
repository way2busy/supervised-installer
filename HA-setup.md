## Install Supervised Home Automation in Docker

Everthing you need can be found [on github](https://github.com/home-assistant/supervised-installer)

I had to install docker compose first with
```bash
sudo apt install docker-compose
```
get Root access so type the command: `sudo -i` then install required prerequisites
```bash
apt-get install \
  apparmor-utils \
  avahi-daemon \
  dbus \
  jq \
  network-manager \
  socat
```
(or run as root with `sudo su` then exit to return to user)

Install (using default command)
```bash
curl -sL https://raw.githubusercontent.com/home-assistant/supervised-installer/master/installer.sh | bash -s
```
will result in config files being located at: _/usr/share/hassio/homeassistant_

If you want to change the file locations use option `-- -d <your path>` like this example:
```bash
curl -sL https://raw.githubusercontent.com/home-assistant/supervised-installer/master/installer.sh | bash -s -- -d /home/boss/volumes/hassio
```
Start portainer with
```bash
docker run -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data --restart always --name=portainer portainer/portainer
```
and check you can access the web interface at [localhost:9000](http://192.168.0.199:9000)
(unless you changed it to something else on the line above to a different port!)

You can connect to homeassistant cli via portainer and use `ha core stop`, then stop the supervisor with
```bash
sudo systemctl stop hassio-supervisor.service
```
You can then stop the other support homeassistant containers using portainer.

Optional stop it restarting at next boot
```bash
sudo systemctl disable hassio-supervisor.service
```
and start with (you need to start homeassistant separately once the supervisor is running)
```bash
sudo systemctl start hassio-supervisor.service
```

## Install and setup Samba
followed bits from both but bound my existing user (boss) instead of creating new one
https://linuxize.com/post/how-to-install-and-configure-samba-on-ubuntu-18-04/
https://www.linuxbabe.com/ubuntu/install-samba-server-file-share
```bash
sudo apt update
sudo apt install samba
```
Once the installation is completed, the Samba service will start automatically. To check whether the Samba server is running, type:
```bash
sudo systemctl status smbd
```

![Screenshot][samba]

[samba]: file:///K:/#IoT/Home%20Assistant/images/samba.jpg "Screenshot"

At this point, Samba has been installed and ready to be configured.
Before making changes to the Samba configuration file, create a backup for future reference purposes:
```bash
sudo cp /etc/samba/smb.conf{,.backup}
```
The default configuration file that ships with the Samba package is configured for standalone Samba server. Open the file and make sure server role is set to standalone server
```bash
sudo nano /etc/samba/smb.conf
```
I made the following changes
changed workgroup:
```bash
workgroup = HOME
```
changed interface to eno1:
```bash
interfaces = 127.0.0.0/8 eno1
bind interfaces only = yes
```
And at the end added the folders we want to share:
```bash
[Hassio]
comment = needs username and password to access
path = /home/boss/volumes/hassio/
browseable = yes
guest ok = no
writable = yes
valid users = @samba

[IoT]
comment = needs username and password to access
path = /home/boss/volumes/IoT/
browseable = yes
guest ok = no
writable = yes
valid users = @samba
```
Next run the following command to check if there’s syntactic errors.
```bash
testparm
```
Rather than create a new user as in guide I have chosen to add my existing user.
First we need to set a separate Samba password for the new user with the following command:
```bash
sudo smbpasswd -a boss
```
Create the samba group.
```bash
sudo groupadd samba
```
And add this user to the samba group.
```bash
sudo gpasswd -a boss samba
```
Create the private share folder.
```bash
sudo mkdir -p /srv/samba/private/
```
The samba group needs to have read, write and execute permission on the shared folder. You can grant these permissions by executing the following command.
```bash
sudo setfacl -R -m "g:samba:rwx" /srv/samba/private/
```
## Install IoT stack (Mosquitto, InfluxDB, Grafana & Node-RED)
Inspiration from Andreas Spiess video #295 Raspberry Pi Server based on Docker, with VPN, Dropbox backup, Influx, Grafana, etc. using https://github.com/gcgarner/IOTstack

Download the repository with:
```bash
git clone https://github.com/gcgarner/IOTstack.git ~/IOTstack
```
then change directory
```bash
cd ~/IOTstack
```
Run the menu script to build our stack of containers - Mosquitto, InfluxDB, Grafana & Node-RED. Then select the Node-RED palettes.
```bash
./menu.sh
```
Then start the stack!
```bash
docker-compose up -d
```
To stop use `docker-compose stop` , this stops without removing containers

[Portainer on port 9000](http://192.168.0.199:9000)

[Home Assistant on port 8123](http://192.168.0.199:8123)

[Node-RED on port 1880](http://192.168.0.199:1880)

[Grafana on port 3000](http://192.168.0.199:3000)
* username admin
* password admin

Mosquitto uses port 1883

InfluxDB on ports 8086 8083 2003

## Misc
```bash
sudo apt-get install tree
tree -L 2 /home/boss
```
![Screenshot][tree]

[tree]: file:///K:/#IoT/Home%20Assistant/images/tree.jpg "Screenshot"
```
└── volumes
    ├── hass_config (un-supervised i.e. single container)
    └── hassio (supervised)
```

## Home Assistant Addons
>Let's try home assistant addons rather than IOTstack

InfluxDB - set SSL to false, enabled web interface on port 8888 a0d7b954_influxdb

Grafana - set SSL to false, enabled web interface on port 3000 a0d7b954_grafana

Mosquitto - nothing

Node-RED - set SSL to false, must set credential_secret: test

InfluxDB
Created db homeassistant
Add user homeassistant with permissions password = test
Add user grafana with permissions password = test

Grafana (can login via webpage using peter / test)
Create data source http://a0d7b954-influxdb:8086

Google drive backup addon https://github.com/sabeechen/hassio-google-drive-backup


## Disable sleep when lid shut (not on docking station)
If you're not logged in as root, you'll need to add "sudo " to invoke the editor as in:
```bash
sudo nano /etc/systemd/logind.conf
```
In the file above, I changed only this line:
```bash
HandleLidSwitch=suspend
```
to this:
```bash
HandleLidSwitch=ignore
```
After saving the file, I ran:
```bash
sudo systemctl restart systemd-logind
```
This immediately resolved the problem.

From <https://www.dell.com/community/Linux-General/Stop-laptop-going-to-sleep-when-closing-the-lid-UBUNTU-Server/td-p/6086201> 

And to turn off the LCD when the lid is shut…
```bash
sudo apt-get install vbetool
```
As for the commands, to turn off add as shortcut:
```bash
sudo vbetool dpms off
```
And to turn on:
```bash
sudo vbetool dpms on
```
From https://help.ubuntu.com/community/vbetool


