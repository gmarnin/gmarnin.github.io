---
title:  "Querying CDP from a Mac"
author: gmarnin
excerpt: "I have a Fluke but"
tags:
  - cisco
  - cdp
  - network
---


As part of our network security plan, we manage all the ethernet ports in the spaces that we support. We have some long standing policies around ethernet ports such as only allowing managed desktops access, binding the MAC address to the port and disabling ports not in use. These policies address various issues such as only allowing authorized devices on our network, preventing desktops from being moving around and preventing unauthorized devices from being connected on our network. Desktops have their Wi-Fi interface [disabled via a configuration profile](https://github.com/gmarnin/Profiles/blob/master/Disable-Wifi-1.0.mobileconfig) installed via a [Munki Conditional Item](https://github.com/munki/munki/wiki/Conditional-Items). Mobile devices can remote into our network via vpn but ethernet is off limits.

The practical impact of these policies is that everytime we setup a new desktop or move devices around we have to configure or reconfigure the ethernet ports too. We have several expensive [Fluke](http://www.flukenetworks.com) devices to query our Cisco switches for [Cisco Discovery Protocol aka CDP](http://www.cisco.com/c/en/us/td/docs/ios/12_2/configfun/configuration/guide/ffun_c/fcf015.html)
which provides the info needed by the networking team. Using the Fluke method works well but has some annoying downsides. Namely, You have to remember to take the Fluke with you when doing fieldwork. We support offices all over town and if you forget the Fluke while across town the job is over before it begins. Worse, sometimes the Flukes don't get returned to their shelf when not in use, get left on desk in someones office or are all in use when I need one. Dead batteries happen too. These are all are common scenarios that are very inconvenient when work has to get done.  

Of course using the Fluke isn't the only way to query Cisco switches for the relevant information. While researching alternate methods, I found [whichSwitch](http://www.computernetworkbasics.com/whichswitch/) which is an gui application that displays CDP information. It works great but has to be installed on every Mac and only runs via the gui. whichSwitch uses [tshark](https://www.wireshark.org/docs/man-pages/tshark.html), a terminal based version of Wireshark, to grab CDP.  [tshark can be used](https://wiki.wireshark.org/CDP) without whichSwitch but it's a dependency that I don't want to manage. The installations can all be automated but is not ideal for my use case. Too much work.

There is a better way. We can capture CDP information without a Fluke, gui app or third party dependencies. I'm not the first person on to post this command but I sure do use it enough  warrant posting it again. A carefully crafted tcpdump command can do it:

`/usr/sbin/tcpdump -nn -v -i en0 -s 1500 -c 1 'ether[20:2] == 0x2000'`

To understand what's going on with all those switches:

- `tcpdump` - Dump the traffic on a network
- `-nn`- Don't convert any addresses
- `-v` - Verbose output
- `-i` - Interface name
- `-s` - Capture 1500 bytes of data from each packet rather than the default of 65535 bytes. Less data = faster run.
- `-c` - Count 1 packet (in this case) and exit
- `ether[20:2] == 0x2000` Check bytes 20 and 21 from the start of the ethernet header for a value of 2000 (hex)

I have posted my [CDP switch script](https://github.com/gmarnin/Mac-Scripts/blob/master/switch_script.sh) for all to use. Along with the above tcpdump command, the script also finds the MAC, IP and link speed for the active ethernet interface. CDP can show some interesting information not easily natively obtainable from the Mac which could come in handy when troubleshooting or documenting your network setup. Examples are which switch port the Mac is plugged into and the vlan the Mac is on. The name of the switch, ip address and physical location are also available. The switch model and duplex setting are returned but are not shown in my script output. I simply don't need that information. To view the raw CDP information returned run `cat /tmp/network.txt`. 

Sample output from the script:
```
$ sudo /Library/ITS/switch_script.sh
Password:

If the Ethernet interface is active, this script can take up to 30 seconds to run.
Still beats using a Fluke in the field. Please be patient.

tcpdump: listening on en0, link-type EN10MB (Ethernet), capture size 1500 bytes
1 packet captured
1495 packets received by filter
0 packets dropped by kernel

Mac Results:
Computer Name: mac02.domain.com
Mac IP (en0) = 172.X.X.X
Mac, MAC Address = 68:5b:35:XX:XX:XX
Link Speed: 1000baseT

Switch Results:
Switch Port = GigabitEthernet1/0/32
Vlan = 120
Switch Location = Basement-Building-3
Switch Name = b01-foo-bar
Switch IP = 172.X.X.X
```
The main advantage is I never forget the script. I always know where it is. If I need to run the script again, I don't have to visit the port in question. I can remote into the Mac and run it. I can then copy the results and send it off to networking. Bonus points for not having any hardware or software dependencies.

In some cases, usually to further network security, CDP might be turned off on the port you are querying. An indication of this is if the script doesn't return any results after 60 seconds.
