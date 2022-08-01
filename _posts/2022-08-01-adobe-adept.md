---
title: "Adobe ADEPT"
category: "reverse-engineering"
tags: ADEPT Adobe DRM
toc: true
toc_sticky: true
---

This mega-post is intended to overview on how the ADEPT protocol works, as discovered by my own efforts and those of Grégory Soutadé and Florian Bach. It will be periodically updated to contain new information.
{: .notice}

For the uninitiated, ADEPT is the protocol that Adobe's flagship DRM product, **Adobe Digital Editions (ADE)**, uses to communicate with both **Eden2 (Adobe Digital Edition activation service)** and with **Adobe Content Server (ACS)** instances run by book distributors.

On it's most basic level, the ADEPT protocol consists of two main parts: Activation and Fulfilment.

## Activation

In order to fulfill books, your device needs to be "activated". There are several methods of going about this, but the simplest is activating anonymously.

**Warning:** Because anonymous activations are, well, anonymous, they cannot be used across multiple devices: if you lose the device, your purchases are gone forever.
{: .notice--warning}

### Signing In

The first step of activation involves signing into your account. This process sets up a bunch of keys and other things that we will use later.

#### Generating Keys

The first thing you need to do is generate the keys you are going to use for your device. These include:

1. The *Device Key*, which is simply 16 random bytes

2. The *License Key*, which is a 1024-bit RSA keypair

3. The *Authentication Key*, which is a 1024-bit RSA keypair

Save these in a safe place, as you will need them later.

#### Constructing Sign-In Data

Now, you need to construct a buffer containing the sign in data, which looks like this:

```
DEVICE_KEY + len(USERNAME) + USERNAME + len(PASSWORD) + PASSWORD
```

The device key is the same one you generated earlier, while the username and password are empty. Thus, for an anonymous account, it should actually look something like this:

```
DEVICE_KEY00
```

You then need to encrypt this whole buffer with the Authentication Certificate.

{% capture auth-cert-notice %}
For simplicity, you can assume the Authentication Certificate will be:

```
MIIEYDCCA0igAwIBAgIER2q5eTANBgkqhkiG9w0BAQUFADCBhDELMAkGA1UEBhMCVVMxIzAhBgNVBAoTGkFkb2JlIFN5c3RlbXMgSW5jb3Jwb3JhdGVkMRswGQYDVQQLExJEaWdpdGFsIFB1Ymxpc2hpbmcxMzAxBgNVBAMTKkFkb2JlIENvbnRlbnQgU2VydmVyIENlcnRpZmljYXRlIEF1dGhvcml0eTAeFw0wODAxMDkxODQzNDNaFw0xODAxMzEwODAwMDBaMHwxKzApBgNVBAMTImh0dHA6Ly9hZGVhY3RpdmF0ZS5hZG9iZS5jb20vYWRlcHQxGzAZBgNVBAsTEkRpZ2l0YWwgUHVibGlzaGluZzEjMCEGA1UEChMaQWRvYmUgU3lzdGVtcyBJbmNvcnBvcmF0ZWQxCzAJBgNVBAYTAlVTMIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDZAxpzOZ7N38ZGlQjfMY/lfu4Ta4xK3FRm069VwdqGZIwrfTTRxnLE4A9i1X00BnNk/5z7C0pQX435ylIEQPxIFBKTH+ip5rfDNh/Iu6cIlB0N4I/t7Pac8cIDwbc9HxcGTvXg3BFqPjaGVbmVZmoUtSVOsphdA43sZc6j1iFfOQIDAQABo4IBYzCCAV8wEgYDVR0TAQH/BAgwBgEB/wIBATAUBgNVHSUEDTALBgkqhkiG9y8CAQUwgbIGA1UdIASBqjCBpzCBpAYJKoZIhvcvAQIDMIGWMIGTBggrBgEFBQcCAjCBhhqBg1lvdSBhcmUgbm90IHBlcm1pdHRlZCB0byB1c2UgdGhpcyBMaWNlbnNlIENlcnRpZmljYXRlIGV4Y2VwdCBhcyBwZXJtaXR0ZWQgYnkgdGhlIGxpY2Vuc2UgYWdyZWVtZW50IGFjY29tcGFueWluZyB0aGUgQWRvYmUgc29mdHdhcmUuMDEGA1UdHwQqMCgwJqAkoCKGIGh0dHA6Ly9jcmwuYWRvYmUuY29tL2Fkb2JlQ1MuY3JsMAsGA1UdDwQEAwIBBjAfBgNVHSMEGDAWgBSL7vCBYMmi2h4OUsFYDASwQ/eP6DAdBgNVHQ4EFgQU9RP19K+lzF03he+0T47hCVkPhdAwDQYJKoZIhvcNAQEFBQADggEBAJoqOj+bUa+bDYyOSljs6SVzWH2BN2ylIeZKpTQYEo7jA62tRqW/rBZcNIgCudFvEYa7vH8lHhvQak1s95g+NaNidb5tpgbS8Q7/XTyEGS/4Q2HYWHD/8ydKFROGbMhfxpdJgkgn21mb7dbsfq5AZVGS3M4PP1xrMDYm50+Sip9QIm1RJuSaKivDa/piA5p8/cv6w44YBefLzGUN674Y7WS5u656MjdyJsN/7Oup+12fHGiye5QS5mToujGd6LpU80gfhNxhrphASiEBYQ/BUhWjHkSi0j4WOiGvGpT1Xvntcj0rf6XV6lNrOddOYUL+KdC1uDIe8PUI+naKI+nWgrs=
```

