---
layout: post
title:  "Customizing PowerShell, Proxy functions and a better Select-String"
date:   2012-02-04
categories: PowerShell
---

I suspect that even regular PowerShell users don’t customize their environment much. By co-incidence, in the last few weeks I’ve made multiple customizations to my systems (my scripts are sync’d over 3 machines, customize one, customize all). Which has given me multiple things to talk about. My last post was about adding persistent history, this time I want to look at Proxy Functions…

`Select-String` is, for my money, one of the best things in PowerShell. It looks through piped text or through files for anything which matches a regular expression (or simple text) and reports back what matched and where with all the detail you could ever need. **BUT** it has a couple of things wrong with it: it won’t do a recursive search for files, and sometimes the information which comes back is **too** detailed. I solved both problems with a function I named **“WhatHas”** which has been part of my profile for ages. I have been using this to search scripts, XML files and saved SQL whenever I need a snippet of code that I can’t remember or because something needs to be changed and I can’t be sure I’ve remembered which files contain it. I use `WhatHas` dozens (possibly hundreds) of times a week. Because it was a quick hack I didn’t support every option that `Select-String` has, so if a code snippet spans lines I have go back to the proper `Select-String` cmdlet and use its `-Context` option to get the lines either side of the match: more often than not I find myself typing `dir -recurse {something} | Select-String {options}`

