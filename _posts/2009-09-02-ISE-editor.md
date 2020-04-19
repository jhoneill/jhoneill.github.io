---
layout: post
title:  "How to load files into the PowerShell ISE editor – or “What’s in your profile”"
date:   2009-09-02
categories: PowerShell
---
_(another in the “Things you’ll want to do with PowerShell V2” series)_

One of the tricks I have been meaning to post for a little while is a neat way
to edit a file in the new PowerShell Integrated Scripting Environment. If you
are working in the “traditional” shell, you can just type `notepad filename` and
you’re editing the file, but how can you have a command line to load up a file in
the ISE? It turns out to be quite easy.  
`$psise.CurrentPowerShellTab.Files.add("C:\myStuff\MyFile.ps1")`

The ISE has an object model and there is an automatic variable which points to
the root of it, so all you have to do call one of its objects, give it the full
path of the file to load and hey presto. But it doesn’t exactly trip off the
fingers. This was the first attempt.
{% highlight powershell %}
function edit {
  param ( $Path )
  $null = $psise.CurrentPowerShellTab.Files.add($Path)
}
{% endhighlight %}

Easy enough, but PowerShell can do much better than that. For starters it can
resolve the path for us, and if it resolves to multiple files (for example
`*.ps1`) then we’ll get all of them.
{% highlight powershell %}
function edit { 
  param ( $Path )
  Resolve-Path $path | ForEach-Object { 
      $null = $psise.CurrentPowerShellTab.Files.add($_.path)
  }
}
{% endhighlight %}
My first instinct was to hand `Resolve-Path` one item at a time, but it is quite
happy being passed multiple paths like `*ps1,*.PsXML` so this version of the
function will open multiple wildcards. It also returns `PathInfo` objects, hence the need to specify their `Path` property in the `add()` This was starting to look like I would
want to be able to pipe things into it. And by now I was realizing that I would
want to use it in the traditional shell as well. So, I made three more
modifications . The first one says I can pass it objects which have a `Path`,
`FileName` or `FullName` property and *that property will be treated as the path*.
The second is because a function can have a `begin`, `process` and `end` block, this
code should go in the `process` block to be _called for each object piped in_, and the
last says if the name of the PowerShell host isn’t the ISE then launch the file
in notepad.
{% highlight powershell %}
function edit {
  param (
    [parameter(ValueFromPipelineByPropertyName=$true)]
    [Alias("FullName","FileName")]
    $Path
  )
  process {
    Resolve-Path $path | ForEach-Object { $_
        if ($host.name -match "\sISE\s") {
            $null = $psise.CurrentPowerShellTab.Files.add($_.path)}
        else {notepad.exe $_.path}
    }
  }
}
{% endhighlight %}

I had the first version working for a few days before I changed it to allow the
value from pipeline *by property name* after reading something on our internal
PowerShell alias (I’d swear it came from James Brundage but I can’t find it). It
means I don’t have to test for different kinds of objects being piped into the
command – I can just anticipate the _property names_. This came into its own last
night when I found I had the same error in about 20 different PS1 files. It was
easy enough to find them all and open them with one command.    
`Select-String -Pattern "ParameterSet" -Path *.ps1 | edit`   
The updated version is now in my profile – and since moving up to version 2, my
profile has become one which works for both ISE and traditional PowerShell –
there are actually 6 profiles, current user, and all user and all hosts / current-host
 – which is either the Shell or ISE host. They are mapped out below –
but you can find them as properties of `$profile`.

`$profile.CurrentuserCurrenthost` = **either**    
<code>&nbsp;&nbsp;&nbsp;&nbsp;$env:userProfile\Documents\WindowsPowerShell\Microsoft.PowerShellISE_profile.ps1</code> **OR**    
<code>&nbsp;&nbsp;&nbsp;&nbsp;$env:userProfile\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1</code>

`$profile.CurrentuserAllhosts`    
<code>&nbsp;&nbsp;&nbsp;&nbsp;$env:userProfile\Documents\WindowsPowerShell\profile.ps1</code>

$profile.AllusersCurrentHost` **either**    
<code>&nbsp;&nbsp;&nbsp;&nbsp;$env:windir\System32\WindowsPowerShell\v1.0\Microsoft.PowerShellISE_profile.ps1</code> **OR**
<code>   $env:windir\System32\WindowsPowerShell\v1.0\Microsoft.PowerShell_profile.ps1</code>

`$profile.AllusersAllHosts`
<code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;$env:windir\System32\WindowsPowerShell\v1.0\profile.ps1</code>

**Bonus:** Looking for the information about profiles I found [this post of
Jeffrey’s](https://devblogs.microsoft.com/powershell/my-powershell_ise-profile/)
on adding custom commands to the menus in the ISE - it showed me how I could do    
<code>&nbsp;&nbsp;&nbsp;&nbsp;$psise.CurrentPowerShellTab.AddOnsMenu.Submenus.Add(</code><br />
<code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"Bulk edit", {Edit(Read-host "Enter file(s)") }, 'Ctrl+E')`</code>    
and have *bulk edit* on a hot key / menu item.
