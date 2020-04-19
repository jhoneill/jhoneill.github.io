---
layout: post
title:  "On PowerShell function design: vague can be good."
date:   2009-09-14
categories: PowerShell
---

There is a problem which comes up in several places in PowerShell – that is
_helping the user by being vague about parameter types_. Consider these examples
from my [Hyper-V library for PowerShell](http://codeplex.com/pshyperv)

1.  The user can specify a machine using a string which contains its name  
    `Save-VM London-DC or Save-VM \*DC, or Save-VM London\*,Paris\*`

2.  The user can get virtual machine objects with one command and pipe these
    into another command  
    `Get-VM –Running | Stop-VM`

3.  The user can mix objects and strings  
    `$MyVMs = Get-VM –server Wallace, Grommit | Where { (Get-VMSettings $_).note –match "LAB1"}`    
    `Start-VM –Wait "London-DC", $MyVMs`

The last one searches servers “Wallace” and “Grommit” for VMs, narrows the list
to those used in "Lab1" and starts London-DC on the local server followed by the
VMs in Lab1. In [a post I made a few days back](/powershell/2009/09/02/ISE-editor.html) about adding Edit to your profile I showed a couple of aspects of about piping objects that became easier in V2 of PowerShell,  
Instead of writing `Param($VM)`, I can now write
{% highlight Powershell%}
Param(
   [parameter(Mandatory = $true, ValueFromPipeline = $true)]
   $VM
)
{% endhighlight %}
`Mandatory=$true` makes sure I have a parameter from somewhere, and
`ValueFromPipeLine` is all I need to get it from the pipeline. PowerShell offers a
`ValueFromPipeLineByPropery` option which looks at the piped object for a property
which matches the parameter name or a declared alias for it. I could use that to
reduce a VM object to its name, but doing so would lose the server information
(which I need in example 3 above) and it gets in the way of piping strings into
functions, so this is not the place to use it.

Allowing an array gives me a problem if an array member expand to more than
one VM (when it's a an with a wildcard). The code for my “Edit” function won’t cope
with being handed an array of file objects or an array of arrays, but it _doesn’t
need to_, because I wouldn’t work like that. But things I’m putting out for
others _need to work the way different users might expect_, this needs to handle
arrays in arrays (like `"london-DC", $myVMs` ) or arrays of VM objects (`$myVMs`),
so time for my old friend recursion, and a function like this:

{% highlight Powershell%}
function Stop-VM {
  param(
    [Parameter(Mandatory = $true, ValueFromPipeline = $true)]
    $VM,
    [String]
    $Server = "."
  )
  process{
    if ($VM –is [String]) {$VM = GetVM –VM $VM –server $server}

    if ($VM –is [Array]) {$VM | ForEach-Object {Start-VM –VM $_ –Server $Server}}
        
    if ($VM -is [System.Management.ManagementObject]) {
        $VM .RequestStateChange(3)
    }
  }
}
{% endhighlight %}
This says, if we got passed a single string (via the pipe or as a parameter), we
get the matching VM(s), if any.    
If we were passed an array , or a string which resolved to an array, we call the function again with each member of that array.    
If we were passed a single WMI object or a string which resolved to a single WMI object, then we do the work required.

There’s one thing wrong with this, and that is that it stops the VM **without any warning** I covered this back here.

# /2009/06/25/more-powershell-a-change-you-should-make-for-v2-1.aspx

It is easy to support `ShouldProcess`; there is level at which `Confirm` prompts are
turned on automatically, (controlled by `$confirmPreference`) and we can say that
the function's impact is `high` – and at the default value the confirm prompt will appear
even if the user doesn’t ask for it.
{% highlight powershell %}
function Stop-VM { 
  [CmdletBinding(SupportsShouldProcess=$True, ConfirmImpact='High')]
  Param(
    [parameter(Mandatory = $true, ValueFromPipeline = $true)]
    $VM,
    [String]
    $Server = "."
  )
  process{
    if ($VM –is [String]) {$VM = Get-VM –VM $VM –server $server}
    
    if ($VM –is [array]) {$VM | foreach-object {Stop-VM –VM $_ –server $server}}

    if ($VM -is [System.Management.ManagementObject] –and 
        $pscmdlet.shouldProcess($VM.ElementName, “Power-Off VM without Saving”) {
        $VM .RequestStateChange(3)
    }
  }
}
{% endhighlight %}
**Nearly there,** but we have two problems still to solve: and a simplification to
make the message. First the simplification; Mike Kolitz (who’s  helped out on
the Hyper-V library), introduced me to this trick : when a function calls
another function using the same parameters (or calls itself recursively) - if
there are many parameters it can be a pain. But PowerShell has a “Splatting”
operator. `@PsboundParameters` puts the contents of a variable into the command.
(James Brundage, who I’ve mentioned before [wrote it
up](https://devblogs.microsoft.com/powershell/how-and-why-to-use-splatting-passing-switch-parameters/).)
You can manipulate `$psboundParameters`, so Mike had a clever generic way of
recursively calling functions.
{% highlight powershell %}
if ( $VM -is [Array]) { [Void]$PSBoundParameters.Remove("VM") ; $VM |
        ForEach-object {Stop-VM -VM $_ \@PSBoundParameters}
}
{% endhighlight %}
In other words, _remove the parameter that is being expanded, and re-call the
function with the remaining parameters_, specifying only the one being expanded.
As James’ post shows it makes life a lot easier when you have a bunch of
switches.

OK, now, the message    
```
  Confirm
  Are you sure you want to perform this action?
  Performing operation "Power-Off VM without Saving" on Target "London-DC".
  [Y] Yes [A] Yes to All [N] No [L] No to All [S] Suspend [?] Help (default is"Y"): 
```

will appear for `every` VM, even if we select _Yes to all_ or _No to all_, because each time
`Stop-VM` is called it gets a new instance of `$pscmdlet`. And, what if we don’t
want the message at all – for example in a script which kills the VMs and rolls back to
an earlier snapshot. Jason Shirk, one of the active guys in our internal
PowerShell alias pointed out first you can have a `–Force` switch and secondly you
don’t need to use the function’s OWN instance of `pscmdlet – why not pass one
instance around? So the function morphed into this
{% highlight powershell%}
function Stop-VM { 
  [CmdletBinding(SupportsShouldProcess=$True, ConfirmImpact='High']
  Param(
    [Parameter(Mandatory = $true, ValueFromPipeline = $true)]
    $VM,
    [String]
    $Server = “.”,
    $PSC,
    [Switch]
    $Force
  )
  process{
    if ($psc -eq $null) {$psc = $pscmdlet}
    if (-not $PSBoundParameters.psc) {$PSBoundParameters.add("psc",$psc)}

    if ($VM –is [String]) {$VM = GetVM –VM $VM –server $server}
    if ($VM –is [array])  {$VM | ForEach-Object {Stop-VM –VM $_ –server $server}}
    if ($VM -is [System.Management.ManagementObject] –and 
            ($force –or $psc.shouldProcess($VM.ElementName, “Power-Off VM without Saving”)) {
        $VM .RequestStateChange(3)
    }
  }
}
{% endhighlight %}
So now, `$PSC` either gets passed in as a parameter or it picks up `$pscmdlet` and then gets passed on
to anything else we call – in this case recursive calls to the same function. And
`–Force` is there to trump everything. And that’s what I have implemented in
dozens of places in my library.

This post originally appeared on my technet blog.
