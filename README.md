![](https://github.com/bobbybudnick/SFS-Infrastructure/blob/main/NETWORK_2POINT2.jpg)

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
On a non dedicated management network overlaid on main network VLANs could be used.  
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
Management/VIP devices are workstation/media/virtualization server.  
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
Enable sshd and use scp because sftp is not available by default.  
copy to router - scp -r root@192.168.1.1:/mnt/sda1/TEST /home/jason/TEST  
copy from router - scp -r jason/ root@192.168.1.1:/mnt/sda1/jason  
Also FTP and samba are available from the NAS page but maybe SSH will use less RAM.  
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

14. Server UPS power survey list  
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
Makes both computer cores have 2 console 2 GUI 2 headless  
May need as spare MID  
May need for demonstration purposes  
Battery difficult to charge from larger battery

20. Redundancy vs responsiveness in multi WAN configurations  
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
Add connection number feature to Streamconfig - use netstat  
Add split results feature to Streamconfig - optimize grep  
Add chat view feature of last 5 lines to Streamconfig - use curl  
Run Streamconfig on II Bravo companion

**The Arrowproof Tunic**  
Focusing on locking down and securing all equipment from outside miscreants.
More scripts and diligence were required.  A closer eye revealed some cracks
in the armor which were interwoven anew.

1. VLAN commentary   
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

2. Further switch improvements  
More network segmentation is ideally needed against Internet intruders.  
Smart managed plus 8 port gs108e is a good choice.  
3 switches are needed with initial thoughts.  
CC Independence/management/backup with decomissioned administration for CC Liberty.  
A managed switch is not strictly required for CC Independence.  
Most independence hosts can use dumb switch connected to server VLAN port on main.  
The backup storage server can use a dedicated connection to the management switch.  
In this way main storage server should also be directly connected to management.  
Only 2 switches are needed now with this approach.  
Management/backup with added or decomissioned administration.  
Setup VLAN on management switch with workstation/main storage/backup storage.

3. Blackwall concept  
Somewhere behind the PFSense firewall.  
And then somewhere behind the core switch firewall and layer 3 VLAN.  
And then somewhere behind the management switch VLAN.  
Lies a shadowy place where storage servers are behind the final line of defense.  
Workstation and main and backup storage are the only denizens of this realm.  
Rogue AI could perhaps be confined here or kept out of here depending on design.

4. The scoop on setuid root  
This is the way of setting executables to execute as root but run by user.  
FreeBSD and Linux cannot directly execute scripts this way only binaries.  
Can edit sudoers file to bypass this.  
Can write small c executable to bypass this.  
Can call setuid root executables directly instead of scripting to bypass this.  
Tested on FreeBSD.  
Copy shutdown to home folder.  
chown root:wheel /home/jason/shutdown to change ownership on FreeBSD  
chown root:root /home/jason/shutdown to change ownership on Linux  
sudo chmod u+s /home/jason/shutdown for setuid root  
/home/jason/shutdown -r 0001010130 shuts down so late that it errors out.  
This is used for testing the shutdown command without a built in test on FreeBSD.  
shutdown -k now can be used to test shutdown on linux without shutting down.  
Of course shutdown -h now shuts both down.  
Sometimes the calling executables directly method does not work.  
This is because some executables call other executables such as with dhclient.  
It is then best to use the sudoers file method in this case.

5. Creating an RAS security layer  
Passwords removed for clients within remote access server.  
This is an extra layer of security in exchange for less convenience.  
If there is a break in to the ras server they will not get very far.

6. Centurion for core switch idea  
Could check for connections on obscure port.  
Could check for connections on common but unused ports.  
Could Block IP with iptables.  
Or could harvest addresses after a while and add to router block lists.  
The honeypot port would need to be opened at main gateway to reach core switch.  
This would largely eliminate bots that portscan.

7. Practical port knocking idea  
First idea was to use netcat on router to listen for a passcode to open ports.  
Similar to Centurion could check for connections on obscure ports.  
Similar to Centurion could check for connections on common but unused ports.  
Could also use custom ping size as trigger.

8. iptables blocking  
iptables can be set to block IP addresses after a set number of attempts.  
These addresses can also be logged.  
Good idea to implement this on each public SSH server at least.  
Need to have a way to disable this for administrative purposes.  
Both main Internet server and cellular static gateway have Internet facing SSH.  
These both are configured with iptables based SSH lockout mechanisms.  
3 logins at max over a one day period are enabled.  
From local login the cellular gateway will need to be reset manually if tripped.  
From local login the internet server can be accessed on secondary if tripped.  
From remote login the first thing should be to clear or change the iptables rules.  
The login on secondary adapter is a practical example of the management network.

9. List of blacklists  
Primary - static list from github hosts.deny blocking project  
Secondary - dynamically generated by captcha mechanism arty  
Tertiary - dynamically generated by core switch Centurion arty - not active  
Quaternary - dynamically generated by cellular gateway SSH arty - not active  
Quinary - dynamically generated by main internet server SSH arty - not active

10. Dealing with bots  
Remember the only bot ingress routes are the static IP connections.  
There are various pages that provide user aggregated hosts.deny lists.  
These can be modified and formatted to plain IP lists if necessary.  
cat hosts.deny | cut -d' ' -f2 > BLACKLIST  
wc -l shows similar amount of lines on 7.8M blacklist file vs 2.6M hosts0.deny.  
Something in the encoding of the text file must have been the difference.  
Important to note that iftop will still show blocked addresses.  
/////  
Solution 1  
Use hosts.deny.  
hosts.deny does work fine with ssh after testing on local network.  
A connection reset by peer error happens when blocked by hosts.deny with SSH.  
Not working to block on proxmox guest when host has modified hosts.deny.  
This solution seems good for the static cellular router.  
This is good because it blocks the internal SSH from most bad guys.  
Also good because it may still block web because it is routed not internal.  
Even if web is not blocked connections should be rare because this is a fallback.  
Looks like a good choice without having to get into something more complicated.  
Still important for internal SSH blocking even when port forwards have been disabled.  
Even so it is just too tedious to maintain blocklists for the cellular gateway.  
However iptables SSH internal connection throttling and logging can be used.  
/////  
Solution 2  
Use port knocking.  
In otherwords do not block problem addresses and use knocking for critical servers.  
/////  
Solution 3  
Use Hardware firewall or semi hardware firewall like PFSense.
PFSense has PFBlockerNG just for this.  
No doubt DD WRT has some similar plugin functionality as well.  
To keep it simple a list alias approach was used instead of PFBlockerNG.  
PFSense has feature to download a list of IP addresses and format them as a table.  
This table is referenced as an alias and can be referred to within firewall rules.  
A simple block rule that involved the table was all that was necessary.  
To add alias go to firewall > aliases and specify web server address.  
Be sure to add the blacklist with just addresses to a web server of your choice.  
Also increase firewall maximum table entries from 200000 to 800000 in PFSense.  
/////  
Solution 4  
Use dynamic firewall rule switching based on network status.  
Also a solution for unwanted web connections from fallback gateway.  
Web connections continue despite cable gateway HTTP block.  
pfsense state reset and filter reload did not help.  
Even changing the forwarded port on PFSense did not help which was suspicious.  
ipv6 was disabled on PFSense.  
Not disabled on static cellular gateway.  
Now disabled on static cellular gateway.  
Early testing with ipv4 showed no connections when cellular was not gateway.  
The thinking was that ipv6 from cellular was problem.  
Yet the problem seems to be connections from cellular anyway.  
Backup /etc/sysctl.conf on static cellular gateway.  
The solution is packets are coming in on cellular but probably not going out.  
This means that ideally the cellular gateway should block connections by default.  
Only during a contingency event do connections need to be made through gateway.  
Testing with cellular gateway disabled revealed no connections.  
Need to monitor status server gateway and take action based on that.  
Should have rule to block connections if status server gateway is cable.  
Should have rule to allow connections if status server gateway is cellular.  
Status server is in same physical cluster or computer core.  
Avoids needing to have a script to test connections.  
Once status server has switched gateways hopefully all VIP computers have as well.  
/////  
Conclusion  
In the end all attempts are not blocked because new addresses appear all the time.  
It is also better than nothing.  
Can be combined with other security methods as part of a greater strategy.

11. Captcha idea  
New addresses pop up all the time that are associated with botnets.  
So the block lists do not catch all rogue addresses.  
Also all addresses are not associated with scans so centurion can not filter them.  
If bots not on blacklist connect to port 80 without a scan then captcha required.  
Captcha needs to be on index.html so it is hopefully seen first.  
Add good hosts that answer captcha correctly to white list.  
Check connections to server resources except index.html and compare to white list.  
Host is bad if they fail captcha or access resources without being on whitelist.  
Simply add bad hosts to local secondary blacklist on web server.  
Whitelist should contain administration device for testing and configuration.  
Whitelist should contain cable gateway so it can retrieve blacklists.  
This would largely eliminate bots that ddos or attempt to upload exploits.

12. Possible captcha alert levels  
Could signal threat level low/medium/high/extreme on status page.  
In practice just level 0 and level 3 are used.  
Level 3 - basic index.html with no captcha and link to content.html.  
For high traffic events that we do not want to risk banning anyone important.  
Level 2 - index.html with nonaggressive captcha.  
When no other resources are accessed there is no ban.  
When captcha not correct there is no ban.  
Banned when - skipped captcha.  
For standard operation.  
Level 1 - index.html with aggressive captcha.  
When no other resources are accessed there is no ban.  
When captcha not correct there is a ban.  
Banned when - bad captcha/skipped captcha.  
For use when under sustained attacks to cull the opposition.  
Level 0 - index.html with captcha that does not allow navigation away without ban.  
Regardless of other resources accessed there is a ban if captcha not correct.  
When captcha not correct there is a ban.  
Banned when - bad captcha/skipped captcha/no captcha.  
For a deadly nuclear blow with no regard for the innocent.

13. Serious things to say to deter hackers  
These items are in addition to the items already on the level 0 captcha HTML page.  
You were caught better luck next time.  
Your IP address is...even if not correct.  
Quote US computer fraud laws.  
Quote US prison conditions.  
Quote US extradition agreements.

14. Basic types of network monitoring  
3 examples of monitoring internally.  
Monitor through switch port mirroring - monitor 1.  
Monitor through virtualization server hypervisor - monitor 2.  
Monitor through inline/transparent firewall - monitor 3.  
2 examples of monitoring at the edge.  
PFSense pftop at cable edge gateway.  
Linux iftop at cellular edge gateway.

15. Basic types of firewalls  
Endpoint firewalls - filter in and out - acts as gateway.  
Internal router firewalls - filter in and out - acts as gateway.  
Inline/transparent firewall (like SFS) - filter in/out - does not act as gateway.  
Proxy/DNS firewalls - filter out - does not act as gateway.

16. Inline/transparent firewall  
Part of a multi layered defense.  
Attacker could take over cellular gateway and be close to critical resources.  
Current - cellular gateway > main switch > core switch > admin switch > storage.  
Proposed - gateway > firewall > main switch > core switch > admin switch > storage.  
To do this would require using brctrl bridging functions in the firewall.  
Ethernet 1 on firewall - subsidiary switch.  
Ethernet 2 on firewall - uplink to main switch.  
Use is for 2 reasons.  
Reason 1 - monitoring of all traffic through firewall hopefully at layer 2 and 3.  
Reason 2 - iptables blocking of cellular gateway connections back inside network.  
This blocking strategy will allow for automatic containment of a hacked gateway.

17. Inline/transparent firewall redundancy vs speed  
Having the 10/100 switchbox connected to uplink means entire cc is speed limited.  
The alternative would be to connect uplink to gigabit connector on monitor.  
The problem with this is it would not allow for the switchbox.  
Battery will die on monitor before cellular gateway so this can not work well.

18. Inline/transparent firewall power concerns  
Manual override switch will allow gateway a direct connection to switch.  
Override important during extended power failure due to firewall laptop battery.  
The gateway battery is setup to last much longer.  
Because of this gateway is an integral part of network during power contingency.  
Because it will likely be needed right away as cable fails when power is out.  
Does represent a tradeoff in loss of unattended power reliability for security.  
Does also represent a tradeoff in transfer speed for security.  
Seems worth the tradeoff as it is strange to have a monitor that does not monitor.  
In otherwords this will allow monitor 3 to realize true potential.  
Also a small commercial UPS would run the firewall laptop for several hours.

19. The mysteries of bridging  
Previously working bridge used scheme where bridge-noIP WAN-IP LAN-IP.  
This bridge does not work when configured that way.  
To troubleshoot set error during ping adapter ip address to 0.0.0.0  
Also to troubleshoot just try the directly connected adapter without bridge.  
Ultimately this setup needed having bridge with ip and both adapters with none.  
Thus the bridge IP becomes what was eth5 IP of 192.168.1.6 in this case.  
Any iptables troubleshooting is a red herring as that is layered on top of this.  
Firewall is not and should not be setup to use cellular gateway.

20. Firewall blocking conundrum  
First off br_netfilter module needs to be loaded to pass layer 2 data to layer 3.  
Initially it was thought better to block connections originating from gateway.  
But this complex solution would involve ebtables and iptables packet marking.  
Just using iptables does not work with this method because interfaces not seen.  
The interfaces are not seen due to complex interaction with br_netfilter.  
Not clear if blocking gateway originating connections would block normal routing.  
Could try the iptables portion of complex method with --state NEW if very paranoid.  
This would also require a way to disable when logging in remotely.  
Blocking ports wholesale will not work because gateway is a critical redundancy.  
One simple way is to just block port 22 because there is no good reason for that.  
The simple way would block any external connections to sensitive internal SSH.

21. Novelty of inline/transparent firewall  
Combines switching/bridging with no configuration of routes because not gateway.  
Provides an extra layer of defense easily.  
Allows for manual bypass switch in case of firewall failure.

22. Internet artillery order of battle  
Emperor of violence  
Chaos sentinel  
Dread cyberia

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

3. Bluetooth adapter failures  
hcitool or bluetoothctl or KDE no devices found in scan  
Faulty device confirmed in another computer  
Device replaced
   
3. Pi 4 display output failure  
No picture on large tv by default  
Use output 0 and turn safe mode on

4. Logitech wireless keyboard failure  
Left click was being held down and left click did not work and many os features broken  
Remove keyboard and reinstall  
Unknown failure mode

5. ZTE mf861 management failure  
192.168.1.1 is unreachable with falkon  
Use Firefox to manage device

6. Static IP failures  
Needs APN modifications for 14476.mcs APN  
Sierra 340 MBIM mode - no DHCP and no connection at first - then SIM MEP locked error  
Sierra 340 AT mode - SIM MEP locked error  
Sierra 313 - flashing orange light and no DHCP and no response to at commands  
ZTE mf861 - success - choose add at bottom of APN settings page for custom APN  
VSVABEFV - does not connect by default or with 14476.mcs APN

7. Linux network monitoring failure  
Knemo does not work anymore  
Simply install the old 0.1 version of network monitor  
It looks different and more classic but it works

8. Pi 4 slow media write failure  
USB 3 flash drive - as low as 13 MB per second - had samba file copy errors also  
High end micro SD - as low as 34 MB per second  
Apparently the peformance of a USB 3 SSD is much greater with the Pi 4  
This would present USB power problems with the current configuration  
This is somewhat more costly  
This would mean finding the exact model that would work properly  
Also not convinced it would be perfectly without errors  
The micro SD speed now is good enough for a staging server

9. Pi 2 CPU speed failure  
By default the later firmware/kernel packaged with Devuan run this at 1200 mhz  
This is way too high and it used to be run around 700 to 900 mhz  
Use the "overclock" function in config.txt to set it around 800 mhz  
900 mhz was too high to avoid the lightning bolt appearing with a decent power setup  
Also disable wifi and Bluetooth in config.txt  
Devices were running stable before but the red light flash is annoying

10. Pi 4 power workaround failures  
Pi 4 devices are setup in unorthodox situations with limited power  
Such as attached to a laptop computer  
Disable wifi and Bluetooth in config.txt  
Set CPU to 1300 mhz from default of 1500 mhz on staging server  
No CPU modifications on remote access server for best speed  
Devices were running stable before but the red light flash is annoying

11. 192.168.1.1 nightmare failure  
There is a conflict here between the ZTE mf861 device and the core switch  
The ZTE mf861 always must have 192.168.1.1  
So change core switch to easy to remember 192.168.1.100 and have DHCP start at 101  
Change all host default gateway to 192.168.1.100

12. MID HDMI failure  
The mobile internet devices are being used to test from outside the network  
FPV HDMI cables are giving consistent issues  
These have given a lot of trouble in the past  
Monoprice short slim cables are almost as small anyway and much more reliable  
The strain relief on the Monoprice cables can be trimmed for more flexibility

13. EFI setup for troublesome computers failure  
For Lattepanda original and Windows tablets and things like that  
/////  
Ways to enter BIOS  
Use Windows to reboot into firmware setup if all else fails  
Use Devuan USB recovery grub console with c at boot and fwsetup at Grub command line  
Use installed Devuan Grub console with c at boot and fwsetup at Grub command line  
Use reboot to firmware setup Grub menu option  
Use BIOS quiet boot off to have prompt for del key to enter firmware setup  
/////   
To enter FreeBSD  
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

14. High power 5v output paradox failure  
No batteries output around 5v  
Almost no USB chargers output around 5.2v for SBC/7inch screen/SSD  
Almost no fixed voltage power supplies output around 5.2v for SBC/7 inch screen/SSD  
A rare adjustable power supply is required along with battery and charger  
Or a rare 5.2v USB charger and commercial UPS combo  
Use Pololu adjustable/medium lead acid/trickle charger  
Alternatively use Canakit 5.2v charger and APC UPS

15. Netgear smart managed pro failure  
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

16. Cloudflare proxy failures  
Was working before proxy  
Most things seem broken due to proxy  
Channels page html largely broken  
All streams on page and through VLC broken  
Working again after disabling proxy  
May have something to do with time to live

17. Ping failure in virtualization and internet server multi WAN script  
Gateway ping is sometimes taking a large delay with these devices now  
This happens after putting the virtualization server on a VLAN  
After rebooting core switch still seeing similar failures  
After rebooting virtualization server still seeing similar failures  
Increase timeout on both scripts  
Now failures in media script  
Latency culprit is definitely second gs116ev2 since reconfiguring VLANs

18. Dell switch limitations and failures  
Monitor mode works fine  
There is no QOS mode  
However VLAN mode is quite useless  
The switch has no internal layer 3 features  
Managed switches sold as layer 2 like Netgear plus have an internal layer 3 "bridge"  
On the Netgear it is as simple as assigning a port to more than one VLAN  
On the Dell that is impossible due to the strict layer 2 nature  
DD WRT makes this possible because it is a true layer 3 switch

19. Switch web monitoring failure  
The Dell switch automatically refreshes the monitoring page  
The Netgear switch does not  
Automatically refreshing normally returns to the main page  
Set Firefox auto refresh page extension to auto click #monitoringSecNav element

20. Cellular device alternate configuration failures  
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
However Sierra 313 does work dial up style with sierra driver  
ZTE mf861 just works well in general and may expose all ports anyway  
VSVABEFV is trash

21. Camera server 2 failures  
Try changing ports - all failed  
Try replacing camera - fixed  
Try replacing with thicker USB extension - not tried  
Try relocating to remove USB extension - not tried

22. VLC reliability failures  
Audio dropouts are untenable and CPU usage is high  
Disabling mitigation - would never be done on VM server in professional setting  
Overclocking up to 3.6 ghz - uses more power on an overloaded charger and battery  
Adding more cores to VM - not sure this will fix because load is single threaded  
Use Firefox on phone for more efficient codec usage - would like to stay with default  
cpufreq on VM server runs CPU at higher than design speed between 3.15 and 3.4 ghz  
It only overclocks to around 3.6 ghz and it is a shock that it is overclocking at all  
It may be the motherboard that is somehow doing the overclocking or an error  
Seems that Proxmox CPU type set to host in VM config helps a large amount  
Before stream 1 without audio was 50% CPU and with audio was 100% CPU  
After CPU type tweaking stream 1 with audio is about 60% CPU  
Still getting TS discontinuity errors even with lower CPU usage  
Still getting audio cutouts even with lower CPU usage  
Still errors after some time with 2 cores  
Still errors even with 3 cores so switched back to 2 cores  
Still audio fails again with an error that says audio codec cannot be found  
Errors do not seem directly performance related so performance fixes may not help  
The only alternatives include forced restarts/codec change/VLC source code fixes  
Need to restart anyway due to the amount of errors bugging up Streamconfig  
Stream 1 is so bugged it does not even play after 10 hours  
Stream 4 still functions after 10 hours  
There is a problem with restarting over long periods  
There could be times where it does or does not work depending on when it restarted  
If at any time it is caught with failed audio then decrease time  
Restart reimplemented for stream 1  
Try 4 hours then 2 then 1 while troubleshooting  
Audio is working with 4 hour restarts

23. Poor Stream Quality Failures - Unsolved  
Should look into improving stream clarity  
Hard to tell what is happening in printer and on workspace  
Compare the output to what top youtubers look like

24. Stream 1 Failures - Unsolved  
HDHomerun had blinking tuner light 1  
HDHomerun power disconnect and reconnect did not fix  
http reconnect did not fix  
Unknown if scheduled restart would have fixed  
Fixed by semi manual stream restart through streamconfig

25. Network Mystery Failures  
netstat hangs too long on reencoder sometimes  
Redundancy script fails on gateway 1 with VM and internet server sometimes  
Connection between staging server and workstation hangs sometimes  
Initial strategy is removal of routes one by one to see which one causes hang  
Now route and netstat do not hang after reboot on reencoder at least  
Now at the same time route and netstat do not hang on virtualization server  
Maybe had some glitch to do with old reencoder instance or rc.local lack of &  
Possibly the timeout delay was just too short

26. Proxmox VM shutdown failures  
Also no virtual console login for vm is a symptom  
Kill process of vm  
ps aux | grep <VMID>  
kill processnumber  
Caused by /etc/rc.local not having & on end of script

27. Proxmox SSH login failures  
permitrootlogin option in SSH config had somehow been changed  
Now changed back and it works fine

28. Multi WAN script potential failures - Unsolved  
Detected on media and virtualization server  
The Internet connection fallback section of the script did not trigger

29. sftp failures  
when bashrc is customized sftp can say received message too long  
Add this to bashrc:  
case $- in  
    *i*) ;;  
      *) return;;  
