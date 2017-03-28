---
title:  "Schedule Restart Configuration Profile"
author: gmarnin
excerpt: "radar or it never happened"
tags:
  - Profile Manager
  - Configuration Profile
---


I had a need to reboot a small lab of student use iMacs every night. My first thought was to use a configuration profile as I try to use them whenever I need to manage a setting. This is the preferred Apple way®️ of managing macOS settings. I don't have an MDM in production and I don't manually create configuration profiles by hand. Instead, I run Apple's Profile Manager inside a VM. I fire it up whenever I need to generate a profile as it does most of the work for me. I then download the profile, run `plutil -convert xml1 /path/to/foo.mobileconfig` to fix the formatting and make minor edits in my text editor if needed. I use [Munki](https://github.com/munki/munki/wiki/Managing-Configuration-Profiles) to install the profile.

So in Profiler Manager Server 5.1.5, I went looking for the setting to schedule reboot. To my surprise, I didn't find it anywhere. In the Energy Saver section, I have the option to sleep or shutdown the Mac on a schedule but no option to restart. In the Energy Saver preference pane in macOS there is a option to restart on a schedule but the option to do so is missing from Profile Manager. It didn't make sense to me why the restart option was missing from Profile Manager.

![No restart option in Profile Manager](https://github.com/gmarnin/gmarnin.github.io/blob/master/images/profile_manager_no_restart_option.png)

To work around this limitation, and because I only had a small lab of iMacs that needed this setting, I could of manually configured each Mac by hand. But I know that won't scale should my need to schedule restart on more Macs grow.

My solution was to configure one Mac with the restart settings that I needed and then, in a moment of #macadminshame, copy off the `/Library/Preferences/SystemConfiguration/com.apple.AutoWake.plist` the Mac created and use [munki-pkg](https://github.com/munki/munki-pkg) to package up the file for Munki to install.

I also filed radar 26844969 requesting that the Profile Manager Energy Saver section match what is available on the Mac. Specifically, I requested the ability to create a restart configuration profile to match the settings that are available in macOS.

Six months later, Apple responded to my radar and asked me to test the latest beta of Server. I cloned my non production Profiler Manager Server VM, updated it to the latest Server beta 5.2, and retested. I'm happy to say that Apple added the ability to create a scheduled restart configuration profile.

![Restart option in Profile Manager]({{ site.url }}{{ site.baseurl }}/images/profile_manager_restart_option.png)

My take away here is simple: If you think filing radars is a waste of time you're wrong. It really works. It may not be a quick process and you may not always get the answer you want but Apple hears you. I encourage you to file early and often.

Sample configuration profile to automatically restart a Mac every weekday at 2am:

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>PayloadContent</key>
	<array>
		<dict>
			<key>PayloadDisplayName</key>
			<string>Energy Saver</string>
			<key>PayloadEnabled</key>
			<true/>
			<key>PayloadIdentifier</key>
			<string>com.apple.mdm.domain.edu.339b00a0-162b-0134-2637-000c29bcc5f4.alacarte.energysaver.0fc3fd60-162d-0134-2638-000c29bcc5f4</string>
			<key>PayloadType</key>
			<string>com.apple.MCX</string>
			<key>PayloadUUID</key>
			<string>0fc3fd60-162d-0134-2638-000c29bcc5f4</string>
			<key>PayloadVersion</key>
			<integer>1</integer>
			<key>com.apple.EnergySaver.desktop.Schedule</key>
			<dict>
				<key>RepeatingPowerOff</key>
				<dict>
					<key>eventtype</key>
					<string>restart</string>
					<key>time</key>
					<integer>120</integer>
					<key>weekdays</key>
					<integer>127</integer>
				</dict>
			</dict>
		</dict>
	</array>
	<key>PayloadDisplayName</key>
	<string>Schedule Restart Overnight</string>
	<key>PayloadIdentifier</key>
	<string>com.apple.mdm.domain.edu.339b00a0-162b-0134-2637-000c29bcc5f4.alacarte</string>
	<key>PayloadOrganization</key>
	<string>ITS</string>
	<key>PayloadRemovalDisallowed</key>
	<false/>
	<key>PayloadScope</key>
	<string>System</string>
	<key>PayloadType</key>
	<string>Configuration</string>
	<key>PayloadUUID</key>
	<string>339b00a0-162b-0134-2637-000c29bcc5f4</string>
	<key>PayloadVersion</key>
	<integer>1</integer>
</dict>
</plist>
```