{% endcapture %}

<div class="notice">
  {{ auth-cert-notice | markdownify }}
</div>

#### Encrypting Private Keys

In order to securly send the Authentication and License keypairs to Adobe, we must encrypt their private keys.

1. Export the private key as **PKCS#8**

2. Encrypt the exported data with the Device Key using `AES-128-CBC`

3. Base64-encode the result

For the public keys, all you need to do is export them as plain PKCS#1 keys, and then Base64-encode them.

#### Sending the Sign-in Request

The sign in request is formatted like this, replacing the placeholders with the data we just generated:

```xml
<?xml version="1.0"?>
<adept:signIn xmlns:adept="http://ns.adobe.com/adept" method="anonymous">
  <adept:signInData>(Encrypted Sign-in Data)</adept:signInData>
  <adept:publicAuthKey>(Public Authentication Key)</adept:publicAuthKey>
  <adept:encryptedPrivateAuthKey>(Encrypted Private Authentication Key)</adept:encryptedPrivateAuthKey>
  <adept:publicLicenseKey>(Public License Key)</adept:publicLicenseKey>
  <adept:encryptedPrivateLicenseKey>(Encrypted Private License Key)</adept:encryptedPrivateLicenseKey>
</adept:signIn>
```

Then, make a POST request to `https://adeactivate.adobe.com/adept/SignInDirect` with a Content-Type of `application/vnd.adobe.adept+xml`. If everything went as planned, the reponse should contain a license key, a license certificate, a user ID, a username, and a PKCS#12 archive. Save these things for later.

**Note:** If the (decrypted) license key returned by the server *does not match* the license key you sent, this means that there was already a license key associated with the account. Make sure to replace the key you generated by the one returned by the server. This should never happen with an anonymous account.
{: .notice--info}

### Activating

Now for the actual activation!

#### Generating information

When activating a device, Adobe collects a whole bunch of information about it. Adobe actually recognizes *two separate devices* during this phase. The first is the device communicating the Activation Request. The second is the "target device" that is being activated. For most use-cases, these will be identical.

##### Device Information

1. A unique fingerprint. (e.g. `4e1243bd22c66e76c2ba9eddc1f91394e57f9f8`)

2. An OS name (e.g. `Windows Vista`)

3. A locale (e.g. `en`)

4. A type (e.g. `standalone`)

5. A target Hobbes version (e.g. `9.3.58046`)

6. A client software version (e.g. `2.0.1.78765`)

7. A product name (e.g. `Adobe Digitial Editions`)

#### Constructing the request

For now, this is all the post contains. I'll update it with the rest of the process at some further time.