esac

30. OpenSSH DNS disable failures  
Very frequent and unnecessary DNS connections are made during local SSH sessions  
Dropbear DNS disable only seems to be an option in source code  
lsh login not working with lsh client or OpenSSH client  
No guarantee that wolfssh and lsh will not be the same way with DNS requests  
One solution might be to block port 53 with iptables  
One solution might be to use regular telnet/telnet with stunnel/telnet-ssl  
Blocking port 53 seems to be the better solution  
sudo iptables -A OUTPUT -p udp --destination-port 53 -j DROP

31. zram failures  
Some anti-user garbage malware named zram is installed now on Devuan  
Does not configure itself in the standard location of /etc/fstab  
Additionally any new swapping functionality is nonsensical in the modern era  
Move zram.ko out of the kernel folder or delete this trash to disable

**The Seer's Knowledge**  
This is networking information organized in a cheat sheet fashion.  For
deploying ever more advanced installations.  It can be added to and probably
will be added to here also.  It is important to take complex information and
word it in a way that is more understandable.

1. Futility of switch redundancy  
A failover at the ethernet port level is extremely impractical for a small network.  
It is also very complicated to think about and implement.  
An enterprise grade switch with redundant internal logic and power supplies are one way.  
The use of dual ethernet and second switch for critical equipment is another way.

