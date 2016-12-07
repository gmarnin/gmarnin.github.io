---
title:  "When AutoPkg Meets a Munki Catalog Error"
author: gmarnin
excerpt: "Munki catalog error caused autopkg munkimport processor error"
tags:
  - munki
  - autopkg
---


I auto run a list of 25 or so [AutoPkg](https://github.com/autopkg/autopkg) recipes twice a day and, if the run finds an update or throws any errors, I have the results emailed to me for review.

I recently updated to AutoPkg 1.0.0. The major feature of 1.0.0 is the [Parent Trust Info](https://github.com/autopkg/autopkg/wiki/AutoPkg-and-recipe-parent-trust-info)  which alerts you to changes made in a parent recipes. I set this up by running `autopkg update-trust-info` against all my recipe overrides. Then I checked all the recipe overrides with `autopkg verify-trust-info` which came back with `OK` for each override. I then ran my recipe list and I got this error for every recipe:

```Error Domain=NSCocoaErrorDomain Code=3840 "Encountered unexpected EOF" UserInfo={NSDebugDescription=Encountered unexpected EOF,
kCFPropertyListOldStyleParsingError=Error Domain=NSCocoaErrorDomain Code=3840 "Malformed data byte group at line 1; invalid hex"
UserInfo={NSDebugDescription=Malformed data byte group at line 1; invalid hex}}
Failed.```

At first I thought I had an AutoPkg trust error because that is all that changed in AutoPkg. To rule out AutoPkg 1.0.0, I downgraded to AutoPkg 0.6.1 and ran my recipe list again. Same error.

I then tested a recipe I never used before: Zoom.munki. I made an override for it, like I do with all my recipes but sans the parent trust info, and ran it:

```The following recipes failed:
    Zoom.munki
        Error in local.munki.Zoom: Processor: MunkiImporter: Error: Error Domain=NSCocoaErrorDomain Code=3840 "Encountered unexpected EOF" UserInfo={NSDebugDescription=Encountered unexpected EOF, kCFPropertyListOldStyleParsingError=Error Domain=NSCocoaErrorDomain Code=3840 "Malformed data byte group at line 1; invalid hex" UserInfo={NSDebugDescription=Malformed data byte group at line 1; invalid hex}}```

Not exactly the same error as before. After closer inspection, it's the MunkiImporter Processor that's at fault. I haven't made any recent changes that would cause MunkiImporter Processor to balk so what is going here? I wasn't sure so I asked [Greg](https://twitter.com/gregneagle) who suspected that I have an invalid plist somewhere in my munki repo. He was right:

`plutil -lint /Volumes/server/munki/catalogs/*`

```/Volumes/macupdates/munki/catalogs/all: Encountered unexpected EOF
/Volumes/macupdates/munki/catalogs/production: OK
/Volumes/macupdates/munki/catalogs/testing: OK```

That surprised me. Not sure how that happen but the fix is an easy one: Run `makecatalogs` which came back clean. I ran my recipe list again and everything was normal.

Curious, I checked for client errors in [munkireport-php](https://github.com/munkireport/munkireport-php) to see how many of my Macs where failing because of this catalog error. Zero. Why? Clients don't use the `all` catalog as [documented on the Munki wiki](https://github.com/munki/munki/wiki/Using-Munki-Catalogs#catalog-naming)

Lesson learned.
