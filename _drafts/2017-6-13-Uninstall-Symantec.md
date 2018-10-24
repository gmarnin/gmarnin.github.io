---
title:  "Uninstalling Symantec with Munki"
author: gmarnin
excerpt: "Your Mac will thank you"
tags:
  - symantec
  - munki
---


Like many organizations we install anti virus on our Macs because of some policy that states to do so. We all know all it really does it eat processor cycles. In our case Symantec Endpoint Protection (SEP) was the software chosen to "protect" us. That was until Symantec decided to change up their academic pricing model and, at our scale, it no longer became affordable.

Practically speaking that means that we need to uninstall Symantec Endpoint Protection from all our PCs and Macs. What is replacing SEP has yet to be determined but while I wait for the replacement I started to figure out how best to remove SEP.

We install SEP using Symantec's [Create Remote Deployment Package tool](https://support.symantec.com/en_US/article.HOWTO92266.html). Even though that KB saids `Starting with Symantec Endpoint Protection (SEP) 12.1.4, you can export and configure a Macintosh client installation package for deployment with third-party software distribution programs Apple Remote Desktop or Casper.` Not surprisingly Munki isn't mentioned nut it works too. Since I use Munki to install SEP my first thought was to have Munki use the same package to [uninstall it](https://github.com/munki/munki/wiki/Pkginfo-Files#uninstallableuninstall_method). In my testing `SEPRemote.pkg` package doesn't uninstall anything. Not surprised, that would of been too easy.

My Google foo lead me to [Symantec's massive uninstall script](https://support.symantec.com/en_US/article.TECH103489.html) currently on version 7.0.60. ![My initial thought when I opened the script:]({{ site.url }}{{ site.baseurl }}/images/symantec_uninstall_script_tweet.png) #Really. I manually ran the script and it works but the obvious problem is that the script is only works when logged in as it interactive and expects someone to answer questions. That's not going to scale. As it turns out David Nelson @dmnelson answered my call or help. He also had this problem and was able to modify an slightly older version of the script to suppress the interactions. He shared this modification with me which saved me lots of time. I made the [changes](link to just the changes, or post them below), [imported the script into munki](https://github.com/munki/munki/wiki/Pre-And-Postinstall-Scripts) to correctly format it for XML. Ran makecatalogs and tested it at the LoginWindow. All good.



[Link to the modified script](gist.github.com)
