---
layout: post
title:  "On writing understandable scripts"
date:   2017-06-02
categories: PowerShell
---
At two conferences recently I gave a talk on “What makes a good PowerShell module”  (revisiting an [earlier talk](https://1drv.ms/p/s!AhfYu7-CJv4e9Q_8zRp8e1w_S149)) the [psconf.eu](http://psconf.eu/) guys have posted a [video](https://www.youtube.com/watch?v=u7BrYQYvH0s&list=PLDCEho7foSooHYGxYqUj2Q6C7usp4aKIQ&index=24) of one I did in Europe and I’ve made the [slides](https://1drv.ms/p/s!AhfYu7-CJv4ejuRdLdJ3ovyIvrDI6A) available (the version I delivered in US used the same slide deck with a different template). .

One of the my points was **Prefer the _familiar_ way to the _clever_ way**. A lot of us like the brilliant PowerShell one-liner (I belong to the “We don’t need no stinking variables” school and will happily pipe huge chains of commands together). But sometimes breaking it into multiple commands means that when you return it later or someone new picks up what you have done, it is easier to understand what is happening.  There are plenty of other examples, but generally _clever_ might be _opaque_; opaque needs comments and somewhere I picked up that what applies to jokes, applies to programming: if you have to explain it, it isn’t that good.

Sometimes, someone doesn’t know the way which is familiar to everyone else, and they throw in something like this example which I used in the talk:
{% highlight powerShell %}
Set-Variable -Scope 1 -Name "variableName" -Value $($variableName +1)
{% endhighlight %}
I can’t recall ever using `Set-Variable`, and why would someone use it to to set a variable to its current value + 1? The key must be in the scope parameter, `–scope 1` means “the parent scope” , most people would write    
`$Script:VariableName ++` or `$Global:VariableName ++`. When we encounter something like this, unravelling what `Set-Variable` is doing interrupts the flow of understanding … we have to go back and say “so what was happening when that variable was set …” 

There are lots of cases where there are multiple ways to do something some are easier to understand but aren’t automatically the one we pick: all the following appear to do the same job
{% highlight powerShell %}
"The value is " + $variable
"The value is " + $variable.ToString()
"The value is $variable"
"The value is {0}" -f $variable
"The value is " -replace "$",$variable
{% endhighlight %}

You might see `.ToString()` and say “that’s thinking like a C# programmer” … but if `$variable` holds a date and the local culture isn’t US the first two examples will produce different results (`toString()` will use local cultural settings).

If you work a lot with the `–f` operator , you might use `{0:d}` to say “insert the first item in ‘short date’ format for the local culture” and naturally write somnething like this    
{% highlight powerShell %}
   "File {0} is {1} bytes in size and was changed on {2}" –f $variable.Name, $variable.Length, $variable.LastWriteTime

{% endhighlight %}

Because the eye has to jump back and forth along the line to figure out what goes into {0} and then into {1} and so on, this loses on readability compared concatenating the parts with + signs, it also assumes the next person to look at the script has the same familiarity with the –f operator. I can hear old hands saying “Anyone competent with PowerShell should be familiar with –f” but who said the person trying to understand your script meets _your_ definition of competence?

As someone who does a lot of stuff with regular expressions, I might be tempted by the final example in that list… but replacing the “end of string” marker ($) to as a way of appending has the resul of excluding people who aren’t happy with regex.  I’m working on something which auto-generates code at the moment and it uses this because the source that it reads doesn’t provide a way of specifying “append”, but has “replace”. I will let it slide in this case, but being bothered by it is a sign that I _do_ ask myself “are you showing you’re clever or are you writing something that can be worked on later”.  Sometimes the only practical way _is_ hard, but if there is a way which takes an extra minute to write and pays back when looking at the code in a few months time... 
  