---
layout: post
title:  "PowerShell functions and when NOT to use them"
date:   2019-04-06
categories: PowerShell
---
When I was taught to program, I got the message that _a function should be a “black box”_: we know what goes in one side, what comes out on the other, we don’t care how inputs become outputs. We learn that these “boxes” can _leak_, a function can see things outside and, in some cases (like connecting to another system) its purpose is to change its environment. But a function <u>shouldn’t manipulate the working variables used by the code which called it</u>. Recently I’ve found myself dealing with PowerShell authors who write like this:

{% highlight powerShell %}
$var_x = 10
$var_y = [math]::pi
do_stuff
$var_i = $var_y * $var_a
{% endhighlight %}

We can’t tell from reading this what `do_stuff` does, it seems to set `$var_a` because that has magically appeared; but does it use `$var_x` and `$var_y`? Does it change them? Will things break if we rename them? The only way to find out is to prise open the box and look inside (read the function definition). If you’re saying “_That function can’t change the value of `$var_x` because it’s not global_” here’s a fragment for you to copy and paste:

{% highlight powerShell %}
function do_stuff {
  Set-variable -Scope 1 -Name var_x -Value 30
}

$var_x = 10
do_stuff
$var_x
{% endhighlight %}

If the function just set `$var_x = $var_x + 20` that would put 30 into a new variable, local to the function  (`$var_x += 20` would add 20 to a new local variable, so the result is different). But it <u>didn’t</u> do that, it specifically said “set this variable in the scope 1 above this one”. That’s how things like `-ErrorVariable` and `-WarningVariable` work. Incidentally if the command setting the variable is in a function in a module, it is a jump of TWO levels to set things in the scope which called it. Recently I saw a good post from [Dave Carrol](https://powershell.anovelidea.org/powershell/how-i-implement-module-variables/) on using the script scope – which is a de-facto module scope as [this older post of Mike’s](https://mikefrobbins.com/2017/06/08/what-is-this-module-scope-in-powershell-that-you-speak-of/) explains – which can help to avoid this kind of thing.

You might wonder “would someone who doesn’t know how to write a function with parameters really use this…?” I’ve encountered it.
Another case where someone should be using parameters or at least making their variables script-scoped or globally-scoped, was this

{% highlight powerShell %}
Function Use-Data {
   $Y = [int]$data.xvalue * [int]$data.xvalue
   Add-Member -InputObject $data -MemberType NoteProperty -Name Yvalue -Value $y
}

$data = New-object pscustomobject
Add-Member -InputObject $data -MemberType NoteProperty -Name Xvalue -Value $x
Use-Data
{% endhighlight %}
Normally we can see the = sign and we know _this named place_ now holds _that value_. But `Set-Variable` and `Add-Member` make that harder to see. We would have one problem fewer to unravel if the writer used `$Global:X` and `$Global:Y`.

An example like the last one can be given a meaningful name, modified to take input through parameters and made to return the result properly. But the function is only called from one place. One of the main points of a function is to **reduce duplication**, but single-use is not an automatic reason to bring a function’s code into the main body of the script which calls it . For example, this:
`if (Test-PostalCodeValid $P) {...}`
saves me reading code which does the validation – there is no need to know _how_ it does it (the sort of regex used in such cases is better hidden); it is enough _that it does_: and the function has a **single purpose communicated by its name**. The problematic functions look like they are the writer’s first mental grouping of tasks (which leads to vague names) and the final product doesn’t benefit from that grouping. The script can’t be understood by reading from beginning to end – it requires the reader to jump back and forth – so _flattening_ the script makes it easier to follow. Because the functions are sets of loosely connected tasks, they don’t have a clear set of inputs or outputs and rely on _leakiness_.

**Replacing a block of code with a black-box whose purpose, inputs and outputs are all clear should make for a _better_ script**. And if those things are unclear, the script is probably _worse_ for putting things in boxes. You need to call a function many times for the tiny overhead in each call to matter, but I hit such a case while I was working on this post. Some users of [Export-Excel](https://www.powershellgallery.com/packages/ImportExcel) work on sheets with over a million cells (I use a 24,000-row x 23 column sheet for tests – 550K cells), and the following code was called for each row of data

{% highlight powerShell %}
$ColumnIndex = $StartColumn
foreach ($Name in $script:Header) {
    Add-CellValue -TargetCell $ws.Cells[$Row, $ColumnIndex] -CellValue $TargetData.$Name
    $ColumnIndex += 1
}
{% endhighlight %}
So, for my big dataset the `Add-CellValue` function was **called 550,000 times** which took about **80 seconds** in total or 150 microseconds per cell, on my machine. I’d say this fragment is clear and easy to work with: for each name in `$header`, that property of `$targetData` is added as a cell-value at the current row and column, and we move to the next column. `Add-CellValue` handles many different kinds of data – _how_ it does so doesn’t matter. This meets all the rules for a good function. BUT… of that 150μS more than 130 is spent going into and out of the function. That **80 seconds becomes about 8 seconds** if I put the function code in the for loop instead of calling out to it. Changes that cut the time to run a script from 0.5sec to 0.4999 sec don’t matter – you can’t use the saved time, and it is better to give up 100μS on each run for the time you save reading clearer code. Changing the time to run scripts from minutes to seconds does matter. So even though using the function was more elegant it wasn’t the best way. As both a computer scientist and an IT practitioner I never forget Jeffrey Snover’s saying _Computer scientists want elegant code; IT pros just want to go home._