2. Load balancers compared to round robin DNS  
Both technologies can be used to provide access to multiple servers.  
The load balancer can check that certain servers are up and can be on or off network.  
Round robin is DNS based and can be on or off network and is not intelligent.

3. Load balancers compared to reverse proxy  
Reverse proxy allows for different servers for subdomains.  
Load balancer allows for multiple servers for the same resource and can bounce between.

4. Reverse proxy compared to port forwarding  
This is new complex technology vs old simple technology.  
Instead of port forwarding subdomains are used.

5. Load balancer disadvantages  
The load balancer itself is a single point of failure.  
Available off the shelf load balancers look very proprietary.  
The real availability problem is one of electrical and internet connection not servers.

6. Possible things to show on monitor devices  
iftop over SSH of web or virtualization server.  
Proxmox status in the web browser.  
Etherape IDS with perhaps a text based output to accompany it.  
The output from RDP/VNC sessions.

7. Multi server availability  
This is the case of needing redundant servers.  
Load balancer on internal network would be good here.  
CARP protocol can allow 2 servers to share an IP and load balance between them.  
Another idea is not to have server redundancy but break the load into smaller servers.  
Also see our custom CARP replacement solution provided here.

8. NAT vs routing  
They are kind of like opposites of each other.  
NAT is multiple IP adddresses on one MAC address.  
Routing is multiple MAC addresses and IP addresses on one interface.

