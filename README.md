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

4. prevent routing through the cellular network  
This is for outbound.  
It should not route through cellular by default but instead use cable default gateway.  
Fundamentally the problem is one of needing multiple gateways.  
A weight needs to be assigned to the gateways favoring the cable connection.  
To prevent inbound routing see cellular bandwidth and response solution.

5. Adequacy of round robin  
Apparently modern browsers work with this pretty well.  
There is misinformation but it should be fine compared to load balancing solutions.  
Load balancing solutions are a single point of failure themselves if improperly setup.

6. Seamless failover capability in theory  
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

**Troubles in the Tapestry**  
This is a catalog of glitches encountered during development.  Some with
networking equipment and some with hosts.  By far the longest time was spent
stuck on these problems so bypassing these will speed up the production of any
networking project.

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
???  
???  
???

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

