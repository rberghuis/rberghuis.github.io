---
layout: post
title: "Reverting from Microsoft 365 Multi-Geo Part 1/4"
date: 2024-06-24
description: "In this post we the WHY, HOW and WHAT you need to do whenever you're considering wind-down of Microsoft 365 Multi-Geo. In this series of articles, I’ll describe my learnings on a client that asked to decommission their Microsoft 365 Multi-Geo setup as preparation for returning to a true single-geo setup. In order to truly be ready, all data stored in their satellite regions needs to be cleared."
---

![Cutting up your Microsoft 365 Multi-Geo environment - generated through Microsoft Designer - © Robbert Berghuis](/assets/images/20240624-Cutting-up-Microsoft-365-Multi-Geo.png)
<figcaption>Cutting up your Microsoft 365 Multi-Geo environment - generated through Microsoft Designer - © Robbert Berghuis</figcaption>

In this series of articles, I’ll describe my learnings on a client that asked to decommission their Microsoft 365 Multi-Geo setup as preparation for returning to a true single-geo setup. In order to truly be ready, all data stored in their satellite regions needs to be cleared. My client has legal requirements to keep certain data for a prolonged period of time, this meant moving data between regions, both active and inactive; the latter being kept in legal retention.

Before we’re kicking off - yes, my client understands that moving data between regions undermines their data residency requirements for which they’ve (well… I’ve) initially set-up the Multi-Geo environment to begin with. Before any data migration, you always have to validate all requirements are met including legal, compliance etc. Writing down my learnings so far quickly turned into a multiple page document, I’ve moved this to a series of articles, each providing a different aspect on the data migration challenges. As a fair warning, these articles are solely covering the technical challenges, and explicitly do not cover any legal implications.

## Why would you do such a thing?
Before diving into the technical stuff, a quick step back into the why. Initially there is no reason to move data between regions as the tenant remains in the Microsoft 365 Multi-Geo 'forest'. Only when you’re actively moving back to a single-geo setup, you need to move data out of your satellite locations back to your main data location.

