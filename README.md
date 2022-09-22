# SFS-Infrastructure
The Software Freedom Solutions Internet facing server infrastructure

Tenets of Resilient Ethernet Micro Networks
-----

**Laying the Fabric**  
This is the basic groundwork for the network.  It was clear early on that
mutiple Internet gateways were required for high availability. Personal
experience has also shown that during digital and real world disasters it is
advantageous to have more than one Internet connection.

1. Approaches to multi WAN  
The first piece of the network fabric must be the ability to switch Internet providers.  
Handling at main router is complicated/stresses pfsense/single router point of failure.  
So handling the changeover with routing and scripting is a must.  
Some terms need to be clarified first.  
Roundrobin is having multiple A records for the same hostname.  
DNS failover is changing a single A record to different IP.  
Dynamic DNS or a client DNS update API is needed.  
Name.com is great but they do not support dynamic DNS/failover/client API.  
However they do support basic round robin as most DNS providers do.  
To give this project the most options it was decided early on to move to Cloudflare.

2. Cellular bandwidth and response solution  
This is the outline of the initial idea for multi WAN.  
Favor cable connection over cellular to keep user experience fast and cellular data low.  
Cellular router http port closed by default which causes clients or failover to skip.  
Script on cellular router monitors site at cable static IP.  
Script on cellular router opens local http port for a time if down.  
Could check every 30 minutes and open port for 30 minutes.  
This would possibly allow for seamless internal and external failover.  
This instead has to be done at the server level because servers can have 1 gateway only.  
There is unpredictability with multiple gateways which the multi wan scripts address.  
It is unclear how to use routing alone to get a reliable result in all situations.  
Could still be done at router but would be redundant.

3. Management network  
Workstation/media/proxmox server are the most important candidates for dual ethernet.  
In this case the management network is dedicated.  
On a non dedicated management network overlaid on main network vlans could be used.  
Interference avoidance and ddos avoidance is an advantage with dedicated.  
Management computers having redundant switch connections is an advantage with dedicated.

4. Prevent routing through the cellular network  
This is for outbound.  
It should not route through cellular by default but instead use cable default gateway.  
Fundamentally the problem is one of needing multiple gateways.  
A weight needs to be assigned to the gateways favoring the cable connection.  
To prevent inbound routing see cellular bandwidth and response solution.

5. Adequacy of round robin  
Apparently modern browsers work with this pretty well.  
There is misinformation but it should be fine compared to load balancing solutions.  
Load balancing solutions are a single point of failure themselves if improperly setup.

6. Failover capability in theory  
Internal - duel weighted gateways and always on cellular.  
Internal example - cable internet fails and computer selects alternate gateway.  
External - round robin/failover dns + cellular bandwidth and response solution.  
External example - cable fails and Cloudflare uses alternate dynamic IP entry for dns.

7. Dual gateway setup concerns  
Configuring all hosts on network for dual gateway would be tedious.  
The management devices should be the only ones configured.  
Management/vip devices are workstation/media/virtualization server.  
A dual link is only needed for cable to both switches.  
Still allows for main switch to fail and management devices still have cable connection.  
Remember it is easier but far less flexible when using dual wan with a single router.

8. Implementing a core switch  
A managed core could create new VLANs for isolation of main and management.  
This approach has redundancy and security.  
A VLAN between cable and management only.  
A VLAN between cable and main only.  
If main switch fails then VIP retain cable on second interface.  
If management switch fails then VIP retain cable and cellular on first interface.  
The specifics involve several tradeoffs here between redundancy and security.

9. DD WRT VLAN implementation  
iptables -nvL lists all rules.  
For DD WRT an important thing to know is a port can not be on more than one VLAN.  
This means some bridging has to be used.

10. Making an open source ethernet hub/switch  
By now it should start to be clear that bridging is the basic of switching.  
Clever use of ethernet adapters and bridging would allow for making a switch.  
All bridged devices are like they are on the same network.  
Perhaps there could be a control to make it more like a hub or more like a switch.

11. Core switch troubleshooting  
If the core switch can ping devices in question itself it is a firewall problem.  
If the core switch can not ping devices in question itself it is a route problem.

12. Power outages effect on WAN  
Often times neighborhood ups systems for cable will be poorly managed.  
Municipal power is needed to run things like switchboxes and repeaters.  
Their battery backup may be nonexistent or only last a few minutes.  
This leads to situations where power being out can lead to internet failures.  
Do not plan or rely on wired providers for high availability in a disaster.

13. Pfsense cable router config  
Static route of 192.168.1.0/24 out of 192.168.4.1 on LAN  
Static route of 192.168.5.0/24 out of 192.168.4.1 on LAN  
Outbound NAT for 192.168.1.0  
LAN access anything rules for 192.168.4.0/24  
LAN access anything rules for 192.168.5.0/24  
LAN access anything rules for 192.168.1.0/24  
Gateway for WAN set to default at cable IP gateway  
Gateway for LAN at 192.168.4.1