9. VLAN vs subnet  
VLANs are layer 2 network segmentation.  
Subnets are layer 3 network segmentation.

10. Software defined WAN  
Mostly concerns connection between remote sites within a company.  
Going away from switched circuits to using business cable/DSL/fiber.  
Comes from DMVPN/IWAN traditionally.  
Performance issues can be challenging with these older solutions.  
Takes logic away from processors in the routers and moves it to other computers.  
The logic referred to is also called the control plane.  
The other computers are sometimes running as a VM on a server in a datacenter.  
Seems like corporate version of replacing home routers with a PFsense computer.  
Can use really high performance hardware with routing now and also scale easily.  
Routing in a sense is static because it is specified beforehand with Vsmart.  
The routing protocol between routers is called OMP and is somewhat like BGP.  
IPsec is used for the data channels between routers but no routing is exchanged.  
Most interestingly it can be used to deal with dual WAN concerns.  
The failover problem with slower backup internet being used is the concern.  
Vsmart appaware can automatically switch connections to the best WAN link.  
Vsmart can do user profiling and auto switch routes for best performance.  
This auto route switching indeed does make the network routing dynamic.  
The network routing is dynamic but in a different way.  
IPsec between routers adds a layer of encryption for extra security.  
App encryption isolation is another feature that keeps channels separate.  
Naturally there are many firewall options available with this technology.  
Obviously no advanced technologies like these in use here but good to know.
 
11. Image writing computer options  
Gaming companion - access to staging  
Monitor 3 - use sftp to send images  
Gaming laptop - access to staging

12. Looped vs on demand programming  
Typically SFS does loops for background jobs.  
But this is not the only way.  
All tasks can be done all at once by a predefined action.  
This could be considered an eventhandler.  
An event happens and processing occurs because of it.  
With a loop the processing usually happens at predefined time intervals instead.  
The web scripts are an example of on demand meant to keep the server more idle.

