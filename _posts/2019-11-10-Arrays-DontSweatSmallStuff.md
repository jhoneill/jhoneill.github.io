---
layout: post
title:  "PowerShell Arrays, performance and [not] sweating the small stuff."
date:   2019-11-10
categories: PowerShell
---
I’ve read a couple of posts on arrays recently [Anthony (a.k.a. the POSH Wolf)](https://theposhwolf.com/howtos/PS-Plus-Equals-Dangers/) posted one and [Tobias had another](https://powershell.one/tricks/performance/arrays). To their advice I’d add **Avoid creating huge arrays, where practical**.  [I’ve written about](/powershell/2017/03/13/ImprovingWithHashTables.html) the problems doing “Is X in the set” with large arrays; hash-tables were a better answer in that case. Sometimes we can avoid storing a big lump of data altogether, and I often prefer designs which do.

We often know this technique is “better” than that one, but we also want code which takes a short **time-to-write**; is clear so that later it has a short **time-to-understand**, but doesn’t take an excessive **time-to-run**. As with most of these triangles, you can **often get two and rarely get all** three. Spending 30 seconds writing something which takes 2 seconds to run might beat something which takes 10 minutes to write and runs in 50 milliseconds. But neither is any good if next week we spend hours figuring out how the data was changed last Tuesday.

Clarity and speed _aren’t_ mutually exclusive, but sometimes there is a clear, familiar way which doesn’t scale up and a less attractive technique which does. Writing something which scales to "bicycle" will hit trouble when the problem reaches "Jumbo Jet" size, and applying "Jumbo" techniques to a "bike" size problem can be an unnecessary burden. And (of course) expertise is knowing both the techniques and where they work (and don’t).

One particular aspect of **arrays in PowerShell** causes a problem at large scale. Building up large arrays one member at a time is something to try to design out, but sometimes it is the most (or only) practical way. PowerShell arrays are created as a fixed size; adding a member means creating a new array, and copying the existing array and one more member to a new array - which gets slower as the array gets bigger. If the time to do each of n operations depends on the number done so far, which is 0 at the start, _n_ at the end and averages _n/2_ during the process, the average time per item is _some_constant * n/2_. Let’s define _k_ as 2* the constant, so average time per item is _kn_  and time to do all n items is _kn²_. The time rises with an exponent of n. People like to say “rises exponentially” for “fast” but this _is_ exponential. You can try this test, the result from my computer appears below. The numbers don’t perfectly fit a square law, but the orders of magnitude do.
```
$hash=[ordered]@{}
foreach ($size in 100,1000,10000,100000) {
  $hash["$size"]=(measure-command {
       $array=@(); foreach ($x in (1..$size)){$array += $x}
  }).TotalMilliseconds;
}
$hash
```

<table cellspacing="0" cellpadding="2" border="1"><tbody>
<tr><td valign="top"><p>Array Size</p></td><td valign="top"><p>Total Milliseconds</p></td></tr>
<tr><td valign="top"><p>100       </p></td><td valign="top"><p>5      </p></td></tr>
<tr><td valign="top"><p>1,000     </p></td><td valign="top"><p>43     </p></td></tr>
<tr><td valign="top"><p>10,000    </p></td><td valign="top"><p>2,800  </p></td></tr>
<tr><td valign="top"><p>100,000   </p></td><td valign="top"><p>310,000</p></td></tr>
</tbody></table>

If 43ms sounds a bit abstract, disqualification rules in athletics say you can’t react in less than 100ms. A “blink of an eye” takes about 300-400ms. It takes ~60ms for PowerShell to generate my prompt, **it’s not worth cutting less than 250ms off  time-back-to-prompt**. Even then a minute’s work to save a whole second only pays for itself after 60 runs. ([I wrote in April](/powershell/2019/04/06/NotUsingFunctions.html) about how very small “costs” for an operation can be multiplied many-fold: saving less than 100ms on an operation still adds up if a script does that operation 100,000 times; we can also see a difference when typing between 50 and 100ms responses, but here I’m thinking of things which only run once in a script).
At 10K array items, the possible saving is a couple of seconds, this is in the band of acceptable times that are slow enough to notice. Slower still and human behaviour changes: we think it’s crashed, or swap to another task and introduce a coffee break before the next command runs. 100K items takes 5 minutes. But even that might be acceptable in a script which runs as a scheduled task. Do you want to guess how long a million would take ?
`$a = 1..999999` will put 999,999 items into an array in 60ms on my machine – `$variable = Something_which_outputs_an_array`   is usually a quick operation.
`$a += 1000000` takes 100ms. **Adding the millionth item takes as long as adding the first few thousand**. The first 100K take a few _minutes_, the last 100K take a few _hours_. And that’s too long even for a scheduled task.

The exponent which makes things scale UP badly means they scale DOWN brilliantly – is a waste of effort to worry about scale if tens of items is a _big_ set, but when thousands of items is a _small_ set it could be critical. Removing operations where each repetition takes longer than the one before can be a big win because these are the root of a exponential execution time.
The following scrap works, but its is unnecessarily slow; it’s not dreadful on clarity but it is longwinded. There are also faster methods than piping many items into foreach-object
```
$results = @()
Get-Stuff | foreach-object {$results += $_ }
return $results
```
This pattern can stem from thinking every function must have a return, which is called exactly once, which isn’t the case in PowerShell. It’s quicker and simpler just as `Get-Stuff,` or if some processing needs to happen between getting and returning the results then something like the following, if the work is done object-by-object:

`Get-Stuff | foreach-object {output_processed $_} `   
or if the work must be done on the whole set:
```
$results = Get-Stuff
work_on $results  #returning  final result
    
```
Another pattern looks like this.
```
put_some_result_in $a
some_loop {
  Operations
  $a += result
}
    
```

this works better as
```
put_some_result in_$a
$a += some_loop {
  Operations
  Output_result
}
```
A lot of cases where a better “add to array” looks like the answer are forms of this pattern and getting down to just one add is a better answer.
When thousands of additions are unavoidable, a lot of advice says use `[Arraylist]` but as Anthony’s post points out, [more recent advice](https://docs.microsoft.com/en-us/dotnet/api/system.collections.arraylist?view=netframework-4.8#remarks) is to use `[List[object]]` or `[List[Type]]`.

## Postscript

At the same time as I was posting this, [Tobias was looking at the same problem with strings](https://powershell.one/tricks/performance/strings). Again building up a 1,000,000 line string one line at a time is something to be avoided and again, it takes a lot longer to add create a new sting which is old-string + one-line when the old-string is big than when it is small, and I found that fitted a square law nicely: 10,000 string-appends took 1.7 seconds; 100,000 took 177 seconds. It takes as long to add 10 strings at lines to 100,000 to 100,010 line as adding the first 3,000 to an empty version. His conclusion – that if you really can’t avoid doing this, using a stringBuilder is much more efficient – is a good one, but I wouldn’t bother with one to join half a dozen strings together.