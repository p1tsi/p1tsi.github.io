---
title: WhatsApp revealing secrets (?)
date: 2025-07-03 19:57:19 +0200
categories: [Reverse Engineering, Mobile, OSINT, WhatsApp]
tags: [post]
image:
    path: assets/img/2025-07-03-WhatsApp-OSINT/WhatsApp_OSINT.png
---


# Intruduction

Lately, I came across an interesting blog post by MobileHackingLab describing a possible Information Disclosure in WhatsApp Messenger application ([https://www.mobilehackinglab.com/blog/silent-osint-whatsapp](https://www.mobilehackinglab.com/blog/silent-osint-whatsapp)).

They show how it is possible to use WhatsApp application in a rooted/jailbroken environment to retrieve the device model of the phone for a chosen phone number.

In particular, at first they inject a Frida script inside the application to bypass SSL pinning. 
Afterwords, they start up a BurpSuite session intercepting the HTTP traffic of the application.
They show that when someone tries to register an account to a freshly installed WhatsApp, the application makes some HTTP
requests to WhatsApp servers. The "incriminated" one is a POST request to `https://v.whatsapp.net/v2/exist`: sending an encrypted and base64-encoded bunch of data at the `ENC` key, result in a json containing, among the others, the key `wa_old_device_name`. 
This happens when the primary account for the chosen phone number is already registered in another
device. That other device's model is reported at that key. This value is used by the application in the subsequent view, when the user is asked to insert the 6-digit security code. That view reports something that sounds like
"Security code sent to \<wa_old_device_name\>". 

All this introduction to say that what caught my attention more in that post are these sentences:
*"The request body is not useful, so easy enumeration via the API is not possible"* and *"If it was easier to enumerate and always returning the same output, it could have been interesting to enumerate all phone numbers (per country) and collect statistics about which phones are most popular for example."*

Then...

![Challenge Accepted](assets/img/2025-07-03-WhatsApp-OSINT/challenge_accepted.png)

# Reverse Engineering

One of the activities I enjoy most is reverse engineering. So I grab all the needed tools ([wormhole](https://github.com/p1tsi/wormhole) among the others) and material and I started my adventure in trying to understand what is behind that ENC value in the request body.

For now I won't report the whole process I followed and I substitute it with this cool GIF ChatGPT gave me:

![Reversing in progress](assets/img/2025-07-03-WhatsApp-OSINT/gear_processing_animation.gif)


# Outcome

At the end, I managed to understand how that request body is crafted and I wrote a Python script that just makes the 'exist'
HTTP POST request without using the whole WhatsApp application + Frida + BurpSuite.

I tested it with a bunch of phone numbers taken from a data leak happened recently and this is the (redacted) outcome:

```json
{
    ...
    <REDACTED>: [
        "iPhone 11",
        "m****************8@icloud.com"
    ],
    <REDACTED>: [
        "iPhone 13 Pro",
        "f****************a@hotmail.com"
    ],
    <REDACTED>: [
        "Samsung Galaxy A52",
        "NA"
    ],
    <REDACTED>: [
        "Honor HONOR x9a 5G",
        "NA"
    ],
    <REDACTED>: [
        "iPhone 15 Pro",
        "s***********7@gmail.com"
    ]
    ...
}
```


# Conclusion

Since MobileHackingLab publicly wrote their post several days ago, I guess Meta is already aware of this issue (I don't even know if I should call it so), and I think all the response bodies Meta's servers return are completely under their control.

Moreover, I gave a very quick review to their Bug Bounty program, but it seems like this case is not in their scope because actually the email field is anonymized and the device model could be not so useful either in targeted attacks without knowing the OS version (unless someone could prove me that I am wrong).

[Here](https://github.com/p1tsi/whatsapp_device_discoverer) you can find the Python script.

One last curious fact: just by knowing the phone number and with the help of another mobile application, I managed to deanonymize the email address for one of the contact.