14. DD WRT core switch config for wrt54gl  
Block vlan 2 and vlan 0 connection - iptables -I FORWARD -i vlan2 -o br0 -j DROP  
VLAN 0 - main network/subnet/switch/LAN and administration  
VLAN 2 - management network/subnet/switch/LAN  
VLAN 3 - cable  
LAN address set to 192.168.1.1 in LAN settings  
VLAN 0 - port 1 / port 4 - assigned to LAN  
VLAN 1 - WAN - assigned to none  
VLAN 2 - port 2 - assigned to none  
VLAN 3 - port 3 - assigned to none  
gateway set to 192.168.4.2 in LAN settings  
VLAN 0 - default bridging  
VLAN 2 - unbridged at 192.168.5.1 - multicast forwarding on - (new) NAT off  
VLAN 3 - unbridged at 192.168.4.1 - multicast forwarding on - (new) NAT off  
Operating mode set to router in advanced routing - is this even needed???  
Set DNS servers in DHCP static DNS fields.  
Uncheck use dnsmasq for dnsmasq.  
Do not set a DNS server for DD WRT itself or it will serve that one also.  
Needed if default route is getting lost - route add default gw 192.168.4.2.  
Enable turning off radio in services.

15. Dynamic DNS  
Should be integrated into cellular bandwidth and response solution script.  
Update a dynamic DNS record for alternate gateway every 30 minutes.  
A scripted approach or manually updating the dynamic DNS website would work.  
The manual method may be ok for devices that do not change IP often.  
Unfortunately this will not work with CGNAT and would require tunneling.  
But for a basic backup hardline this should do.

16. Outbound routing for VIP  
Assumes a main switch and a management switch.  
Primary interface - default gw (whichever ip this is) - low metric - cable  
Secondary interface - default gw (whichever ip this is) - medium metric - cable  
Primary interface - default gw (whichever ip this is) - high metric - cellular

17. Carrier grade NAT nightmare  
Cellular providers are really giving paying consumers half of a connection.  
A business connection required is required to fully use a cellular connection.  
Due to nat being used at the carrier level with no user forwarding controls.  
ATT/Tmobile/Verizon all require credit/business checks for business connection.  
Fixed by ordering an att static ip sim card from protectli.  
Cellular business data is expensive.  
Will need a way cut off after exceeding cellular data usage limit.

18. Avoiding switching loops  
Also called a "layer 2 storm" or "frame storm" or something like that.  
When making a connection and all the switches light up that is the problem.  
Switches cannot connect in a way that they have more than one link to each other.  
A typical switch network should look like a tree with branches coming from main.  
Typically link aggregation technologies in advanced switches prevent this.  
Possible to have high reliability without link aggregation but takes thought.  
Multiple network adapters are the only good solution without link aggregation.  
These allow for a total switch failure with still existent connectivity.

19. Terminology and interchangeability of concepts  
Network/subnet/switch/LAN  
These are all referring to the same thing in this case.

20. Network status  
The backup internet server will host apache with network status page.  
This page can be updated during emergency events.  
This will display updated content even when other dynamic content is offline.

21. Content control  
Serve dynamic/static content/network status over cable connection.  
Serve static content and network status over cellular connection.  
Cut connection to reencoder to stop dynamic content.  
Cut connection to web server at 500 MB usage.

22. Explaining metrics  
Metrics still work to direct traffic when there is more than one good gateway.  
Metrics do not work as expected when a bad gateway of low metric (high priority) exists.  
Metrics are only used because it is a good way for route del default to knock a gateway off table.

23. Layer 3 switching  
This is the point that the device would traditionally be configured further for layer 3 segmentation.  
This is the true job of a core switch.  
The blocking of certain IP addresses and networks would be done here.  
But the existing layer 2 segmentation seems way more than enough already.

24. FreeBSD initial working multi WAN implementation  
Primary interface - static route to first network with /24 - first gateway to cable - on FIB 0 default  
Secondary interface - static route to second network with /24 - second gateway to cable - on FIB 1  
Secondary interface - static route to second network with /24 - gateway to cellular - on FIB 2  
Ping gateways with setfib and the corresponding FIB number.  
Ping hosts without specifying FIB number.  
In otherwords only use FIB for Internet connections.  
User applications have to be started with a FIB to use the right gateway.  
Servers can start and listen on a certain FIB.  
net.add_addr_allfibs=0 - not sure of the ramifications or whether it would work with setting 1.  
Sometimes the ethernet should be unplugged and the IP set then plugged back.

25. FreeBSD proposed broken multi WAN implementation  
Do not use FIB system.  
netstat -nr -f inet - show FreeBSD routing table.  
Test current gateway with test route.  
If test works then delete test and move on to next section.  
If test fails then remove gateway.  
Signal that the next section should be ran.  
Sleep for a time.  
Only run next sections if signaled by earlier section.  
Try to add test routes 0 to 2 inside connected network with a timeout.  
If a test works then delete test and add gateway.  
This is not going to work without a redesign due to the wrong assumptions made above.  
Would need to be written like the reimplementation above but without metrics.  
Probably not going to work without FIB regardless.

26. Linux proposed broken multi WAN implementation  
Link local works because a route is not added unless there is a connection.  
Because of this connection check routes can be added and removed dynamically.  
Test routes 0 to 2 inside connected network with a timeout.  
If a test works then move to next test.  
If a test fails then remove gateway and message that gateway is removed.  
Then try to add all gateways with a timeout.  
Only good gateways will be readded.  
Sleep for a time.

27. Linux multi WAN reimplementation  
route does not block from adding bad hosts as expected.  
route also does not check that gateways are good before adding them.  
However a combination of ip show and ping worked as expected.

