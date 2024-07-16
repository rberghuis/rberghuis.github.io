---
layout: post
title: "Enabling Windows Copilot within the EU"
date: 2024-05-02
description: "UPDATED June 21st, 2024! In this post I'll share my findings on how to bypass the Country-level restrictions imposed by Microsoft that disables Windows Copilot. Next to how, I'm also sharing a PowerShell script that can help enable it for you!"
---

**2024-06-21** The Copilot for Windows icon is being moved to the System Tray (controlled feature release). Testing my earlier script revealed that 'enforcing' the icon to show now requires a restart of the Windows File Explorer - so alongside some new registry keys, I've just pushed a new version to the GitHub repository on my [GitHub repository](https://github.com/rberghuis/WindowsCopilot) - enjoy!

![Windows Copilot in System tray - © Robbert Berghuis](/assets/images/20240621-WindowsPowerShellTray.png)
<figcaption>Windows Copilot in System tray - © Robbert Berghuis</figcaption>

Also want to get your hands on Windows Copilot even though Microsoft has not yet released it in the EU? Well this is your moment 😁 I've been using Windows Copilot for quite a while now and happily test the new functionality Microsoft releases. The thought crossed my mind that more people want to test this, so I've tried to make it incredibly accessible by writing all required changes into a single PowerShell script to remove the 'Country-level' block and forcing some settings / policies ✨ - find it on my [GitHub repository](https://github.com/rberghuis/WindowsCopilot)
 
The code is provided as-is but I'm open to receive feedback on the coding itself and learn if it worked for you! It should also work for those that don't yet have a Copilot for Microsoft 365 license, as it will (then) default to the Microsoft Copilot Web experience (formerly known as Bing Chat Enterprise). 

As it's still in preview and Microsoft prevented roll-out in the EU to ensure it’s release adheres to EU Data Privacy laws, please take due care when using this functionality. It's Copilot, not auto-pilot - you remain ultimately responsible❗

![Windows Copilot enablement on a EU-based Windows device - © Robbert Berghuis](/assets/images/20240621-WindowsPowerShellTray.png)
<figcaption>Windows Copilot enablement on a EU-based Windows device - © Robbert Berghuis</figcaption>