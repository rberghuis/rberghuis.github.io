---
layout: post
title: "Upgrading to the Microsoft Azure Information Protection Unified Labeling v3 client"
date: 2024-04-19
description: "In this short post, we’ll dive into the why and how of the upgrade to the V3 Unified Labeling client."
---

![Image created through Microsoft Copilot showing a shiny new system - © Robbert Berghuis](/assets/images/20240419-shinynewsystem.png)
<figcaption>Image created through Microsoft Copilot showing a shiny new system - © Robbert Berghuis</figcaption>

In this short post, we’ll dive into the why and how of the upgrade to the V3 Unified Labeling client.

The Unified Labeling client is software that has a long history amongst folks concerned with Data Security. Historically, organizations have deployed this to provide labeling functionality in the Microsoft Office productivity apps, add a scanner to classify unstructured data on file shares etc. and provide a PowerShell module. April 11th marked the day that we waived goodbye to the Office integration, and the release of the v3 client. 

## What does the V3 client provide?
The new software package includes the Information Protection Scanner, PowerShell, File Labeler (previously AIP Classify & Protect) and Viewer and can be downloaded from [here](https://www.microsoft.com/download/details.aspx?id=53018).

It’s still in public preview and General Availability (GA) release is expected in May 2024. Although the functionality seems similar to the v2, but adds the all important supported from Microsoft. The thing is, the v2 client that currently provides for the Non-office stuff will deprecate as of June 30, 2024. As there are still many use cases for the Scanner, File Labeler and PowerShell module - organizations need to act today to deploy the new version from the moment it hits GA.

## Why and how to migrate?
Let’s get one thing straight, although the v2 version was able get updates through Microsoft Update, it will not auto-upgrade to the v3 version. Organizations need to package, test, deploy, validate the upgrade within (roughly) a 7-week time frame and whilst doing so, uninstall the v2 version.

Migration steps are made available by Microsoft [here](https://learn.microsoft.com/en-us/purview/upgrade-scanner-migrate) and whilst most organizations have made preparations already, I also notice some that didn’t.

The below timeline is based on the well-known ring deployment principle and assumes the organization has 4 rings. It also uses an n+7 days principle I see across enterprises, based on the initial release of software in an attempt to avoid bugs.

![Release schedule visualization - © Robbert Berghuis](/assets/images/20240419-releaseschedule.png)
<figcaption>Release schedule visualization - © Robbert Berghuis</figcaption>

## Summary
To wrap-up:
- v2 support ends on June 30, 2024
- v3 preview was released on April 11, 2024
- as previews only provide limited support, it’s not advisable to run these in full-fledged production deployment
- v2 will not auto-upgrade to v3
- v3 is expected to be GA as of May 11, 2024
- you can already start packaging and creating the uninstall (v2) and install (v3) today

## Relevant reading materials
The below is a non-exhaustive list of additional reading materials
- [Retirement notification for the Azure Information Protection Unified Labeling add-in for Office](https://techcommunity.microsoft.com/t5/security-compliance-and-identity/retirement-notification-for-the-azure-information-protection/ba-p/3791908)
- [Azure Information Protection and the Information Protection Modernization Journey ](https://techcommunity.microsoft.com/t5/security-compliance-and-identity/azure-information-protection-and-the-information-protection/ba-p/3671070)
- [Microsoft Purview Information Protection client - Release management and supportability](https://learn.microsoft.com/en-us/purview/information-protection-client-relnotes)