28. DD WRT core switch config for ac1750 r6400v2-3  
VLAN 0 seems bugged on this device so we use VLAN 1 instead and shift others up by 1.  
VLAN 1 - main network/subnet/switch/LAN and administration  
VLAN 2 - management network/subnet/switch/LAN  
VLAN 4 - cable  
LAN address set to 192.168.1.1 in LAN settings  
VLAN 1 - port 1 / port 4 - assigned to LAN  
VLAN 2 - WAN - assigned to none  
VLAN 3 - port 2 - assigned to none  
VLAN 4 - port 3 - assigned to none  
gateway set to 192.168.4.2 in LAN settings  
VLAN 1 - default bridging  
VLAN 3 - unbridged at 192.168.5.1 - multicast forwarding on - (new) NAT off  
VLAN 4 - unbridged at 192.168.4.1 - multicast forwarding on - (new) NAT off  
Operating mode set to router in advanced routing - is this even needed???  
Set DNS servers in DHCP static DNS fields  
Uncheck use dnsmasq for dnsmasq.  
Do not set a dns server for DD WRT itself or it will serve that one also.  
If default route is getting lost - route add default gw 192.168.4.2 - not needed now???  
Shifted up one like before so VLAN 2 is now VLAN 3.  
Block vlan 2 and vlan 0 connection - iptables -I FORWARD -i vlan3 -o br0 -j DROP  
Stayed on old initial firmware so not to push luck.  
Turning off radio enable in services

29. Copy from DD WRT  
Enable USB/enable USB storage/enable automatic drive mount.  
Enable sshd and use scp like scp root@192.168.1.1:/mnt/sda1/TEST /home/jason/TEST  
Also ftp and samba are available from the nas page but maybe ssh will use less ram.  
Sadly rsync does not work by default.

**Sewing the Servers**  
This is merging the network together in a coherent way.  There was more of a
focus on getting power usage and battery life under control.  A small
reorganization was needed with the addition of extra gateways resulting in the
network2point0 initiative.

1. Cellular configuration  
Both cellular gateways should be on the 192.168.1.0 network and have separate addresses.  
192.168.5.1 will remain as the alternate cable gateway.  
Small amount of redundancy lost because if main switch down all cellular down to NOC.  
Redundancy was regained with emergency routing capability of monitor 1 Bluetooth PAN.  
However cellular will still be up in both computer cores because of their switches.  
Redundancy gained by datacenters having their own Internet connections if needed.

2. VIP gateways - all other devices still only use cable gateway 1  
Three Linux multi WAN scripts.  
One FreeBSD multi WAN script.  
Cable gateway 1 is favored first for VIP.  
Cable gateway 2 is favored second for VIP.  
Cellular gateway with static IP is favored third or fourth for VIP.  
Cellular gateway with dynamic IP is favored third or fourth for VIP.  
Servers will favor the static cellular first while consumption devices favor CGNAT.

3. VIP devices - these are the only devices with high battery life and dual ethernet  
Workstation has 2 adapters with one on each network.  
Media has 2 adapters with one on each network.  
Virtualization server has 2 adapters with one on each network.  
Backup internet server has 2 adapters with one on each network.

4. The concept of sole computers among each group with high battery life  
Only the backup internet server needs high battery life.  
This would match the idea that the main internet server has high battery life.  
Each cellular router also needs high battery life.  
Cable router does not need high battery life because it usually fails when power is out.

5. Central power administration  
The idea being that most devices could be shutdown when away from the network.  
Would work similar to apcupsd but have more features.  
Could listen with netcat.  
Wake on LAN could be used for most traditional computers.  
Scripting for shutdowns could be used on all computers.  
Ethernet controlled power switches could be used for basic single board computers.  
The network power switches seem to add more complexity than they are worth.  
Kdialog windows with buttons with a separate window showing status changes.

6. Concept of physical disaster isolation  
This is what the network 2point0 initiative is partially based on.  
Forward independent SSH port to a device in CC Independence.  
Independence will be an "independent CC" with a dedicated inbound Internet connection.  
With Liberty in closer proximity to the NOC it uses the NOC for inbound Internet.  
Devices at CC liberty could message when they are reduced to outbound only CGNAT.

7. Power reduction initiative system 1  
Was 5 amps and dropped to 4 amps by unplugging MID charger for 25 hours.  
Turning off custom laptop monitor with button on back would save further power.  
Could potentially drop to 1 amp for 100 hours.

8. Power reduction initiative system 2  
Turn receiver off to go from 58w to 18w and 5 amps to 2 amps for 50 hours.  
Setting laptop monitor to power off would save further power.  
Could potentially drop to 1 amp for 100 hours.

9. Power reduction initiative system 3  
Can not change due to server draw but currently 17 hours.  
Replacing server with newer more efficient system would be save further power.  
Could potentially drop to 1 amp for 100 hours.

10. Power reduction initiative system 4  
.55a at 13.8v nothing connected except switch and stepdown with nothing connected.  
.65a at 13.8v switch and stepdown with backup internet server.  
.95a at 13.8v switch and stepdowns with backup internet server and router for 100 hours.

