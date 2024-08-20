---
layout: post
title: "Copilot and the Office Domain-Specific Language - Part 2"
date: 2024-08-20
description: "In our attempt to deepdive into the inner workings of Copilot for Microsoft 365, we already explored the foundation provided by the Office Domain-Specific Language. In this post, we will dive into how this ties into Copilot for Microsoft 365 when used in the Office Productivity apps."
---

![Designing an abstraction layer of an application - generated through Microsoft Designer - © Robbert Berghuis](/assets/images/20240820-Copilot-Designer.png)
<figcaption>Designing an abstraction layer of an application - generated through Microsoft Designer - © Robbert Berghuis</figcaption>

Previously, we've discussed what the Office Domain Specific Language (ODSL) entails and just in case it slipped your mind, you can find that [here]({{ '/2024/08/12/Copilot-and-the-Office-Domain-Specific-Language.html' | absolute_url }}){:class="plausible-event-name=Post+Internal"}, and I'd highly recommend reading it prior to this.

We've ended with the conclusion that the ODSL is a specialized language designed to interact with Microsoft Office applications. It provides a set of commands and structures that make it easier to automate tasks, manipulate data, and integrate different Office applications. The power of the ODSL lies in its specificity. Because it’s designed specifically for the Office suite, it can do things that would be more complex or even impossible when relying on a general-purpose language. Next to that, as it creates an abstraction layer it's probably way easier for Microsoft to introduce new features / functionality in Copilot for Microsoft 365.

In this post we'll explore how the ODSL integrates with Microsoft Copilot for Microsoft 365.

> **DISCLAIMER** Copilot for Microsoft 365 is a SaaS offering and Microsoft only discloses that much of the inner workings, which means the content here might not be a 100% correct. It’s my understanding on how it works based on all the information I’ve digested.

## How does Copilot for Microsoft 365 incorporate the ODSL?
The ODSL is quite a powerful tool to encode the application-specific user intent and communicates it in a way that can be executed by the application. But what happens when a user sends a prompt through one of the Microsoft Office Productivity Apps? How does the ODSL fit in? How does Microsoft utilize this tool? Simply put, it creates a bridge between the Large Language Model and the Office productivity apps. With that consulting answer, we're not getting smarter.

There's more involved but want to highlight a couple of aspects starting with something called the **Copilot Augmentation loop**, followed by **Responsible AI** in an attempt to demystify that and focussing on the **output generation** which is where most of the magic happens.

1. When a user sends a **prompt** (instruction) to the Copilot interface provided in PowerPoint such as ```Convert the text on the slides to a specific font```, the prompt with the current context of the slide is send to the **Copilot Augmentation Loop**. The Copilot Augmentation Loop is responsible for initializing the **Office Solutions Framework control** and provides a way of coordinating all the various types of data and services consumed by the application.
1. The **prompt** is then checked against the **Responsible AI check**, which consists of stuff like guard lists, Content Moderator and Microsoft-managed blocklists. The Responsible AI check is done in the beginning to help avoid working on instructions that are in violation, examples include self-harm, drug use etc.
1. As a **commanding intent**, the **prompt** is then classified into various categories and extract if the users' intent and classification would **require context** from the source. Source-context is needed when the content of the file is required to fulfill the task, such as _where_ (x, y coordinates) to place the text on the current slide.
1. The **prompt** is then compared against the **ODSL Sample Prompt bank**, as there's absolutely no need nor desire to re-invent the wheel on every Copilot-interaction - right? 😉
1. Copilot then **generates the code** to fulfill the user' intent.
1. The **generated code** is then parsed and run through the **Abstract Syntax Tree** (AST) to identify any errors in an attempt to resolve. As with all Large Language Models, there can be some hallucinations and this is where Microsoft tries to **prevent hallucinations**.
1. Ultimately, the **response** is sent to your Office client as a response ready for processing.

A visual representation of this is shown below.

![Microsoft Copilot Augmentation Loop - © Robbert Berghuis](/assets/images/20240820-Copilot-Augmentation-Loop.png)
<figcaption>Microsoft Copilot Augmentation Loop - © Robbert Berghuis</figcaption>

I'm aware of the other components I didn't describe, and truthfully, we've only hit the surface of the Copilot Augmentation Loop and its components in this article. This initial run-down of how it works should already remove the black box feeling you might have on how Copilot for Microsoft 365 interacts with your prompt and data within Office Apps like Word, Excel and PowerPoint.

## What can we learn from this?
In short, the ODSL bridges the gap between the Large Language Model and Office APIs, making stuff happen. Between the lines you’ll also read that this is part of the requirements –if not fully- to **use a specific Microsoft Office Release channel**. As the ODSL functionalities are also 'just' a features and the instructions the Office API can receive are vital for Copilot to actually function and perform the magic we've become used to.

We all already know that Microsoft only release features using the update cadence imposed by the release channel. Some of these required features haven't yet reached the Semi-Annual update channel most Enterprises use, requiring them to switch to the Monthly Enterprise channel when they want to deploy Copilot for Microsoft 365. Obviously, this is not something to take lightly as there are many reasons to have devices use the Semi-Annual update channel and you'll need to confirm if you can switch those to the Monthly Enterprise without introducing risks that could negatively affect business critical processes.

Stay tuned for more!