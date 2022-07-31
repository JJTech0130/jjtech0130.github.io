---
title: "Adobe ADEPT"
category: "reverse engineering"
tag: "ADEPT"
---

This post is intended to be a high-level overview on how the ADEPT protocol works, as discovered by my own efforts and those of Grégory Soutadé and Leseratte10.

For the uninitiated, ADEPT is the protocol that Adobe's flagship DRM product, **Adobe Digital Editions (ADE)**, uses to communicate with both a **Adobe's license server (Eden2)** and with **Adobe Content Server (ACS)**.

<!-- Move everything pertaining to the individual methods availible to a separate post? -->
In order to fulfill books, your device needs to be "activated". There are several methods of going about this, including activating anonymously, using a pre-existing Adobe ID, or a third-party login provider.

> **Warning:**
>
> There is a limit to the number of devices that can be activated with each account. By default, this limit is 4 stand-alone devices, as well as an additional 4 *tethered* devices.
>
> This limit can theoretically be increased by contacting Adobe Customer Service.

Anonymous activations are by far the simplist method, as they don't require any kind of out-of-band sign-up, but they have a significant disadvantage in that they cannot be used on multiple devices.

> **Note:**
>
> Anonymous activations can be "upgraded" into full Adobe IDs, provided that the target Adobe ID has not been used with ADE before.
