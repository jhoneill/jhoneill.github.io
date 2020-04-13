---
layout: post
title:  "Exit, Throw, Return, Break and Continue. A Round up."
date:   2019-08-21
categories: PowerShell
---

PowerShell recognises all the words in the title. I’ve [previously written](/powershell/2019/01/30/dont-just-throw.html) about the problem with assuming that throw will exit from a script or function. Normally it will, but if the error action preference has been changed it might not. So I now put `return` after `throw` to prevent execution running on.

##  Making exceptions for exceptions
`return` is one of those Powershell commands that people get upset about; we can **break normal rules in order to handle exceptions**, I tend to avoid using `throw`, unless I expect something to _want_ to catch the error. I prefer this:    
`Write-Warning "Couldn't do what you wanted with $parameter." ; return`   
to this:    
`throw "$parameter is Evil" ; return`

But when it isn’t really an exception… I was taught that this:
```
if ($result) {return}
Nextcommand
etc
FinalCommand
```
is **wrong** – I can hear “that’s just ‘goto end’, and you know `goto` is evil” and the right way is:
```
if (-not $result) {
  Nextcommand
  etc
  FinalCommand
}
```
However when the “etc” part of that code goes on for a whole screen it gets hard to see that _nothing else happens_ if `$result` evaluated as true; in that case  putting in a comment (“If result was set, the work is complete”) and using return can make things clearer. I’ll come back to this later.

## Break and stopping with “Continue”

There are two other sort-of “goto” commands which can be allowed: `Continue` has one use if you use `trap` instead of `try/catch`, which is to resume execution at the line after the error. It has another use in loops; to save doing the job of an `if` which runs a large amount of the script conditionally, `continue` says “Skip the rest of the work for this item and <u>Continue-with-the-next one</u>.”, it has a companion, `break` which says “Skip the rest of the work on for this item, and <u>don’t bother with any remaining ones</u>.”  Here’s a slightly contrived example, finding primes with the sieve of Eratosthenes

{% highlight powershell %}
$Primes = @()
foreach ($i in 2..100) {
    $isPrime = $true
    foreach ($p in $Primes) {
        if ($i % $p -eq 0 ) {$isPrime = $false; break}
    }
    if (-not $isPrime) {continue}
    Write-Verbose "$i is prime" -Verbose
    $primes += $i
}
{% endhighlight %}
There are two nested loops. The inner one looks at each of the primes already found and sees if the number being looked at divides by any of them. Once we have found one we don’t need to look at any of the others, `break` gets us out of that for loop, but not out of the Outer loop or the script or function that contains it. 
The outer loop does something if the number IS prime, but it uses `continue` to go on to the next number – I said the example was contrived, the if would normally be written the other way round without using `continue`.

## Switch Statements
Both `break` and `continue` work in a `switch` statement if you are using `switch` against a file, `break` stops looking and ignores the rest of the file, and `continue` stops for the current line and continues for the next. I’ve saved the fragment below as deleteMe.Ps1 so it reads itself …

{% highlight powershell %}
switch -Regex -file .\deleteme.ps1 {
    "w" {"Contains W" ; break}
    "o" {"Contains o" ; break}
    default {"Default msg"}
}
{% endhighlight %}
The first line matches on the W in `switch` so it outputs “Contains W” and the statement stops.
If I replace each `break` with `continue` I get
“Contains W”, “Contains W”, “Contains O”, “Default msg”, and “Default msg”.
I.e. each line line is processed for a maximum of one match; And if I remove break/continue, the second line matches both W in the quotation marks and the O  in “contains” so I get
“Contains W”, “Contains W”, “Contains O”, “Contains O”, “Default”, and “Default”.

But that is the less common way to use switch this is more usual:

{% highlight powershell %}
$s = "Hello World"
switch -Regex ($s) {
    "w" {"Contains W" ; }
    "o" {"Contains o" ; }
    default {"Default msg"}
}
{% endhighlight %}
Here, without break or continue the value matches two values and outputs “Contains W”, “Contains O”, because only one value is examined, both **break or continue have the same effect**. Default only gets run if nothing matches. Often I’ll see a switch statement that could be written like this: 
{% highlight powershell %}
switch ($birthday.tostring("dddd")) {
   "Monday"      {"fair of face"}
   "Tuesday"     {"full of grace"}
   "Wednesday"   {"full of woe"}
   #etc
}
{% endhighlight  %}
and although the values don’t overlap and all there isn’t an “Output this for ‘none of the above’”  (we’ve written the case for each of the days) the writer has carefully added `Break` or `Continue` to each of the blocks and added a default block which only contains Continue. Does putting these things in and being absolutely explicit make things clearer ? I don’t think so. Putting in an empty “else” is just more to read; and the continue is a stylistic tick – because it is needed sometimes, and it is harmless when it is not needed why not put it in always?  It’s more typing, more to read and some people will focus on the tick. 

