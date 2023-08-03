---
title: "iMessage, explained"
category: "reverse-engineering"
tags: iMessage Apple
redirect_from:
  - /reverse-engineering/imessage-overview/
  - /imessage
---

This blog post is going to be a cursory overview of the internals iMessage, as I've discovered during my work on [`pypush`](https://github.com/JJTech0130/pypush), an open source project that reimplements iMessage.

I gloss over specific technical details in the pursuit of brevity and clarity. If you would like to see how things are specifically implemented, check out the [`pypush` repository](https://github.com/JJTech0130/pypush) as I mentioned above. It's a pretty cool project, if I do say so myself. Make sure to check it out!

If you still end up with any questions, feel free to ask me in the [`pypush` Discord](https://discord.gg/BVvNukmfTC)

### the foundational layer
One of the most foundational components of iMessage is Apple Push Notification Service (APNs). You might have encountered this before, as it is the *same service* that is used by applications on the App Store to receive realtime notifications and updates, even while the app is closed.

However, what you probably *didn't* know about APNs is that it is *bidirectional*. That's right, APNs can be used to send push notifications as well as receive them. You can probably already tell where this is going, right?

Internally, after a device connects to APNs it will receive a "push token". This token can be used to route notifications to that specific device.

**Note:** This token is technically different then the token you receive when using the `application:didRegisterForRemoteNotificationsWithDeviceToken:` API. That token is scoped for per-app use, and is requested using the bundle ID of the application. However, it is basically used for the same purpose.
{: .notice--info}

When sending push notifications to a device, you also need to specify the *topic* a message is for. This usually looks like a Bundle ID, and for iMessage it's `com.apple.madrid`. When a device connects to APNs, it sends a filter message instructing the server on which messages it wants delivered to it.

**Note:** The APNs server is also known as the APNs *Courier*. The filter message includes several lists of topics, for each of the different possible states. It may want a topic to be `enabled`, `opertunistic`, `paused`, or `disabled`
{: .notice--info}

APNs is not only used for the actual message delivery part of iMessage. Using a pseudo-HTTP layer on top of APNs, IDS (which will be explained in a moment) can send queries and receive responses over APNs as well.

One tricky note that I will mention is that in order to connect to APNs, you need a client certificate issued by the Albert activation server.

### the keyserver
The next piece of this puzzle is IDS. As far as I can figure out, this stands for IDentity Services, though I don't think there is any official confirmation on that.

**Note:** You may also see it referred to as ESS. This is confusing because the APNs topic *FaceTime* uses is specifically called `com.apple.ess`. Moving on...
{: .notice--info}

IDS is used as a keyserver for iMessage, as well as a few other services like FaceTime. Remember, iMessage is E2E encrypted, so the public keys of each participant must be exchanged securely.

The first step in registering for IDS is getting an authentication token. This requires giving the API your Apple ID Username and Password.

**Note:** As 2FA is now standard, it had to be retrofitted into the IDS API. There are 2 options for this: the legacy option, in which a 2FA code is directly appended to the password, and the "GrandSlam" option. In the GrandSlam option, "Anisette data" is used to prove you are the same device and thus do not need to enter the 2FA code again. You then receive a Password Equivalent Token (PET) which can be used as if it was the password + 2FA code.
{: .notice--info}

After one has gotten an authentication token, it must be immediately exchanged for a longer lived authentication certificate. This certificate allows registering with IDS, but it is not yet enough to perform key lookups.

Perhaps the most important step of the IDS setup process is *registration*. This is where public encryption and signing keys are uploaded to the keyserver, as well as various other "client data" about what features the device supports.

When making an IDS registration request, a binary blob called "validation data" is required. This is essentially Apple's verification mechanism to make sure that non-Apple devices cannot use iMessage.

**Warning:** In order to generate the "validation data", pieces of information about the device such as its serial number, model, and disk UUID are used. This means that not all validation data can be treated equivalently: just like with Hackintoshes, the account age and "score" determine if an invalid serial can be used, or if you get the "customer code" error.
{: .notice--warning}

**Note:** The binary that generates this "validation data" is highly obfuscated. `pypush` sidesteps this issue by using a custom mach-o loader and the Unicorn Engine to emulate an obfuscated binary. `pypush` also bundles device properties such as the serial number in a file called data.plist, which it feeds to the emulated binary.
{: .notice--info}

After registering with IDS, you will receive an "identity keypair". This keypair can then be used to perform public key lookups.

When performing a lookup, you provide the account(s) that you would like to look up, and receive a list of "identities". Each of these identities corresponds to a device registered on the account, and includes important details such as its public key, push token, and session token.

**Warning:** Session tokens are necessary to send messages to a device. They essentially prove that you made a recent lookup, because the session token expires. Session tokens cannot be shared, as they can only be used by the account that performed the lookup request.
{: .notice--warning}

### message encryption
Now, we've setup the basics of iMessage. We have enough that we can look up the public keys of another user, as well as publish our own. Now we just need to put it together with APNs to send and receive messages!

In order to receive messages, one simply filters the APNs connection to `com.apple.madrid` and sends the active state packet.

Depending on which capabilities you advertised in your IDS registration, as well as the iOS version of the sending device, you may receive messages in the legacy (pre-iOS 13) `pair` encryption format, or in the new `pair-ec` format. While the `pair` format is much more documented and easier to implement, it does not provide forward secrecy using "pre-keys" (similar to Signal) as the new `pair-ec` format does.

You can then decrypt the message as described in several papers, and as implemented in `pypush`. Verifying the message signature is optional, but is obviously important if you intend your client to be secure.

Sending messages is pretty much the inverse to receiving them. Keep in mind that you can chose to individually send out messages to each recipient, or you can bundle all the different recipients and their respective encrypted payloads into a giant bundle, which APNs will split up for you. 

**Note:** Another thing to keep in mind is that message are delivered to all participants in a conversation, *including the other devices on your own account*.
{: .notice--info}

One more thing to keep in mind that is often overlooked when sending messages is that the AES key is not entirely random: it is tagged with an HMAC. Your message will fail to decrypt on newer devices if you use an entirely random AES key.

And that's pretty much it! As I mentioned, this blog post is designed to give you a good idea of how the iMessage protocol fits together, so that you can explore the `pypush` code, not directly explain every technical detail.

### resources and attribution
Many people and prior works have helped me understand iMessage. Here is a brief list, in no way exhaustive:
+ [IMFreedom Knowledge Base: iMessage](https://kb.imfreedom.org/protocols/imessage/)
+ [M. Frister: `pushproxy`](https://github.com/mfrister/pushproxy)
+ [Nicolás: `apns-dissector`](https://gitlab.com/nicolas17/apns-dissector)
+ [QuarkSlab: iMessage Privacy](https://blog.quarkslab.com/imessage-privacy.html)
+ [Garman et al. Chosen Ciphertext Attacks on Apple iMessage](https://www.usenix.org/system/files/conference/usenixsecurity16/sec16_paper_garman.pdf)
+ [NowSecure: Reverse Engineering iMessage](https://www.nowsecure.com/blog/2021/01/27/reverse-engineering-imessage-leveraging-the-hardware-to-protect-the-software/)
+ [Elcomsoft: iMessage Security and Attachments](https://blog.elcomsoft.com/2018/11/imessage-security-encryption-and-attachments/)
+ [Eric Rabil's `open-imcore`](https://github.com/open-imcore)
+ [The Apple Wiki: Apple Push Notification Service](https://theapplewiki.com/wiki/Apple_Push_Notification_Service)
+ [Mihir Bellare and Igors Stepanovs: Security under Message-Derived Keys: Signcryption in iMessage](https://par.nsf.gov/servlets/purl/10200009)
+ [Apple Platform Security: How iMessage sends and receives messages securely](https://support.apple.com/lt-lt/guide/security/sec70e68c949/web)
+ [Nicolás: Apple IDS payload keys](https://gist.github.com/nicolas17/559bec0d8e636f93f62cca844ee94ada)
+ [Various people on the Hack Different Discord](https://discord.gg/NAxRYvysuc)

This blog post has been reworked from [its original version]({% post_url 2023-04-19-imessage-overview-original %}).
{: .notice}
