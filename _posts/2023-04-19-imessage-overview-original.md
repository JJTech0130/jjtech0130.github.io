---
title: "iMessage Overview (Original)"
category: "reverse-engineering"
tags: iMessage Apple
hidden: true
---
My project right now is reverse engineering iMessage. And I don't mean like a half usable POC that only works with Macs. I mean the real thing: a fully open demo that can run on any computer.

Sounds far-fetched? It really isn't. When you get down to it, iMessage isn't that complex. I should be able to get it working soon! *knocks on wood*

First and formost, here's the [repo where I'm experimenting](https://github.com/JJTech0130/pypush) and the [Hack Different Discord](https://discord.gg/hackdifferent) where I'm working on this in real-time.

### Prior Art
Everyone knows what iMessage is, but not many know how it works.
A few people have done reasearch on this before, and I've heavily borrowed from them.
Here's an inexhaustive list:
+ [IMFreedom Knowledge Base](https://kb.imfreedom.org/protocols/imessage/)
+ [M. Frister](https://github.com/mfrister/pushproxy)
+ [Nicolás](https://gitlab.com/nicolas17/apns-dissector)
+ [QuarkSlab](https://blog.quarkslab.com/imessage-privacy.html)
+ [Garman et al.](https://www.usenix.org/system/files/conference/usenixsecurity16/sec16_paper_garman.pdf)
+ [NowSecure](https://www.nowsecure.com/blog/2021/01/27/reverse-engineering-imessage-leveraging-the-hardware-to-protect-the-software/)
+ [Elcomsoft](https://blog.elcomsoft.com/2018/11/imessage-security-encryption-and-attachments/)
+ [Eric Rabil](https://github.com/open-imcore)
+ [The Apple Wiki](https://theapplewiki.com/wiki/Apple_Push_Notification_Service)
+ There's probably others too. If I forgot you, leave a comment.

## Construction
So, the iMessage protocol. Here's what happens when a device is first setting up:
1. The device asks Albert for a "push certificate". Basically, it uses a key obfuscated in the binary itself to prove to Albert that this is "legitimate Apple software". [I defeated this about 2 weeks ago](https://gist.github.com/JJTech0130/647705a968fe0f9d1633c32a6c5a8c8d).
2. The device connects to Apple Push Notification Service using the aformentioned push certificate. It receieves a "push token" which allows notifications to be routed to it.
3. The device uses Grand Slam Authentication to authenticate the user's Apple ID and recieves a "Password Equivalent Token (PET)". [I wrote something for this last year](https://github.com/JJTech0130/grandslam).
4. The device makes a request to the iCloud Setup server to exchange the PET for an IDS "authentication token". 
5. The device uses the IDS "authentication token" to recieve an "authentication certificate" for the user's account.
6. The device sends a registration request to IDS (signed by both the "authentication certificate" and the "push certificate") containing its new public keys and other public information. In exchange, it recieves an "IDS certificate" which it can then use to perform lookup requests. Additionally, this information is public and can be looked up by other users.


This is a new way of writing blog posts, where I slowly add to them over time. Hopefully it means that even if I get distracted, it's still useful for others.