`Break` and `Continue` work **anywhere**. `Switch`, `while`, `for`, and `foreach` statements handle them as “exit from this statement”, other commands (including `if` and the `ForEach-Object` cmdlet) treat them as “exit from where this is running”  So I could write this:    
`Write-Warning "Couldn't do what you wanted with $parameter." ; break`    
or this:    
`Write-Warning "Couldn't do what you wanted with $parameter." ; continue`     

But using “Continue” to exit from a function or script is showing off in a “I know a trick that I bet you don’t” kind of way. It hinders when someone else is dealing with my code. Lately I find I keep repeating the importance of clarity, some people like to say “Imagine the next person who looks at this is an axe wielding maniac who knows where you live”; I imagine that the next person will be me, and people will be screaming that something is broken, it’s late, I’m tired and I have forgotten ever writing the script.    

Since I’ve returned to the example that used `return`, it’s worth taking a moment to mention that [I have written about implicit or explicit return](/powershell/2017/06/16/WriteOutput-and-Return.html) before. Some people habitually write `return $result` as the last line of their function / script ; which sets other people’s teeth on edge. I tend to only write that if, somewhere before the end of a function / script I would otherwise write
```
$result
return
```

As I said at the start, my computer science training would tell me that I should write this way : 
```
#try a quick way to get the result
$result = simpleCommand
if (-not $result) {
    complex | pipeline -of "commands"
}
else {$result}
```
But I think the return in the next example is OK: it is clearer so say “if we got a result return it, otherwise do X, Y and z;” than to write it the other way around “if we didn’t get the result do X,Y and Z, otherwise return the result”. (If one clause is simple and one is complex, put the simple clause in the IF ).   
```
#try quick way
$result = simpleCommand
if ($result) {return $result}       
complex | pipeline -of "commands"  
```
but this next return is unnecessary
```
$Result = complex | pipeline -of "commands"     
return $result     
```
## And finally … Exit
And then there is `Exit`. Exit says “Leave what you are running” at the PowerShell prompt it is _Leave PowerShell_, in a script it is _Leave the script_. `Exit` can return an _exit code_. If a script wants to tell another script or PowerShell itself what happened it should really send output or throw errors; some people really don’t like seeing `Exit` in a script and often it’s just old habits refusing to die. Codes don’t help fix a problem “Error 4096 occurred” doesn’t help users understand what _did_ go wrong but makes them feel worse for not knowing 4095 other things that _might_ have gone wrong.  But sometimes an error code is the only way to to tell something which called the script what happened.

However in a script _exit_ doesn’t always behave as people expect:     
`PowerShell "something"` is treated as `PowerShell –Command "something"` which  works like this:

-  It starts PowerShell.exe (Windows Powershell - the cross platform / core version is pwsh),
-  It runs the command and returns any output
-  Because -command was specified and –NoExit wasn’t, PowerShell exits. If the last command ran to completion the exit code is 0; if the last command threw a terminating error the exit code is 1.
So

-  If I run `PowerShell –Command "1/0"` from an existing instance of PowerShell $LASTEXITCODE is 1 ;
-  If I run `PowerShell –Command "1/0 ; hostname"`  $LASTEXITCODE is 0 because the last command ran to completion and past errors are forgotten.
-  If I run `PowerShell –Command "MyScript.ps1"` the rules don’t change. PowerShell returns an exit code of 1 or 0 depending on whether the last (only) command threw an error. If the script ends with exit 123 then $LASTEXITCODE is 123 <u>in that instance</u> of PowerShell and in that instance something else can see that exit code.    
Then when that <u>instance of PowerShell</u> exits, it follows the standard rules – the Exit code from the script is lost.

I’m sure someone must want this behaviour, but there are multiple ways to get the script’s exit code back; one is to make a command which runs the script, and explicitly exits with the code it returns, like this:
`powershell 'MyScript.ps1; exit $lastexitcode'`  
A better way is to tell PowerShell this is **not a command, but a file**. That <u>does</u> return the the error code from the script      
Don’t run  `PowerShell 'MyScript.ps1'` instead run      
`PowerShell –File 'MyScript.ps1'`.

In PowerShell \[core\] 6 and later file is the default `Pwsh 'MyScript.ps1'` implies `–File ` and the _command_ version needs the explicit `–command`. It's good to get into the habit of being explicit with the switches in either environment, so if/when a script/command changes between powershell and pwsh versions it always does as you expect.

One final way is to put `$host.setshouldExit(123)` in the script. This time, when PowerShell exits it has been primed to leave with a specific code. Which is better ? Of course, it depends. It you want to test the script by running it in PowerShell and looking at `$lastExitCode` then using exit in the script might be better, but it relies on others not just running `powershell MyScript` The second way (with information written to error or verbose) avoids that, and as a bonus lets you set what code should go back from PowerShell to the caller if a terminating error happens in specific section of code, and then change the code for the next section and so on.
