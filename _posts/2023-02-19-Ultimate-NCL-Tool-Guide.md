
---
layout: post
title: "NCL Tool Guide, Tips, and Tricks"
date: 2023-02-19
tags: Competition, NCL, Tools
description: NCL Guide!
---

Hey there! 
<br>
In honor of my favorite competition, NCL, I've put together an 'Ultimate Tool Guide'.  My goal is to provide a one stop shop to learn about/access some amazing tools, but not provide any direct answers for the competition. <br>
Layout:
Each tool has a table, with links to relevant resources, and of course the tool itself. I imagine this guide will be pretty long, so CTRL + F to search for the tool you need. <br>

Last but not least, this is a living document, parts of this guide will change as I come across new tools and resources, so please check in often for any changes! Happy reading! <br>
[Register for NCL here!](https://cyberskyline.com/events/ncl)<br><br>
~ryanq47
<br>
## Open Source (OSINT)
---
#### __Google:__
No, I'm not kididng. Google will be your best friend for anything OSINT. Google dorking serves as an excellent content filter, and allows you to find some really cool stuff. <br>


| Site| Link|
| ------ | ------ |
|Google | [Google](https://google.com) |
| __Writeups__ ||
| *"Google Dorking: A guide for hackers & pentesters"* | [HTB Google Dorking](https://www.hackthebox.com/blog/What-Is-Google-Dorking)|
| __Additional resources__ | |
| Google Dorking DB | [Google Hacking Database](https://www.exploit-db.com/google-hacking-database) |

<br>

#### __ChatGPT:__
ChatGPT's data may be from 2021, but it still has a ton of relevant information. In essence, it's a better way to Google. If you are a lucky enough to get access to BingChat, then I'd use that, as it can actively search the internet. (Pro tip, set [Edge as your default browser to speed up the wait](edge://settings/defaultBrowser)) <br>

It is important to realize that these tools aren't always accurate, it can't hurt to verify the information if possible. 
<br>

| Site| Link|
| ------ | ------ |
|ChatGPT | [chat.openai.com](https://chat.openai.com/) |
| BingChat Waitlist | [bing.com/new](https://www.bing.com/new)|

## Password Cracking
--- 
<br>
Save for OSINT, password cracking is my favorite NCL category - I mean what's cooler than going full  hacker mode and cracking passwords?! Getting back on track, there are a variety of great password cracking tools, both locally, and online. 


#### __Crackstation:__
CrackStation is an online based password cracker, that uses a massive database of pre-computed hashes. It's a great starting point, and should retreive the majority of weak passwords, but once salting is involved, you've met its limits.

Pros:
- Easy to use, no setup
- Very Fast

Cons:
- Limited to unsalted hashes
- Limited hash algorithms 


| Site| Link|
| ------ | ------ |
|Crackstation | [CrackStation](https://crackstation.net/) |
| __Writeups__ | |
| *"Salted Password Hashing - Doing it Right"* | [Link](https://crackstation.net/hashing-security.htm) |
| __Files__ | |
| Crackstation Wordlists | [Wordlist](https://crackstation.net/crackstation-wordlist-password-cracking-dictionary.htm)|

<br>

## Cryptography
---
The Cryptography section starts off fairly easy, but quickly ramps up in difficulty, especially once you reach the Steganography section.

__Cipher vs Hash:__ <br>
	Hashing & Ciphers are conceptually similar, however they have a key difference. Hashing is a one way operation - i.e. __it's not meant to, or needed to be decrypted.__ Windows password authentication is a good example, when you type in your password, it is hashed (usually with NTLMv1 or v2) then compared to the stored hash in the SAM file. This way, your plaintext password is never left floating around the OS. <br>
	Ciphers, on the other hand, use a shared secret code, and they are __intended to be decrypted.__ Think of messaging applications that use end-to-end encryption, they'd be useless if all you recieved was a hashed message, or had to crack it. (RIP battery life)


#### dcode.fr:
From their website:
	"dCode is the universal site for decipher coded messages, cheating on letter games, solving puzzles, geocaches and treasure hunts, etc." - and oh boy is it ever useful in any cyber competition, especifically the 'Cipher Identifier'. 

Pros:
- One location to identify (and decode) just about any cipher
- Great breadowns on how the ciper & hash algorithms work<br>
Cons:
- Initially in French, need to translate, or go to /en in the site
- Not the most intuitve to use at times

| Site| Link|
| ------ | ------ |
|Dcode.fr | [Dcode (English)](https://www.dcode.fr/en) |
| __Quick Links__| |
| Cipher Identifier | [dcode.fr/cipher-identifier](https://www.dcode.fr/cipher-identifier)

#### CyberChef
CyberChef is the swiss army tool of the internet, it can do just about anything you ask it to, from sending HTTP requests, to Cipher Decryption, and much more. Honestly, CyberChef is helpful on just about every section of NCL, but I included it under Cryptography as that's what I utilize the most. <br>

| Site| Link|
| ------ | ------ |
|CyberChef | [CyberChef](https://gchq.github.io/CyberChef/) |
| __Quick Links__| |
| Recipes | [CyberChef Recipes](https://github.com/mattnotmax/cyberchef-recipes)
| Base64 Decode | [CyberChef Base64](https://gchq.github.io/CyberChef/#recipe=From_Base64('A-Za-z0-9%2B/%3D',true,false)&input=YUdWc2JHOGdkR2hsY21VaENncEhaVzVsY21Gc0lFdGxibTlpYVNFPQ)

#### Additional Tools

Here's some additional tools I've used in a pinch, most of these are redundant, but are handy to have in your back pocket.

| Site| Link|
| ------ | ------ |
|Base64 Decode/Encode | [base64decode.org](https://www.base64decode.org/) |
| URL Encode/Decode | [urldecoder.org](https://www.urldecoder.org/)

---
