---
layout: post
title:  "More on writing clear scripts: Write-output and return … good or bad?"
date:   2017-06-16
categories: PowerShell
---

My [last post](/powershell/2017/06/02/Understandable-Scripts.html) talked about writing understandable scripts and I read a piece entitled [Let’s kill Write-Output](https://get-powershellblog.blogspot.co.uk/2017/06/lets-kill-write-output.html?) by Mark Krauss (actually I found it because Thomas Lee Tweeted it with “And sort out return too”).

So let’s start with one practicality. You can’t remove a command which has been in a language for over 10 years unless you are prepared for a lot of pain making people re-write scripts. `Write-Output` has an alias `echo` which was put there for people who come from other scripting languages, and start by asking “How do I print to the console”. But if removing it altogether is impractical, we can **advise people to avoid it**, write rules to catch it in the script analyser and so on. **Should we?** And when is a good idea to use it ?

Mark points out he’s not talking about `Write-Host`, which should be kept for limited scenarios: if you want the user to see it by default, but it **isn’t part of the output** then that’s a job for `Write-Host`, for example with my [Get-SQL](/powershell/databases/2017/01/29/Sharing-GetSQL.html) command,    
`$result = Get-SQL $sqlQuery` writes a message: `42 rows returned` to the console but the output saved into `$result `is the 42 rows of data. Mark gives an example:    
{% highlight powershell %}
  Write-Output "PowerShell Processes:"
  Get-Process -Name PowerShell
  
{% endhighlight  %}
and says it is better written as  
{% highlight powershell %}
  "PowerShell Processes:"
   Get-Process -Name PowerShell
   
{% endhighlight  %}
And this is actually a case where Write-host should be used … why ? Let’s turn that into a function.
{% highlight powershell %}
Function Get-psProc {
  "PowerShell Processes:"
  Get-Process -Name"*PowerShell*"
}
  
{% endhighlight  %}
Looks OK doesn’t it ? But it outputs two different types of object into the pipeline. All is well if we run Get-psProc on its own, but if we run    
<code>&nbsp;&nbsp;&nbsp;&nbsp;Get-psProc | ConvertTo-Csv</code>    
It returns
```
#TYPE System.String
"Length" ...
"21" ...   
  
``` 
The next command in the pipeline saw that the first object was a string and that determined its behaviour for all the following objects. “PowerShell processes” is <u>decoration you want the user to see but isn’t part of the output</u>. [That earlier  post on understandable scripts](/powershell/2017/06/02/Understandable-Scripts.html) came from a talk about writing good code and the one of the biggest problems I find in other peoples code is **fixation with printing to the screen**. That leads to things like the next example – which is meant to read a file and say how many lines there are and their average length.
{% highlight powershell %}
  $measurement = cat $path | measure -average Length
  echo ("Lines read    : {0}"    -f $measurement.Count  )
  echo ("Average length: {0:n0}" -f $measurement.Average) 
   
{% endhighlight  %}
This runs and it does the job the author intended but I’d suggest they might be new to PowerShell and haven’t yet learnt that Output is not the same as “stuff for a user to read” (as in the previous example) , and they feel their output must be printed for reading. Someone more experienced with PowerShell might just write:    
<code>&nbsp;&nbsp;&nbsp;&nbsp;cat $path| measure -average Length</code>    
If they aren’t bothered about the labels, or if the labels really matter   
{% highlight powershell %}
cat $path | measure -average Length |
     select @{n="Lines Read";e="count"}, @{n="Average Length";e="average"}
    
{% endhighlight %}
If this is something we use a lot, we might **change the aliases to cmdlet names**, specify parameter names and save it for later use. And it is re-usable, for example if we want to do something when there are more than x lines in the file, where the previous version can only return text with the number of lines embedded in it. **Resisting the urge to print everything is beneficial** and that gets rid of a lot of uses of `Write-output` (or echo).
Mark’s post has 3 beefs with `Write-Output`:
1.  Performance. It _is_ slower but rarely _noticeably_ so, so I’d discount this.
2.  Security / Predictability – `Write-Output` can be redefined, and that allows for something malign or just buggy.    
True, but it also allows you to redefine it for logging. debugging and so on. So you could use a proxy Write-output for testing and the standard one in production. So this is not exclusively bad
3.  The false sense of security. He says that explicitly returning stuff is held to be better than implicit return, which implies    
`Write-Output $result`  is better than just  `$result`    
But no-one says you should write `cat $path | Write-Output` it’s obviously redundant, but when you don’t isn’t that implying output ?

My take on the last point is piping output into `Write-Output` (or `Out-Default`) is a tautology “Here’s some output, take it and output it”. It’s making things **longer but not clearer**. If using `Write-Output` _does_ make things clearer then it is a sign the script is hard to read and at least needs some comments, and possibly some redesign. [Joel Bennett sums up the false sense of security part in a sentence](https://github.com/PoshCode/PowerShellPracticeAndStyle/issues/46#issuecomment-236727676)  "_while some people like it because it highlights the spots where you intentionally output something — other people argue it’s presence distracts you from the fact that other lines could output._"  [Thanks Joel, that would have taken me a paragraph!]
This is where Thomas’ comment about `return` comes in; it tells PowerShell to bail out of a function, and there are many good reasons for doing that, it also has a _two in one_ syntax :     
`return $result`    
is the same as    
`$result`    
`return`   
When I linked to Joel above he also asks the question whether, as the last lines of a function, this   
`$output = $temp + (Get-Thing $temp)`    
`return $output`    
is better or worse than    
`$output = $temp + (Get-Thing $temp)`    
`$output`

Not many people would add `return` to the second example – it’s redundant. But if you store the final output in a variable there is some logic to using `return` (or `Write-Output`) to send it back. But is it making things any clearer to store the result in a variable ? or it just as easy to read the following ?    
`$temp + (Get-Thing $temp)`

As with `Write-Output`, sometimes using `return $result` makes things **clearer** and sometimes it’s a habit from other programming languages where functions must **return results in a single place** – so multiple parts must be gathered and then returned. Here’s something which combines the results of 3 queries and returns them:
{% highlight powershell %}
  $result  = (Get-SQL $sqlQuery1)
  $result += (Get-SQL $sqlQuery2)
  $result +  (Get-SQL $sqlQuery3)
  
{% endhighlight%}

The first line assigns an array of database rows to a variable, the second appends more rows and the third _returns_ these rows together with the results of a third query. You need to look at the operator in each line to figure out which sends to the pipeline. Arguably this it is clearer to replace the last line with this:
{% highlight powershell %}
  $result += (Get-SQL $sqlQuery3)
  return $result
  
{% endhighlight%}

When there are 3 or 4 lines between introducing `$result` and putting it into the pipeline this is OK. But lets say there are 50 lines of script between storing the results of the first query and appending the results of the second. **Has the script been made clearer by storing a partial result** … or would you see something being appended to `$result` and look further up the script for where it was originally set and anywhere it was changed? This example does nothing with the combined segments (like sorting them) we’re just following an old habit of only outputting in one place. Not outputting anything until we have everything can mean it takes a lot longer to run the script – we could have processed all the results from the first query while waiting for the second to run. I would dispense with the variable entirely and use
{% highlight powershell %}
  Get-SQL $sqlQuery1
  Get-SQL $sqlQuery2
  Get-SQL $sqlQuery3
  
{% endhighlight%}
If there is a lot of script between each I’d then use a `#region `around the lines which lead up to each query being run    
{% highlight powershell %}
#region build query and return rows for x
#etc etc
Get-SQL $sqlQuery1
#endregion  
   
{% endhighlight%}
so when I collapse the outlining regions in my editor I see
{% highlight powershell %}
#region build query and return rows for x
#region build query and return rows for y
#region build query and return rows for z
  
{% endhighlight%}
Which gives me a very good sense of what the script is doing at a high level and then I can drill into the regions if I need to. If I do need to do something to the combined set of rows (like sorting) then my collapsed code might become
{% highlight powershell %}
#region build query for x and keep rows for sorting later
#region build query for y and keep rows for sorting later
#region build query for z and keep rows for sorting later
#region return sorted and de-duplicated results of x,y and z
   
{% endhighlight%}
Both outlines give a sense of where there should be output and where any output might be a bug.

### In conclusion. 
When you see _lots of_ `echo` / `Write-Output` commands that’s usually a bad sign – it’s usually an indication of **too many formatted strings** going into the pipeline, but `Write-Output` is **not automatically bad** when used sparingly – and used _properly_ `return` isn’t bad either. But if you find yourself adding either for clarity it should make you ask “Is there a better way”. 
