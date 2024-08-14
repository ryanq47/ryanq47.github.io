---
title: Ding-O-Tron
date: 2024-08-10 00:00:01 +0600
categories: [CTF-Writeups, USCG-CTF-2024]
tags: [web]     # TAG names should always be lowercase
---

# Intro

In the 2024 USCG-CTF, one of the web challenges that caught my attention was the intriguingly named "Ding-O-Tron." At first glance, it appeared to be a simple click-based web application, but I knew there had to be more lurking beneath the surface. This writeup walks you through how I tackled the challenge, breaking down the steps I took to uncover the hidden secrets within the web app.

# Methodology and Approach
---
## Initial Exploration

Because this was a web challenge, I began by exploring the functionality of the WebApp to understand its purpose. It quickly became apparent that its sole purpose was to ding when clicked until the user reaches over 9000 clicks, at which point it provides a reward (most likely the flag).

## Console Examination

Opening the browser console revealed more details about the WebApp. Immediately, there was a status message: `[INFO] Ding-O-Tron Now Active!`, originating from a `wasm_exec.js` file. This suggested that WebAssembly was being used in some capacity in the WebApp.

Given tsuto's reputation for leaving subtle hints in web challenges, I decided to keep an eye on the console while interacting with the WebApp.

![image](https://github.com/user-attachments/assets/771a0e3e-a2eb-4ee1-9270-da9033f862b2)

Repeatedly clicking the bell leads to an interesting new message: `[ERROR] You're dinging too quickly!`. 

![image](https://github.com/user-attachments/assets/0ca2147a-d199-4d91-9351-3928f3b29370)

Investigating further, I found that on each click, the `window.ding()` function was called.

![image](https://github.com/user-attachments/assets/6f19310b-72fc-44fd-86e8-f5c5225c02a1)

At this point, I downloaded the entire challenge using a recursive `wget` command, which allowed me to take a closer look at the code. 

`wget -r https://uscybercombine-s4-web-ding-o-tron.chals.io/`. 

## Code Inspection

Searching for the `ding()` function revealed a few useful items, primarily the `main.js` file, which handles the interaction with `ding.wasm`.


![image](https://github.com/user-attachments/assets/7c253487-0bca-4332-a2f6-7f3e87b74551)

![image](https://github.com/user-attachments/assets/e9444189-9588-4841-869a-f7b3a5d227cd)

Inside `main.js`, there were various functions, most of which were irrelevant for obtaining the flag, except for `giveFlag()`, which was "leaked" in a comment, in `main.js`. When called, it prints to the console: `[LOL] Did you think it would be that easy? Can you find my secret hidden function?` and plays the "Emotional Damage" sound bite - which was a great touch. 

## Function Hunting

The latest console message suggested finding a "secret hidden function." There are a couple of ways to find functions in a web challenge: by searching the source code or by examining the Window interface in the console. Decompiling/reversing the `ding.wasm` binary seemed a bit impractical for a web challenge, so exploring the Window interface was the more efficient option.
## Window Interface

Typing `window` into the console listed everything the window could interact with. At the bottom of the following screenshot, all the functions from `main.js` were listed as expected. However, there was also a `superSecretFunction_585b46b3b6aecf7a` function, which, when expanded, was being called from the `wasm_exec.js` file. This seemed promising.

![image](https://github.com/user-attachments/assets/8d347c61-e7bc-45cc-ab37-80e23f5de89d)

![image](https://github.com/user-attachments/assets/86fce88b-4cf0-4d0c-b23d-f295a3fb14af)

Calling this function in the console did the trick, and revealed the flag!

![image](https://github.com/user-attachments/assets/cfce955a-cedd-46eb-989e-333647097958)


## Final Thoughts
---
This was quite the fun challenge, and sparked a bit of an interest in web assembly for me. My first attempt at this challenge, during the USCG, I got caught in the rabbit hole of trying to decompile the WASM, and eventually moved on to other challenges. However, it feels great to have come back and solved it this time around!