11. Switch power usage list  
name - voltage - watts used  
Netgear 8 port unmanaged 4 devices connected - 1.8w  
Netgear 8 port smart managed pro gs108t 7 devices connected - 4w  
Netgear 5 port smart managed plus gs105e 5 devices connected - 2w  
Netgear 16 port smart managed plus - fairly low  
Dell powerconnect 2708 7 devices connected - 9w

12. Main UPS power survey list  
Main switch  
Management switch  
Workstation  
HD Homerun  
Antenna amplifier

13. Media UPS power survey list  
Media laptop  
Receiver  
Fire stick

14. Server UPS survey list  
Virtualization server processing unit only  
Subsidiary switch

15. CC Liberty device survey list  
2 GUI  
2 console  
2 headless

16. CC Independence device survey list  
2 GUI  
2 console  
2 headless

17. Master networking list  
Dell switch - 192.168.2.1 default IP - admin user name - no password  
Netgear switch - 192.168.0.239 default IP - no user name - password password  
ZTE mf861 - 192.168.1.1 default IP - no user name - no password  
VSVABEFV device - 192.168.1.1 default IP - admin user name - no password

18. Network2point0 practical steps list  
NOC Freedom - Odroid c0 makeshift tablet becomes monitor 1  
CC Liberty - install switch  
CC Liberty - move cold spare cellular router and android phone from NOC freedom  
CC Liberty - Pi 4 of old backup internet server becomes headless staging server  
CC Independence - install switch  
CC Independence - Pi 3 device with 5" screen becomes backup internet server  
CC Independence - move kitchen laptop to monitor position  
CC Independence - Lattepanda of old monitor 1 becomes backup storage server  
II Alpha - setup Toshiba laptop companion  
II Bravo - shutdown staging server on Lenovo companion laptop

19. MID router unsuitability list  
Complex design wasted as router  
Fairly unstable USB  
Small screen for GUI interactivity in the data center  
Makes both data centers have 2 console 2 GUI 2 headless  
May need as spare MID  
May need for demonstration purposes  
Battery difficult to charge from larger battery

20. Redundancy vs responsivesness in multi WAN configurations  
Multi WAN redundancy is only needed for VIP devices.  
Multi WAN responsiveness is needed for servers on both static IP addresses.  
Redundancy reqires an elaborate script to detect network failures and take action.  
Responsiveness hopefully only requires the 2 default static gateways set up.  
In testing the behavior with 2 gateways was poor and or unpredictable.  
This is similar to testing from the initial multi WAN scripts.  
So a compromise script was made to be not as complicated as the multi WAN scripts.  
This script checks for the primary gateway having an internet connection.  
From that information it decides whether to use cellular or cable.

21. Cellular static IP routing  
Bridging was a red herring and unnecessary and was causing duplicate packets.  
Anything going to the cellular device will go to the DMZ address instead.  
Any items that need to go from the DMZ address will need to be sent using NAT.  
Need to have an IP for the ethernet interface which will be the gateway for other hosts.  
Need to have the correct ip of the DMZ address set on the WAN interface.  
Need to have the default gateway of 192.168.1.1 set out of the WAN interface.  
Need to have ipv4 forwarding turned on.  
Need to have masquerading set for WAN interface.  
Need to have destination or outbound NAT aka DNAT for the forwarded IP addresses.

22. Outside network troubleshooting  
For the case when the administrator is away from the network.  
Try all services at domain name and rely on round robin to work.  
Try direct to cable IP SSL.  
Try direct to cellular IP SSH.  
Wait on administrative backdoor request to dynamic DNS address/admin email/admin SMS.

23. Texting from the command line  
Use textbelt for free or paid SMS or use voip.ms SMS API.  
Set IP permission to 0.0.0.0 in voip.ms API settings.  
Set password in voip.ms API settings.  
Turn on API in voip.ms API settings.  
Install python pip on freebsd by installing py39-pip-22.1.2.  
Install requests library on freebsd with pip install --user requests.  
Use custom python script that interfaces with API.  
Call program with python3.9.  
Edit python script with did number that will be originating the text from voip.ms.  
Edit python script with destination number.  
Edit python script with SIP user name and API password at bottom.  
Do not edit anything further.

24. Multi WAN survey list  
Workstation needs complex script  
Media needs complex script  
Internet server VM needs complex script  
Reencoder VM does not need script  
RAS has simple multi WAN  
Cable router SSL does not need script  
Static cellular router SSH does not need script  
CCTV does not need script  
Backup internet server needs complex script  
Virtualization server needs complex script

25. Administrative backdoor  
Script could text/email when one CGNAT gateway left.  
Script could make outbound connection to some server.  
Script could make connections to a dynamic DNS address.  
Dynamic address connections are still useful but limited due to cellular CGNAT.  
Meaning a MID or phone could not make a meaningful connection for troubleshooting.

26. Power systems variety  
There are various systems set up now that need a battery backup.  
These are not attached to any larger UPS and lack an internal battery.  
DZS 5v stepdown was not enough for Lattepanda/7" screen/USB SSD.  
1 - Medium lead acid/pololu stepupdown/auto or marine trickle charger  
2 - 2 parallel 26650 lion/adafruit powerboost 1000c for power supply and charging  
3 - Abenic 12v lion blue/dzs 5v stepdown/12v brick charger

27. Good old screen  
Screen runs like a new virtual terminal.  
A good alternative to nohup and disown.  
Exit with exit and detach with ctrl-a-d and resume with screen -r.

