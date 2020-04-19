---
layout: post
title:  "More PowerShell: A change you should make for V2. (#2)"
date:   2009-07-22
categories: PowerShell
---
PowerShell Modules: A change you should make for V2. (#2) 
2 minutes to read
A few days back [I wrote about](/powershell/2009/06/05/Change-For-PSV2.html)
PowerShell version 2’s ability to confirm whether it should be changing
something. Since I was writing something which would some pretty drastic changes
, supporting –WhatIf and –Confirm for almost no effort seems like a huge win.

The next thing I wanted to cover was modules. I’ve written some quite large
function libraries in PowerShell V1 and I met a few problems which V2 solves by
use of **modules**.

1.  Collaboration. One script with 100 functions isn’t easy to collaborate on. A
    module lets you load multiple script files as one, but different people can
    work on each.
2.  Loading of formatting and type extensions – these XML files can be loaded
    from scripts, but when they are loaded more than once things get untidy.
    Modules can load them along with the code.
3.  While we’re on code, modules allow the loading of a mixture of script files
    and DLLs in one go. Module DLLs don’t need to be registered as snap-ins do,
    so deployment is easier.
4.  Loading at all: an environment variable defines module paths, and you use
    import-module NAME, educating people on dot sourcing was a pain (and part of
    the reason for having a single monolithic file)
5.  Discovery : finding all the functions in a script needed some creativity.
    Now you can just do get-command –module Name

You can turn a script into a module simply by creating a **manifest file**,
which is a text file with a .PSD extension. At its simplest it looks like this
```
@{ ModuleVersion = "1.0.0"
   NestedModules = "Helper.ps1"
```

But there is no reason why there should only be one nested module. So, to
collaborate with different people owning different functions, you just have a
long list in the manifest. Here’s the (only slightly edited) manifest for a
project which I’m just about to publish.
```
@{  ModuleVersion     = "1.0.0"
    NestedModules     = "Firewall.ps1" , "Helper.ps1" , "Licensing.ps1",
                        "network.ps1" , "menu.ps1" , "Remote.ps1",
                        "WinConfig.ps1", "windowsUpdate.ps1", "WinFeatures.ps1"
    GUID              = "{75c6f959-23a1-4673-8ee9-e61e21ff8381}"
    Author            = "James O'Neill"
    CompanyName       = "Microsoft Corporation"
    Copyright         = "© Microsoft Corporation 2009. All rights reserved."
    PowerShellVersion = "2.0"
    CLRVersion        = "2.0"
    FormatsToProcess  = "Config.format.ps1xml" 
}
```

If my manifest file is named `Configurator.psd`, all I need to do is create a
`configurator` sub-folder in one of the folders pointed to by the Environment
variable `PSModulePath` then I can just load it with     
`import-module configurator`    
And I can get rid of the functions if I want to with     
`Remove-Module configurator`   
And if I reload them the fact that I am loading a `.format.ps1XML` file for the second time isn’t a problem as it would be when I
load it from a script. No need to dot source, and the functions are discoverable with 
`Get-command –module configurator` .
Of course, different people can be working on `Firewall.ps1` and `Licensing.ps1` - collaboration problem solved.

As you can see there are quite a few things you can add to the manifest file
over and above the basics – things like FormatsToProcess and TypesToProcess, so
to make it easier to build the file there is a `New-ModuleManifest` cmdlet. There
is plenty more to read about modules, but for starters look at [this post of
Oisin’s](http://www.nivot.org/2008/12/30/PowerShellCTP3AndModuleManifests.aspx)
on Module Manifests.

This post originally appeared on my [technet blog](https://docs.microsoft.com/en-gb/archive/blogs/jamesone/powershell-modules-a-change-you-should-make-for-v2-
22/07/2009) in 2009