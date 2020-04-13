---
layout: post
title:  "PowerShell. Don’t Just Throw"
date:   2019-01-30
categories: PowerShell
---
I write  `; return`  every time that I put `throw` in my PowerShell scripts. I didn’t always do it. Sooner or later I’ll need to explain why.

First off: when something throws an error it is kind-of ugly, and it can stop things that we don’t to be stopped, sometimes **Write-Warning is better** than throw.  But many (probably most) people don’t realise the assumption they’re making when they use _throw_.

Here’s a simple function to demonstrate the point
{% highlight powershell%}
function test {
    [cmdletbinding()]
    Param([switch]$GoWrong)
    Write-verbose "Starting ..."
    if ($GoWrong) {
        write-host "Something bad happened"
        throw "Failure message"
    }
    else {
        Write-Host "All OK So Far"
    }
    if ($GoWrong) {
        write-host "Something worse happens. "
    }
    else {
        Write-Host "Still OK"
    }
}
{% endhighlight %}

So some input causes an issue and to prevent things getting worse, the function throws an error. I think almost everyone has written something like this (and yes, I’m using `Write-Host` – those messages are decoration for the user to look at not output , I could use `Write-Verbose` with `–Verbose` but then I’d have to explain… why the verbose switch, just try it )

I can call the function    
<code>&gt;test<br/>
All OK So Far<br/>
Still OK</code>

or like this    
<code>&gt;test -GoWrong<br/>
Something bad happened<br/>
<span style="color:red">Failure message<br/>
At line:9 char:9<br/>
\+         throw "Failure message"</span></code>

Exactly what’s expected – **where’s the problem?** no need to put a return in is there ?
Someone else takes up the function and they write this.

{% highlight powershell%}
Function test2 {
    Param([switch]$Something)
    $x = 2 + 2 #Really some difficult operation
    test -GoWrong:$Something
    return $x
}
{% endhighlight %}
This function does some work, calls the other function and returns a result

<code>&gt;Test2<br/>
All OK So Far<br/>
Still OK<br/>
4<br/>
</code>

But _some_ input results in a problem.

<code>&gt;test2 -Something<br/>
Something bad happened<br/>
<span style="color:red">Failure message<br/>
At line:9 char:9<br/>
\+         throw "Failure message"</span></code>

That `throw` in the first function was for protection but it has lost some work. And the author of Test2 doesn’t like big lumps of “blood” on the screen. What would you do here? I know what I did, and it wasn’t to say “Oh somebody threw something, so I should try to catch it” and start wrapping things in `try {} catch {}`. I said “One quick change will fix that!”
{% highlight powershell%}
    test -GoWrong:$Something -ErrorAction SilentlyContinue
{% endhighlight %}

Problem solved.

What do you think happens if I run that command again ? I’m certain a lot of  people will get the answer wrong, and I’m tempted to say copy the code into PowerShell and try it, so that you don’t read ahead and see what happens without thinking about it for a little bit.  Maybe if I waffle for a bit… Have you thought about it ? Really ? OK, This is what happens.

<code>&gt;test2 -Something<br/>
Something bad happened<br/>
Something worse happens.<br/>
4</code>

The change got rid of the ‘blood’, and the result came back. But… the second message got written – execution continued into exactly the bit of code which had to be prevented from running. Specifying the error action **stopped the throw doing anything.**

Discovering that made me put a return after every throw, even though it should be redundant more than 99% of the time. And I now think any test of error handling should include changing the value of `$ErrorActionPreference` to ensure things stop when the preference is for all errors to silently continue.
