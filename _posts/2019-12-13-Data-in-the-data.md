---
layout: post
title:  "Parameters and putting the data in the data."
date:   2019-12-31
categories: PowerShell
---
In the last post said my IT career began in the Mesolithic era; a recent discussion reminded me of something from my days as an expert in SharePoint Portal Server and a talk I gave back in 2002 which would be the Neolithic. Back then I tried to explain **don’t put data in the field names** back then I was talking about document metadata but it applies anywhere

![A column per version](/assets/colPerValue.png)

In the example above each possible value for a single question “which year are you in” has become its own yes/no question. “Are you in Year 7”, then “Are you in year 8”… it makes it awkward to ask for a list of pupils younger than X and so on; it’s just not a very good way. If multi-value fields are an option, it is much better to have one for “Subjects”, not separate tick boxes for “French”, “Geography”, “History”. But we do see skills databases with a column for every skill. And so on.

The PowerShell angle came about in a discussion about how parameter sets should behave. The person I was talking to had a fragment of script which looked roughly like the following pseudo-code

{% highlight powerShell %}
function Send-Message {
  param(
    [string] $Message,
    [Parameter(ParameterSetName = 'High')] [switch]$HighPriority,
    [Parameter(ParameterSetName = 'Low')]  [switch]$LowPriority
  )
  $Priority = 1
  if     ($HighPriority) { $Priority ++ }
  elseif ($LowPriority)  { $Priority -- }

  Invoke-sender $message $Priority
}
{% endhighlight  %}
His complaint was that PowerShell does not deduce that the script should run with neither HighPriority nor LowPriority. (Because he has a _Normal_ priority which works if _neither_ is specified) and to avoid a message
<code><font color="#c0504d">Parameter set cannot be resolved using the specified named parameters</font></code> he needs to either specify a default set, or say that one of the two parameters is mandatory.
Of course when a language has this sort of behaviour some people come to rely on it: if the options were “ID” and “Name” and providing neither meant some action would change all items without filtering by either, a parameter resolution error is the only thing standing between a mistyped command and disaster.

Experienced programmers / scripters will know that code sometimes grows in this way: first it has
`[switch]$HighPriority`
giving “ordinary” behaviour (send at priority of 1) and a switch to invoke “special mode” and run extra code (raising priority to 2). All is as it should be.
Then someone says “we should support low priority” and the parameters become
```
[switch]$HighPriority
[switch]$LowPriority
```
This means any existing scripts can retain `–HighPriority`. But things are already going wrong. We no longer have “Special” and “Ordinary”, selected by one switch (two choices), but priority 1,2 or 0 selected using two switches (four choices) “Both” is a nonsensical option so to prevent the user setting both, the parameters become
```
[Parameter(ParameterSetName = 'High')] [switch]$HighPriority,
[Parameter(ParameterSetName = 'Low')]  [switch]$LowPriority
```
At this point PowerShell doesn’t know there should be a third option. Both parameters are required in their respective sets but neither is marked as mandatory. If HighPriority were mandatory, PowerShell would interpret _neither_ to be the “Low” set with its optional parameter omitted. If a set can’t be used without a parameter, that parameter really should be mandatory, but when a set has a single parameter specifying the parameter selects the set. Keeping <u>one</u> set with _everything_ optional allows that set to be selected as a default if nothing can be inferred from the parameters which _are_ there.
I think it is better style to declare a third set, with no members of its own, to be the default. The extra set or the the “defaultable” set both side step the error when high and low are omitted, but they don’t make it clear that that results in a valid, safe, priority of 1. It is better to collapse the two switches into one parameter which can take any of the three values with a default, like this.

{% highlight powerShell %}
function Send-Message {
  param(
    [string] $Message,
    [ValidateSet('Normal','High','Low')]$Priority = 'High'
    )
  Switch ($Priority) {
    "Low"   {Invoke-Sender $message 0 }
    "High"  {Invoke-Sender $message 2 }
    default {Invoke-Sender $message 1 }
  }
}
{% endhighlight %}
High, Medium and Low will tab complete, making it obvious to the user that there are 3 choices, and it is equally obvious to someone maintaining the code. There’s no need to worry about parameter-sets, and the code is shorter. The following will keep the –HighPriority switch if existing scripts are using it:
{% highlight powerShell %}
  [parameter(DontShow)]
  [switch]$HighPriority
{% endhighlight %}