28. Internet server forwarding situation  
Backup replaces main dynamically.  
The problem here is that you can only forward one port to one IP.  
Subdomains may be a solution.  
Failing that the redundant switch is going to have to just to be for outbound.  
So cellular and cable router will just forward to 192.168.1.8.  
In case of main switch failure due to failure/power supply/hack the following occurs.  
Main internet server will not SMS because it still has management gateway.  
Backup internet server will not see main and take over via subsidiary switch.  
Core switch ping indicates core switch or main switch could be down.

29. Etherape view optimization  
The view gets blown out or washed out when LAN transfers take place.
Switch size mode to square root and link width to 1.0 and multiplier to the middle.

30. Website relaunch aka transition to production aka going live list  
Check passwords  
Log into Cloudflare for telemetry  
Start both virtual machines and verify server redundancy transition  
Start streamconfig on monitor 2 or monitor 3  
Open ports on cable router

31. Round Robin results  
Enter an a or aaaa record for each round robin IP address all on the same domain.  
A ping will continue to one address until stopped.  
Then a ping will continue to the other address until stopped.  
As expected the browser caches DNS records itself so behavior is good.  
In otherwords every other page does not fail to load.

32. The path of a contingency packet  
What should happen is as follows.  
Cable connection goes down.  
Multi WAN script on servers will eventually decide to switch gateway to static cellular.  
A round robin DNS request will eventually arrive at the cellular router.  
Because server gateway matches cellular router which sends request with DNAT a connection is made.

33. Streamconfig and 2 way web communication list  
HTML chat submission form  
Auto refreshing iframe to show chat  
Bash script to handle form submission instead of Python this time  
Should show busy before timeout is reached to prevent spam  
Clear text file when it gets too large  
Clear text file after 1 week  
Add connection number feature to streamconfig - use netstat  
Add split results feature to streamconfig - optimize grep  
Add chat view feature of last 5 lines to streamconfig - use curl  
Run streamconfig on II bravo companion

34. VLAN commentary   
Only VIP hosts will have server VLAN guest access.  
In otherwords they will be on the main VLAN and the server VLAN.  
Independence switch will need to stay on server VLAN 1 because it is a dumb switch.  
Liberty switch is using a shared simple VLAN scenario with main switch.  
A very simple scenario would be separating web server from other host on switch.  
This case is doing this on one switch but leading to another with similar segregation.  
This does expose VIP hosts on the network to possible attacks from hacked servers.  
However it allows VIP hosts access to 4 gateways and full administration access.  
In true need of protection are innocent guest clients and internal storage servers.  
It is a tradeoff between security and utility.  
However as explained there is still some segmentation to the network.  
Another hardened area of the network could be a VLAN on management switch.  
Clients there could access the internet through gateway 2 but be heavily shielded.  
Should be no loops because each VLAN will act like an independent switch.  
Loops start when devices share VLANs with multiple connections to router.

**Troubles in the Tapestry**  
This is a catalog of glitches encountered during development.  Some with
networking equipment and some with hosts.  By far the longest time was spent
stuck on these problems so bypassing these will speed up the production of any
networking project.

1. Display manager failures on Pi 2 Devuan  
SDDM does not start at all  
Lightdm starts with black screen  
Use /etc/inittab to automate login  
Use bashrc to run startx with a message and a delay
 
2. Boot failure on Lattepanda FreeBSD  
Drop to boot loader  
Set hint.uart.0.disabled="1"  
Set hint.uart.1.disabled="1"  
Setup in /boot/device.hints

3. Bluetooth adapter failures  
hcitool or bluetoothctl or kde no devices found in scan  
Faulty device confirmed in another computer  
Device replaced
   
4. Pi 4 display output failure  
No picture on large tv by default  
Use output 0 and turn safe mode on

5. ZFS mount failure on Lattepanda FreeBSD  
Unique disk ids are not used so usb install is da0 and usb target is da1  
When installer is removed then da1 becomes da0 but fstab references da1  
Recovery can be done without the installer  
At failed boot shell - mount -u /  
At failed boot shell - zfs mount -a  
Edit fstab with vi changing da1 to da0

6. xorg start failure on Lattepanda Freebsd  
/dev/dri/card0: No such file or directory errors  
Install drm-kmod and edit rc.conf to load i915kms

7. KDE start failure on Lattepanda FreeBSD  
Only a black screen with mouse cursor appears  
onestart dbus to create /var/lib/dbus/machine-id  
Now dbus can be stopped permanently and KDE will start  
This used to be the way it works but sadly now dbus must always run

8. FreeBSD KDE favorites failure  
Delete kactivitymanager stuff in config and local  
Switch launchers with show alternatives to have changes take effect

9. Logitech wireless keyboard failure  
Left click was being held down and left click did not work and many os features broken  
Remove keyboard and reinstall  
Unknown failure mode

10. ZTE mf861 management failure  
192.168.1.1 is unreachable with falkon  
Use Firefox to manage device

11. Static IP failures  
Needs APN modifications for 14476.mcs APN  
Sierra 340 MBIM mode - no DHCP and no connection at first - then SIM MEP locked error  
Sierra 340 AT mode - SIM MEP locked error  
Sierra 313 - flashing orange light and no DHCP and no response to at commands  
ZTE mf861 - success - choose add at bottom of APN settings page for custom APN  
VSVABEFV - does not connect by default or with 14476.mcs APN