So why? Easily said, it has something to do with the EU & EFTA Data Boundary. My client wants to move back to the “EU Forest” to ‘leverage’ the restrictions of the EU & EFTA Data Boundary accordingly as this helps them meet certain requirements going forward. [The documentation](https://learn.microsoft.com/en-us/privacy/eudb/eu-data-boundary-learn) clearly states as much:

> Customers who have purchased Multi-Geo Capabilities are not in scope for the EU Data Boundary even if their tenant is listed as being in a country or region in the EU or EFTA. 

 It technically can’t be part of the EU & EFTA Data Boundary, as that boundary aims to prevent data transfers outside the EU, whilst the sole purpose of Microsoft 365 Multi-Geo is to facilitate data residency requirements in other regions than the main region.

Returning to a single-geo setup is not an easy feat as there’s a lot of transformations required at Microsoft – for which the supplier bears all the liability of successful completion and all the risk of business disruptions that could negatively affect any service level agreements they’ve contractually agreed upon. Moving back to a single-geo setup is something Microsoft has only done on a few rare occasions.

Let's shift our focus to what you can do - as ‘the customer’ you’ll need to dispose of all data stored in any satellite location, or move all data you want to preserve. It’s up to you to decide if you want to move data back into your main data location. Upon return to a single-geo setup, you can’t access data outside your tenant region – which is what you can do when you’re in the multi-geo forest. My client wants to retain all data stored in their satellite locations, so I'm tasked on ensuring everything is moved.

This article series will provide my insights into the challenges to complete cross-region data moves and prepare the tenant for whatever Microsoft must do to technically move the tenant from the “multi-geo forest” to the “EU (Data Boundary) forest”. I can share up front that disabling the SharePoint Online satellite locations and removing the multi-geo licenses from the tenant will not automatically move the tenant back to a single-geo setup.

For reference, 'satellite locations' is the terminology used by Microsoft to identify locations where data can reside in your Multi-Geo tenant, other than the main ‘central’ location. In the capture below, you can see that the main location is EUR (or actually EMEA) and there are two satellite locations active, one in North America and one in Asia-Pacific. 

![Microsoft 365 Multi-Geo view - © Robbert Berghuis](/assets/images/20240624-Microsoft-365-Multi-Geo.png)
<figcaption>Microsoft 365 Multi-Geo view - © Robbert Berghuis</figcaption>

Side note, I’m not entirely sure if referring to these as “forests” is factually correct, but the concept of different forests will resonate with most technical experts. A visual representation would be something like shown in the image below. Keep in mind that this only concerns the storage of data, not the processing of data.

!["Forest" concept of Microsoft 365 Multi-Geo - © Robbert Berghuis](/assets/images/20240624-Forest-concept-Multi-Geo.png)
<figcaption>"Forest" concept of Microsoft 365 Multi-Geo - © Robbert Berghuis</figcaption>

## How would someone approach this? Asking for a friend…
With any engagement, the key to success is setting up an approach and validating that prior to its executing. We’ve done our due diligence and aligned with Microsoft directly to ensure our approach was valid. High-level this is roughly what it looks like:
- Move all active data *
- Move all inactive data that you need (or want) to preserve
- Validation
- Clean-up (e.g. licenses, email aliases, SharePoint Online satellite locations, etc.)

*Active means accessible by regular users whereas inactive data is only kept in legal retention (ie. SharePoint Online Preservation Hold Library, Soft-deleted mailbox, etc.)

Moving data between regions sounds easy, but what is it that you need to move? Well, for that you need to have an understanding of where and how data is stored. The scope of Microsoft 365 multi-geo services currently encompasses Exchange, OneDrive for Business, SharePoint Online and Teams. So, roughly speaking, it’s either Exchange Online or SharePoint Online. The below summary is probably not complete, but it gives an idea on where data is stored. 

## Exchange Online
- **Outlook** - items in user, shared, resource, and Microsoft 365 Group mailboxes like emails, - calendar items, tasks
- **Teams chats** - individual chats, group chats, private channel meeting chats, and “interactions - with Copilot for Microsoft 365” are stored in the user mailbox (or when absent, the - corresponding 'component shared mailbox')
- **Teams messages** - Teams Channel messages are stored in the corresponding Microsoft 365 Groups
- **Viva Engage user messages** - private messages and community message notifications
- **Viva Engage Community messages** - discussions etc. are stored in the corresponding Microsoft - 365 Groups
- **Public folders and Skype for Business** - here for completeness, for those that still use it

## SharePoint Online
- **Group connected sites** - files shared through Teams Channels, Viva Engage communities or Outlook groups are all stored in the corresponding Microsoft 365 group connected site
- **Classic, communication and non-group connected sites** - files uploaded including those in default document libraries such as ‘Site Assets’
- **OneDrive for Business** - files uploaded in your personal OneDrive for Business, and those shared through Connected experiences like Teams Chats, Viva Engage private chats. This also includes (Stream) videos uploaded in 1:1 Teams chats etc.

OK, cool! How about my Loop Workspace? Strictly speaking its not enabled for Multi-Geo service but its stored as a .loop file on SharePoint Online, so… 😊 Anyway, as mentioned it’s a rough outline and when you’re faced with the same challenge, you’ll need to do a proper assessment of what is used and where the data is stored.

## How is the data location managed?
I'm well aware not everybody has experience with Multi-Geo setups, a quick summary on this.

The Preferred Data Location attribute exists on objects like users, SharePoint sites and Microsoft 365 groups - which in turn determines in which region the data should be stored. How this attribute is managed obviously depends on the organization’ setup. However, it’s common for organizations to manage this through their Identity Governance & Administration solution, and when utilizing Active Directory to map it through “Entra ID Connect” or “Entra Connect Sync” to the managed object.

What is managed through an Identity Governance & Administration solution is also different for each organization. Some will manage everything; some have a demarcation on managing (personal and non-personal) identities. When the object is not managed through Identity Governance & Administration, such as a self-provisioned Outlook Group, the Preferred Data Location of the user creating the object (creator) is used to determine the Preferred Data Location of the object to Outlook group (etc.) and the object is then provisioned in the corresponding region. When the Preferred Data Location attribute is not set, it will always default to the main (tenant) data location, also known as the central location. If it is set but invalid, it will also default to the main data location.

Don’t have an Identity Governance & Administration solution or the solution doesn’t manage the attribute used to set the Preferred Data Location? There’s someone somewhere with (at worst case) an Excel list that manages or automation that writes the Preferred Data Location to facilitate the Multi-Geo setup – find that person, entity or process owner. 

## To be continued...
"Aw... you're already stopping?" - I hear you! You're wondering where that technical stuff is that I promised to talk about in the beginning of this article. No stress - in the next article we'll dive into moving active data across regions and how to validate those moves.

> Disclaimer the information is provided as-is without warranty of any kind. It's an article containing learnings, not a step-by-step manual