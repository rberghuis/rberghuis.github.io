---
layout: post
title: "Reverting from Microsoft 365 Multi-Geo Part 2/4"
date: 2024-07-01
description: "In this post will explore moving active data. As a reminder, this is data that a user can still access, as opposed to data only kept in soft-deleted mailboxes and SharePoint Online Preservation Hold Library etc. Active user mailboxes can also hold data a user cannot access such as Teams Chats that have been copied as part of a Compliance Retention policy to a part of the Recoverable Items Folder of the user mailbox"
---

![Cutting up your Microsoft 365 Multi-Geo environment - generated through Microsoft Designer - © Robbert Berghuis](/assets/images/20240624-Cutting-up-Microsoft-365-Multi-Geo.png)
<figcaption>Cutting up your Microsoft 365 Multi-Geo environment - generated through Microsoft Designer - © Robbert Berghuis</figcaption>

In this series of articles, I will describe my learnings on a client that asked to decommission their Microsoft 365 Multi-Geo setup as preparation for returning to a true single-geo setup. To truly be ready, all data stored in their satellite regions needs to be cleared. My client has legal requirements to keep certain data for a prolonged period of time, this meant moving data between regions, both active and inactive; the latter being kept in legal retention.

 
If you missed the [first]({{ '/2024/06/24/Reverting-from-Microsoft-365-Multi-Geo-Part-1.html' | absolute_url }}) article in this series, I would highly recommend following the link and start at the beginning. It describes the why and provides a generic approach.

In this article, we will explore moving active data. As a reminder, this is data that a user can still access, as opposed to data only kept in soft-deleted mailboxes and SharePoint Online Preservation Hold Library etc. Active user mailboxes can also hold data a user cannot access such as Teams Chats that have been copied as part of a Compliance Retention policy to a part of the Recoverable Items Folder of the user mailbox.

Moving active data is probably the easiest, as an admin can (still) manage the Preferred Data Location and thereby enabling you to initiate cross-region data moves accordingly. Yes, obviously there are several challenges when moving data, but this does not require a massive amount of customization, expert coding, or external tooling. Cross-region data moves are native to Microsoft 365 – so by far the ‘easiest’ when seen from a technical perspective.

How you manage this attribute depends on your setup. If you manage it on-premises and have ‘mapped’ the attribute accordingly within “Entra ID Connect” or “Entra Connect Sync”, then obviously you need to update the authoritative source.

