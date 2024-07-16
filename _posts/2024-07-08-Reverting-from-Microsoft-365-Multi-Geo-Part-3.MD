---
layout: post
title: "Reverting from Microsoft 365 Multi-Geo Part 3/4"
date: 2024-07-08
description: "In this post we will explore moving inactive data and as reminder, this is data that a regular user can no longer access as it's kept in retention only. As opposed to data a user can access, which can be actively used or is sitting there consuming storage capacity whilst being redundant, obsolete or trivial."
---

![Cutting up your Microsoft 365 Multi-Geo environment - generated through Microsoft Designer - © Robbert Berghuis](/assets/images/20240624-Cutting-up-Microsoft-365-Multi-Geo.png)
<figcaption>Cutting up your Microsoft 365 Multi-Geo environment - generated through Microsoft Designer - © Robbert Berghuis</figcaption>

In this series of articles, I’ll describe my learnings on a client that asked to decommission their Microsoft 365 Multi-Geo setup as preparation for returning to a true single-geo setup. To truly be ready, all data stored in their satellite regions needs to be cleared. As they have legal requirements to keep certain data for a prolonged period of time, this meant moving data between regions, both active and inactive; the latter being kept in legal retention.

If you missed the [first]({{ '/2024/06/24/Reverting-from-Microsoft-365-Multi-Geo-Part-1.html' | absolute_url }}) or [second]({{ '/2024/07/01/Reverting-from-Microsoft-365-Multi-Geo-Part-2.html' | absolute_url }}) article in this series, I'd highly recommend following the provided links! The first describes the why and a generic approach, and the second dives into active-data geo-moves.

In this article, we'll explore moving inactive data and as reminder, this is data that a regular user can no longer access as it's kept in retention only. As opposed to data a user can access, which can be actively used or is sitting there consuming storage capacity whilst being redundant, obsolete or trivial.

Now we’re getting somewhere. Inactive data is data you still have accessible in Microsoft 365, either because the object is still in soft-deleted state or because the organization requires data to remain available to fulfill legal (retention) requirements and any object reference is already hard-deleted. You can’t update the Preferred Data Location of a user that is no longer actively represented in Entra ID, or from a Group-connected site for which the group is long gone. This is where things become interesting and equally frustrating. Microsoft has some documentation on this topic, but not as clearly described as their documentation on moving active data across regions.