12. FreeBSD KDE failures  
Log out glitches mouse/KDE hyper slow/screen flicker worse/USB drive access errors  
Disable baloo in /usr/local/etc/xdg/autostart  
This was only a workaround and the real problem was low Lattepanda 5v voltage

13. Freebsd CPU speed unpredictability  
CPU speed jumps around even when set to 480 mhz with sysctl dev.cpu.0.freq  
Disable powerd from starting in rc.conf

14. FreeBSD network monitoring failure with 1366x768  
Knemo does not work anymore  
Ksysguard shows network but all widgets except netspeed widget do not work  
A very roundabout way can be used to show a decent network monitor  
Add a spacer in the panel in the position the monitor will go  
Add a tab in kysguard with 1 column and 2 rows and add re0 download and upload  
Turn off status bar  
The tab can be named network  
Force position to 975x700  
Force size to 125x75 - this will not fully shrink the window yet  
Force no titlebar and frame  
Force keep above other windows  
Force skip taskbar  
Cycle between no menubar and menubar with ctrl-m - this should fully shrink the window

15. FreeBSD network monitoring 2 failure with 1024x600  
Same as before but with 125x65 size and 690 x 530 position  
No cycling menu on and off tricks this time either  
Simply turn menu bar off along with status bar

16. Linux network monitoring failure  
Knemo does not work anymore  
Simply install the old 0.1 version of network monitor  
It looks different and more classic but it works

17. Pi 4 slow media write failure  
USB 3 flash drive - as low as 13 MB per second - had samba file copy errors also  
High end micro SD - as low as 34 MB per second  
Apparently the peformance of a usb 3 ssd is much greater with the pi 4  
This would present usb power problems with the current configuration  
This is somewhat more costly  
This would mean finding the exact model that would work properly  
Also not convinced it would be perfectly without errors  
The micro SD speed now is good enough for a staging server

18. Pi 2 CPU speed failure  
By default the later firmware/kernel packaged with Devuan run this at 1200 mhz  
This is way too high and it used to be run around 700 to 900 mhz  
Use the "overclock" function in config.txt to set it around 800 mhz  
900 mhz was too high to avoid the lightning bolt appearing with a decent power setup  
Also disable wifi and Bluetooth in config.txt  
Devices were running stable before but the red light flash is annoying

19. Pi 4 power workaround failures  
Pi 4 devices are setup in unorthodox situations with limited power  
Such as attached to a laptop computer  
Disable wifi and Bluetooth in config.txt  
Set CPU to 1300 mhz from default of 1500 mhz on staging server  
No CPU modifications on remote access server for best speed  
Devices were running stable before but the red light flash is annoying

20. 192.168.1.1 nightmare failure  
There is a conflict here between the ZTE mf861 device and the core switch  
The ZTE mf861 always must have 192.168.1.1  
So change core switch to easy to remember 192.168.1.100 and have DHCP start at 101  
Change all host default gateway to 192.168.1.100

21. FreeBSD ethernet trickery failure  
Sometimes after a switch goes offline ethernet will get confused  
Physically disconnect interface  
Bring up interface and set IP manually  
Physically reconnect and set gateway if necessary

22. MID HDMI failure  
The mobile internet devices are being used to test from outside the network  
FPV HDMI cables are giving consistent issues  
These have given a lot of trouble in the past  
Monoprice short slim cables are almost as small anyway and much more reliable  
The strain relief on the Monoprice cables can be trimmed for more flexibility

23. EFI setup for troublesome computers failure  
For Lattepanda original and Windows tablets and things like that  
/////  
Ways to enter bios  
Use Windows to reboot into firmware setup if all else fails  
Use Devuan usb recovery grub console with c at boot and fwsetup at Grub command line  
Use installed Devuan Grub console with c at boot and fwsetup at Grub command line  
Use reboot to firmware setup Grub menu option  
Use BIOS quiet boot off to have prompt for del key to enter firmware setup  
/////   
To enter freebsd  
Disk configuration is Devuan and Windows on EMMC and FreeBSD ZFS on micro SD and USB 3  
Probably best to use these steps  
Enter firmware setup using one of 5 methods  
Choose boot override option for USB 3 instead of micro SD  
/////  
Boot fix 1  
Grub is bugged after messing around with operating systems  
Boot up in rescue mode from USB installer and mount system root  
Mount and or create EFI partition as first partition  
Run grub-install now that the system is automatically chrooted  
/////  
Boot fix 2  
After a fresh install grub still fails with a command line  
Grub prefix is looking for a "debian" folder which does not exist  
This prefix appears to be hardcoded into the Grub EFI files  
Copy the /boot/efi/devuan folder to "debian" to fix  
/////  
Boot fix 3  
After a fresh install windows still boots and the BIOS boot order is not respected  
The BIOS has no respect for you so delete or move the Microsoft folder in EFI partition  
Anything can be chosen in BIOS but it does not matter  
Essentially a multi boot through BIOS is not possible  
The operating system loader must enable a multi boot situation

24. High power 5v output paradox failure  
No batteries output around 5v  
Almost no USB chargers output around 5.2v for SBC/7inch screen/SSD  
Almost no fixed voltage power supplies output around 5.2v for SBC/7 inch screen/SSD  
A rare adjustable power supply is required along with battery and charger  
Or a rare 5.2v USB charger and commercial UPS combo  
Use Pololu adjustable/medium lead acid/trickle charger  
Alternatively use Canakit 5.2v charger and APC UPS

