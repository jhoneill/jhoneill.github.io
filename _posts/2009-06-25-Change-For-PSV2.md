---
layout: post
title:  "More PowerShell: A change you should make for V2. (#1)"
date:   2009-06-05
categories: PowerShell
---

There will be a couple more posts on changes for V2, of PowerShell but I want to
get something really clear up front:  
All V1 PowerShell should work in V2. Should meaning unless you have been
*really* stupid or are *very* unlucky; you won’t *NEED* to change anything.  
OK, now we’re clear on that here’s something you’re going to want to change if
you ever write functions which change the state of the system…

In PowerShell V1 I had to explain to people that anything which produced results
would have its output go to the console if it wasn’t sent anywhere else.
Sometimes that meant explicitly writing something to the console so the user
could see it without it getting merged into output that was used in another
function (see [This is not an an output](/powershell/2007/08/28/Pipes-This-is-not.html)).
With more knowledge of PowerShell people would discover the `Write-Verbose` and
`Write-Debug` cmdlets, so the user could set a variable and get more output or
less. This was good, but it meant setting a variable globally, and setting it
back when you were done – unlike compiled cmdlets which could use common
parameters like `–Verbose` and `–Debug`.  
The other thing I’d explain for V1 was that potentially harmful things could
take `–Confirm` and `‑Whatif` switches. I regularly use this:
{% highlight Powershell%}
dir \*.jpg | ren -NewName {$_.name -replace "IMG_","DIVE"} –WhatIf
{% endhighlight %}
it renames photos, in this case from “IMG_1234” to “DIVE_1234” but because I
don’t trust myself not to louse up the typing of the bits I’m replacing, I can run
it with `–WhatIf`, and if that works, recall the line and delete `–Whatif` to do it
for real. Fantastic, but if you wrote a function in version 1 you had to do it
all yourself. That changes in version 2 and it is *really* easy, here is a
function which uses the new feature
{% highlight Powershell%}
function Disable-AutoPageFile{
  [CmdletBinding(SupportsShouldProcess=$True)]
  param()
    $pc = Get-WmiObject -class win32_computerSystem -Impersonation 3 -EnableAllPrivileges
    $pc.AutomaticManagedPagefile = $false
    If ($psCmdlet.shouldProcess("Local computer" , "Disable automatic page file")) {
         $pc.Put() | Out-Null 
    }
}
{% endhighlight %}

Easy stuff, get a WMI object, change a property , save it back. But this turns
off the Windows PageFile on a server, serious stuff. So, we want `–WhatIf`
`–Confirm` and so on. The first line in the code-block hooks the function up with
the support it needs (it seems you must have a `param()` statement, even if it is
empty when you use this) and then `$psCmdlet.shouldProcess` does the magic so
let’s run the command:
```
  PS \> Disable-AutoPageFile -whatif
  What if: Performing operation "Disable automatic page file" on Target "Local computer".

  PS \> Disable-AutoPageFile -Confirm Confirm
  Are you sure you want to perform this action?
  Performing operation "Disable automatic page file" on Target "Local computer".
  [Y] Yes [A] Yes to All [N] No [L] No to All [S] Suspend [?] Help (default is "Y"):
  
``` 
Woo hoo ! I’m used to PowerShell saving me time, but how much has this just
saved me ? No need put `–Confirm` or `–Whatif` switches (and the rest) into Param() and **no need to code for them**, `shouldprocess()` tells the
function if it should go ahead with the change to the system. In fact the amount
of time it would have taken to do this before meant `I simply wouldn’t have
bothered`: now it’s so easy the only excuse for NOT doing it is if you need to
keep working with V1. And bluntly, this is a reason to go to V2 at the first
opportunity.

[Jeffrey put up a
post](https://devblogs.microsoft.com/powershell/test-pscmdlet/) on `pscmdlet` some
way back, you can modify the `cmdletbinding` line to match mine above and have a
play with `shouldprocess` as well. You can see the text in the call to
`ShouldProcess` got turned into the “operation” and “target” parts in the message,
but here’s the summary of what it does with the different switches.

<table>
<tr><td><code>-Confirm</code></td><td> Result returned depends on user input, uses the message as a prompt</td></tr>
<tr><td><code>-WhatIf</code></td><td> Always returns false, the message is displayed prefixed with WHATIF:</td></tr>
<tr><td><code>-Verbose</code></td><td> Always returns true, the message is displayed in an alternate colour prefixed with VERBOSE (as with <code>Write-Verbose</code>)</td></tr>
<tr><td>&lt;none of these&gt;</td><td>Always returns true</td></tr>
</table>
This post originally appeared on my [TechNet
blog](https://docs.microsoft.com/en-gb/archive/blogs/jamesone/more-powershell-a-change-you-should-make-for-v2-1) in 2009
