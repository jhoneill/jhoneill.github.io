---
layout: post
title:  "TabCompletion++ or TabCompletion#"
date:   2013-07-05
categories: PowerShell
--- 
One of the tricks of PowerShell V3 is that it is even cleverer with Tab completion / intellisense than previous versions, though it is not immediately obvious how you can take control of it.  When I realised what was possible I had to apply it to a Get-SQL command command I had written. I wanted the ISE to be able to give me something like this

<a href="/assets/tab-expansionISE.png"><img style="float:left;" alt="ISE Tab-Expansion" align="left" src="/assets/tab-expansionISE.png" width="260" height="115" /></a>

I happened to have Jan Egil Ring's [article on the subject for PowerShell magazine](http://www.powershellmagazine.com/2012/11/29/using-custom-argument-completers-in-powershell-3-0/) (where he credits [another article by Tobias Weltner](http://www.powertheshell.com/dynamicargumentcompletion/) ) open in my browser, when I watched Jason Shirk gave a talk covering the module he has published [via GitHub](https://github.com/lzybkr/TabExpansionPlusPlus) named TabExpansion++. This module includes custom tab completers for some PowerShell commandss which don’t have them and some more for legacy programs. Think about that for a moment, if you use `NET STOP` in PowerShell instead of in CMD, tab completion fills in the names of the services. Yes, I know PowerShell has a `Stop-Service` cmdlet, but if you’ve been using the `NET` commands since the 1980s (yes, guilty) why stop using them?

More importantly Jason has designed a framework where **you can easily add your own tab completers** – which are the **basis for intellisense**. On loading, his module searches all ps1 and psm1 files in the paths `$env:PSModulePath` and `$env:PSArgumentCompleterPath` for functions with the `ArgumentCompleter` attribute – I’ll explain that shortly.  When it finds one, it extracts the function body and and `imports` it into the TabExpansion++ module. If I write argument completer functions and save them with my modules (which are in the PSModulePath) then when I load TabExpansion++ … whoosh! my functions get tab-completion.

A lot of my work at the moment involves dealing with data in a MySQL database, and I have installed the MySQL ODBC driver, and I wrote a function Named `Get-SQL` (which I can just invoke as SQL).
When I first wrote it, it was simple enough: leave an ODBC connection open as a global variable and pump SQL queries into it. After a while I found I was sending a lot of     
`Select * From table_Name` queries, and so I gave it a `–Table` parameter which would be built into a `Select` query and a `–gridview` parameter which would send the data to the PowerShell grid viewer. Then I found that I was doing a lot of     
`Desc table_name` queries, so I added a `-Describe` parameter. One way and another the databases have ended up with long table names which are prone to mistyping, this seemed like a prime candidate for an argument completer, so I set about extending TabExpansion++ (does that make it TabExpansion#? if you haven’t noticed with C# the # sign is ++ ++ one pair above the other).

It takes 4 things to make a tab completer function. **First:** one or more `ArgumentCompleter` attributes
{% highlight powershell %}
[ArgumentCompleter(Parameter   = 'table',
                   Command     = ('SQL','Get-SQL'),
                   Description = 'Complete Table names for Get-SQL , for example: Get-SQL -GridView -Table ')]
                   
{% endhighlight %}
This defines the parameter that the completer works with – which must be a single string. If the completer supports multiple parameters, you must use multiple `ArgumentCompleter` attributes. 
And it defines the command(s) that the completer works with. The definition can be a string, an array of strings, or even a ScriptBlock.     
The **Second** thing needed is a `param` block  that understands the parameters passed to a tab completer.    
{% highlight powershell %}
param($commandName, $parameterName, $wordToComplete, $commandAst, $fakeBoundParameter)

{% endhighlight %}

The main one here is `$wordToComplete` – the partially typed word that tab-completion is trying to fill in. However, as you can see in the screen shot, it is possible to look at the parameters already completed and use them to produce the list of possible values.
`$wordToComplete` -is used in the **third part**, whch is the body that gets those possible parameter value. So in my function I have something a bit like this…
{% highlight powershell %}
  $parameters = Get-TableName | Where-Object { $_ -like "$wordToComplete*" } | Sort-Object
{% endhighlight %}

And the **final part** is to return the right kind of object to tab completion process, and Jason’s module has a helper function for this
{% highlight powershell %}
    $parameters | ForEach-Object {
        $tooltip = "$_"
        New-CompletionResult $_ $tooltip
    }
{% endhighlight %}

There is the option to have different text as the tool tip – in some places `$tooltip` – which is shown in intellisense – would be set to something other than the value being returned. Here I’ve kept it in place to remind me, rather than calling    
`New-CompletionResult $_ $_`

And that’s it. Unload and reload TabExpansion++ and my SQL function now knows how to expand `-Table.` I added a second attribute to allow the same code to handle `-Describe` and then wrote something to get field names so I could have a picklist for `–OrderBy` and `–Select` as well. With `-Select` intellisense doesn’t pop up a second time if you select a name and enter a comma to start a second; but tab completion works. Here’s the finished item:
{% highlight powershell %}
Function SQLFieldNameCompletion {
    [ArgumentCompleter(Parameter   = ('where'),
                       Command     = ('SQL','Get-SQL'),
                       Description = 'Complete field names for Get-SQL , for example: Get-SQL -GridView -Table ')]
    [ArgumentCompleter(Parameter   = ('select'),
                       Command     = ('SQL','Get-SQL'),
                       Description = 'Complete field names for Get-SQL , for example: Get-SQL -GridView -Table ')]
    [ArgumentCompleter(Parameter   = ('orderBy'),
                       Command     = ('SQL','Get-SQL'),
                       Description = 'Complete field names for Get-SQL , for example: Get-SQL -GridView -Table ')]
    param($commandName, $parameterName, $wordToComplete, $commandAst, $fakeBoundParameter)
    if ($DefaultODBCConnection) {
        $TableName = $fakeBoundParameter['Table']
        Get-SQL -describe $TableName | Where-Object { $_.column_name -like "$wordToComplete*" } | 
            Sort-Object -Property column_name | ForEach-Object {
                $tooltip           = $_.COLUMN_NAME + " : " + $_.TYPE_NAME
                New-CompletionResult $_.COLUMN_NAME $tooltip
        }
   }
}
{% endhighlight %}
Which all saves me a few seconds a few dozen times a day.