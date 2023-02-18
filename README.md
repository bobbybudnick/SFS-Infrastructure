![](https://github.com/bobbybudnick/SFS-Infrastructure/blob/main/NETWORK_2POINT2.jpg)

# SFS-Infrastructure
The Software Freedom Solutions Internet facing server infrastructure

Please read our network whitepaper which details the work leading up to this stage in the network rollout:  
https://github.com/bobbybudnick/SFS-Infrastructure/blob/main/SFS_NETWORK_WHITEPAPER_1.md

The time had come to put some lessons learned in the major new expansion to the test.  Manageability, performance and blocking had become major concerns.  In terms of management there was a discordant arrangement of roles leading to some nodes being overloaded at their respective jobs.  Wiring had become a moderate issue and aesthetics and power draw were still in need of improvement.  Due to some legacy systems still in use and several nodes overloaded with jobs performance had taken a hit on the network overall.  Systems such as classic single board computers have been relegated to largely single task backup networking roles.  Blocking routines across the network were in need of a cleanup and being more aggressive at their task.  A blocklist unification was necessary along with identifying the need to block more problem areas.

design rules and major features and improvements and hallmarks  
multiple switch connections for high availability/redundancy/resiliency nodes  
multiple wan connections for high availability/redundancy/resiliency nodes  
all nodes employ a screen except for iot and layer 2 switches  
multiple firewall layers for static gateway paths - core and transparent firewall  
clustered with physical separation between nodes  
monitoring devices placed throughout network  
all nodes are low power  
all nodes except iot devices are custom  
no enterprise equipment  
low cost  
minimal usage of commercial ups systems  
each computer core and the noc uses 4 nodes with 2 graphical and 2 console  
improved software and added blacklist for persistent threats  
monitor 1 hardware upgrade  
media hardware upgrade  
monitor 3 hardware upgrade  
virtualization server power usage improvements  
no more than 2 specialized roles per node  
extensive usage of bsd  
small custom ups systems drive 2 small systems and are dc to dc  
large custom ups systems drive 1 large system and use an inverter  
multi wan and server connectivity improvements

types of power - current - future relevant devices only  
usb from larger computer - monitor 1/cgnat cellular router/staging  
commercial ac connection - companion 1 and 2  
internal battery and custom ups - virtualization/workstation/media  
internal battery and commercial ups - storage  
internal battery and no ups - monitor 3  
custom ups only - backup internet/static cellular router  
commercial ups only - gaming desktop/cable router/monitor 2  
commercial ups and custom ups - core

types of power - newest proposal  
classed as follows  
1-internal battery/custom ups(inverter) - work/media  
2-custom ups only(dc to dc) - virt/backup/static/cgnat  
3-internal battery/no ups - monitor3/ras-staging/storage(*)/companion1/companion2  
4-commercial ups only - gaming desktop/cable router/monitor2/core/monitor1  
*uses a digital charger for now so is attached to ups to survive flickers

windows lack of viability  
companion 1 was going to be used for cctv  
however getting stunnel to work is going to be too much  
also companion 1 has a slow old hard drive  
also this is an opportunity to eliminate all commercial systems

ksysguard network monitoring failures  
5.24.6/5.95.0 is freebsd version and 5.20.5/5.78.0 is linux version  
after cycling the ksysguard menu with ctrl-m on freebsd the window shrinks more  
on linux the window does not shrink further  
possibly a problem with the version numbering  
thus the network monitoring applet must be larger and the bottom off screen  
may have something to do with long network adapter names

next gen blocking system  
bruce banner  
important because blacklist 2 does not handle port scans or partial connections  
this will heavily throw off those hacker search engines  
the dialog should show current status and may need to turn back on with delay  
was going to run on monitor but may be better to run on core switch  
backup trigger for deactivation will be disabling the rule in the webgui remotely  
green - no blocking/yellow - all but server 1 and 2/red - all servers
                                
recorder nightmares  
when vlc shows the error no space left on device it means usb is overwhelmed  
unbelievably changing one of the webcams from the usb 3 to usb 2 ports worked  
this is because the usb 2 and usb 3 controllers of some computers are separate  
2 usb 3 ports will not work because they are not running at 3.0 bandwidth  
a virtual usb 2 interface is shared among all usb 3 ports on same controller

dc to dc problems  
internal battery only makes sense on inverter powered systems with ac adapters  
ac adapters do not allow for internal battery discharge alongside ups battery  
charging the battery all the time would mean it discharges with ups battery  
not charging the battery all the time would mean it would self discharge  
turning the battery switch on from time to time to charge internal is too tedious  
dual battery advantage lost if power cut during transfer so should use one battery  
in extended outage or when doing system maintenance custom ups power can be lost  
would be best to have a reliable battery transfer switch that does not lose power

proxmox backup concept  
mount ext4 usb drive manually  
add mountpoint as storage folder in datacenter view  
specify backups as option for this folder in datacenter view  
enable storage in datacenter view  
now the storage can be chosen as a backup location  
use stop as the most reliable mode for backup  
there will be some delay before the backup as network scripts are run

extra notes on server redundancy  
scripts live in /var/lib/vz/snippets and are not backed up with vm  
bind script - qm set 100 -hookscript local:snippets/SFS_SERVER_REDUNDANCY_1  
script 1 is the scheduler for script 2 and persistency script  
script 2 needs to run in parallel with the worker to stop it after a time  
persistency script also runs in parallel with the worker  
persistency script could be more accurately called the try again script  
script 3 is the one for backup and listens for comms and pings

konsole crash in swrast.dri.so on monitor 1  
try opengl 2.0 - fail  
try compositor on - fail  
try opengl 3.1 - fail  
try xrender - fail  
use alternate terminal - pass  
possibly related to the hacks needed to run pi 4 on pi 3 image

core switch mixups  
net.ifnames on vim 3 is set to 0 so the old ethernet naming scheme applies  
one solution for the mixed up ethernet may be to go to the new scheme  
but it is not clear how to edit the kernel command line  
it is too dangerous to try any editing the kernel command line anyway  
however editing /etc/udev/rules.d/70-persistent-net.rules works  
need to reboot once after a reboot ideally to lower the resolution for some reason

nossh initiative  
ssh logins are a liability if a zero day exploit is found  
the following nodes have no real reason for an ssh server  
media  
companion 1  
companion 2  
ras-staging  
backup internet server  
workstation  
monitor 2  
cable gateway  
gaming desktop  
cgnat cellular gateway  
core switch needs ssh for switch administration  
static cellular gateway needs ssh for remote administration during contingency  
monitor 1 needs ssh as it is frequently accessed  
internet server needs ssh as it is the outside ssh server  
monitor 3 needs ssh for firewall administration  
storage server needs ssh for transfers from workstation

inline firewall switching explanation  
monitor 3 is always connected to switch with main ethernet on port 2  
cc independence uplink goes into manual switch first  
uplink is switched between independence switch and monitor 3 extra ethernet  
when manual switch is set to setting 1 then the uplink flows direct to the switch  
when manual switch is set to setting 2 then the uplink flows to monitor 3  
thus all traffic is forced to use monitor 3 for an uplink on setting 2  
the manual switch does limit physically separated group interconnects to 100M

proxmox required packages  
net-tools  
iftop

etherape responsiveness initial  
this program was never meant to be pressed into being an ids  
it only highlights new connections made after a certain time  
with a busy network it often does not respond to test connections on server 1  
this is because of the whitelist itself and whitelist computers making connections  
whitelist computer connections pin server on board so there are no announcements  
one fix is to reduce the timings in the etherape preferences timing section  
changing them all to 5 seconds is not bad  
however the display is very busy

etherape responsiveness conclusion  
may be best to be more specific with grep searches for 192.168.1.8  
that way no whitelist is needed  
also the timing could be set to default or higher this way  
a higher timing would make the display more readable  
will be more responsive because whitelist will not be canceling some notices

blocking overview  
BLACKLIST 1 - internet curated list - internet server  
BLACKLIST 2 - SFS list - core switch  
WHITELIST - SFS list - internet server  
BADFOLKS on core switch - access any server but 1 and 2 on yellow and all on red  
GREYFOLKS on core switch - access restricted shell  
WEBFOLKS on core switch - access internet server web server  
trigger for BADFOLKS - any access during alert levels  
trigger for GREYFOLKS - any access without being on WHITELIST  
trigger for WEBFOLKS - any access without being on WHITELIST

restricted shell security upgrade program  
add ip to whitelist on successful login on internet server  
log all attempts and compare to whitelist on internet server - no need  
add restricted shell addresses to GREYFOLKS on core switch  
do not block attempts to restricted shell with iptables however  
check GREYFOLKS against whitelist on agent action and add to blacklist if needed  
add GREYFOLKS informational output to switch assistant  
this method has the advantage of nabbing even portscans to ssh  
run even on condition green because connections to ssh are most egregious  
this modifies the existing core switch scripts and does not require new scripts

linphone quirks part deux  
be sure to set the "realm" or just called the server address below the login field  
tls did not login successfully  
tcp logged in successfuly but could not complete a call  
udp works perfectly  
missed call notifications in the program itself are dismissed with previously tab

12v x86 sbc roundup  
udoo x86 ultra  
lattepanda delta 2/3/alpha  
odroid h3/h3+  
zima board

webfolks implementation  
send port 80 SRC connections to iptables WEBFOLKS with logging only no blocking  
check against whitelist after a time and add WEBFOLKS without a match to blacklist  
disable blacklist on internet server because this is redundant now  
this has the advantage of grabbing even port scans to port 80  
similar disadvantage to existing in that it could run before captcha answered  
run even during condition green due to traffic volume and need to differentiate  

software changes  
customize amazon video to linux for media  
setup stunnel for cctv on media (test client/server)  
setup cctv script on media  
setup power control (central/listener)  
transition backup internet server to bsd (agent/redundancy)  
setup voip on monitor 1 and integrate with ai and add list exception  
setup firewall on monitor 3  
setup recorder script on monitor 3  
fix multi wan on media  
fix simple multi wan on ras-staging  
deprecate multi wan on backup internet server/virtualization server  
deprecate config on camera 1/camera 2/ras/staging  
setup config.txt on ras-staging  
setup samba on ras-staging  
recompile vim 3 4.9 kernel to add iptables recent/logging to core switch  
added bruce banner (conditions/scheduler/action/assistant) to core switch  
configure udhcpd for static leases on core switch  
setup portrait mode on vim 3  
modify captcha agent for restricted shell security upgrade program  
modify captcha for restricted shell security upgrade program  
added ssh helper for restricted shell security upgrade program

security by obscurity  
switch agent action  
captcha

sanitized  
sms.py  
worker 1  
uploader.py  
vnc server  
server redundancy 3  
cctv  
central power administration  
central power administration listener  
central power administration listener bsd

to do - completed  
break down former switch administration tablet  
temporarily decomission ras and staging servers  
decomission backup storage server  
decomission workspace camera server  
decomission printer camera server  
break down early hackberry mid  
decomission monitor 3  
reorganize cc independence  
decomission old companion 2  
add battery to new companion 2  
move new companion 2 in place and setup  
transition critical equipment to utility power  
transition media equipment to utility power  
move media ups  
move main ups  
remove commercial ups from living room  
remove commercial ups from desk 1  
remove commercial ups from desk 2  
move former gaming laptop in place of monitor 3  
remove redundant connection from backup internet server  
disable multi wan script on backup internet server  
add audio device/hub/step down fan on companion 1  
copy old ras 8gb drive to new ras-staging 256gb drive  
move ras-staging in place and install desktop software  
disable kwallet and calibrate touchscreen on ras-staging  
build core switch  
rtl8153 not supported by vim 3 4.9 kernel so switched with ax device  
break down comtemporary pc stick mid  
build companion 2  
decomission old companion 2  
move companion 2 in place and setup  
move core switch in place  
convert udoo/5" into virtualization  
convert staging server into monitor 1  
backup virtual machines  
decomission virtualization server  
move server ups  
remove inverter from server ups  
reorganize cc liberty  
replace cc liberty switch with 8 port unmanaged  
add stepdown for cgnat cellular gateway to battery  
run one dc to dc cable from inverter for media - instead use inverter  
run one dc to dc cable from inverter for workstation - instead use inverter  
run one dc to dc cable from inverter for monitor 1 - instead connect to gaming ups  
run one dc to dc cable from battery(!) for static cellular gateway  
run one dc to dc cable from battery(!) for cgnat cellular gateway  
run one dc to dc cable from battery(!) for virtualization server  
run one dc to dc cable from battery(!) for backup internet server  
run one dc to dc cable from battery(!) for cc liberty switch  
setup weather on companion 2  
copy virtual machines to virtualization server  
setup server redundancy on virtualization server  
decommission media  
move new media in place and setup  
setup email on companion 1  
setup central power administration on ras-staging  
setup central power administration on ii bravo companion  
setup central power administration on monitor 3  
setup central power administration for bsd on ii alpha companion  
setup central power administration for bsd on monitor 2  
setup central power administration for bsd on storage server

NOC Freedom - Updated

![](https://github.com/bobbybudnick/SFS-Infrastructure/blob/main/IMG_20221121_1235361.jpg)

CC Liberty - Updated

![](https://github.com/bobbybudnick/SFS-Infrastructure/blob/main/IMG_20221121_1240499.jpg)

CC Independence

![](https://github.com/bobbybudnick/SFS-Infrastructure/blob/main/IMG_20220922_1558196.jpg)

II Alpha

![](https://github.com/bobbybudnick/SFS-Infrastructure/blob/main/IMG_20220922_1556189.jpg)

II Bravo

![](https://github.com/bobbybudnick/SFS-Infrastructure/blob/main/IMG_20220922_1559411.jpg)

