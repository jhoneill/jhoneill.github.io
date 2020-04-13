---
layout: post
title:  "Powershell Piped Parameter Peculiarities (and a Palliative pattern!)"
date:   2016-11-30
categories: PowerShell
--- 
Writing some notes before sharing a PowerShell module,  I did a quick fact check and rediscovered a hiccup with piped parameters and (eventually) remembered writing a simplified script to show the problem – 3 years ago as it turns out. The script appears below: it has four parameter sets and all it does is tell us which parameter set was selected: There are four parameters: A is in all 4 sets, B is in Sets 2,3 and 4, C is only in 3 and D is only in set 4. I’m not really a fan of parameter sets but they help intellisense to remove choices which don’t apply. 
{% highlight powershell %}
function test { 
[CmdletBinding(DefaultParameterSetName="PS1")]
param (  [parameter(Position=0, ValueFromPipeLine=$true)]
         $A
         [parameter(ParameterSetName="PS2")]
         [parameter(ParameterSetName="PS3")]
         [parameter(ParameterSetName="PS4")]
         $B,
         [parameter(ParameterSetName="PS3", Mandatory)]
         $C,
         [parameter(ParameterSetName="PS4", Mandatory)]
         $D
)
$PSCmdlet.ParameterSetName
}
{% endhighlight %}
So lets check out what comes back for different parameter combinations    
`> test  1`    
`PS1`    
No parameters or parameter A only gives the default parameter set. Without parameter C or D it can’t be set 3 or 4, and with no parameter B it isn’t set 2 either.

`> test 1 -b 2`    
`PS2`    
Parameters A & B or parameter B only gives parameter set 2, – having parameter B it must be set 2,3 or 4 and but 3 & 4 can be eliminated because C and D are missing. 

`> test 1 -b 2 –c 3 `    
`PS3`    
Parameter C means it must be set 3 (and D means it must be set 4) ; so lets try piping the input for parameter A

`> 1 | test`     
`PS1`    
`> 1 | test  -b 2 -c 3`    
`PS3`

So far it’s as we’d expect.  But then something goes wrong.    
<code>1 | test  -b 2 <br/>
<span style="color:red">Parameter set cannot be resolved using the specified named parameters</span></code>

Eh ? If data is being piped in, PowerShell <u>no longer <i>infers</i> a parameter set from the <i>absent</i> mandatory parameters</u>.  Which seems like a bug. And I thought about it: why would piping something change what you can infer about a parameter not being on the command line? Could it be uncertainty whether values could come from properties the piped object ? I thought I’d try this hunch   
{% highlight powershell %}
  [parameter(ParameterSetName="PS3", Mandatory,ValueFromPipelineByPropertyName=$true)]
  $C,
  [parameter(ParameterSetName="PS4", Mandatory,ValueFromPipelineByPropertyName=$true)]
  $D
{% endhighlight %}
This does the trick – though I don’t have a convincing reason why two places not providing the values works better than one – (in fact that initial hunch doesn’t seem to stand up to logic). This (mostly) solves the problem– there could be some odd results if parameter D was named “length” or “path” or anything else commonly used as a property name. I also found in the “real” function that adding `ValueFromPipelineByPropertyName` to too many parameters – non mandatory ones – caused PowerShell to think a set had been selected and then complain that one of the mandatory values was missing from the piped object. So just adding it to every parameter isn’t the answer.
