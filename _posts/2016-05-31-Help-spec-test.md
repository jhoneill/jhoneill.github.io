---
layout: post
title:  "Help = Spec = Test"
date:   2016-05-31
categories: PowerShell
tags:
    - Testing
    - Pester
    - Devops
    - BDD
---
Going back for some years – at least as far the talks which turned into the [PowerShell Deep Dives book](https://www.amazon.co.uk/PowerShell-Deep-Dives-Jeffery-Hicks/dp/1617291315/ref=sr_1_1?ie=UTF8&qid=1464174210&sr=8-1&keywords=powershell+deep+dives) – I have told people **Start Help Early** (especially when you’re writing anything that will be used by anyone else).   
In the face of time pressure, documentation is the first thing to be cut – but this isn’t a plea to keep your efforts from going out into the world undocumented.    
**Help specifies what the command does, and help examples are [User Stories](https://en.wikipedia.org/wiki/User_story)** – a short, plain-English description of something someone needs to do.   
Recently I wrote something to combine the steps of setting up a Skype for business (don’t worry – you don’t need to know S4B to follow the remainder) – the help for one of the commands looked like this
{% highlight PowerShell %}
<#
.SYNOPSIS
Sets up a S4B user including telephony, conference PIN and Exchange Voice mail
.EXAMPLE
Initialize-CsUser –ID bob@mydomain –PhoneExtension 1234 –pin 2468 –EnterpriseVoice
Enables a pre-existing user, with enterprise voice, determines and grants the 
correct voice policy, sets a conferencing PIN, updates the Phone number in AD, 
and enables voice mail for them in Exchange.
#>
{% endhighlight %}
I’ve worked with people who would insist on writing user stories as “Alice wants to provision Bob… …to do this she …”  but the example serves well enough as _both_ **help** for end users _and_ a **specification** for one use case: after running the command  user “bob” will

-  Be enabled for Skype-for-Business with Enterprise Voice – including “Phone number to route” and voice policy
-  Have a PIN to allow him to use voice-conferencing
-  Have a human-readable “phone number to dial”  in AD
-  Have appropriate Voice Mail on Exchange

The starting point for a Pester test (the Pester testing module ships with PowerShell V5, and is downloadable for earlier versions) ,  is set of simple statements like this – the thing I love about Pester it is so human readable.

<code><span style="color:#0000ff;">Describe</span><span style="color:#8b0000;"> &quot;Adding Skype for business, with enterprise voice, to an existing user&quot;</span><span style="color:#000000;">&#160; {</span>        <br /><span style="color:#006400;">### Code to do it and return the results goes here</span>        <br /><span style="color:#0000ff;">&#160;&#160;&#160; It</span><span style="color:#8b0000;"> &quot;Enables enterprise voice with number and voice policy&quot;</span><span style="color:#000000;"> {&#160;&#160;&#160; </span><span style="color:#000000;">} </span>         <br /><span style="color:#0000ff;">&#160;&#160;&#160; It</span><span style="color:#8b0000;"> &quot;Sets a conference PIN&quot;</span><span style="color:#000000;">&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; {&#160;&#160;&#160; } </span>        <br /><span style="color:#0000ff;">&#160;&#160;&#160; It</span><span style="color:#8b0000;"> &quot;Sets the correct phone number in the directory&quot;&#160;&#160;&#160;&#160;&#160;&#160;&#160; </span><span style="color:#000000;">{&#160;&#160;&#160; }</span>        <br /><span style="color:#0000ff;">&#160;&#160;&#160; It</span><span style="color:#8b0000;"> &quot;Enables voice mail&quot; </span><span style="color:#000000;">&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; {&#160;&#160;&#160; }</span>        <br />}</code> 

The “doing” part of the test script is the command line from the example (through probably with different values for the parameters).   
Each thing we need to check to confirm proper operation is named in an **It** statement with the script to test it inside the braces. Once I have my initial working function, user feedback will either add further user stories (help examples), which drive the creation of new tests or it will refine this user story leading either to new **It** lines in an existing test (for example “It Sets the phone number in AD in the correct format”) or to additional tests (for example “It generates an error if the phone number has been assigned to another user”)

In my example running the test a second time proves nothing, because the second run will find everything has already been configured, so a useful thing to add to the suite of tests would be something to undo what has just been done. Because **help and test are both ways of writing the specification**, you can start by writing the specification in the test script – a simplistic interpretation of “Test Driven Development”.  So I might write this

<code><span style="color:#0000ff;">Describe</span><span style="color:#8b0000;"> &quot;Removing Skype for business from a user&quot;</span><span style="color:#000000;">&#160;&#160; {</span><br />
<span style="color:#006400;">### Code to do it and return the results goes here&<br /></span>
<span style="color:#0000ff;">&#160;&#160;&#160; It</span><span style="color:#8b0000;"> &quot;Disables S4B and removes any voice number&quot;</span><span style="color:#000000;">&#160;&#160; {&#160;&#160;&#160; </span><span style="color:#000000;">} –Skip          <br /></span><span style="color:#000000;">&#160;&#160;&#160; </span><span style="color:#0000ff;">It</span><span style="color:#8b0000;"> &quot;Removes voice mail&quot; </span><span style="color:#000000;">&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; {&#160;&#160;&#160; </span><span style="color:#000000;">} –Skip</span> <br />}</code>

The `–Skip` prevents future functionality from being tested. Instead of making each command a top-level `Describe` section in the Pester script, each can be a second-level Context section.

<code><span style="color:#0000ff;">Describe</span><span style="color:#8b0000;"> &quot;My Skype for business add-ons&quot;</span><span style="color:#000000;"> {</span>         <br /><span style="color:#0000ff;">&#160;&#160;&#160; Context</span><span style="color:#8b0000;"> &quot;Adding Skype for business, with enterprise voice, to an existing user&quot;</span><span style="color:#000000;">&#160;&#160; {...</span><span style="color:#000000;">}</span>         <br /><span style="color:#0000ff;">&#160;&#160;&#160; Context</span><span style="color:#8b0000;"> &quot;Removing Skype for business from a user&quot;</span><span style="color:#000000;">&#160; {...</span><span style="color:#000000;">}</span>         <br /><span style="color:#000000;">}</span></code>

So… you can start work by declaring the functions with their help and then writing the code to implement what the help specifies, and finally create a test script based on the Help/Spec OR you can start by writing the **specification as the outline of a Pester test script**, and as functionality is added, the help for it can be populated with little more than a copy and paste from the test script.
Generally, the higher level items will have a help example, and the lower level items combine to give the _explanation_ for the example. As the project progresses, each of the `It` commands has its `–Skip` removed and the test block is populated, **to-do** items show up on the on the test output as **skipped**.

<code><font color="#9b00d3">Describing My Skype for business add-ons          <br />
&#160;&#160; Context Adding Skype for business, with enterprise voice, to an existing user</font></code><br />
<code><font color="#00ff00">
&#160;&#160;&#160; [+] Sets the phone number to call in the directory 151ms<br />
&#160;&#160;&#160; [+] Enables enterprise voice with the phone number to route and voice policy&#160; 33ms<br />
&#160;&#160;&#160; [+] Sets a conference PIN&#160; 18ms<br />
&#160;&#160;&#160; [+] Enables voice mail&#160; 22ms</font><br /></code>
<code><font color="#9b00d3">&#160;&#160; Context Removing Skype for business from a user</font></code><br />
<code><font color="#a5a5a5">&#160;&#160;&#160; [!] Disables S4B and removed any voice number 101ms<br />
&#160;&#160;&#160; [!] Removes voice mail 9ms<br /></font></code>
<code><font color="#000000">Tests completed in 347ms <br />Passed: 4 Failed: 0 Skipped: 2 Pending: 0</font></code>

With larger pieces of work it is possible to use `–skip` and an empty script block for an `It` statement to mean different things (Pester treats the empty script block as “Pending”), so the test output can shows me which parts of the project are done, which are unfinished but being worked on, and which aren’t even being thought about at the moment, so it compliments other tools to keep the focus on doing the things that are in the specification. But when someone says “Shouldn’t it be possible to pipe users into the remove command”, we <u>don’t just go and write the code</u>, we don’t even stop at writing and testing. We bring the example in to show that way of working.
