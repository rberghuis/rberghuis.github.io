---
layout: post
title: "Group-based licensing"
date: 2024-07-22
description: "In this post we will explore how licenses are assigned to objects such as groups, how Microsoft stores your configuration on what plans you want enabled and provides some input on how to get a grip on Microsoft License assignments through some minor automation using PowerShell."
---

![Happily creating PowerShell scripts - generated through Microsoft Designer - © Robbert Berghuis](/assets/images/20240722-Happy-Coding.png)
<figcaption>Happily creating PowerShell scripts - generated through Microsoft Designer - © Robbert Berghuis</figcaption>

Whenever Microsoft adds new 'Plans' (services / capabilities / features) to existing licenses, the groups that assign the license are automatically updated to also allow (/enable) the new Plans. This often releases functionality in an uncontrolled manner.

Unfortunately, I've not found a solid method of getting a notified of these changes **up-front** from any of the Microsoft systems whenever a license plan changes. This leads to license management through luck, powered by coincidence. To regain some control, we can obviously revert to automation and in this post, we'll explore the logic of how plans are enabled (or rather, disabled) within license assignments. The bare minimum this leads to, is that someone can confirm if the new plans are to remain enabled or should be disabled until you've run your due diligence checks.

## Walk before we run
Before we can write any PowerShell whatsoever, we first need to understand how licenses are assigned to groups and how Microsoft stores the enabled and disablement plans. Exploring this through Graph Explorer will get you started.

![Microsoft Graph Explorer view on license SKU and plans - © Robbert Berghuis](/assets/images/20240722-GraphExplorer-LicenseView.png)
<figcaption>Microsoft Graph Explorer view on license SKU and plans - © Robbert Berghuis</figcaption>

On a group with an ID, we can see a license is assigned referenced by the SKU ID. Next to that, a list of GUIDs is presented representing the (service) plans that are **disabled**. This means that all others plans that are part of this SKU are apparently enabled.

So, we got a list of GUIDs but how does that translate back to human-readable display names? One option is to use [this page](https://learn.microsoft.com/en-us/entra/identity/users/licensing-service-plan-reference){:class="plausible-event-name=Post+External+Microsoft+Learn"} which references all licenses and their service plans.

The SKU ```99cc8282-2f74-4954-83b7-c6a9a1999067``` references<sup>1</sup> ```Microsoft 365 E5 Extra Features``` and one of the disabledPlans ```a1ace008-72f3-4ea0-8dac-33b3a23a2472``` references<sup>1</sup> ```Microsoft Clipchamp```. All other plans listed on the aforementioned site are then always enabled, but which ones?

<small><sup>1</sup> these are the display names at the time of writing - they will undoubtedly change...</small>

There's another call we can make against the Microsoft Graph that lists all subscriptions and their (service) plans.

![Microsoft Graph Explorer view on directory subscriptions - © Robbert Berghuis](/assets/images/20240722-GraphExplorer-SubscriptionView.png)
<figcaption>Microsoft Graph Explorer view on directory subscriptions - © Robbert Berghuis</figcaption>

Each plan that is part of license has the serivcePlanID which is also used in the ```disabledPlans``` that we picked up on earlier.

## Logical approach
Now that we know how Microsoft presents the data on license assignment level, and that we can find all GUIDs of plans tied to the license. We can create some logic to act on this. We always know what we want to enable, and everything else should be disabled. What we can disable will change over time, so we don't actually know what to disable up-front. It's a bit of odd thinking, but the disabledPlans list should therefore contain all plans you're not explicitly enabling.

1. Define a list of what we are expecting, probably the most non-technical way would be a CSV-file where you describe the Entra ID Groups by Object ID (GUID), License (skuId) and Enabled Plans (servicePlanId).
1. Retrieve the available Licenses from the tenant (subscribedSku) to facilitate building a list of available Service Plans.
1. Imports your CSV-file that outlines the expected results
1. Builds a list of expected DisabledPlans, which is the residual list after removing all 'expected enabled plans' from the list of all available plans
1. Loops through each group to determine if the expected state meets the current state
1. Then subsequently determine (in order):
    1. If the expected License SKU is assigned
    1. If the expected disabled service plans are indeed disabled
    1. If the expected enabled service plans are indeed **not** disabled
1. For anything out of the order do... something smart 😉

> **Note**: Within Microsoft Entra ID, when assigning the same license to the same group, the license is consolidated into 1 and the enabled/disabled plans are combined.

The intermitted result can be found on my [GitHub repository](https://github.com/rberghuis/M365LicenseManagement){:class="plausible-event-name=Post+External+GitHub+rberghuis+M365LicenseManagement"}, whilst still providing room for improvement it should give you a decent head start! I've taken the liberty of including some additional logic to avoid end-user introduced issues (PEKBAC) in the input-file with regards to specifying the same License (sku) on the same group (GUID) multiple times with same or different Enabled Plans (servicePlanId).