13. In the event of core switch failure  
CC Liberty and CC Independence switches connect to main switch.  
So connectivity can happen from main switch > subsidiary switch > fallback gateway.

14. RAS addendum  
Restart procedure in case the password is forgotten.   
This is what would normally be done with docker programs.  
Show running containers - sudo docker container list   
Stop running container - sudo docker container stop idofcontainer  
Show all containers - sudo docker container list -a  
Remove all stopped containers - dangerous - sudo docker container prune  
However guacamole keeps configuration in a persistence folder in home directory.  
This directory is referenced at docker startup and is at /home/docker/guacamole  
This contains among other things the postgresql database.  
This would need to be erased or renamed to reset passwords.

15. The end of Haiku OS  
Does not boot at all with any options with PC stick.  
Was very unstable with Asus gaming laptop.  
Was fairly unstable with Gateway laptop.  
So is not going to work moving forward as monitor 3.  
Unable to be used in a professional setting until changes are made.

16. An effort to replace all commercial computers  
Commercial computers can be expensive and a liability.  
Custom computers are more flexible with ultimately wider parts availability.  
Administration tablet will be the 6th after router.  
Monitor 2 will be the 5th after administration tablet.  
This leaves 4 commercial computers.  
2 companions are bad candidates because of how much space they save already.  
Media could be replaced by fast USB 3 SBC as AIO or laptop but with high cost.  
Monitor 3 could be replaced by a custom laptop.

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
Datacenter layout document showing overview of hosts/remote admin/website/redundancy  
Switch index document showing ports and roles  
Map document showing host IP and hostname  
WAN info document showing details such as static IP and netmask  
Credentials document showing service login details  
Gateways document that lists internet connections with different info fields  
HAIKU_FIXUP (LEGACY) - miscellaneous script deals with Haiku OS as network monitor  
HAIKU_REMOTE (LEGACY) - handles one line SSH login  
bashrc - workaround screen sizing issue on virtualization server  
tmux.conf - dotfile for customizing tmux status bar  
TMUX_STATUS_HAIKU_BATTERY - script for battery on tmux status bar  
TMUX_STATUS_HAIKU_MEMORY - script for showing memory on tmux status bar  
WEATHER - this miscellaneous script runs the weather on a monitor  
2. Multi WAN  
A reliable network starts with reliable Internet connections.  However, consider
if all network computers truly need backup gateways or only a subset.  This is
another tradeoff between hardware features in network devices and scripting.  
LINUX_MULTI_WAN - several script examples for different Linux computer roles  
FREEBSD_MULTI_WAN - uses the unique FreeBSD fibs system  
/////  
ROUTER_CGNAT - continuation of mid script now streamlined  
ROUTER_CGNAT_HELPER - actuate PI status light and write status message over iftop  
/////  
ROUTER_STATIC - continuation of mid script with several extra features  
ROUTER_STATIC_WORKER - controls port forward switching based on network status  
ROUTER_STATIC_HELPER - chooses program to run in tmux which is iftop in this case  
tmux.conf - displays extra data for cellular routing as dotfile for root folder  
TMUX_STATUS_DATA - backend script for tmux data usage computation  
TMUX_STATUS_GATEWAY - backend script for tmux gateway path computation  
/////  
ROUTER_BTPAN - for emergency routing from an Android phone  
DD_WRT_FIXUP 1 and 2 - handles network segmentation among other things  
rc.local.VIRTUALIZATIONSERVER - auto start multi wan  
rc.local.BACKUPINTERNETSERVER - auto start multi wan and redundancy and status  
rc.local.INTERNETSERVER - auto start multi wan  
rc.local.RAS - auto start simple multi wan and container  
rc.local.REENCODER - auto start Streamlauncher  
rc.local.MONITOR3 - start cctv recorder and inline/transparent firewall  
SMS.py - sends administrative alerts through VOIP.ms to an SMS number  
config.txt.STATIC - low level system configuration for Pi 2  
config.txt.CGNAT - low level system configuration for Pi 2  
ports.conf - configure primary Internet server for administration on port 8080  
SFS_INLINE_FIREWALL - implement transparent firewall for extra security  
3. Streaming  
The result of an effort to obtain a modernized website with multimedia features.
Data and power usage from a modest streaming system does not have to be excessive.
Websites like Youtube are giving creators ever more control of their streams
while simulateneously exerting control on those same creators.  These tools
allow for at home streaming.  
RECORDER - webcam slave script  
STREAMCONFIG - easy to use and informative stream front end  
STREAMLAUNCHER - daemon to handle stream startup and restarts  
Windows VLC shortcut - 2 clicks to start network streaming a windows display  
config.txt.CAMERA - low level system configuration for Pi 3 camera server  
boot.ini.CAMERA - low level system configuration for Odroid C2 camera server  
4. Chat  
There was a need identified for 2 way website communication.  This is useful
for fan feedback during streams or for business messaging or even just a
visitor's sign in area.  The amount of external calls has been kept to a
minimum to keep the potential attack surface low.  Like everything else
presented here this is a no javascript solution.  
CHAT - handler bash script  
CHAT.txt - example text file shown in iframe  
CHAT.html - example html meant as iframe  
index.html.CHAT - example html main page  
ABUSE_TIMESTAMP - helps to prevent spamming  
RESET_TIMESTAMP - helps to prevent spamming and over large files  
5. Upload  
The existing SFS uploader has been moved here to be with the other internet
server relevant information.  Allows for file upload and upload status
monitoring.  This also uses a small amount of external calls and implements a
basic security scheme.  Like everything else presented here this is a no
javascript solution.  
index.html.UPLOAD - example html component that displays data  
UPLOADER.py - python component that handles upload  
HELPER_UPLOADER - bash component that does status updates  
6. Server Redundancy  
Typically CARP is used in the enterprise to enable the takeover and replacement
of one server to another.  This can be crudely simulated with some thoughtful
scripting.  While this does work a solution like this would not scale well to a
large organization.  This scripting involves a complex chain that all needs to
be implemented correctly to work.  
SFS_SERVER_REDUNDANCY_1 - Proxmox hook script that starts pre and post script  
SFS_SERVER_REDUNDANCY_2 - run on virtualization server pre VM boot  
SFS_SERVER_REDUNDANCY_3 - runs on backup internet server  
SFS_SERVER_REDUNDANCY_PERSISTENT - runs on virtualization server post VM boot  
WORKER_1 - launches netcat for redundancy 2 and persistent to backup server  
7. Remote Access Server  
While not strictly required versus forwarding ports this seems like an
interesting alternative.  It allows for things like putting computers with
questionable security status behind another system rather than being directly
on the Internet.  A computer with moderate power is useful here for better remote
desktop smoothness.  Setup Windows hosts for RDP manually.  
ports.conf.RAS - open ports for SSL proxy  
SFS-SSL.conf.RAS - setup SSL proxy  
VNC_SERVER - auto start VNC for Linux host  
config.txt.RAS - low level system configuration for Pi 2  
8. Backup Internet Server  
A more simple non virtualized backup internet server can take over while the main
server undergoes maintenance.  Another idea is to use the backup as a network
status server while the main server is operational.  This server can reside on
a different IP with appropriate forwarding or a more complex redundant
arrangement can be used like the server redundancy scripts.  
index.html.STATUS.EXAMPLE - arranges status page to be displayed in iframe  
STATUS.html.TEMPLATE - displayed in iframe and updated dynamically by network agent  
NETWORK_AGENT - handles the backend status work and updates html  
config.txt.BACKUPINTERNERSERVER - low level system configuration for Pi 2  
9. Staging Server  
Acting as a kind of network scratchpad the staging server is a miscellaneous
bin for files.  It can use minimal security as it is a temporary file holding
point only and is meant for internal use only.  Of chief importance is to
support a server like this one with relevant archival servers.  SAMBA allows
for easy networking and file exchange with clients of different operating
systems.  
smb.conf - samba configuration for easy access and no login dialogs  
config.txt.STAGINGSERVER - low level system configuration for Pi 4  
10. Power Control  
It is advantageous to have fine tuned centralized energy management for a
number of reasons.  The network can be moved to a moderate energy state when
the network administrator is absent.  Additionally a low overall energy state
can be quickly achieved during an adverse power event.  Enables remote mains
voltage monitoring and could be tailored for battery voltage monitoring.  Other
future applications include IOT electric switch control/POE control/WOL control.  
CENTRAL_POWER_ADMINISTRATION - server script running in workstation background  
CENTRAL_POWER_ADMINISTRATION_HELPER - heavy lifting of logging in and shutdown  
LISTENER - example client  
11. Network AI  
Star Trek has inspired the creation of this currently passive "AI".  It only
talks and does not listen.  This voice assistant is tied into phone
communications by way of KDE Connect and tied into the network courtesy of
Etherape.  A basic ethernet hub or a managed switch is a must to use this.  This 
is a continuation of an earlier effort to add a voice assistant to the
MID project.  
SFS_AI - main logic  
AI_MESSAGE_DIALOG - present a dialog for incoming messages of various types  
12. Captcha  
Bots are a threat on the modern Internet and one of the best ways to deal with
them is the unfortunate use of a captcha.  This captcha does not use javascript
or anything running client side.  Has the ability to detect bypassing.  Should
be used as part of a multi-tiered security strategy.  Requires a gateway or
firewalling device to read the blocklist it generates to be useful.  
CAPTCHA - CGI script at the core of the initiative  
CAPTCHA_AGENT - run the CGI script at intervals in case it is not run otherwise  
index.html - example page  
WHITELIST - necessary list for comparison  
BLACKLIST_2 - the list for reading by blocking devices