Disclaimer! Any and all code is provided on an as-is basis, without warranty of any kind and can be found on my [GitHub repository](https://github.com/rberghuis/M365MultiGeo). It makes little sense in showing stuff without providing parts of the code, and then requiring you to type it over into your favorite IDE. 

## Exchange Online
There are two states that we’ll explore here, both are focused on mailboxes for which the object in Entra ID is long gone. Why? Well, if you still have the object in a soft-deleted state in Entra ID, just restore it and migrate it as “active data” – way easier. Here are the two states we’ll explore:

- Mailbox with or without a single Primary archive
- Mailbox with auto-expanded archive

Regardless if the current configuration allows for auto-expanding archives today, that doesn’t mean it was not allowed in the past. Also, almost equally regardless, providing the option for auto-expanding archives doesn’t mean every mailbox in your scope has an auto-expanded archive. When the configuration allows for it, doesn’t mean it always creates one. Exchange Online will only auto-expand the archive, when the archive is reaching its limits (90 out of 100 GB) and the auto-expanding archive feature is enabled.

Okay, so what gives? It matters a lot! As mailboxes with an auto-expanded archive require a different approach. Before diving into that, you might wonder – how do I know which approach to use for any particular mailbox?

### Finding your marks
Hitting your marks and ensuring you’re spending your time on something that actually makes sense. How do you distinguish between a mailbox with an auto-expanded archive and those without? Easy.

...

Right? Okay, remember from the previous article I’ve written about Mailbox Locations in relation to the database that particular part of the mailbox is stored. The database giving away in which region etc. but.. did.. you.. also see the type of mailbox? Yes, we can identify the type of mailboxes used by an object. An auto-expanding archive is nothing more than an additional archive attached to the main archive, which Microsoft calls Auxiliary Archives, or ‘AuxArchive’.

So provided you have your set of mailboxes you need to migrate, the provided code will allow you to identity those that have an auto-expanded archive. Additional bonus, as these are inactive mailboxes they inherently also will not grow in size – meaning if they don’t need an auto-expanded archive today, they also don’t need one tomorrow. I've uploaded some code into the [GitHub repository](https://github.com/rberghuis/M365MultiGeo) to help you identify mailboxes with or without an 'AuxArchive'. 

![Grouping based on presence of 'AuxArchive' - © Robbert Berghuis](/assets/images/20240708-PowerShell-Grouping-AuxArchive.png)
<figcaption>Grouping based on presence of 'AuxArchive' - © Robbert Berghuis</figcaption>

The mailboxes without an auto-expanded archive have a different approach when compared to those with such an archive. So, let's explore!

### Migrating inactive mailboxes without auto-expanded archives
Approaching mailboxes without an auto-expanded archive is fairly easy. The recommended approach provided through Microsoft Learn is to make them active again. This then provides you with the ability to use the built-in tooling of Microsoft to initiate and complete the cross-region move. Lastly, you’ll want to confirm all retention policies are correctly applied and when confirmed, delete the object representation in Entra ID in order to make the mailbox inactive again.

Restoring the mailbox isn’t too difficult, but there are some caveats to consider. Most are documented pretty well by Microsoft, so on the GitHub repository is some code to mass-restore the mailboxes in scope for this.

Some highlights – this will stop the Managed Folder Assistant from processing, set the correct mailbox type details etc. and ensure the mail address is ‘unknown’ for outsiders. You don’t want to receive new emails for a mailbox that was inactive.

As this essentially restores the mailbox in-place attaching a new object to it in Entra ID, you can use the earlier provided information to track cross-region migration accordingly. 

###  Migrating mailboxes with an auto-expanded archive
Yoda would say "difficult, auto-expanded archive are". When trying to use the previously provided approach of restoring the mailbox as-is, you’ll find that this is not an option for mailboxes with an auto-expanded archive. There are other options to restore the mailbox, I know and already hear you're probably thinking "just run a mailbox restore request and be done", right?. Nice try – but will also fail, I know as I tried. The Primary mailbox can be restored using mailbox restore requests, but the Online Archive be restored cannot as shown in the capture below. 

![Error shown when restoring an auto-expanded archive - © Robbert Berghuis](/assets/images/20240708-PowerShell-Error-Restore-Autoexpanded-Archive.png)
<figcaption>Error shown when restoring an auto-expanded archive - © Robbert Berghuis</figcaption>

> Error : Unable to perform Mailbox Restore as one of the mailboxes is a Large Archive. You can use eDiscovery to search & export data that has been preserved in inactive mailboxes.
A 'large archive' is layman's terms for an auto-expanded archive.

Okay, so now what? Well, here I’m a bit in doubt what is the best approach. There are multiple roads to Rome. One of which, is to restore the mailbox and use Purview Compliance Search & Export to export the Online Archive from the source and import it into the new mailbox again - perhaps even create multiple destinations to avoid waiting for the Online Archive to auto-expand. Exporting the Online Archive then requires a bit of KQL magic, which I’ve captured below and was only slightly adapted from the [Microsoft provided code](https://techcommunity.microsoft.com/t5/exchange-team-blog/content-search-for-targeted-collection-of-inactive-mailbox-data/ba-p/3719422). Obviously, I'm you can find my version on the [GitHub repository](https://github.com/rberghuis/M365MultiGeo).

![KQL query generation targeting all folders in the Online Archive - © Robbert Berghuis](/assets/images/20240708-PowerShell-KQLQuery.png)
<figcaption>KQL query generation targeting all folders in the Online Archive - © Robbert Berghuis</figcaption>

Are there other options? Let your imagination run wild! I'm trying to persuade Microsoft to re-attach an auto-expanded archive to an active mailbox. As that saves me from exporting the content, then validating the export is valid & complete, import it and re-run the validation on the destination to detect and solve any data corruption that might happen in the process. You might want investigate 3rd party tools to do the job for you.

### So, are we done with Exchange Online?

What about Unified Groups that have been deleted from Entra ID, don't they have a mailbox component? They are just mailboxes of type GroupMailbox, so you should be able to find them quite easily using the Get-Mailbox like

```powershell
Get-Mailbox -GroupMailbox -InactiveMailboxOnly -Filter $filter
```

Unfortunately, the -GroupMailbox parameter is not yet supported in the Get-ExoMailbox cmdlet which is a shame considering that works way faster in the majority of cases. The Verb-Exo* cmdlets are based on REST API which amongst other benefits outperform their related legacy cmdlets, as such and when possible use Get-ExoMailbox over Get-Mailbox. For those that are unaware, well-formed cmdlets contain a Verb and a Noun, example: Get-ExoMailbox the verb is Get and the noun is ExoMailbox. 

![Error when using GroupMailbox parameter in the Get-ExoMailbox cmdlet - © Robbert Berghuis](/assets/images/20240708-PowerShell-Error-GroupMailbox.png)
<figcaption>Error when using GroupMailbox parameter in the Get-ExoMailbox cmdlet - © Robbert Berghuis</figcaption>

> Get-ExoMailbox : GroupMailbox is not a supported parameter.

Similar for other mailbox types like resources etc. you’ll need to find the ones you want or need to move across regions and act accordingly.

## SharePoint Online
SharePoint Online is a lot trickier, for a variety of reasons one of which is that I’m nowhere near a SharePoint expert. It’s something that I’m still trying to figure out the options for, but most likely using 3rd party tools best serve your needs. Simply put, restoring group-connected sites for which the group no longer exists is a nightmare on its own. Microsoft does not provide native functionality to do this at-scale.

OneDrive for Business on first glance seems to be rather straight forward. As long as you can re-create a user object with the same UserPrincipalName, you can use the built-in Diagnostic Tooling in Microsoft 365 Admin center to re-map the old OneDrive for Business to the newly created user object. Then it’s an active object again and can be moved accordingly. Essentially, this is similar to the mailbox recovery action we’ve seen in Exchange online (without auto-expanded archives) – in simple terms: re-activate the OneDrive for Business and assign it to a user. Doing this at scale is a nightmare again, but it's an option to consider.

## To be continued...
"Aren't you done?" I have at least one more article up my sleeve that will look into Purview Compliance Retention policies and validating these are applied on the objects you've 'restored' before you make them inactive again. We don't want to spend days on this and end-up inadvertently deleting the data 😉

> Disclaimer the information is provided as-is without warranty of any kind. It's an article containing learnings, not a step-by-step manual 
