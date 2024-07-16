---
layout: post
title: "Reverting from Microsoft 365 Multi-Geo Part 4/4"
date: 2024-07-15
description: "In this post we will explore validating the Purview Compliance retention policies are active on the moved objects. This is especially important when you have restored objects from an inactive state, as you do not want to inadvertently lose data because some background process did not properly apply your expected retention policy."
---

![Cutting up your Microsoft 365 Multi-Geo environment - generated through Microsoft Designer - © Robbert Berghuis](/assets/images/20240624-Cutting-up-Microsoft-365-Multi-Geo.png)
<figcaption>Cutting up your Microsoft 365 Multi-Geo environment - generated through Microsoft Designer - © Robbert Berghuis</figcaption>

In this series of articles, I will describe my learnings on a client that asked to decommission their Microsoft 365 Multi-Geo setup as preparation for returning to a true single-geo setup. To truly be ready, all data stored in their satellite regions needs to be cleared. As they have legal requirements to keep certain data for a prolonged period of time, this meant moving data between regions, both active and inactive; the latter being kept in legal retention.

If you missed the [first](_posts/2024-06-24-Reverting-from-Microsoft-365-Multi-Geo-Part-1.MD), [second]() or [third]() article in this series, I would highly recommend following the provided links! The first describes the why and a generic approach, the second dives into active-data geo-moves and the third explores moving inactive data across regions.

In this article, we will explore validating the Purview Compliance retention policies are active on the moved objects. This is especially important when you have restored objects from an inactive state, as you do not want to inadvertently lose data because some background process did not properly apply your expected retention policy.