**Forbidden Tomes**  
There are certain documents that for reasons of security and practicality will
be unique for every network.  

Legend  
/////  
Some listed here will be examples that must be edited with custom credentials.  
These would be a bad loss but a password change away from being fixed.  
These are marked as RESTRICTED.  
/////  
Some examples displayed here will be unique to the network but non-confidential.  
Best kept close to the chest because a network design can not be easily changed.  
These are marked as PROPRIETARY.  
/////  
Some even are so secret that they are not posted at all but merely referenced.  
These are protected through security through obscurity whatever that is worth.  
These are marked as SECRET.

1.Recorder - RESTRICTED  
2.Server redundancy 3 - RESTRICTED  
3.SMS python script - RESTRICTED  
4.Streamconfig - RESTRICTED  
5.Streamlauncher - RESTRICTED  
6.VNC server - RESTRICTED  
7.Server redundancy worker 1 - RESTRICTED  
8.Uploader CGI script - RESTRICTED  
9.Haiku remote SSH login script - RESTRICTED  
10.RAS SSL configuration - RESTRICTED  
11.RAS ports configuration - RESTRICTED  
12.Switch index - PROPRIETARY  
13.Network map - PROPRIETARY  
14.Cable IP document - PROPRIETARY  
15.Numbers and passwords - PROPRIETARY  
16.PFSense configuration XML - SECRET  
17.Unused hostname ideas - SECRET  
18.Captcha CGI script - SECRET

**Fruits of the Quiltmaker**  
This is a list of hosts showing what can be expected with a typical end result.  A product of the 
struggle to get the network under control.  There is enough here to build on and fill the switch
ports further than they already are or stop building and just enjoy what has been created.

1. Devuan Pi 2 network monitor - installed  
TCP mode of Etherape was crashing with Lattepanda but seems to work with Pi 2  
Needs port mirroring turned on in switch and all ports fed to the monitor port  
Internal VNC server started with KDE startup control panel  
Runs Network AI

2. Devuan laptop network monitor 3 - installed  
Shows the weather  
Runs business email  
Hosts phone  
Assists with development  
Does backend 3d models work  
Now directly monitors the CC as an inline/transparent firewall  
/proc/sys/net/bridge/bridge-nf-call-iptables should be 1 for layer 2 to 3 for firewall  
The following describes operations but also shows how to use stunnel  
Runs CCTV stream - install stunnel 4 - move key to /etc/stunnel - create stunnel.conf  
Generate key - openssl req -new -x509 -nodes -out ~/STUNNEL.pem -keyout ~/STUNNEL.pem  
stunnel.conf - client - client = yes/accept=8080/connect=serveraddress:7777  
stunnel.conf - server - accept=7777/connect=127.0.0.1:8080  
Run stunnel4 on server/run stunnel 4 on client/client connects to http://localhost:8080  
Do not forget the http on the front for VLC or it will not connect

3. Devuan reencoder server - virtual machine  
Take h264 HD input OTA with hdhomerun and reencode  
Use simple encoding like MPEG TS for sending from webcam server  
Reencode for web friendly output with mobile support and low bandwidth  
Reencoding will begin automatically at virtual machine startup  
Streamconfig script will allow for turning streams on and off and labeling  
Monitor 2 can run streamconfig over SSH - may be better for kitchen to run Streamconfig  
This allows for clients to deal only with the reencoder and not subsidiary devices  
This also allows for absolute control over the streams  
This also allows for better monitoring of server performance  
theora with ogg on windows slave - causing reencoder transcoding errors  
mp4v with mpeg ts on windows slave - pretty good and also what the webcam slaves use  
h264 with mp4 on windows slave - less network but maybe less reliable  
Required software - SSH/VLC/psmisc

4. Devuan Internet server - virtual machine  
Hosts web and SSH  
At startup signals are sent to to backup internet server to move to parking IP

