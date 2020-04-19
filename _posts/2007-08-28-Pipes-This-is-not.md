---
layout: post
title:  "Powershell again. Pipes, and 'This is not an output'"
date:   2007-08-28
categories: PowerShell
---
My experiences with PowerShell continue and every so often realize that I never
really understood something I thought I'd “got” some while back. I've spent most
of my time trying *not* to use the `Write-` cmdlets to send output to the screen.
After all, if I want to write "Hello, World" to the screen I can have a function
{% highlight powershell %}
function hw {
    'Hello, World'
}
    
{% endhighlight %}
No need for a `Print`, `Write`, `Echo` or any other command: in PowerShell, an output
with nowhere else to go ends up on the screen.     
If I want formatted output,
`Format-List` and `Format-Table` do a great job, especially since I "discovered"
calculated fields. Since I always used to get my queries in Access to do loads
of work, shoving the work onto  `Format-Table` seems natural enough, even if the
syntax is little weird. A calculated field is written as a hash table:    
`@{label="Text", Expression={some code} }`. Of course, "Some code" can do *anything
you want* and what it returns goes in the table. So, I had a bright idea over
the weekend. I can have a menu using  `Format-Table`. Here the code:
{% highlight powershell %}
function Choose-Process {
  $global:Counter=-1
  $proc=Get-Process
  Format-Table -inputobject $Proc @{ Label = "ID"; Expression={($global:counter++) }} , processName
  $proc[(Read-Host "Which one ?")]
}

{% endhighlight %}
So I set a counter, get a list of processes, and then format a table; in
formatting the table I *output and increment the counter*. Then I ask the user to
input a number to choose one and that gives me the offset into the array of
processes of the one to return. When I run it, here's what I get:
```
    PS C:\Users\Jamesone\> choose-proc
    ID ProcessName
    -- -----------
    0 audiodg
    1 CcmExec
    Which one ?: 1

    Handles NPM(K) PM(K) WS(K) VM(M) CPU(s)   Id ProcessName
    ------- ------ ----- ----- ----- ------   -- -----------
        744     29 13128 10560    92        2204 CcmExec
        
```
Of course, I want to save the result, so I type in `$proc=choose-proc` and I get this:    
<code>&nbsp;&nbsp;&nbsp;&nbsp;Which one ?:</code>    
What the ... ? **Where did my menu go ?** And the answer is: *the menu was
OUTPUT*. Where did I tell PowerShell that `Format-Table` *wasn't* to send to
standard output but `$proc[offset]` *was*? I didn't: so *both* ended up being
stored in the result.

Anyone who's explored PowerShell's *providers* will have found there is one for
variables. *Now I get it*  
`$varName= MyFunction` is equivalent to `MyFunction > Variable:varName`.    
If it goes to standard output it goes into the variable. This behaviour can work for or against you depending on the situation, and *here*
I need to pipe `Format-Table` into `Out-Host` to force output to go the **console and not to standard Output**

I guess I was still *thinking in Basic*, where the last line of your function
usually returns the result. PowerShell's Outputs are different, they go down the
"Standard Output" pipe unless told otherwise and standard output ends up on the
screen if the "end of pipe" is left Open; just like you'd expect from a Shell.
And since I was [quoting my university
lecturers](/powershell/2007/08/19/Paradigms-of-VB.html) a little while ago
here's something which was written on the board in our first programming class.

*It is practically impossible to teach good programming style to students that
have had prior exposure to BASIC; as potential programmers they are mentally
mutilated beyond hope of regeneration.*    
( Edsger Dijkstra )

This post originally appeared on my [TechNet
blog](https://docs.microsoft.com/en-gb/archive/blogs/jamesone/powershell-again-pipes-and-this-is-not-an-output) in 2007