---
layout: post
title:  "Powershell and Paradigms of VB"
date:   2007-08-19
categories: PowerShell
---
I have talked about experiences from long ago in other posts and, here's
another. When I was at University, one of my tutors, Dr Brian Lings, used the
word "Paradigm" in a lecture. And we all went "WHAT !" and went running to
dictionaries to find out what he meant. He actually used it in the context of
looking at something written in one language but actually having the
way-of-doing-things that you'd see in another.

I was reminded of this when I was asked to help with some PowerShell scripts for
the OCS resource kit. I was sent a script which looked like this.
{% highlight PowerShell %}
# Get all SIP domains
$SIPDomains = Get-WmiObject -class MSFT_SIPDomainData

# For every SIP domain in environment write SIP domain info to standard output
foreach ($SIPDomain in $SIPDomains) {
    write-host "*******************************"
    write-host SIP domain: $SIPDomain.Address
    write-host "*******************************"
    write-host \`t Default Domain : $SIPDomain.DefaultDomain
    write-host \`t Authoritative : $SIPDomain.Authoritative
}
{% endhighlight %}
Getting the WMI Object class `MSFT_SIPDOMAINdata` gives a collection of Objects:
one per domain. So, we store that collection in a variable and use a `foreach` loop to
go through each object and write its properties to the screen. Simple.

Well… except for that fact that `Write-Host` (despite the comment) isn't sending to "Standard Output" and can't be redirected to a file. We
can't use  
<code> &nbsp;&nbsp;&nbsp;&nbsp; Get-SIPDomains &gt; domains.txt</code>    
But even fixing that, this isn't PowerShell "style" - though I'm still learning
what that is, exactly. It's written like we'd write VB.

Using LISP at university meant that grew to like one function to feed into the
next, so piping functions into each other seems natural and PowerShell gives me
cmdlets like `Format-Table` to take the results of `Get-WMIobject` and reduce the
`For` loop to one line.

`Get-WmiObject -class MSFT_SIPDomainData | Format-Table Address, Authoritative, DefaultDomain.`

I'm beginning to see why they named it POWER shell; one of the other scripts I
need to write for the OCS resource kit needs to get information from the event
log. I was presently surprised to find PowerShell has a `Get-EventLog` cmdlet.
With `Where-Object` to filter the events I can do this in one line.

This post orignally appeared on my [technet blog](https://docs.microsoft.com/en-gb/archive/blogs/jamesone/powershell-and-paradigms-of-vb) in 2007