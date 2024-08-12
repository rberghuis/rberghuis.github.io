---
layout: post
title: "Copilot and the Office Domain-Specific Language"
date: 2024-08-12
description: "Everywhere you look, you'll see people writing blogs and articles on emerging features in Microsoft Copilot for Microsoft 365. Whilst I applaud these, I always found the blogs lacking fundamental knowledge on how Copilot for Microsoft 365 <b>actually</b> works inside the Microsoft Office Productivity apps like Word and PowerPoint. In this post I'll take you along the journey of the Office Domain-Specific Language, what that has to do with Copilot for Microsoft 365 and uncover quite a bit of the reasoning behind the various requirements Microsoft imposes when you want to use Copilot for Microsoft 365."
---

![Designing an abstraction layer of an application - generated through Microsoft Designer - © Robbert Berghuis](/assets/images/20240812-Copilot-Designer.png)
<figcaption>Designing an abstraction layer of an application - generated through Microsoft Designer - © Robbert Berghuis</figcaption>

Whenever I read a blog, post, article, etc. on Copilot for Microsoft 365 - I get the feeling we're all feature-craving technologists. As a self-proclaimed technical expert, I can't shake the feeling that we're not sharing and thereby not discussing the **inner workings of Copilot for Microsoft 365** publicly. Not understanding the technical implementation, denies us from truly grasping the capabilities, features, limitations and prevents us to envision the future roadmap. This in turn means we're not bringing our best selves to our clients and stakeholders.

I've searched around a bit, admittedly not that extensive, and cannot find anyone writing about how Copilot for Microsoft 365 works from a (deep-)technical perspective. As it's a Software-as-a-Service (SaaS) offering from Microsoft, I can understand someone's reservations to some degree, but it shouldn't evolve into the abstention of knowledge-sharing I see today. Microsoft publishes just about everything for anyone to read/digest and with some substantiated assumptions - we should be able to draw-up quite a decent picture. Let me be amongst the first that opens the proverbial ball and take you on a dance in which we'll deep dive on how some of the technology works.

> **DISCLAIMER** as mentioned, Copilot for Microsoft 365 is a SaaS offering and Microsoft only discloses that much of the inner workings, which means the content here might not be a 100% correct. It’s my understanding on how it works based on all the information I’ve digested.

## Copilot for Microsoft 365 position within the AI Shared Responsibility Model
Before diving into the Office Domain-Specific Language, I wanted to quickly take a moment to highlight the shared responsibility model of AI in general.

![AI Shared Responsibility Model - © Microsoft](/assets/images/20240812-AI-Shared-Responsibility-Model.png)
<figcaption>AI Shared Responsibility Model - © <a href="https://learn.microsoft.com/en-us/azure/security/fundamentals/shared-responsibility-ai" class="plausible-event-name=Post+External+Microsoft+Learn">Microsoft</a></figcaption>

You might ask yourself "why are you showing this?". We all understand that Copilot for Microsoft 365 is a SaaS solution, and whilst the inner-workings are opaque to its users - it is (probably) still build using resources we _can_ also use when using Azure AI or when building our own AI solution on Azure (IaaS or PaaS).

The reason I'm showing this, is in order to understand how Copilot for Microsoft 365 works, we can leverage the plethora of knowledge shared on Microsoft Learn describing the various sub-components. I've dissected these individual services and extrapolated how they would operate in Copilot for Microsoft 365, next to reading the wealth of whitepapers, service trust documentation etc. The resulting body of knowledge is what I'm using to describe my findings on how Copilot for Microsoft 365 technically works.

## Office Domain-Specific Language
As mentioned, we're opening the ball starting with the Office Domain-Specific Language (ODSL). Some people already know what it is, others are reading this combination of words for the very first time. [Wikipedia](https://en.wikipedia.org/wiki/Domain-specific_language){:class="plausible-event-name=Post+External"} tells us:
> A domain-specific language (DSL) is a computer language specialized to a particular application domain. This contrasts with a general-purpose language (GPL), which is broadly applicable across domains. [..] Domain-specific languages allow solutions to be expressed in the idiom and at the level of abstraction of the problem domain. [..] In programming, idioms are methods imposed by programmers to handle common development tasks.

