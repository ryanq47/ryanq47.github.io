---
title: Timing is Everything
date: 2024-08-10 00:00:00 +0600
categories: [CTF-Writeups, USCG-CTF-2024]
tags: [forensics]     # TAG names should always be lowercase
---

# Intro
---

In this writeup, I take on a forensics challenge from the USCG CTF 2024, titled "Timing is Everything." The challenge provides a PCAP file that initially seems to showcase routine network communication between two hosts using the ICMP protocol, but is infact hiding data via a clever trick. I don't want to spoil said trick in the intro, so read on!


# Methodology and Approach
---
## Initial Exploration

Forensics challenges have a million ways to approach them, depending on the challenge type. This challenge gives us a PCAP, which usually means wireshark.

The PCAP is pretty simple, we see that there's only two hosts talking to each other, and the only protocol used between them is ICMP. 

![[Pasted image 20240731131940.png]]
![[Pasted image 20240731132036.png]]

Scrolling through each packet reveals that each one is the exact same! That's odd, networks don't usually work like that. The one thing that changes, is *when* the packets are sent. That, along with the fact that the challenge is called `timing is everything` probably means that there's something hidden in the timing of these packets.

## Dissecting Timestamps

There's a few ways to potentially hide data in timestamps, but the first thing I notice, is that the second packet is exactly `0.083000` seconds after the first. If we look at the rest of the packets, all of them are unusually exact for a PCAP. Maybe `0.083000` could be a number for something?

I checked out a UTF-8 Table, and would you look at that, `83` translates into `S`. All the flags are in the format of `SIVUSCG{FLAG}` - so this is a promising lead.

Just to verify, I checked the time on the next packet as well, `0.156000`, which translates to `œ` in UTF-8. That can't be right? 

`0.156000` happens to be bigger than `0.083000`, in fact, each packets timestamp is subsequently larger. If we go ahead and subtract  `0.083000` from `0.156000`, we get `0.73`, which translates to `I`. That fits the flag format! 

## Scripting it out
This PCAP is small enough that the flag could be extracted by hand, but if there were thousands of packets, like in a real world scenario, it's inefficient to do so. I did a bit of python scripting, and with some help from the scapy library, was able to pull the timestamp of each packet, and subtract them as needed. 

![[Pasted image 20240731133453.png]]
Running this script, we get the following:

![[Pasted image 20240731133531.png]]

```
083 073 086 085 083 067 071 123 084 049 109 049 11 057 095 049 053 095 051 118 051 114 121 116 104 049 11 057 125
```

If I toss those numbers in CyberChef, and select the `From Decimal` operation, I get the following:

![[Pasted image 20240731133642.png]]

## Script Bug

Well, that's close... but why are there newlines? Turns out there's a bug in my script, when I strip the trailing 0's, it messes with some of the numbers, specifically turning `110` into `11`

I could fix this, but the most efficient route in this case is to just add those 0's back in CyberChef:

![[Pasted image 20240731133852.png]]

```
SIVUSCG{T1m1n9_15_3v3ryth1n9}
```

Bingo! There we go, we've got the flag! 

## Final Thoughts
---
This was an incredibly fun challenge. I love the concept of using timestamps to hide data, it's a sweet little trick. I question the applicability of this to a real world scenario, mainly due to unavoidable/unpredictable latency over networks, but nevertheless, still a cool idea. 

