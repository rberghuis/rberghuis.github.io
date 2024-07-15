---
title: "Sensitivity labels and Copilot for Microsoft 365"
date: 2024-01-17
---

![Image illustrating Microsoft Productivity Apps, Copilot and Purview Sensitivity Labels - © Robbert Berghuis](_images\20240117-microsoftproductivityappscopilotandlabels.png)

Many organizations are using Microsoft Purview Sensitivity labels to classify their unstructured data for quite a while now. Some organizations are also exploring Microsoft Copilot for Microsoft 365, but how do these two services work together? In this post I'm providing some of my own learnings on this topic from first-hand deployment experience and guidance on how to resolve some of the issues I've seen to date.

**Update February 20, 2024**: Microsoft published an update at [Considerations for deploying Microsoft Purview data security and compliance protections for Microsoft Copilot | Microsoft Learn](https://learn.microsoft.com/en-us/purview/ai-microsoft-purview-considerations) of which I want to highlight the following:
> The AIP add-in from the Azure Information Protection unified labeling client isn't supported for Microsoft Copilot. This add-in for Office apps is [retiring April 2024](https://techcommunity.microsoft.com/t5/security-compliance-and-identity/retirement-notification-for-the-azure-information-protection/ba-p/3791908). If you still use this add-in for labeling in Office apps, see [Migrate the Azure Information Protection (AIP) add-in to built-in labeling for Office apps](https://learn.microsoft.com/en-us/purview/sensitivity-labels-aip) to benefit from Copilot support and other new capabilities. 

## Microsoft Purview Sensitivity Labels
I've been grateful for the many opportunities to help enterprises deploy Microsoft Security & Compliance at scale, including the solution we know today as Microsoft Purview Sensitivity Labels. Back when I started on this journey, it was called Azure Information Protection (or Secure Islands IQProtector for the insiders...), which at some point in time transitioned into the Microsoft Compliance solution and renamed to Microsoft Unified Sensitivity Labels. Skip forward a few years, and we've landed on Microsoft Purview Sensitivity Labels which amongst other capabilities, still provides core functionality of classifying your unstructured data, including Office documents.

Why is this relevant you ask? Well, back when it was used to called Azure Information Protection, organizations required to deploy the Azure Information Protection client to their workstations, which amongst other functionality contained an add-in for the Microsoft Productivity apps (Word, Excel etc.). This add-in later transitioned into the Microsoft Information Protection Unified Labeling client which in turn already entered maintenance-mode back in 2022 and will be retired (for the Microsoft Productivity apps) in April 2024.

Microsoft has been working hard over the last years to reach feature parity between the add-in and the built-in capability within the productivity apps. As always, new features enter the Office release system based on channels, from Beta all the way to Semi-Annual channel. This can be one of the causes that resulted for some companies to delay the transition activities required to move the built-in labeling experience. I happen to know a few companies that still use the add-in, instead of the built-in capability. If you're interested in reading more on this, I'd strongly recommend a visit to the [Microsoft Purview Customer Experience Engineering (CxE)](https://microsoft.github.io/ComplianceCxE/playbooks/AIP2MIP/CompareAIP2MIP/) site, where they've extensively detailed out the feature parity.

## Copilot for Microsoft 365
In comes Copilot for Microsoft 365 on which many people have blogged about, so let's get down to the nitty gritty when it comes to Sensitivity labels in combination with the add-in. Whenever you, your organization or client are still using the Microsoft Information Protection Unified Labeling client, you'll experience that the Copilot-functionality isn't covered in all Microsoft Productivity desktop apps. The below bullets contain some of my findings:

- **Copilot for Word** will cease to exist whenever the Microsoft Unified Labeling add-in is active for the end-user. Launching word in 'safe mode' will ensure the Copilot interface will launch as that disables the add-in. However, any attempts of activating the Microsoft Unified Labeling add-in will fully remove the Copilot for Word experience.
- **Copilot for Excel** is visually present in the ribbon (only) but greyed out. Meaning the functionality isn't provided in the Desktop app. Similar to Word, when the Microsoft Unified Labeling add-in is not active, the functionality is working as expected.

![An image showing the difference between Excel desktop and Excel web, when using the Microsoft Unified Labeling add-in - © Robbert Berghuis](_images\20240117-excelvsexcelonline.png)
An image showing the difference between Excel desktop and Excel web, when using the Microsoft Unified Labeling add-in - © Robbert Berghuis

- **Copilot for PowerPoint** works! Whenever you're in need of transforming a Word-document into a presentation, the highest label of the source documents is also applied to the resulting PowerPoint deck. This is called label inheritance (more can be found on Microsoft Learn) and works, however it isn't reflected in the add-in bar. It is shown correctly in the Ribbon underneath the Sensitivity button and upon saving and re-opening the presentation, the label is also correctly reflected in the add-in bar. 

![An image showing the label inheritance 'flaw' in PowerPoint when using the Microsoft Unified Labeling add-in - © Robbert Berghuis](_images\20240117-powerpoint.png)
An image showing the label inheritance 'flaw' in PowerPoint when using the Microsoft Unified Labeling add-in - © Robbert Berghuis

I can't speak on behalf of Microsoft but can probably share that in my conversations with the Microsoft Product Group they've been strongly advising to transition to the built-in labeling client. If you need more information on the retirement of the Microsoft Unified Labeling add-in, I'd suggest to read the following [Microsoft Tech Community publication](https://techcommunity.microsoft.com/t5/security-compliance-and-identity/retirement-notification-for-the-azure-information-protection/ba-p/3791908).

## How to transition?

Amongst the huge set of possibilities, I've found that using the Office Cloud Policy service best fits my needs to-date. The Office Cloud Policy service settings can be pushed based on the used License Group that you also use to enable the Copilot for Microsoft 365 licenses and these policies take precedence over settings pushed through Group Policy Objects.

1. "Use the Sensitivity feature in Office to apply and view sensitivity labels" should be set to Enabled. This will enforce the use of the built-in sensitivity label feature.
1. "Use the Azure Information Protection add-in for sensitivity labeling" can be set to Disabled, although not always required - please refer to the [AIP Exception page at Microsoft Purview Customer Experience Engineering (CxE)](https://microsoft.github.io/ComplianceCxE/playbooks/AIP2MIP/AIPException/) site.
1. "List of managed add-ins" should be updated to stop the add-in from initializing upon the start of the application. Commonly, companies deploy a policy to forcefully enable the 'MSIP.[AppName]' (Word, Excel, etc.) add-in and although its functionality won't work due to the introduction of the earlier changes, it will still consume time/resources to initialize upon launching the desktop application.

Please note that these only cover some parts of the technical steps. Also, always consider the People, Process and Governance aspects of any change.