Disclaimer! Any and all code is provided on an as-is basis, without warranty of any kind and can be found on my [GitHub repository](https://github.com/rberghuis/M365MultiGeo). It makes little sense in showing stuff without providing parts of the code, and then requiring you to type it over into your favorite IDE. 

## Exchange Online
Exchange Online mailboxes are moved automatically whenever the Preferred Data Location attribute is changed. Within Exchange Online, the MailboxRegion property is used to specify the Preferred Data Location. Whenever the database geolocation no longer matches the MailboxRegion value, the mailbox in question is put into a relocation queue which will process the geo-move accordingly.

### Finding your marks
It is one thing to instruct the service to initiate the cross-region move, but another to validate if the move was finalized. Luckily, Exchange Online allows you to filter on the MailboxRegion property… oh wait 😊

I am always in favor of making our lives easier through automation, so first align with your Identity and Access Management team that you will utilize an attribute that can be used to filter, such as an Extension Attribute (known as Custom Attribute in Exchange Online). Then use that attribute for the purpose of tracking the Mailbox Region. It will take some time if you are hitting 100k+ mailboxes but totally worth it in my opinion. This mostly is an issue for those that are not already synchronizing this attribute where you can already filter it from using the Get-ExoMailbox cmdlet etc.

Under the assumption you selected CustomAttribute1, then the below example depicts how to “sync” (one-time) the Mailbox Region value with a filterable property. Keep in mind, there are more objects than just mailboxes that you will need to consider. 

![PowerShell to 'copy' the Mailbox Region to a Custom Attribute - © Robbert Berghuis](/assets/images/20240701-PowerShell-Copy-Region-to-Attribute.png)
<figcaption>PowerShell to 'copy' the Mailbox Region to a Custom Attribute - © Robbert Berghuis</figcaption>

A few days later… going forward you can now filter based on the Mailbox Region that you have mapped to your filterable property. Just adjust the filter to something like the below to find the objects not hosted in the central location.

![PowerShell to filter on the value of a Custom Attribute - © Robbert Berghuis](/assets/images/20240701-PowerShell-Custom-Filter.png)
<figcaption>PowerShell to filter on the value of a Custom Attribute - © Robbert Berghuis</figcaption>

A quirky side note, I really like the use of $hitlist and $hit as variable names, as anyone without PowerShell knowledge will read $ as an S... anyway, moving on 😉

### Initiating and validating a cross-region mailbox move

Now that you can find the mailboxes you want to move, you can issue changes to the Mailbox Region accordingly. Just ensure that whichever mailbox you target, you also update the Extension Attribute accordingly, so you can find it again. Maybe write something like ‘Moving’. When the object is synchronized and the region is 'mapped', update the authoritative source, and await the synchronization. If Entra ID is authoritative for the object you are targeting, then update through Update-MgUser and Update-MgGroup accordingly. Once the change has been made to update the Preferred Data Location and this differs from where the mailbox data is stored, then Exchange Online will automatically put the mailbox in the queue.

> Update-MgUser  -UserId $hit -PreferredDataLocation 'EUR'
> Update-MgGroup -GroupId $hit -PreferredDataLocation 'EUR'

Once the move has been initiated, adjust the filter to fetch and then retrieve the mailboxes with a variety of properties. Using Get-ExoMailbox really helps speed things up, as it only retrieves a default limited set of properties alongside anything you specifically identify.

![PowerShell to get in-scope mailboxes with a set of additional properties - © Robbert Berghuis](/assets/images/20240701-PowerShell-GetScope.png)
<figcaption>PowerShell to get in-scope mailboxes with a set of additional properties - © Robbert Berghuis</figcaption>

After fetching the mailboxes, you can use a simple Select-Object cmdlet to identify what has been moved, and what has not moved yet. The numbers shown are obviously altered, as I am not going to disclose which client this concerns.

![PowerShell to determine 'geo-move' state - © Robbert Berghuis](/assets/images/20240701-PowerShell-Determine-Move-Status.png)
<figcaption>PowerShell to determine 'geo-move' state - © Robbert Berghuis</figcaption>

Or fetch the nitty gritty details, if you are wondering which part of the mailbox is still being moved.

![PowerShell Grid-view on mailboxes with their database(s) - © Robbert Berghuis](/assets/images/20240701-PowerShell-Database-OutGridView.png)
<figcaption>PowerShell Grid-view on mailboxes with their database(s)- © Robbert Berghuis</figcaption>

I almost hear you thinking, is this worth describing? Nah, not really. Would I consider this accurate? No, also not - would you?! There is a catch as it will only report on the Primary and Main Archive mailbox locations, not taking the location of the Auxiliary archives etc. into account. These can still be hosted in a different region and is something you will need to take into consideration when initiating cross-geo moves. When reading the Mailbox Locations property, you will find a list of semi-column separated strings.

![PowerShell listing the Mailbox Location - © Robbert Berghuis](/assets/images/20240701-PowerShell-AuxArchive-list.png)
<figcaption>PowerShell listing the Mailbox Location - © Robbert Berghuis</figcaption>

 Reading this list is something you will need to get used to, and based on some minor reading I think the following information is provided:

- **ID** - Some ID of the mailbox location object, an auto-expanded archive can have multipl- AuxArchives accordingly, but this number (then) does not increase. Maybe 1 is a Boolean fo- 'available' and 0 being not available but is attached/affiliated to the object?
- **Mailbox GUID** - GUID of the mailbox location object
- **Mailbox Type** - Mailbox type, can be Primary, Main Archive, Aux Archive, etc.
- **Exchange Online FQDN** - Assumption, this is the FQDN of the Exchange Online Databas- **Availability Group (DAG) or the load balancer
- **Exchange Online GUID** - Assumption, this is the GUID of the database hosted on the DAG

Dissecting this will allow you to confirm if all mailbox databases attached to the object reside in the expected mailbox region. When that does not match, the move has not yet been completed. Just a reminder, the code for this can be found on the [GitHub repository](https://github.com/rberghuis/M365MultiGeo).

![PowerShell showing the list of all databases used, for all mailboxes - © Robbert Berghuis](/assets/images/20240701-PowerShell-List-all-regions.png)
<figcaption>PowerShell showing the list of all databases used, for all mailboxes - © Robbert Berghuis</figcaption>

If we were to zoom in on the second line where it shows multiple databases used. Zooming into shows me exactly which parts of the mailbox are still moving between regions. It is just a matter of time before Exchange Online finalizes the migrations. Obviously, there is still room for improvement in this script, but it will get you started.

### Let us assume we migrate the mailbox, what is the user impact?
Whilst the move can take hours to in my case several days to a week per mailbox, depending on the amount of mailboxes queued and type of mailboxes, there is no real impact for end-users during the migration. The migration itself is mostly transparent to the user. Does that mean there is no impact at all? No, as I said mostly 😉

- Outlook for Mac users might temporarily loose access to their Online Archive during the migration.
- Certain feature limitations are in effect for Exchange Online such as accessing mailboxes across regions through Outlook on the Web is only possible in a separate browser window as described in this Microsoft Support article. 

### So, are we done with Exchange Online?
No, well... it depends if you are using more/other types of mailboxes (it depends... sorry, I am in consultancy).

Let us look at Microsoft Teams for a bit. When you create a new team in Teams, it will create a Microsoft 365 (Unified) Group in Entra ID, which consists of a 'Group Mailbox' in Exchange Online and a 'group-connected site' in SharePoint Online. Upon creation, the Preferred Data Location attribute of the entity creating the team is used and one-time replicated onto the underlying objects. Luckily, most of the logic for moving can be re-used, with the main difference that for Microsoft 365 Groups, you will need to use ‘Unified Group’ as opposed to ‘(Exo)Mailbox’. Exchange Online uses the same principles on Mailbox Location and Mailbox Region. Initiating the cross-region move results into queuing the mailbox etc.

Is there more than Microsoft Teams? Also consider other mailboxes like Resources (Room, Equipment), Scheduling (Bookings) etc. In the previous section I have provided a lot of information already, and this is (still) an article - not a manual 😉 

## SharePoint Online
Speaking of Microsoft Teams, these also have a SharePoint Online Group-connected site. So that is where we will continue our story from this example. As long as we are talking about active groups, there is a one-on-one mapping directly from the ‘Unified Group’ to the related SharePoint Online site. So, finding your target should be a walk in the PowerShell – hint: there is a set of properties on each ‘Unified Group’ for which the name starts with ‘SharePoint’, one of them is ‘SharePointSiteUrl’.

What is definitely different when compared against Exchange Online, is that SharePoint Online sites are **not** moved across geographic areas automatically. There are at least two reasons for that:
- Creating the SharePoint Online Satellite location is a manual step an admin needs to perform, whereas Exchange Online ‘satellite locations’ are provisioned automatically. Why is it manual? Well, you will need to provide the URL for your Satellite location. So, a North-America location might be suffixed accordingly, to specify that location as ‘```contoso-NAM.sharepoint.com```’. Microsoft cannot guess your preference and the URL needs to be free/available, you will need to initiate its creation.
- Moving can only happen when files are not open, the files-in-use need to be closed before the move start and then there is a read-only window of 4-6 hours (officially) during the migration. Next to that, URLs will change as you are literally moving to a different SharePoint Online ‘forest’ and whilst SharePoint Online provides for a native ‘forwarders’ upon migrating sites (including personal OneDrive for Business sites), those will obviously only work as long as the satellite location remains operational. What you are reading is a requirement for proper user communication, adoption & awareness, aligning support etc.

### Initiating and validating a cross-region site move
So, under the assumption you have done your due care and due diligence in ensuring you are change management colleagues have spread the word – you can do technical stuff again. There are three cmdlets you need to remember:

- *Start-SPOSiteContentMove* for non-Group connected sites
- *Start-SPOUnifiedGroupMove* for Group connected sites
- *Start-SPOUserAndContentMove* for OneDrive for Business sites

It is also worth noting that the changes to the Preferred Data Location need to synchronize across the estate. It is best practice to wait at least 24 hours before committing changes in SharePoint Online.

If you are targeting the same ‘Unified Group’ we processed in our Exchange Online topic earlier, this has a group-connected site in SharePoint Online. Assuming you have already updated the group within Exchange Online, you do not need to update the property in SharePoint Online (again) as this attribute is synchronized between the two platforms. However, it is recommended to wait 24 hours for the internal SharePoint Online synchronizes. After that, just kick-off the migration with:

> Start-SPOUnifiedGroupMove -GroupAlias $hit -DestinationDataLocation "EUR"

The migration itself has four steps, of which the third step takes the longest. You can retrieve the move-status through Get-SPOUnifiedGroupState for Group connected sites, and you are unsurprisingly Get-SPOSiteContentMoveState is for non-Group connected sites etc.

<ol>
    <li><b>Ready to trigger</b> – the move has not started yet</li>
    <li><b>Scheduled</b> – the move was triggered, but is still in the queue</li>
    <li><b>InProgress</b> – migration is in progress</li>
    <ul>
        <li>3.1 Validation</li>
        <li>3.2 Back-up</li>
        <li>3.3 Restore</li>
        <li>3.4 Clean-up</li>
    </ul>
    <li><b>Success</b> or <b>Failed</b> – depicts the end-state of the migration</li>
</ol>

Permissions etc. are migrated as-is, so nothing changes. The Most Recently Used files are even operational. Any link to the site, folder or document remains functional as long as the forwarders are operational, which depend on the presence of the satellite location where you have "moved away" from.

Awesome! So, this means you can migrate all group-connected sites with ease, right? Well, almost... I almost sounded like your typical consultant again with a ‘that depends’ answer – dodged the bullet this time I guess. You know those amazing features Microsoft released to provide for Private and Shared Channels? Well, they are officially non-group connected sites and need to be migrated separately using the non-group connected site cmdlet: Start-SPOSiteContentMove. Take proper care in setting up your migration approach, to avoid strange-ish behavior with Teams split across data region, depending on the type of channel someone created.

### How about OneDrive for Business?
Same story as non-Group connected sites, just remember to connect to the source SharePoint Online admin endpoint through PowerShell. Meaning if you want to move a OneDrive site from the NAM region to your main tenant location at ```contoso-admin.sharepoint.com```, and your NAM region is ```contosoNAM-admin.sharepoint.com```, connect to the ```contosoNAM-admin.sharepoint.com``` to initiate the move to another region (from NAM).

Meanwhile, the mailbox of the user will be moved to the correct region depending on the length of the queue. This has lesser user impact, so nothing to worry about.

### Let us assume we migrate a site, what is the user impact?
Frankly, it is quite something. Some key-takeaways are to take into consideration when planning the cross-region moves are:
- The move takes anywhere between 2 to 6 hours and during this time, the location and all files- are read-only. Requiring any open files to be closed and all files checked-in
- Any SharePoint Online sites that are created as part of Shared and Private teams channels- need to be migrated separately.
- SharePoint Online Apps need to be re-instated, and Power Apps need to be re-created or- re-attached, both requiring a manual action after the data migration has finalized

On the plus-side, a lot of stuff also works out-of-the-box! Such as:
- All sharing links remain functional
- OneDrive for Business sync app is auto-updated to reflect the new locations / URLs
- Teams, SharePoint Mobile app and Power Automate are auto-updated
- Most Recently Used files list in Office Apps (Word, Excel, PowerPoint) is auto-updated

As with any migration, it all comes down to proper planning, communication, and awareness.

## To be continued...
"But wait, what about inactive data?" - I know right! In the next article we will dive into moving inactive data across regions and how to validate those moves.

> Disclaimer the information is provided as-is without warranty of any kind. It is an article containing learnings, not a step-by-step manual 