5. Devuan Pi 2 cellular router/MID - installed  
For router usage with metered static IP business cellular device  
Script type is router static  
Further network specific information is in dual WAN notes - SSH on 1001 same as cable webgui

6. FreeBSD Lattepanda custom convertible tablet backup storage server - installed  
Provides a clone of data on main storage server  
Is physically and logically separated from main storage server  
2 Lattepanda original died during production of this device  
Important to boost incoming 5v to around 5.25v for stability with heavy USB load

7. FreeBSD tablet switch administration device - installed  
Available for real time switch administration with monitoring/stats/QOS/VLAN  
Udoo x86 Ultra SBC with 8GB RAM and internal gigabit ethernet  
Attached to different subnet that is the same as switch  
Has 7" touch screen but is currently limited to landscape orientation  
Windows tablet switch administration device 1 - retired

8. Devuan media laptop - installed  
Attached to Fire Stick with HDMI capture and VLAN  
Connected to home theater

9. Devuan gaming custom laptop - installed  
Takes games on the go  
Cable router administration device

10. Devuan Pi 4 headless staging server - installed  
Holds only data that is yet to be sorted and archived  
No redundancy but the copying computers keep until stored

11. FreedBSD storage server custom laptop - installed  
Holds 2 copies of network files  
Also exports 1 or 2 copies and 1 gen back copies kept

12. FreeBSD workstation custom laptop - installed  
Holds 2 copies of files  
Exports to various secure means  
Does frontend 3d models work  
Core switch administration device  
Runs Central Power Administration

13. Proxmox luggable virtualization server - installed  
With web/SSH - dedicate 2 cores  
With reencoder - dedicate 2 cores  
With development and testing environments - 2 cores left over  
net-tools package needed for Linux Multi WAN  
Cannot setuid root scripts - use sudo locally or /etc/rc.local at boot  
Create /etc/rc.local with exit 0 at end and #!/bin/sh -e at beginning  
Edit linux multi WAN interface names and metric priorities  
Reference Linux Multi WAN with rc.local  
Import VMWare - qm importovf 100 /tmp/exported-vm.ovf local-zfs  
Create bridge and assign IP and gateway of primary interface and gateway  
Create second bridge and assign IP and gateway of primary interface and no gateway  
Assign guests 2 interfaces linked to these bridges  
This means guests need the dual WAN scripts for maximum redundancy  
USB 3 failed after shutdown and restart - unplug/shutdown/restart/replug/restart  
92w at bios - probably near worst case  
64w at proxmox console - probably near idle  
Currently using 5a adjustable power supply with 300w true sine inverter and battery  
Other options - going dc to dc with a picopsu or similar  
Other options - replacing power supply with large switch based adjustable  
Other options - replacing power supply with existing backup and ordering another  
The DC to DC option would need a solar charge controller or large voltmeter for current  
The replacement power supply options both include a screen for showing current

14. Devuan Pi 3 headless camera server - installed  
Pi 4 nearly freezes but fixed with setpci -s 01:00.0 0xD4.B=0x41  
Need a pure HDMI connection for Pi 4 without too many adapters  
Pi 4 not working with multiple webcams as USB 2 bus which is on all ports and is shared  
Pi 3 not working with multiple webcams - slower HD capture than Pi 4 but could use 2 SD  
Odroid c0 works ok - USB hub ports are limited and slightly temperamental  
Pi Zero is too slow and has limited USB ports  
Lattepanda original is not going to work as expected  
Lattepanda Alpha died while preparing for testing  
Development laptop has bandwidth error for 2 streams despite separate busses  
Pi 3 will do printer camera only since odroid c0 was stable with it  
Development laptop will do CCTV only  
View with mplayer - mplayer http://address -fps 15 -demuxer lavf -noaspect  
View with VLC - use GUI which will prompt for password if needed  
Increasing threads will take more cpu time if vlc is overloaded  
Increasing threads will take less cpu time if vlc is underloaded  
Performance can mislead when above 100% usage but more threads always help  
Install v4l-utils and VLC  
Add user to video group

15. Debian Odroid C2 headless camera server - installed  
One Odroid c2 has broken micro USB and requires barrel connector  
Odroid c2 has shared USB 2 bus and thus has bandwidth issues and cannot use 2 cameras  
Odroid c2 will do workspace camera only

16. Devuan Pi 4 headless guacamole remote access server - installed  
Install docker - sudo apt-get install docker.io  
Install one click docker image for guacamole - sudo docker pull jwetzell/guacamole:arm64  
sudo docker run -p 8080:8080 -v /home/docker/guacamole:/config jwetzell/guacamole:arm64  
Before the colon is the host directory and after the colon is the guest folder name  
Much faster to start up and to use gaming computer with arm64  
User and password and security mode any and ignore server certificate for RDP  
Just password and port 5900 for VNC with x11vnc  
Ctrl alt shift for Guacamole menu  
Required x11vnc startup script for monitor 1 and turning on RDP for gaming desktop  
x11vnc script starts with KDE  
Docker is auto started in /etc/rc.local with a small sleep in front  
Open shell in container - docker exec -it container-name sh - debug only  
Login as postgres not guacamole - psql -d guacamole_db -U postgres - debug only  
Create root account in database - CREATE USER root; - debug only  
Docker containers are non persistent by default  
docker run v switch is important for persistence  
Configs and the databases are created here on the host  
Apparently the container scripting looks for the folder name and responds  
This causes settings to save properly and the database to stop being recreated  
Red herring - FATAL: role "root" does not exist  
Solution - dedicate a local VM to viewing VNC and capture with VLC - bad performance and waste of CPU core  
Solution - pipe x11vnc to VLC - piping-vnc-web might work but there is no RDP solution  
Solution - Guacamole uses VNC for same or different connection as remote - probably best  
Not quite sure where we were going with the solutions above but keeping it in there anyway  
Ideally there should be no host interaction required  
There is no option for connection encryption so a proxy must be set up  
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout guacamole-selfsigned.key -out guacamole-selfsigned.crt  
Copy SFS-SSL.conf to sites-available/a2ensite SFS-SSL/a2enmod ssl proxy proxy_http proxy_wstunnel  
SFS-SSL.conf basically references the certificate and key just created and internal site and port being proxied  
Edit ports.conf to add 3001 below 443 for SSL and TLS