Disclaimer! Any and all code is provided on an as-is basis, without warranty of any kind and can be found on my [GitHub repository](https://github.com/rberghuis/M365MultiGeo). It makes little sense in showing stuff without providing parts of the code, and then requiring you to type it over into your favorite IDE. 

##  Microsoft Purview Compliance portal
Obviously, the first step is to confirm the policies you are expecting are at least found when using the Policy Lookup tool found in the Purview Compliance portal. This provides you a quick overview on the policies where the concerning object is in-scope. 

![Policy lookup for a particular user in Purview admin portal - © Robbert Berghuis](/assets/images/20240715-Purview-Policy-Lookup-GUI.png)
<figcaption>Policy lookup for a particular user in Purview admin portal - © Robbert Berghuis</figcaption>

The next step is to confirm if the policy distribution was successful, and this is preferably done through PowerShell. The Distribution Status has three states: Success, Pending or Error. When it states 'Error', the Distribution Results will provide detailed insights into where the error occurred. Historically, the detailed error messages were not visible in the portal and I know these are visible in the new (public preview) Purview portal. However, I prefer PowerShell as this allows me to further investigate the objects that are in an error state at scale.

![Fetching Retention Compliance policy status and any errors through PowerShell - © Robbert Berghuis](/assets/images/20240715-PowerShell-Get-Retention-Compliance-Policies.png)
<figcaption>Fetching Retention Compliance policy status and any errors through PowerShell - © Robbert Berghuis</figcaption>

Now comes the caveat you have been waiting for... what if, both are not actually representing the truth...? Wait what now?! You have read correctly, these tools only confirm if the object should be in-scope, and that the policy was distributed to the underlying system. They do not confirm if the policy was successfully applied on the concerning object. Let's explore 😊

If you are working with retention policies on adaptive scopes based on region or other attributes that might have changed due to the cross-region move, this is also the moment in time to validate and confirm the correct policies are applied. As a side note, be aware that creating a new policy to replicate the retention behavior of the source region is obviously part of the options, but you will need to keep that policy until all objects covered by it are deleted (or covered by another superseding policy).

## Exchange Online
Reading the applicable retention policies on object level requires knowledge on how they are reflected on the mailbox object, which for whatever reason is not documented clearly. I can only guess you are expected to trust Microsoft on this and that for 999/1000 times this works without failure. That one time it does not could leave your client or organization at a state where they cannot fulfill their legal requirements as the data is not kept in retention.

Okay, you claim something does not work and now you are going to add additional work on my plate for what? In the spirit of "pictures or it didn't happen", here is an active example of the two mailboxes in scope of the earlier shown policies, notice something different? Both mailboxes are in-scope of the same Retention Compliance policies, but this is not reflected. 

<< IMG >>
![Retention Compliance Policy not applied to the mailbox - © Robbert Berghuis](/assets/images/20240715-PowerShell-Policy-not-applied.png)
<figcaption>Retention Compliance Policy not applied to the mailbox - © Robbert Berghuis</figcaption>

The trick of reading the retention policies is fetching them through the following cmdlet.

> Export-MailboxDiagnosticLogs -ComponentName HoldTracking -Identity $hit

This gives you a JSON-formatted string which translates to a list of objects for which one the properties is a GUID. This GUID can be translated back to the Retention Compliance Policy and will obviously require some minor string modifications. 

<< IMG >>
![Extracting the "Holds" through Mailbox Diagnostic Logs - © Robbert Berghuis](/assets/images/20240715-PowerShell-Extracting-Holds.png)
<figcaption>Extracting the "Holds" through Mailbox Diagnostic Logs - © Robbert Berghuis</figcaption>

For that mailbox in question, this part of the Mailbox Diagnostic Logs shows as empty as captured below. There can be many reasons for this, including replication time etc. However, if you are facing the same, I would advise you to raise a case at Microsoft Support to get this sorted.

<< IMG >>
![No "Holds" applied based on the Mailbox Diagnostic Logs - © Robbert Berghuis](/assets/images/20240715-PowerShell-No-Holds-Applied.png)
<figcaption>No "Holds" applied based on the Mailbox Diagnostic Logs - © Robbert Berghuis</figcaption>

That is it for Exchange Online, and with Group Mailboxes (or Unified Groups) we also might want to validate the corresponding SharePoint Online site!

## SharePoint Online
It is no surprise that SharePoint Online is way too complicated again. There is a Microsoft internal tool that would allow you to list the Retention Policies applied onto a SharePoint site using the cmdlet Get-TenantSPSiteHoldInfoV2. However, being an internal-only tool - you cannot have it 😊 This is as much as I can share about it.

> Get-TenantSPSiteHoldInfov2 -TenantId $id -SiteURL $url

As not everyone has access to the Microsoft internal tools, we need to find another way to accomplish this. So what are our options? We have already seen that the Purview Compliance portal just does an 'include/exclude' look-up of policies applied, so that is not 100% trustworthy. Is there an alternative? There are at least two methods I have seen so far that would sort-of fulfill your needs, but the challenge is scale again. Both are fine if you are only validating a handful of sites, but everything beyond 10 sites is tedious and everything above 100 sites is simply impossible.

The first option is to leverage the API look-up also provided in the SharePoint Online admin center, which provides you with insight if the site itself can be deleted or not. The below being an example on a per-site basis. 

![Compliance policy applied to the site - © Robbert Berghuis](/assets/images/20240715-Compliance-Policy-Applied.png)
<figcaption>Compliance policy applied to the site - © Robbert Berghuis</figcaption>

> This site has a compliance policy set to block deletion.

When you have a list of sites, you can call the API in bulk to fetch the details faster. It is a Boolean value, so should be easy enough to process for anyone with a little bit of PowerShell knowledge.

![Leveraging in the API through PowerShell to fetch the "Is Deletable" value - Robbert Berghuis - © Robbert Berghuis](/assets/images/20240715-SharePoint-Online-API.png)
<figcaption>Leveraging in the API through PowerShell to fetch the "Is Deletable" value - Robbert Berghuis - © Robbert Berghuis</figcaption>

The second option is to leverage the native [Microsoft Support diagnostic tooling](https://aka.ms/PillarInvalidRetention), which will list the applicable retention policies. I am not entirely sure if this is just a Purview Compliance policy look-up, or something checked within SharePoint Online itself.

![SharePoint Online Diagnostics through Microsoft Support- © Robbert Berghuis](/assets/images/20240715-Microsoft-Support-Diagnostic.png)
<figcaption>SharePoint Online Diagnostics through Microsoft Support - © Robbert Berghuis</figcaption>

After you have confirmed the data is subject to the expected Purview Compliance retention policies, you can mark the objects as inactive again.

## Okay, so all data was moved, now what?
We started this series with the desire to move to a single-geo setup of Microsoft 365 and looked at the setup of Microsoft 365 Multi-Geo referring to this as a separate forest. My client demanded to ensure all data remains accessible, so we explored options to move active and inactive data across regions.

Assuming you have secured all data you want to retain; the next steps are seemingly easy: turn off the SharePoint Online satellite location and remove the "Office 365 Multi-Geo" licenses from the tenant. However, that does not trigger the move from the Multi-Geo forest back to whichever forest tied to the tenant main location. It remains in that forest with all benefits and limitations.

The last thing you need to do, is contact Microsoft Product Group through the existing Support lines with the ask to migrate your tenant back to the "main single-geo forest". This is something only Microsoft can do and in this particular case, only after its completion will the tenant adhere (again) to the EU Data Boundary.

## Summary
We've looked what it takes to move active and inactive data across regions and managing the Preferred Data Location attribute accordingly. We've discussed the differences between active and inactive data, and I recon you'll acknowledge the need for separating the two to simplify the overal approach. We've also explored validating that all data was transferred and that the retention policies you're expecting are also applied onto the objects. We didn't dive into end-user communication, which is essential for (at least) all "active data moves". I also didn't disclose my thoughts on 3rd party tooling that reduce the efforts required for moving inactive data residing on SharePoint Online sites. There's also still more to the process than what I've described, but it should cover most of the technical bits and pieces.

I'll leave you with this rough outline on steps to take when attempting to wind-down multi-geo 
![Flow showing some of the steps to wind-down Microsoft 365 Multi-Geo - © Robbert Berghuis](/assets/images/20240715-Wind-down-flow.png)
<figcaption>Flow showing some of the steps to wind-down Microsoft 365 Multi-Geo - © Robbert Berghuis</figcaption>


I also want to repeat that this is not a manual, but just some of my learnings that I wanted to share. If you have any questions, feel free to reach out!

## Closing words
That is all folks. I hope the articles provided you with valuable insights, tools to run through multi-geo moves and Purview Compliance policy retention validation, or both and more. As always, I appreciate any and all feedback and insights you might have.

As a reminder, any and all code is provided on an as-is basis, without warranty of any kind and can be found on my [GitHub repository](https://github.com/rberghuis/M365MultiGeo). 