26. Netgear smart managed pro failure  
Types of VLANs  
802.1x/q layer 2  
Standard layer 3  
Port based  
/////  
Definitely did not want to do 802.1x  
/////  
Did not want to do layer 3 VLAN because it requires changing up IP addresses  
It may not even work  
This is how DD WRT handles VLANs  
In DD WRT this works well if you understand and are willing to change IP addresses  
It requires that each VLAN have a corresponding subnet  
/////  
Port based unavailable with gs108t  
Netgear says that smart managed pro provides same features as plus with more options  
This is a conflict  
Port based works perfect with the smart managed plus switches

27. KDE start failure on AMD Ryzen Pro 2400GE on FreeBSD  
gpu-firmware-kmod-somethingsomethingsomething was installed  
xf86-video-amdgpu-somethingsomethingsomething was installed  
xorg.conf specified modesetting as driver with no busid field  
rc.conf specified kld_list="/boot/modules/amdgpu.ko"  
xinitrc specified exec ck-launch-session startplasma-x11

28. Cloudflare proxy failures  
Was working before proxy  
Most things seem broken due to proxy  
Channels page html largely broken  
All streams on page and through vlc broken  
Working again after disabling proxy  
May have something to do with time to live

29. Ping failure in virtualization and internet server multi WAN script  
Gateway ping is sometimes taking a large delay with these devices now  
This happens after putting the virtualization server on a VLAN  
After rebooting core switch still seeing similar failures  
After rebooting virtualization server still seeing similar failures  
Increase timeout on both scripts  
Now failures in media script  
Latency culprit is definitely second gs116ev2 since reconfiguring VLANs

30. Dell switch limitations and failures  
Monitor mode works fine  
There is no QOS mode  
However VLAN mode is quite useless  
The switch has no internal layer 3 features  
Managed switches sold as layer 2 like Netgear plus have an internal layer 3 "bridge"  
On the Netgear it is as simple as assigning a port to more than one VLAN  
On the Dell that is impossible due to the strict layer 2 nature  
DD WRT makes this possible because it is a true layer 3 switch

31. Switch web monitoring failure  
The Dell switch automatically refreshes the monitoring page  
The Netgear switch does not  
Automatically refreshing normally returns to the main page  
Set Firefox auto refresh page extension to auto click #monitoringSecNav element

32. Cellular device alternate configuration failures  
Test going back to wvdial style connection  
lsusb -t shows driver  
disablembimglobal option set to 0 in usb_modeswitch.conf  
This allows /dev/ttyUSBX to appear on MBIM devices  
Will always be an unblocked connection if no carrier grade NAT is enabled  
ZTE mf861 - ssim - broken (works with cdc) - disablembim irrelevant with cdc_ether  
VSVABEFV - msim - broken (terrible!!!) - disablembim irrelevant with cdc_ether  
Sierra 340 - lsim - ttyUSB1 works well but screen does not show - cdc_mbim  
Sierra 313 - lsim - ttyUSB3 is perfect - disablembim irrelevant with sierra  
So this mode is only relevant to making Sierra 340 work better  
However sierra 313 does work dial up style with sierra driver  
ZTE mf861 just works well in general and may expose all ports anyway  
VSVABEFV is trash

**The Seer's Knowledge**  
This is networking information organized in a cheat sheet fashion.  For
deploying ever more advanced installations.  It can be added to and probably
will be added to here also.  It is important to take complex information and
word it in a way that is more understandable.