Ok, but what problem does it solve? Specifically in Copilot for Microsoft 365 in the Microsoft Productivity apps, the underlying Large Language Model needs to _understand_ (prompt) and _fulfill_ (response) the user intent. Understanding the users' intent isn't the challenge here. It's transforming the intent into actions to be executed in the Office App in question for which a translation needs to happen. Large Language Models excel at understanding the intent but aren't trained in encoding the intent into application-specific programmatic instruction such as [Office-JS2](https://learn.microsoft.com/en-us/office/dev/add-ins/reference/javascript-api-for-office){:class="plausible-event-name=Post+External+Microsoft+Learn"}. This is where Domain-Specific Languages (DSL) enter the playing field, as these provide the required abstraction to the level that the Large Language Model can generate a text-based response that leads to intent fulfillment.

The ODSL is a language designed by Microsoft for their Office suite, which abstracts the underlying APIs to easily instruct the Large Language Model on how to use these. Whilst designing this language, they've tried to develop it in such a way that requires the least amount of words (or tokens), as that's one of the main bottlenecks of Large Language Models. A much-used example is the hypothetical prompt ```transform all text on all slides to font 'Aptos'```, which could result into the following ODSL instruction / code:
```shell
# Get all slides
slides = select_slides()
# Get all text from slides
text = select_text(scope=slides)
# Format text
format_text(textRanges=text, fontName="Aptos")
```

Let's put this into perspective by reviewing how to accomplish the same using PowerShell.
```powershell
# Adding office assembly
Add-Type -Path $env:WINDIR\assembly\GAC_MSIL\office\15.0.0.0__71e9bce111e9429c\office.dll

# Create application object
$PowerPoint = New-Object -ComObject PowerPoint.Application

# Open the file
$PPT = $PowerPoint.Presentations.Open("C:\Users\robbert.berghuis\example-file.pptx")

# For each slide in the file
Foreach ($Slide in ($PPT.Slides)) {
  # Get all shapes on the slide
  Foreach ($Shape in ($Slide | Where-Object { $null -ne $_.Shapes }).Shapes) {
    # Get all TextFrame(s) within the shapes
    Foreach ($TextFrame in ($Shape | Where-Object { $null -ne $_.TextFrame }).TextFrame) {
      # Get all TextRanges within the TextFrames
      Foreach ($TextRange in ($TextFrame | Where-Object { $null -ne $_.TextRange }).TextRange ) {
        # Write FontName
        $TextRange.Font.Name = "Aptos"
      }
    }
  }
}

# Save, close and exit
$PPT.Save()
$PPT.Close()
$PowerPoint.Quit()
```

The resulting **Foreach**-loops (plural) are somewhat similar to the provided example ODSL instruction, and you'll probably agree the ODSL version is way easier to read when compared to the PowerShell-code. In the below 'recording' I'm showcasing how this works by altering the font from ```Webdings``` to ```Aptos```.
![PowerShell code to update the Font on all slides - © Robbert Berghuis](/assets/images/20240812-PowerShell-PowerPoint-Text-Formatting.gif)
<figcaption>PowerShell code to update the Font on all slides - © Robbert Berghuis</figcaption>

Let's get to the point I'm working towards. The **ODSL is a specialized language** designed to interact with Microsoft Office applications. It provides a set of commands and structures that make it easier to automate tasks, manipulate data, and integrate different Office applications. The power of the ODSL lies in its specificity. Because it’s designed specifically for the Office suite, it can do things that would be more complex or even impossible when relying on a general-purpose language. Next to that, as it creates an abstraction layer it's probably way easier for Microsoft to introduce new features / functionality in Copilot for Microsoft 365.

Stay tuned for more on how Microsoft Copilot for Microsoft 365 incorporates the ODSL!