A while back I saw a couple of presentations on Proxy functions (there’s a [good post about them here by Jeffrey Snover](https://devblogs.microsoft.com/powershell/extending-andor-modifing-commands-with-proxies/)): I thought when I saw them that I would need to implement one for real before I could claim to understand them, and after growing tired of jumping back and forth between Select-String and WhatHas, I decided it was time to do the job properly creating a proxy function for `Select-String` and keep `WhatHas` as an alias.

There are 3 bits of background knowledge you need for proxy functions.

1.  Precedence. Aliases beat Functions, Functions beat Cmdlets. Cmdlets beat external scripts and programs. A function named `Select-String` will be called instead of a cmdlet named `Select-String` – meaning a function can replace a cmdlet simply by giving it the same name. That is the starting point for a Proxy function.
2.  A command can be invoked as `moduleName\CommandName`. If I load a function named “Get-Stuff” from my profile.ps1 file for example, it won’t have an associated module name but if I load it as part of a module, or if “Get-Stuff” is a cmdlet it _will_ have a module name.
`Get-Command get-stuff | format-list name, modulename`
will show this information You can try
`> Microsoft.PowerShell.Management\get-childitem`
For yourself. It looks like an invalid file-system path, but remember PowerShell looks for a matching Alias, then a matching Function and a then a matching cmdlet before looking for a file.
3.  Functions can have a process block (which runs for each item passed via the pipeline) a begin block (which runs before the first pass through process, and an end block (which runs after the last item has passed through process.) Cmdlets follow the same structure, although it’s harder to see.

Putting these together **A function named Select-String can call the Select-String cmdlet, but it must call it as Microsoft.PowerShell.Utility\Select-String** or it will just go round in a loop. In some cases, calling it isn’t quite enough and PowerShell V2 delivered the steppable pipeline which can take a PowerShell command (or set of commands piped together) and allow us to run its begin block , process block, and end block, under the control of a function. So a Proxy function looks like this :

{% highlight powerShell %}
Function Select-String {
  [CmdletBinding()]
  param  (  Same Parameters as the real Select-String
            Less any I want to prevent people using
            Plus any I want to add
         )
  begin  {  My preliminaries
            Get $steppablePipeline
            $steppablePipeline.begin()
          }
  process { My Per-item code against current item ($_ )
            $steppablePipeline.Process($_)
          }

  end     { $steppablePipeline.End
            My Clean up code
          }
}
{% endhighlight %}

What would **really help** would be something produce a function like this template, and fortunately it is built into PowerShell: it does the whole thing in 3 steps: Get the command to be proxied, get the detailed metadata for the command and build a Proxy function with the meta data, like this:

{% highlight powerShell %}
  $cmd=Get-command select-string -CommandType cmdlet
  $MetaData = New-Object System.Management.Automation.CommandMetaData ($cmd)
  [System.Management.Automation.ProxyCommand]::create($MetaData)
{% endhighlight %}

The last command will output the Proxy function body to the console, I piped the result into Clip.exe and pasted the result into a new definition
Function Select-String { }
And I had a proxy function.

At this point it didn’t do anything that the original cmdlet doesn’t do but that was a starting point for customizing.
The auto-generated parameters are be formatted like this

{% highlight powerShell %}
  [Parameter(ParameterSetName='Object', Mandatory=$true, ValueFromPipeline=$true)]
  [AllowNull()]
  [AllowEmptyString()]
  [System.Management.Automation.PSObject]
  ${InputObject},
{% endhighlight %}
And I removed some of the line breaks to reduce the screen space they use from 53 lines to about half that.
The ProxyCommand creator wraps parameter names in braces just in case something has a space or other breaking character in the name, and I took those out.
Then I added two new switch parameters `-Recurse` and `-BareMatches`.

Each of the Begin, Process and End blocks in the function contains a `try...catch` statement, and in the try part of the begin block the creator puts code to check if the -OutBuffer common parameter is set and if it is, over-rides it (why I’m not sure) – followed by code to create a steppable pipeline, like this:

{% highlight powerShell %}
  $wrappedCmd = $ExecutionContext.InvokeCommand.GetCommand('Select-String',
                                                           [System.Management.Automation.CommandTypes]::Cmdlet)
  $scriptCmd = {& $wrappedCmd @PSBoundParameters }
  $steppablePipeline = $scriptCmd.GetSteppablePipeline($myInvocation.CommandOrigin)
{% endhighlight %}
I decided it would be easiest to build up a string and make that into the steppable pipeline . In simplified form

{% highlight powerShell %}
  $wrappedCmd        = "Microsoft.PowerShell.Utility\Select-String "
  $scriptText        = "$wrappedCmd @PSBoundParameters"
  if ($Recurse)      { $scriptText = "Get-ChildItem @GCI_Params | " + $scriptText }
  if ($BareMatches)  { $scriptText += " | Select-Object –ExpandProperty 'matches' " +
                                      " | Select-Object -ExpandProperty 'value'   " }
  $scriptCmd         = [scriptblock]::Create($scriptText)
  $steppablePipeline = $scriptCmd.GetSteppablePipeline($myInvocation.CommandOrigin)
{% endhighlight %}

`& commandObject` works in a scriptblock: the “&” sign says  “run this” and if _this_ is a command object that’s just fine: so the generated code has `$scriptCmd = {& $wrappedCmd @PSBoundParameters }` where` $wrappedCmd` is a command object.
but when I first changed the code from using a script block to using a string I put the original object $wrappedCmd inside a string. When the object is inserted into a string, the conversion renders it as the unqualified name of the command – the information about the module is lost, so I produced a script block which would call the function, which would create a script block which would call the function which… is an effective way to cause a crash.

The script above won’t quite work on its own because
(a) I haven’t built up the parameters for `Get-Childitem`. So if `-recurse` or `–barematches` are specified I build up a hash table to hold them, using taking the necessary parameters from what ever was passed, and making sure they aren’t passed on to the `Select-String Cmdlet` when it is called. I also make sure that a file specification is passed for a recursive search it is moved from the path parameter to the include parameter.
(b) If `-recurse` or `-bare` matches get passed to the “real” `Select-String` cmdlet it will throw a “parameter cannot be found” error, so they need to be removed from `$psboundParameters`.

This means the first part of the block above turns into

{% highlight powerShell %}
  if ($recurse -or $include -or $exclude) {
     $GCI_Params = @{}
     foreach ($key in @("Include","Exclude","Recurse","Path","LiteralPath")) {
          if ($psboundparameters[$key]) {
              $GCI_Params[$key] = $psboundparameters[$key]
              [void]$psboundparameters.Remove($key)
          }
     }
     # if Path doesn't seem to be a folder it is probably a file spec
     # So if recurse is set, set Include to hold file spec and path to hold current directory
     if ($Recurse -and -not $include -and ((Test-Path -PathType Container $path) -notcontains $true) ) {
        $GCI_Params["Include"] = $path
        $GCI_Params["Path"] = $pwd
     }
   $scriptText = "Get-ChildItem @GCI_Params | "
}
else { $scriptText = ""}
{% endhighlight %}


And the last part is
{% highlight powerShell %}
if ($BareMatches) {
  $psboundparameters.Remove("BareMatches")
  $scriptText += " | Select-object -expandproperty 'matches' | Select-Object -ExpandProperty 'value' "
}
$scriptCmd = [scriptblock]::Create($scriptText)
$steppablePipeline = $scriptCmd.GetSteppablePipeline($myInvocation.CommandOrigin)
{% endhighlight %}

There’s no need for me to add anything to the process or end blocks, so that’s it – everything `Select-String` originally did, plus recursion and returning bare matches.
