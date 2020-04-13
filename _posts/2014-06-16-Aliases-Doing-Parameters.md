---
layout: post
title:  "A trick with Powershell aliases–making them do parameters"
date:   2014-06-16
categories: PowerShell
--- 
The first thing that everyone learns about PowerShell Aliases is that they just **replace the name** of a command, aliases don’t _do_ parameters.
`DIR` is an alias for `Get-ChildItem`; you can’t make an alias `RDIR` for `Get-ChildItem –Recurse`. If you want that you need a function.
To quote Redmond’s most famous resident[^1] **I canna change the laws of physics, Captain, but I can find ye a loophole.**

I wrote a function which I use <u>a lot</u> – 100+ times some days – named `Get-SQL`.    
Given an unknown command `xxx`,  PowerShell will see if there is a command `Get-xxx` before reporting an “Not recognized” error, so I usually just run it as `SQL` without the to `Get-`.    
The function talks to databases and sessions it keeps connections open between calls: connections tend to named after places in South America, so to **open a session** I might run    
{% highlight powershell%}
> SQL -Session Peru -Connection DSN=PERU
{% endhighlight %}
and then to find out the **definition of a table** I use   
{% highlight powershell%}
> SQL -Session Peru -Describe Projects
{% endhighlight %}

I’ve previously [written about Jason Shirk’s tab expansion++](/powershell/2013/07/05/TabCompletionPlusPlus.html) which gets a list of tables in that session available to `Describe` (or be Selected from, or Updated or Inserted into), and provides tab-expansion or an intellisense pick list: this is incredibly useful when you can’t remember if the table is named “project”, or “Projects”, and tab expansion++ does the same job for field names so I don’t have to remember if the field I want is named “ID_Project”, “Project_ID”, “ProjectID” or simply “ID”.

`Get-SQL` has a **default** session: I might use`-Session Peru 20` times in a row but still want to leave the default connection alone.    
I found myself thinking ‘I want “Peru” to be an alias for `Get-SQL -Session Peru`.    
And one line: `New-Alias -Name $session -Value something` inside Get-SQL could set it all up for me when I make the connection.’
As I said **we all know you can’t do that with an alias,** but doing this with functions is – bluntly – a nightmare, creating functions on the fly is possible but awkward, and [Tab expansion++](https://github.com/lzybkr/TabExpansionPlusPlus) wouldn’t know how it was supposed to work with them (it _does_ figure out aliases). Defining the functions in advance for each data source is would give me a maintenance headache…

Then I had a flash of inspiration: if I needed this to work for a built-in cmdlet, I’d need to create a [proxy function](/powershell/2012/02/04/ProxyFunctions.html) … but `Get-SQL` is already a function. So, if I can write something in the function to check _how it was invoked_ it can say “A-ha! I was called as ‘Peru’ and ‘Peru’ is the name of a database session, so I should set `$session` to ‘Peru’.” Whatever the _alias_ is, provided there is a _connection_ of the <u>same name</u> it will get used. This turns out to be almost laughably easy.

In my Get-SQL function the `$session` Parameter is declared like this
{% highlight powershell%}
[ValidateNotNullOrEmpty()]
[string]$Session = "Default"
{% endhighlight %}


A function can find out the name was used to invoke it by looking at `$MyInvocation.InvocationName`. If `Get-SQL` is invoked with no value provided for `-session` the value of `$Session` will be set to  ‘Default’: if that is the case and there is a database session whose name matches the invocation name then that name should go into `$Session`, like this:   
{% highlight powershell%}
if ($Session -eq "Default" -and  $Global:DbSessions[$MyInvocation.InvocationName])
    {$Session = $MyInvocation.InvocationName}
{% endhighlight %}

Of course the **parameter is not part of the alias definition** – but the function can detect the alias and set the parameter internally – **the laws stand, but I have my loophole**. Although it’s split here into two lines I think of the IF statement as one line of code.    
When Get-SQL creates a connection it finishes by calling `New-Alias -Name $session -Value Get-SQL –Force`. So two lines give me what I wanted.   
[Tab expansion++](https://github.com/lzybkr/TabExpansionPlusPlus) was so valuable, but stopping here would mean its argument completers don’t know what to do – when they need lists for fields or tables they call `Get-SQL`, and this worked fine when a `-Session` parameter was passed but I’ve gone to all this trouble to get rid of that parameter, so now the completers will try to get a list by calling the function using its canonical name and the default connection. There is a different way to find the invocation name inside an argument completer – by getting it from the parameter which holds the *C*ommand *A*bstract *S*yntax *T*ree, like this:   

{% highlight powershell%}
$cmdnameused = $commandAst.toString() -replace "^(.*?)\s.*$",'$1'
if ($Global:DbSessions[$cmdnameused]) {$session = $cmdnameused}
else {set $session the same way as before} 
{% endhighlight %}

`> Peru –Describe`     

Will pop up the list of tables in that database for me to choose “Projects”. There was a fair amount of thinking about it, but as you can see, only four lines of code. Result!

[^1]: James Doohan – the actor who played “Scotty” in Star Trek actually lived in Redmond – ‘home’ of Microsoft, though Bill Gates and other famous Microsoft folk lived elsewhere around greater Seattle. So I think it’s OK to call him the most famous resident of the place.