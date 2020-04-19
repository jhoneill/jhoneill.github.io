---
layout: post
title:  "Do the job 100 times faster with Parallel Processing in PowerShell"
date:   2016-12-06
categories: PowerShell
---

_It’s a slightly click-baity title, but I explain below where the 100 times number comes from below. The module is on the [PowerShell gallery](https://www.powershellgallery.com/packages/Start-parallel) and you can install it with `Install-Module -Name Start-parallel`_

Some of the tasks we need to do in PowerShell involve firing off many similar requests and waiting for their answers – for example getting status from lots computers on a network. It might take several seconds to do each one – maybe longer if the machines don’t respond. Doing them one after the other could take ages. If I want to ping all 255 addresses on on my home subnet, most machines will be missing and it will take 3 seconds to time out for each of the 200+ inactive addresses. Even if I try only one ping for each address it’s going to take 10-12 minutes, and for >99% of that time my computer will be waiting for a response. Running them in parallel could speed things up massively.
Incidentally the data which comes back from ping isn’t ideal, and I prefer this to the `Test-Connection` cmdlet.
{% highlight powerShell %}
Function QuickPing {
    param ($LastByte)
    $P = New-Object -TypeName "System.Net.NetworkInformation.Ping"
    $P.Send("192.168.0.$LastByte") | where status -eq success | select address, roundTripTime
}

{% endhighlight %}
PowerShell allows you to start multiple processes using Jobs and there are places where Jobs work well. But it only takes a moment to see the flaw in jobs: if you run
{% highlight powerShell %}
    Get-Process *powershell*
    Start-Job -ScriptBlock {1..5 | ForEach-Object {start-sleep -Seconds 1 ; $_ } }
    Get-Process *powershell*
    
{% endhighlight %}
You see that the job creates a new instance of PowerShell… doing that for a single ping is horribly inefficient – jobs are better suited to tasks where run time is much longer than the set-up time AND where we don’t want run _lots_ running concurrently. In fact I’ve found creating large numbers of jobs tends to crash the PowerShell ISE; so my first attempts at parallelism involved tracking the number of jobs running and keeping to a maximum – starting new jobs only as others finished. It worked but in the process I read [this by Boe Prox](https://learn-powershell.net/2012/05/13/using-background-runspaces-instead-of-psjobs-for-better-performance/) and [this by Ryan Witschger](http://www.get-blog.com/?p=189) which led me to a better way: _RunSpaces and the RunSpace factory_.
MSDN defines a RunSpace as “_the operating environment where the command pipeline of the [PowerShell object](https://docs.microsoft.com/en-us/dotnet/api/system.management.automation.powershell?view=pscore-6.2.0) is invoked_”; and says that the PowerShell object allows applications that programmatically use Windows PowerShell to create pipelines of commands, invoke them and access the results. The _factory_ can create **single RunSpaces, or a pool of RunSpaces**. So a program (or script) can get a PowerShell object which says “Run this, with these named parameters and these unnamed arguments. Run it asynchronously (i.e. start it and don’t wait for it complete, give me some signal when it is done), and in a space from this pool.” If there are more things wanting to run than there are RunSpaces, **the pool handles queuing** for us.

Thus the idea for `Start-Parallel` was born.  I wanted to be able to do this:   
`Get-ListOfComputers | Start-Parallel Get-ComputerSettings.ps1`   
or this:
`1..255 | Start-Parallel -Command QuickPing -MaxThreads 500`   
or even pipe PS objects or hash tables in to provide multiple parameters to the same command.

`-MaxThreads` in the second example says "create a pool where 500 pings can be in progress", so every _QuickPing_ can be running at the same time (performance monitor shows a spike of threads). So how long does it take to do 255 pings now? 240 inactive addresses taking 3 seconds each gave me ~720 seconds and the version above runs in a little under 7, so a that’s a **100 fold speed increase!**  This is pretty consistent with what I’ve found with polling servers over the couple of years I’ve been playing with `Start-Parallel` – things that would take a morning or an afternoon run in a couple of minutes.

You can install it from the [PowerShell gallery](https://www.powershellgallery.com/packages/Start-parallel). Some tips:

-  `Get-ListOfComputers | Start-Parallel Get-ComputerSettings.ps1`    
works better than    
`$x = Get-ListOfComputers ; Start-Parallel -InputObject $x -Command Get-ComputerSettings.ps1`    
if Get-ListOfComputers is slow, we will probably have the results for the first computer(s) before we have been told the last one on the list to check.
-  **Don’t** hit the same same service with many requests in parallel – unless you `want` to mount a denial of service attack.
-  Remember that **RunSpaces don’t share anything** – the parallel RunSpaces won’t load your profile, or inherit anything from the session which launches them. And there is no guarantee that every module out there always behaves as expected if run in multiple RunSpaces simultaneously. In particular if, “QuickPing” is defined in the same PS1 file which runs `Start-Parallel`, then `Start-Parallel` is defined in the global scope and can’t see `QuickPing` in the script scope. The work round for this is to use   
`Start-Parallel –scriptblock ${Function:\QuickPing}`
-   Some commands by their nature specify a computer. For others it is easier to define a script block inside another script block (or a function) which takes a computer name as a parameter and runs   
`Invoke-Command –ComputerName $computer –scriptblock $InnerScriptBlock`
- I don’t recommend running `Start-Parallel` inside itself, but based on very limited testing it does appear to work.

You can install it by running `Install-Module -Name Start-parallel`