**Instructions for Incantations**  
The amount of scripts needed to enable the required functionality was a
suprise.  In a certain way these scripts are replacing typical enterprise
hardware networking options.  Scripts are sorted into a folder according to
associated features.  After copying make these executable.
1. Architecture  
This may include scripts that are expected to run during contingency events.
It is important to have a disaster response plan for your IT department to 
immediately respond to certain emergencies.  In the dark with alarms beeping
when you smell smoke and the wind is howling outside is too late to be
researching information.  You do not want to be hauling heavy batteries up
several flights of stairs during tornado conditions with emergency lighting only.
You do not want to be wondering if your last 2 computers will keep running when
most of your power sources have failed and there is a twister outside.  
Architecture document showing overview of hosts/remote admin/website/redundancy  
Switch layout document showing ports and roles  
Map document showing host IP and hostname  
WAN info document showing details such as static IP and netmask  
Credentials document showing service login details
2. Multi WAN  
A reliable network starts with reliable Internet connections.  However, consider
if all network computers truly need backup gateways or only a subset.  This is
another tradeoff between hardware features in network devices and scripting.  
Linux Multi WAN - several script examples for different Linux computer roles  
FreeBSD Multi WAN - uses the unique FreeBSD fibs system  
Router cgnat - continuation of mid script now streamlined  
Router static - continuation of mid script with several extra features  
Router worker - needed to assist the static router for data usage computation  
DD WRT fixup 1 and 2 - handles network segmentation among other things  
rc.local.VIRTUALIZATIONSERVER - auto start multi wan  
rc.local.BACKUPINTERNETSERVER - auto start multi wan and redundancy and status  
rc.local.INTERNETSERVER - auto start multi wan  
rc.local.RAS - auto start simple multi wan and container  
SMS.py - sends administrative alerts through VOIP.ms to an SMS number  
3. Streaming  
The result of an effort to obtain a modernized website with multimedia features.
Data and power usage from a modest streaming system does not have to be excessive.
Websites like Youtube are giving creators ever more control of their streams
while simulateneously exerting control on those same creators.  These tools
allow for at home streaming.  
Recorder - webcam slave script  
Streamconfig - easy to use and informative stream front end  
Streamlauncher - daemon to handle stream startup and restarts  
Windows VLC shortcut - 2 clicks to start network streaming a windows display
4. Chat  
There was a need identified for 2 way website communication.  This is useful
for fan feedback during streams or for business messaging or even just a
visitor's sign in area.  The amount of external calls has been kept to a
minimum to keep the potential attack surface low.  Like everything else
presented here this is a no javascript solution.  
CHAT - handler bash script  
CHAT.txt - example text file shown in iframe  
CHAT.html - example html meant as iframe  
Index.html.CHAT - example html main page  
Abuse timestamp - helps to prevent spamming  
Reset timestamp - helps to prevent spamming and over large files  
5. Upload  
The existing SFS uploader has been moved here to be with the other internet
server relevant information.  Allows for file upload and upload status
monitoring.  This also uses a small amount of external calls and implements a
basic security scheme.  Like everything else presented here this is a no
javascript solution.  
Index.html.UPLOAD - example html component that displays data  
UPLOADER.py - python component that handles upload  
HELPER_UPLOADER - bash component that does status updates  
6. Server Redundancy  
Typically CARP is used in the enterprise to enable the takeover and replacement
of one server to another.  This can be crudely simulated with some thoughtful
scripting.  While this does work a solution like this would not scale well to a
large organization.  This scripting involves a complex chain that all needs to
be implemented correctly to work.  
SFS server redundancy 1 - Proxmox hook script that starts pre and post script  
SFS server redundancy 2 - run on virtualization server pre VM boot  
SFS server redundancy 3 - runs on backup internet server  
Persistent - runs on virtualization server post VM boot  
Worker 1 - launches netcat for redundancy 2 and persistent to backup server  
7. Remote Access Server  
While not strictly required versus forwarding ports this seems like an
interesting alternative.  It allows for things like putting computers with
questionable security status behind another system rather than being directly
on the Internet.  A computer with moderate power is useful here for better remote
desktop smoothness.  Setup Windows hosts for RDP manually.  
Ports configuration RAS - open ports for SSL proxy  
SFS-SSL.conf.RAS - setup SSL proxy  
VNC server - auto start VNC for Linux host  
8. Backup Internet Server  
A more simple non virtualized backup internet server can take over while the main
server undergoes maintenance.  Another idea is to use the backup as a network
status server while the main server is operational.  This server can reside on
a different IP with appropriate forwarding or a more complex redundant
arrangement can be used like the server redundancy scripts.  
Index html example - arranges status page to be displayed in iframe  
Status html - displayed in iframe and updated dynamically by network agent  
Network agent - handles the backend status work and updates html  
config.txt.BACKUPINTERNERSERVER - raspberry pi specific power saving boot options  
9. Staging Server  
Acting as a kind of network scratchpad the staging server is a miscellaneous
bin for files.  It can use minimal security as it is a temporary file holding
point only and is meant for internal use only.  Of chief importance is to
support a server like this one with relevant archival servers.  SAMBA allows
for easy networking and file exchange with clients of different operating
systems.  
SMB configuration - samba configuration for easy access and no login dialogs  
config.txt.STAGINGSERVER - Raspberry pi specific power saving boot options  
10. Power Control  
It is advantageous to have fine tuned centralized energy management for a
number of reasons.  The network can be moved to a moderate energy state when
the network administrator is absent.  Additionally a low overall energy state
can be quickly achieved during an adverse power event.  Enables remote mains
voltage monitoring and could be tailored for battery voltage monitoring.  Other
future applications include IOT electric switch control/POE control/WOL control.  
Central power administration - server script running in workstation background  
Listener - example client
11. Network AI  
Star Trek has inspired the creation of this currently passive "AI".  It only
talks and does not listen.  This voice assistant is tied into phone
communications by way of KDE Connect and tied into the network courtesy of
Etherape.  A basic ethernet hub or a managed switch is a must to use this.  This 
is a continuation of an earlier effort to add a voice assistant to the
MID project.  
SFS AI - main logic  
AI connection dialog - present a dialog for phone reconnection  
AI persistence dialog - present a dialog to dismiss persistent notifications

**Forbidden Tomes**  
There are certain documents that for reasons of security and practicality will
be unique for every network.  Some listed here will be sanitized examples that
absolutely must be edited with custom credentials.  Some examples displayed here
will be unique to the network but non-confidential.  
1.Recorder  
2.Server redundancy 3  
3.SMS python script  
4.Streamconfig  
5.Streamlauncher  
6.VNC server  
7.Worker 1  
8.Uploader python script  
Remember to edit the passcodes in these files.

**Fruits of the Quiltmaker**  
This is a list of what can be expected with a typical end result.  A product of
the struggle to get the network under control.  There is enough here to build
on and fill the switch ports further than they already are or stop building and
just enjoy what has been created.