Marking a parameter as don’t show hides it in tab completion - useful for a deprecated option.  If it is specified and the priority is not, then Priority can be set accordingly.
{% highlight powerShell %}
  if (-not $PSBoundParameters.ContainsKey("Priority") -and $Highpriority) {
    $Priority = "High"
  }
{% endhighlight %}

While I was a drafting this post I came across [Adam Driscoll’s Selenium module](https://github.com/adamdriscoll/selenium-powershell/) which had an example which takes this problem up a level. I started work on supporting Selenium from PowerShell back in 2013/14 and when I dug some work out of my archive I found I had done near enough the same thing– it’s the natural way for the code to evolve, so don’t go inferring that this is (to borrow a phrase) “_[to point out] how the strong man stumbles, or where the doer of deeds could have done them better._”
Selenium is a test framework for loading web pages into a browser and checking their content and there are multiple ways to specify how an element should be found on the page (by ID, By XPath and so on), so you need to say find _this_, which is an ID, or find _that_, which is a class name and so on. Adams’s code covers more options than my old version did and the param block looks like this:

{% highlight powerShell %}
  [Parameter(ParameterSetName = "ByCss")]
  $Css,
  [Parameter(ParameterSetName = "ByName")]
  $Name,
  [Parameter(ParameterSetName = "ById")]
  $Id,
  [Parameter(ParameterSetName = "ByClassName")]
  $ClassName,
  [Parameter(ParameterSetName = "ByLinkText")]
  $LinkText,
  [Parameter(ParameterSetName = "ByPartialLinkText")]
  $PartialLinkText,
  [Parameter(ParameterSetName = "ByTagName")]
  $TagName,
  [Parameter(ParameterSetName = "ByXPath")]
  $XPath
{% endhighlight %}

All legal and valid. There are 8 parameter sets - if we needed to segment on something else it would become 16 or 24 or 32 sets. In the body of the function there is

{% highlight powerShell %}
  if ($PSCmdlet.ParameterSetName -eq "ByName") {
      $Target.FindElements([OpenQA.Selenium.By]::Name($Name))
  }

  if ($PSCmdlet.ParameterSetName -eq "ById") {
      $Target.FindElements([OpenQA.Selenium.By]::Id($Id))
  }
{% endhighlight %}

And this repeats for each parameter set. Using the logic I’ve already talked about I reduced the parameters to two
{% highlight powerShell %}
  [ValidateSet("CssSelector", "Name", "Id", "ClassName", "LinkText", "PartialLinkText", "TagName", "XPath")]
  [string]$By = "XPath",

  [Alias("Name", "Id", "ClassName","LinkText", "PartialLinkText", "TagName","XPath")]
  [Parameter(Position=1,Mandatory=$true)]
  [string]$Selection,
{% endhighlight %}
The old syntax of `–XPath "something"` has become `–By Xpath –selection "something"` or `–By Xpath "something"` (because “something” will be assumed to be selection) or simply `"something"` (because `–By` defaults to "Xpath".) and the main part of the code becomes one line

$Target.FindElements([OpenQA.Selenium.By]::$By($Selection))
The static method is selected using a parameter value, which is validated to be one of the allowed methods; and the value passed to the method always comes from the same parameter.
But this won’t (yet) work with the original syntax. However, because `-By` isn’t mandatory,and I have created aliases for the new selection parameter using the “lost” names the new version can still be called with  `-xpath "Something"`. It needs a little extra code (which appears below) to recognise that has happened and set `$By` to the right value. First it finds the name which was used to call the function (it might be an alias, or the function might be renamed), then if the `–By` parameter hasn’t been supplied and the command line reads `InvocationName  <<anything except ‘>’, ‘|’ or ‘;’>> –ParameterAlias` it captures the parameter alias and puts it into `$by`
{% highlight powerShell %}
  $mi = $MyInvocation.InvocationName
  if(-not $PSBoundParameters.ContainsKey("By") -and
        ($MyInvocation.Line -match  "$mi[^>\|;]*-(Name|Id|ClassName|LinkText|PartialLinkText|TagName|XPath)")) {
    $By = $Matches[1]
  }
{% endhighlight  %}
Net, I shortened the function by about 60 lines, and with this flourish, kept it compatible.