---
title:  "My Bootstrappr Workflow"
author: gmarnin
excerpt: "I found a use for chroot"
tags:
  - bootstrappr
  - imaging
  - munki
  - netboot
---

### See Ya NetBoot

With Apple's introduction of the [T2](https://support.apple.com/en-us/HT208862) processor, a long standing tool in many MacAdmin's toolkit has taken a big hit. The T2 introduces [Secure Boot](https://support.apple.com/en-us/HT208330) which renders Macs with the chip unable to be NetBoot. Although Secure Boot can be disabled, it is non trivial to do so and clearly non advisable. In summary, Apple considers NetBoot a security risk and NetBoot is no longer a viable option going forward. Apple's preferred NetBoot replacement is DEP and MDM. For now, my preferred replacement is [Bootstrappr](https://github.com/munki/bootstrappr) from [Greg Neagle](https://managingosx.wordpress.com). 

Before the T2, I was NetBooting via [BSDPy](https://github.com/bruienne/bsdpy/tree/windows) into [Imagr](https://github.com/grahamgilbert/imagr). Imagr was running a simple workflow that took a new Mac, named it and created a new [Munki](https://github.com/munki/munki) manifest based off a template using a forked version of a [Munki-Enroll](https://github.com/gmarnin/munki-enroll). Finally, Imagr installed a few setup packages to boot the Mac into Munki's bootstrap mode and the rest is history. This workflow has served me well for a for a few years. 

Naming the Mac is an important step in our imaging workflow. I know some environments don't care what the Mac is named, but we do. We have stable hostnames and use the name to key off for many of our other systems. The name codifies the physical location of the Mac on our sprawling campus. We also use it for Active Directory binding, as well as the identifier for the Munki manifest name and other stuff too. 

### Why Bootstrappr? 
Bootstrappr is a bare-bones tool to install a set of packages and scripts on a target volume. So why use it over DEP and MDM? In short, because it is quick, easy, non-fragile and it just works. I can still keep the entire imaging process on our internal network which also means I still control the whole process. Bootstrappr can replace NetBoot and my Imagr workflow. I simply feed Bootstrappr the same packages Imagr was using and I'm done. Well, done until it came time to give the Mac its name before the Munki bootstrap mode takes over. Out of the box, Bootstrappr has no support for naming a Mac. 

### My Workflow
Bootstrappr is a simple bash script because it is made to run in Recovery mode where a lot of other scripting languages and binaries are not available. With an idea I got from [Vaughn Miller](http://www.vaughnemiller.com) at the hallway track at [PSU MacAdmins Conference](https://macadmins.psu.edu), I could modify the bootstrappr.sh script to ask for the computer name and write it to a file on the boot drive. For me, that addition looks like this:

```
# Name the Mac. Works in conjunction with the 1_bootstrappr_macname_munki_enroll-1.0.pkg to set the name and generate a munki manifest before munki runs. 
echo
echo "Please give the Mac a name:"
read computer_name
/usr/bin/touch "/Volumes/${SELECTEDVOLUME}"/Users/Shared/computer_name.txt
echo $computer_name > "/Volumes/${SELECTEDVOLUME}"/Users/Shared/computer_name.txt
```

Vaughn was using [Outset](https://github.com/chilcote/outset) to run a script on first boot to read the file that Bootstrappr set. While I love me some Outset, I didn't want it as dependency in this workflow. I was looking to replace my Imagr workflow all within my Bootstrappr workflow. With a hat tip to [Mike Lynn](https://twitter.com/mikeymikey), I started looking at `/usr/sbin/chroot` to allow me to run `scutil` on the Macintosh HD volume from Recovery mode.

I created a package named `1_bootstrappr_macname_munki_enroll-1.0.pkg` which Bootstrappr [runs first](https://github.com/munki/bootstrappr#order). The payload drops `bootstrappr_macname_munki_enroll.sh` in `/Users/Shared/` and a `postinstall` which executes it. 

`bootstrappr_macname_munki_enroll.sh` reads the `computer_name.txt` file which contains the name, sets the name and generates the Munki manifest using `Munki-Enroll`. A perfect hat trick!

`/Users/Shared/bootstrappr_macname_munki_enroll.sh`:

```
#!/bin/sh

# This script is part of our bootstrappr workflow. It sets the Mac name, and gens a munki manifest based on munki-enroll.

ComputerName=`cat /Volumes/Macintosh\ HD/Users/Shared/computer_name.txt`
Serial=`/usr/sbin/ioreg -l | grep IOPlatformSerialNumber | cut -d'"' -f4`

echo "ComputerName is $ComputerName"
echo "Serial is $Serial"

# Name the Mac 
/usr/sbin/scutil --set ComputerName "$ComputerName"
/usr/sbin/scutil --set HostName "$ComputerName"
/usr/sbin/scutil --set LocalHostName "$ComputerName"

echo "The Mac has been named to $ComputerName"

# mDNSResponder is not available in the chroot'ed environment and so all DNS lookups will fail. Using the ip address as workaround.
# Using -k option, (--insecure) because ssl cert is based on the DNS name. Doesn't work without -k
SUBMITURL="https://ip.address/munki/munki-enroll/enroll.php"

	/usr/bin/curl -vv -k -H 'Authorization:Basic KeyGoesHere' --max-time 5 --silent --get \
	-d computername="$ComputerName" \
	-d serial="$Serial" \
	"$SUBMITURL"
```

The `postinstall` script uses `/usr/sbin/chroot` and cleans up:

```
#!/bin/sh

# Macintosh HD is hard coded because new Macs have shipped with that HD name since forever
/usr/sbin/chroot /Volumes/Macintosh\ HD /Users/Shared/bootstrappr_macname_munki_enroll.sh

/bin/sleep 10

# Clean up
/bin/rm /Volumes/Macintosh\ HD/Users/Shared/computer_name.txt
/bin/rm /Volumes/Macintosh\ HD/Users/Shared/bootstrappr_macname_munki_enroll.sh
```

In testing, I found this works well in 10.13 and on 10.14 beta 4. In 10.12, `curl` throws an error that I didn't investigate. 10.12 isn't really a concern for me because Apple isn't shipping anymore 10.12 Macs except for maybe the Mac Mini.

The other packages Bootstrappr installs are (in order):

```
1_bootstrappr_macname_munki_enroll-1.0.pkg
2_create_local_admin-1.1.pkg
3_munki-config-1.1.pkg
4_munkitools-3.3.1.3537.pkg
5_munki-bootstrap-1.0.pkg
6_SuppressSetupAssistant-1.0.pkg
7_disable-login-popups-1.3.pkg
```

### Update

After imaging several dozen Macs using Bootstrappr with the chroot method described above, I ultimately found the chroot method of naming the Mac unreliable. 
I haven't been able to pin down why some Macs get named and some don't. I needed a more consistent workflow and so I changed things up again. I'm still having Bootstrappr
ask for and write the computer name to `/Users/Shared/computer_name.txt` because that always works. I stopped using `chroot` and moved the naming of the Mac to
to Munki, using a `postinstall_script` in a `nopkg` that is in my `site_default` manifest:

```
 <key>installcheck_script</key>
    <string>#!/bin/sh

# Check if computer_name file exists. Abort if it doesn't exist. 

if [ ! -f /Users/Shared/computer_name.txt ]; then
    echo "/Users/Shared/computer_name.txt not found!"
    exit 1
fi

    </string>
    <key>postinstall_script</key>
	<string>#!/bin/bash

# This script is part of our bootstrappr workflow. It sets the Mac name, and gens a munki manifest based on munki-enroll.

ComputerName=`cat /Volumes/Macintosh\ HD/Users/Shared/computer_name.txt`
Serial=`/usr/sbin/ioreg -l | grep IOPlatformSerialNumber | cut -d'"' -f4

# Name the Mac
/usr/sbin/scutil --set ComputerName "$ComputerName"
/usr/sbin/scutil --set HostName "$ComputerName"
/usr/sbin/scutil --set LocalHostName "$ComputerName"

SUBMITURL="https://ip.address/munki/munki-enroll/enroll.php"

	/usr/bin/curl -vv -k -H 'Authorization:Basic KeyGoesHere' --max-time 5 --silent --get \
	-d computername="$ComputerName" \
	-d serial="$Serial" \
	"$SUBMITURL"

# Remove the file computer_name.txt bootstrappr file
/bin/rm /Users/Shared/computer_name.txt

exit 0
    </string>
```

I hope you find this workflow useful or at least gives you an idea of what Bootstrappr can do. If you have a similar workflow that improves on mine, please let me know. 

Happy imaging! 