17. Windows gaming desktop - installed  
Gaming in a very fixed location  
Acts as wintendo with no web browsers or productivity software installed  
One way is allowing web a Guacamole view only login but that is dangerous and dirty  
Ultimately the idea was to use the RDP server on gaming desktop for streams and remote  
Seems like RDP cannot use simultaneous connections so screen recording is out  
This also means that when guacamole is logged in the user is logged out and the reverse  
RDP can still be used for remote access  
Third party program needs to be installed to view the screen simultaneously on Windows  
Guacamole screen recording seems awkward for live streaming anyway  
Recording records first then would need guacenc to convert the recording to a m4v  
guacenc is not even installed in the container  
Piping VNC - complicated/requires work to automate/same thing but worse than Guacamole  
VLC desktop streaming is the way forward  
Have to transcode or there would be no practical way to transfer such data on network  
h264/ogg/mpeg4 are similar in performance and .25 scale is better speed than 1  
Set commandline shortcut for VLC and place in Windows startup folder  
Reencoder will do final transcoding for mobile friendly to save power and centralize  
vlc screen:// :screen-fps=10.000000 :live-caching=300 :sout=#transcode{vcodec=theo,vb=800,scale=0.25,scodec=none}:http{mux=ogg,dst=:8080/} :no-sout-all :sout-keep
Thus VLC and RDP run on the gaming desktop

18. Devuan convertible laptop companion - installed  
Assists the gaming computer  
Connected to secondary audio system

19. Windows convertible laptop companion - installed  
Assists the media computer  
Another windows computer for general usage

20. Devuan Pi 2 backup internet server - installed  
Idea 1 - carp or something similar would be able to do this also  
Idea 2 - use manual ethernet switch - flawed - primary has no access to cellular  
Idea 3 - start on demand device - flawed - primary has no access to cellular  
Idea 4 - auto determine best IP - flawed - have to be on correct physical network  
Idea 5 - connect each switch with manual switch - flawed - can only forward 1 IP on 80  
Idea 6 - use internetwork messaging to signal backup server to change to parking IP  
Parking IP is used for network status when main server is running  
Server IP is used for server replacement when main server is down  
Show timestamp  
Show current IP  
Show ping to 192.168.5.1(is core up?)  
Show apcaccess  
Set to 192.168.1.125  
ping for 192.168.1.1  - if no 192.168.1.1 then set to 192.168.5.125  
ping for 192.168.5.8  - if no 192.168.5.8 then set 192.168.5.8  
Wait for shutdown signal and set to 192.168.5.125 - best option - use netcat  
ping for 192.168.5.8??? - would not be reliable because we would often ping ourselves  
ping for 192.168.5.9 - detect reencoder VM??? - would not allow work on just reencoder  
If yes 192.168.1.1 then set 192.168.1.125  
ping for 192.168.1.9  - if no 192.168.1.8 then set 192.168.1.8  
Wait for shutdown signal and set to 192.168.1.125 - best option - use netcat  
ping for 192.168.1.8??? - would not be reliable because we would often ping ourselves  
ping for 192.168.1.13 - detect reencoder VM??? - would not allow work on just reencoder  
This is a slightly older pseudocode overview but still generally applies  
Setup for listening with nc instead of pinging to switch back to parking  
Still uses ping for switch to server  
LoadModule cgi_module /usr/lib/apache2/modules/mod_cgi.so - allow scripts- not needed - script independent  
HttpProtocolOptions Unsafe - fix python error handling - not needed - script independent  
ports.conf add Listen 8080 behind Listen 80 - listen on port 8080 also  
Packages - ntp/apache2/apcupsd

21. Pi 2 cellular router/retired MID - installed  
For router usage connected to 1 of 2 unlimited CGNAT phones  
Runs console only for better performance with Pi 3 and easy management  
Script type is router CGNAT  
Connect with USB/turn on USB tethering/dhclient on computer  
LG - presents USB ACM device at /dev/ttyACM0 that uses cdc_ether driver  
Blackberry - same but uses cdc_ncm/cdc_wdm/cdc_mbim  
Addendum - cdc_ncm bind failure with android phone  
Only happens with Blackberry not LG  
Works with old x86 debian live environment  
Does not work on any newer environment  
Switch sim between LG and Blackerry - impractical due to wear and tear  
Use live environment in isolation - impractical due to increasingly older software  
Finding the exact kernel when the failure happens is important for 2 reasons  
For bug fixing reasons  
The file will hopefully be new enough to install on a newer OS  
2015-02-02 release date with kernel 3.18 works correctly - too old for packages  
2016-2-29 release date with kernel 4.1 fails with bind error - too old for packages  
2020-2-14 release date with kernel 4.19 fails with bind error - packages work  
Thus the problem started with kernel 4.1 at least for the raspberry Pi  
The case of modifying Devuan Chimaera pi 2 32 bit with old kernel  
Just copying old kernels to new Pi 2 image does not boot  
The entire boot folder was copied over including kernels  
Now the old kernel boots on new Devuan on Pi 2 but no kernel modules available  
/lib/modules needs to be copied over  
Not working on pi 3 which makes sense because there is no kernel support at that time  
Pi 3 router replaced with Pi 2 router

22. FreeBSD AIO network monitor 2 - installed  
Acts as status board or "master systems display"  
Terryza PC Stick Z8350 4GB RAM USB 3 gigabit ethernet 32" 1080p display  
Monitor 2 visualizations as follows:  
Terminal with iftop  
Virtualization server web page  
News stream from HDHomerun  
Status HTML page  
Haiku OS laptop network monitor 2 - retired  
Runs Proxmox web interface - console login through web does not work - use SSH  
Falkon is available but is worse than the system default Webpositive browser  
Proxmox console can be minimized and interface set for 3 columns for better view  
pkgman full-sync updates system  
pkgman install falkon installs Falkon for instance  
qemu is actually available from haiku depot if a browser is really needed  
Troubleshooting Haiku  
Current problem is freeze of all input with OS still working  
All except safe mode with hda disabled - instant lock at VM server  
All except safe mode and apm with hda disabled - instant lock at VM server  
All except safe mode and apm/acpi with hda disabled - instant lock at VM server  
Only apm/acpi/hda disabled - works ok initially  
All except safe mode and don't call with hda disabled - instant lock at VM server  
All except safe mode and disable local apic with hda disabled - works ok initially  
Did crash after several days - trying the same without local io apic or whatever  
Was forced to replace old Asus laptop with old Gateway Laptop that works better  
There are still issues with memory leaks in the Webpositive browser however

23. DD WRT core switch  
Has gigabit switch integrated  
Could in theory be used as a wireless access point also

24. Pfsense cable router - installed  
The cable gateway and main Internet connection  
Custom luggable laptop with no battery but still has a basic UPS  
As noted below an extensive UPS is not needed for hardline connections  
Hardline connections can and will fail when the power goes out  
Samsung laptop - retired  
Set for relatively short battery life due to unreliability of hard lines in outages

25. MID  
The Mobile Internet Device has emergency Internet sharing capability  
SIM cards for alternative MIDs are in use elsewhere

26. Fire Stick  
Quarantined by VLAN  
Lives in II Alpha and is connected by HDMI to media computer

27. HDHomerun  
Reliably takes ATSC OTA and displays it in a convenient webpage  
Also works pretty well interfacing with VLC

28. HP Color Laser Printer  
This has been working well for over a decade  
Has only needed a toner replacement one time and was not expensive

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

