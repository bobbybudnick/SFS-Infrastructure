# SFS-Infrastructure
The Software Freedom Solutions Internet facing server infrastructure

Tenets of Resilient Ethernet Micro Networks
-----

**Laying the Fabric**  
This is the basic groundwork for the network.  It was clear early on that
mutiple Internet gateways were required for high availability. Personal
experience has also shown that during digital and real world disasters it is
advantageous to have more than one Internet connection.

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
Switch layout document showing ports and roles  
Map document showing host IP and hostname  
WAN info document such as static IP and netmask  
Credentials document with service login details
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
Index.html - html component that displays data  
UPLOADER.py - python component that handles upload  
HELPER_UPLOADER - bash component that does status updates  
6. Server Redundancy  
Typically CARP is used in the enterprise to enable the takeover and replacement
of one server to another.  This can be crudely simulated with some thoughtful
scripting.  While this does work a solution like this would not scale well to a
large organization.  This scripting involves a complex chain that all needs to
be implemented correctly to work.  
SFS server redundancy 1 - Proxmox hook script that starts pre and post script  
SFS server redundancy 2 - run on virtualization server pre vm boot  
SFS server redundancy 3 - runs on backup internet server  
Persistent - runs on virtualization server post vm boot  
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
config.txt.STAGINGSERVER - raspberry pi specific power saving boot options  
10. Power Control  
It is advantageous to have fine tuned centralized energy management for a
number of reasons.  The network can be moved to a moderate energy state when
the network administrator is absent.  Additionally a low overall energy state
can be quickly achieved during an adverse power event.  Enables remote mains
voltage monitoring and could be tailored for battery voltage monitoring.  Other
future applications include IOT electric switch control/POE control/WOL control.  
Central power administration - server script running in workstation background  
Listener - example client

**Forbidden Tomes**  
There are certain documents that for reasons of security and practicality will
be unique for every network.  Some listed here will be sanitized examples that
absolutely must be edited with custom credentials.  Some examples displayed here
will be unique to the network but non-confidential.  
1.Recorder  
2.Server redundancy 3  
3.SMS  
4.Streamconfig  
5.Streamlauncher  
6.VNC server  
7.Worker 1  
Remember to edit the passcodes in these files.

**Fruits of the Quiltmaker**  
This is a list of what can be expected with a typical end result.  A product of
the struggle to get the network under control.  There is enough here to build
on and fill the switch ports further than they already are or stop building and
just enjoy what has been created.

