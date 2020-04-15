---
layout: post
title:  "Including PowerShell classes in modules. A quick round-up"
date:   2020-04-2
categories: PowerShell
---

Recently I have been adding PowerShell classes to modules and this post is what I’ve learned about where the class code should be located and how different ways of loading it have odd looking side effects. At some point I’ll make a list of people who should get some credit for educating me, because I would not have discovered all of it on my own.

From the start, PowerShell has been able to use the `Add-Type` command to compile a type written in C# (or Visual Basic) and load it into the PowerShell session like any other class. Windows PowerShell 5 added the ability to code classes in PowerShell and there’s little, if any, change in version 6 and 7. However there are some quite unusual behaviours when classes are used in modules,  which don’t seem well documented. The following applies to the classes written in PowerShell (not the ones loaded with Add-Type)

1.  A PowerShell class is only visible outside its own module if EITHER.    
   a. It is loaded in the PSM1 file (not dot sourced into the PSM1) AND the module is loaded with the using module directive. OR    
   b. It is loaded from a PS1 file using the `ScriptsToProcess` section of the PSD1 manifest.
2.  If a module is loaded by a script which specifies Using module, classes from its PSM1 are only visible in the script’s scope (inside a ps1, unless the file is Dot sourced.)
The same scope issues/rules apply if the class is loaded from `ScriptsToProcess` and the module is loaded with `Import-module`
3.  If a class used as a parameter attribute in a function is not visible in the scope where that function is called it cannot be abbreviated by dropping the Attribute from its name.
In other words, a parameter attribute which has been shortened from `[ValidateWibbleAttribute()]` to `[ValidateWibble ()]` may cause failures if the module is loaded in the wrong way.    
point [2] above should make it clear that sooner or later any module _will_ be loaded the wrong way.

**Based on this**
1.  My view of a .PSM1 file as a small “loader” – which I think is a common one - needs to at least be revised enough to allow for Classes to be included in it.
I’m still not keen on having _everything_ in the PSM1 and will probably continue to keep functions in their own PS1 files, but functions can be successfully dot sourced into the PSM1 and classes cannot.
2.  `Using module X` is preferable to `Import-Module X`, where the module has some requirement which means it won’t run on legacy Windows PowerShell.
In other words,V4 and earlier are still out in the field – and for modules which work with those versions you might prefer `import` to `using`. However V4 won’t import a module with classes so with modules which need V5/6/7, sticking to Import-Module doesn’t gain anything and makes classes invisible
3.  I’m no longer trimming ‘attribute’ off the names of parameter attribute class names when declaring parameters. I’m not sure at the moment if there is any reason to use attribute in the class name (it appears not).
