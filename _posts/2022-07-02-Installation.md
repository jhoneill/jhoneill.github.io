---
layout: post
title:  "Getting installed Software with PowerShell"
date:   2022-07-02
tags: 
    - PowerShell
---

Sadly, "what is on this computer?" isn't the simple question it seems to be. Discovering what has been installed and what patches have been applied can be found in multiple ways - they overlap but on their own none provides a complete least of original installations + patches.

## The way NOT to do it

One method I *have* used, and still see recommended is to use  `Get-CimInstance Win32_Product`.  
**Don't**  
It is slow, but that is a symptom  rather than the real reason to avoid it- it does a **consistency check** of all installed software, which is slow and can change it! There is [more detail here](https://xkln.net/blog/please-stop-using-win32product-to-find-installed-software-alternatives-inside/ ). If this method gave a full list, maybe we could forgive it, but it doesn't.

## The commonly recommended way

A better way, and one which finds things others don't, is to check the `uninstall` areas of the registry. There are two system ones, and potentially 2 more for each user, the link I just gave has a script to check users other than the current one. I've used variations of the following code to check for some years now - I don't know the original source:

{% highlight PowerShell %}
 @('HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*',
   'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*',
   'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*',
   'HKCU:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*') | where {Test-path $_} |
    Get-ItemProperty |  Where-Object {$_.DisplayName -and -not $_.SystemComponent} |
        Sort-Object DisplayName |
            Select-Object -Property @{n='Name';     e='DisplayName'   },
                                    @{n='Version';  e='DisplayVersion'}, "Publisher",
                                    @{n='ID';       e='PSChildName'   }
    
{% endhighlight %}

There are four branches, 64-bit and 32-bit in both HKey_CurrentUser, and in HKey_LocalMachine.
Some 64-bit *programs* are installed by a 32-bit *installer* (and I guess the reverse is possible), and
these are listed in the 32-bit branch of the registry - `WOW6432.`  
I don't have any 32-bit "user installations" and the `Test-Path` avoids an error.  
There are some stub entries which don't give helpful information, and the `where` in the code above
filters those out, together with items which are flagged as components that are installed as part of something else.  
Other properties are left to the publisher, "Version" is missing from a few, "Size" from more, and
install dates - where present - are a mixture of yyyyMMdd and 3/31/2022 US format.  
This source replicates the list shown in Control Panel -it's **good** but it is **not complete**,
it has *some* drivers but not all of them, none of the Windows store apps appear here, and we don't see updates and hotfixes.

![Software from the Registry's uninstall branches](/assets/Install_reg.png)

## Winging it with wing-et

*(Side note - the & character is the letters E and t combined so I'm tempted to write it as Wing-& )*  
WingEt is a relatively new package manager for Windows, with a frankly dreadful command line app,
(written in C++ for that authentic 1980s feel). It needs to run in a elevated session even when not
making changes (a side effect of its poor design.) It can get a very similar list to the one just
described, with a bonus that software distributed through the winge-t repository is called out and
where newer versions are available they are highlighted. Sorting the output was too much trouble
for the team that wrote it, and because it doesn't output objects, parsing the output to sort in
PowerShell is your problem<sup>\*</sup>. It truncates names and IDs when printing, so even if you get
something that you *can* sort, some data will be broken. But if you're using it, it's another way
to get a partial listing.

![Winget - broken names and IDs, unsorted, marginally better than nothing](/assets/Install_winget.png)

## Filling in missing *apps*

Windows store applications can be checked with :

{% highlight PowerShell %}
     Get-AppxPackage -AllUsers | Sort-Object Name ,Version |
        Select-Object Name, Version, Publisher,  NonRemovable, SignatureKind | Format-Table
    
{% endhighlight %}

Some apps are signed by the store rather than by the developer, and these will have a GUID for
the publisher; for dev-signed apps the publisher is a distinguished name - which I'd assume is
the one on the cert that signed the app.  
The names here can be quite cryptic: for example, I had "Age of Empires" installed which has a
name of `Microsoft.MSPhoenix`. Each app has a manifest file and in some cases this holds
the app's and publisher's display names as literal text, in others it looks like a link to a
resource in one of the app's binary files. The list from the previous code fragment can be improved
by filling in names where they *can* be found in the manifests, and using parts of the existing
attributes where the manifest doesn't help.  
Filtering out items which have "IsFramework" or "NonRemovable" flags set gives a list which is
more like the apps we see installed. There is a `Get-AppxPackageManifest` command but it doesn't
work with this script, so I cheat and read the manifest XML directly.

{% highlight PowerShell %}
Get-AppxPackage -AllUsers | where {-not ($_.IsFramework -or $_.NonRemovable)} |
     Select-Object Name, Version, InstallLocation, Publisher, NonRemovable, IsBundle, IsFramework -Unique | ForEach-Object {
         $props = ([xml](Get-Content (Join-path $_.InstallLocation "Appxmanifest.xml"))).Package.Properties
         if ($Props.DisplayName -notlike 'ms-resource*') {
                Add-Member -InputObject $_ -NotePropertyName DisplayName -NotePropertyValue $props.DisplayName
         }
         else { Add-Member -InputObject $_ -NotePropertyName Displayname -NotePropertyValue ($_.name -replace "^.*?\.","")}
         if ($Props.PublisherDisplayname -notlike 'ms-resource*') {
                Add-Member -InputObject $_ -NotePropertyName Pub -NotePropertyValue $props.PublisherDisplayname -PassThru
         }
         else { Add-Member -InputObject $_ -NotePropertyName Pub -NotePropertyValue ($_.publisher -replace '^cn=(.*?),.*$','$1') -PassThru}
     } | Sort-Object DisplayName | 
          select-Object @{n='Name';e='DisplayName'}, Version, @{n='Publisher';e='Pub'} 
    
{% endhighlight %}

![Appx Packages installed for users](/assets/Install_store_user.png)

These packages record their dependencies, and we see can which items are used most - some of them will have "NonRemovable" or "IsFrameWork" set so won't show up in the previous list.

{% highlight PowerShell %}
$packNameToId = @{} 
Get-AppxPackage | ForEach-Object { 
        $packNameToId[$_.PackageFullName] = $_.Name 
        foreach ($d in ($_.Dependencies| Sort-Object -Unique)) {[PSCustomObject]@{"Name"=$_.PackageFullName; DependsOn=$d}} 
    } | Group-Object Dependson -NoElement | Sort-Object Count | 
        Select-Object -Property Count, @{n='shortname'; e={$packNameToId[$_.name] }} | where-object shortname
    
{% endhighlight %}

![Appx Package dependencies](/assets/Install_store_depend.png)

`Get-AppxPackage` returns a list which is a mixture of apps added by users and apps that Windows pre-installs, but it is not the complete list because some apps are *staged* on the computer: a store app may be *installed* for the user (e.g. "Age Of Empires" downloaded from the store), *Staged and installed* (e.g. "Microsoft.Paint"), *Staged but either never installed or uninstalled* (e.g. "Microsoft.YourPhone") or staged with a *different version* installed.

{% highlight PowerShell %}
    Get-AppxProvisionedPackage  -Online | Sort DisplayName, Version |
        Select-Object @{n="Name";e="DisplayName"},Version,PublisherId
    
{% endhighlight %}

Using the manifest to translate names is only slightly different for provisioned packages, and again not all packages have the name as simple text, instead of a distinguished name, there is a cryptic ID for each publisher, but it does show us that when we see "8wekyb3d8bbwe" attached to a name means "published by Microsoft".

{% highlight PowerShell %}
Get-AppxProvisionedPackage  -Online | 
    Select-Object @{n='Name';e='DisplayName'}, Version, InstallLocation, PublisherID -Unique |  ForEach-Object {
        $props = ([xml](Get-Content ($_.InstallLocation -replace "%SystemDrive%", $env:SystemDrive  ) )).Package.Properties
        if ($Props.DisplayName -and $Props.DisplayName -notlike 'ms-resource*') {
               Add-Member -InputObject $_ -NotePropertyName Displayname -NotePropertyValue $props.DisplayName
        }
        else { Add-Member -InputObject $_ -NotePropertyName Displayname -NotePropertyValue ($_.name -replace "^.*?\.","")}
        if ($Props.PublisherDisplayname -and $Props.PublisherDisplayname -notlike 'ms-resource*') {
               Add-Member -InputObject $_ -NotePropertyName PublisherDisplayname -NotePropertyValue $props.PublisherDisplayname -PassThru
        }
        else { Add-Member -InputObject $_ -NotePropertyName PublisherDisplayname -NotePropertyValue $_.PublisherID -PassThru}
    } | Sort-Object DisplayName | ft @{n='Name';e='DisplayName'}, Version, @{n='Publisher';e='PublisherDisplayName'},InstallLocation
    
{% endhighlight %}

![Staged Appx Packages](/assets/Install_store_staged.png)

## Updates

There's more! In Windows PowerShell 5 you can get installed software using the `Get-Package` cmdlet and specify either the *Programs* or *MSI* providers - combined these give you the same as the registry query above. However there is also an *MSU* provider for updates. This includes hotfixes (of which more in a moment), drivers and a some other components of interest. The MSU provider was not been ported to PowerShell 6 and 7, but we can get the same effect like this.

{% highlight PowerShell %}
    $updateSession  = New-Object -ComObject Microsoft.Update.Session
    $updateSearcher = $updateSession.CreateUpdateSearcher()
    $updates        = $updateSearcher.QueryHistory(1, $updateSearcher.GetTotalHistoryCount() ) |
        Where-Object ResultCode -eq 2 | Select-object @{n="Name"; e="Title"},
            @{n='KB';         e={if ($_.Title -match '(KB\d{5,})') {$matches[1]} }},
            @{n='Version';    e={if ($_.Title -match '(-\s+v?|version\s+)(\d+(\.\d+)+)') {$matches[2]} }},
            Date, Description |
                Sort-Object @{e={$_.kb -as [boolean]}}, title,version
    $updates | ft -a
    
{% endhighlight %}

![The first updates - showing drivers](/assets/Install_updates1.png)

Above shows the *first* entries returned which includes drivers, and below are *later* entries showing KB items.

![The first updates - showing drivers](/assets/Install_updates2.png)

The KB numbers for hotfixes, and the version numbers for other things are embedded in the title, so the code above extracts those. The update process tags each update with a result-code: 0 for not started, 1 for running, 2 for succeeded, 3 for succeeded with errors, 4 for failed, 5 for aborted, and the `where` in the code above filters to only successes. On my machine which I'd had about a month when I wrote this, I can check how many KBs have been installed with  
`$updates.kb | Sort-Object -Unique | measure`
\- it was 17 at that point, rather higher now.

There is a `Get-Hotfix` command in PowerShell, which is really just calling `Get-CimInstance win32_QuickFixEngineering` but on my machine it only returns 4 items

{% highlight PowerShell %}
    $kbs = $updates.kb | Sort-Object -Unique
    Get-CimInstance Win32_QuickFixEngineering | Where-Object HotFixID -NotIn $kbs
        
{% endhighlight %}

Of the four only one is an extra, the other three are in the updates query, so I'm not sure this is is a useful test.
And the list of drivers found by checking updates is long, yet incomplete: `Get-WindowsDriver` returns a more complete list and with a little processing it can look like the values returned as updates.  

{% highlight PowerShell %}
Get-WindowsDriver -Online  | 
    select-Object @{n='Name';e={$_.ProviderName + " - " + $_.ClassName + " - " + $_.version}}, Version, 
                  @{n='Description';e={"{0} {1}  driver update released in  {2:MMMM yyyy}" -f $_.providerName, $_.ClassName,$_.Date}} 
    Get-WindowsDriver -Online  | Sort-Object -Unique  ProviderName, ClassName, Date, OriginalFileName | 
        Format-Table -Autosize  ProviderName, ClassName, Version, Date, @{n='File';e={Split-Path -leaf $_.OriginalFileName }} 
    
{% endhighlight %}

![Get-WindowsDriver - processed to the same format as updates](/assets/Install_drivers.png)

All the above have been designed to produce similar enough output that I can pull them into a single spreadsheet and have something that is easy to de-duplicate. My month-old system already had 373 rows and you'd think that was all. But it doesn't cover PowerShell modules. 
{% highlight PowerShell %}
 Get-Package -ProviderName PowerShellGet
    
{% endhighlight %}
Will get a list of PowerShell modules, run in PowerShell 7 it lists only the modules installed in 7, so if some were installed from *Windows* PowerShell, the command needs to be run in both versions. Other products will have ways of installing scripts/macros/templates/presets and so on, and I'm not looking here at things installed into the Windows-Subsystem for Linux.  

**But wait** - there's no `Quick Assist`, no `SSH client`, no `WordPad` (has anyone used WordPad in the last 20 years?) - these have their own page in settings. And they are found with `Get-WindowsCapability` each name ends with "~~~~0.0.1.0" so the code below reduces it to just the name. Most of the items returned are recognizable as items on that page in settings, but there are some extra network components which the UI *doesn't* show.
{% highlight PowerShell %}
    Get-WindowsCapability -Online | Where-Object state -eq installed |  
       ForEach-Object {$_.name -replace '~.*$',''}  | Sort-Object -Unique
    
{% endhighlight %}

At one time these components were accessed through the Optional features part of control panel, rather than via the settings app. Whether that's installation or Windows configuration I'm not sure, but for completeness the list of items which still use the control panel method can be found with  
{% highlight PowerShell %}
     Get-WindowsOptionalFeature -Online  | where-object state -eq enabled | sort-object FeatureName |ft
    
{% endhighlight %}

These last two add 79 more items on my machine.

<sup>\*</sup> The Crescendo toolkit for making PowerShell interfaces into legacy commands was meant for this and there is a module named (Cobalt)[https://www.powershellgallery.com/packages/Cobalt] built with it for winget. And I do know it is meant to be Win Get :-)