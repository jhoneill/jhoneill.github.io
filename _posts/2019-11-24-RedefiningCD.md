---
layout: post
title:  "Redefining CD in PowerShell."
date:   2019-11-24
categories: PowerShell
---

For some people my IT career must seem to have begun in Mesolithic times – back then we had a product called [Novell Netware](https://en.wikipedia.org/wiki/NetWare) (and Yorkshiremen of a certain age will say _“Aye, and Rickets and Diphtheria too”_). But I was thinking about one of of Netware’s features recently; well as the traditional `cd ..` for the parent directory Netware users could refer to two levels up as ..., three levels up as .... and so on. And after a PowerShell session going up and down a directory tree I got nostalgic for that. And I though...

-  I can convert some number of dots into a repetition of “..\” fairly easily with regular expressions.
-  I’ve recently written a blog post about argument transformers and
-  I already change cd in my profile, so why not change it a little more ?

By default, PowerShell defines `cd` as an alias for `SET-Location` and for most of the time I have been working with PowerShell I have set `cd-` as an alias for `POP-Location`, deleted the initial cd alias (until PowerShell 6 there was no `Remove-Alias` cmdlet, so this meant using `Remove-Item Alias:\cd –force`) and created a new alias from `cd` to `PUSH-location` , so I can use `cd` in the normal way but I have `cd-` to re-trace my steps.
To get the exta functionality means attaching and **Argument transformer** to the parameter where it is declared, so I would have to make “new cd” a _function_ instead of an -alias). The basic part of it looks like this:-

{% highlight powerShell %}
function cd {
    <#
.ForwardHelpTargetName Microsoft.PowerShell.Management\Push-Location
.ForwardHelpCategory Cmdlet
#>
    [CmdletBinding(DefaultParameterSetName='Path')]
    param(
 [Parameter(ParameterSetName='Path', Position=0,
      ValueFromPipeline=$true, ValueFromPipelineByPropertyName=$true)]
 [PathTransform()]
 [string]$path
    )
    process {
 Push-Location @PSBoundParameters
    }
}
{% endhighlight %}
The finished item (posted [here](https://gist.github.com/jhoneill/47f5151b22a1dabb4ddc79c083162f77)) has more parameters – it is built like a [proxy function](/powershell/2012/02/04/ProxyFunctions.html), it forwards help to `Push-Location’s` help. If the path is “–”(or a sequence of – signs)  `Pop-Location` is called for each “–”, so I can use a bash-style to cd  - as well as cd-  and `Push-Location` is only called if a path is specified.
If the path isn’t valid I don’t want the error to say it occurred at a location in the function so I added a validate script to the parameter.

The key piece is the `[PathTransform()]` attribute on the path Parameter – it comes from a [class, with a name ending “attribute”](/powershell/2019/09/23/ClassyParameters.html) (which can be omitted when writing the parameter attribute in the function). Initially the class was mostly wrapping around one line of code
{% highlight powerShell %}
class PathTransformAttribute : System.Management.Automation.ArgumentTransformationAttribute {
    [object] Transform([System.Management.Automation.EngineIntrinsics]$EngineIntrinsics,
  [object] $InputData)
     {
 return $InputData -replace "(?<=^\.[./\\]*)(?=\.{2,}(/|\\|$))"  ,  ".\"
    }
}
{% endhighlight  %}

The `class` line defines the name and says it descends from the `ArgumentTransformationAttribute` class;
the next line says it has a `Transform` method which returns an object, and takes parameters `EngineIntrinsics`, and `InputData`
and the line which does the work is a regular expression. In Regex:
`(?<=AAA)(?=ZZZ)`
says find the part of the text where looking behind you, you see AAA and looking ahead, you see ZZZ; this doesn’t specify anything to select between the two, so “replacing” it doesn’t remove anything it is just “insert where…”.
In the code above, the look-behind part says ‘the start of the text (“^”), a dot (“\.”), and then dots, forward or back slashes (“[./\\]”) repeated zero or more times (“*”) ’ ;  and the look ahead says ‘a dot (“\.”) repeated at least 2 times (“{2,}”) followed by / or \ or the end of the text (“/|\\|$”).
So names like readme...txt won’t match, neither will ...git but ...\.git will become ..\..\.git. .

BUT ...\[tab\] and doesn’t expand two levels up – the parameter needs an **argument completer** for that. Completers take information about the command line – and especially the current word to complete - and return `CompletionResult` objects for tab expansion to suggest.
PowerShell has 5 ready-made completers for Command, Filename, Operator, Type and Variable. Pass any of these completers a word-to-complete and it returns  CompletionResult objects – for example you can try
[System.Management.Automation.CompletionCompleters]::CompleteOperator("-n")

A simple way to use for one of these is to view help in its own window, a feature which is returning in PowerShell 7 (starting in preview 6); I  like this enough to have a little function, `Show-Help` which calls  `Get-Help –ShowWindow`. Adding an argument completer, my function’s command parameter means it tab-completes matching commands.

{% highlight powerShell %}
function Show-Help {
  param (
    [parameter(ValueFromPipeline=$true)]
    [ArgumentCompleter({
 param($commandName, $parameterName,$wordToComplete,$commandAst,$fakeBoundParameter)
 [System.Management.Automation.CompletionCompleters]::CompleteCommand($wordToComplete)
    })]
    $Command
  )
  process {foreach ($c in $Command) {Get-Help -ShowWindow $c} }
}
{% endhighlight %}

The completer for `Path` in my new `cd` needs more work and there was a complication which took little while to discover: **PSReadline caches alias parameters** and their associated completers so after the `cd` alias is replaced my profile I need to have this:

{% highlight powerShell %}
if (Get-Module PSReadLine) {
    Remove-Module -Force PsReadline
    Import-Module -Force PSReadLine
    Set-PSReadlineOption -BellStyle None
    Set-PSReadlineOption -EditMode Windows
}
{% endhighlight %}
You might have other `psreadline` options to set.
I figured that I might want to use my new completer logic in more than one command, and I also prefer to keep anything lengthy scripts out of the Param() block, which led me to use an **argument completer class**. The outline of my class appears below:

{% highlight powerShell %}
class PathCompleter : System.Management.Automation.IArgumentCompleter {
    [System.Collections.Generic.IEnumerable[ System.Management.Automation.CompletionResult]] CompleteArgument(
     [string]$CommandName,
     [string]$ParameterName,
     [string]$WordToComplete,
     [System.Management.Automation.Language.CommandAst]$CommandAst,
     [System.Collections.IDictionary] $FakeBoundParameters
    )
    {
 $CompletionResults = [System.Collections.Generic.List[ System.Management.Automation.CompletionResult]]::new()

 # populate $wtc from $WordToComplete

foreach ($result in
    [System.Management.Automation.CompletionCompleters]::CompleteFilename($wtc) ) {
      if ($result.resultType -eq "ProviderContainer") {$CompletionResults.Add($result)}
 }
 return $CompletionResults
    }
}
{% endhighlight  %}

The `class` line names the class and says it implements the `IArgumentCompleter` interface, Everything else defines the class’s `CompleteArgument` method, which returns a collection of completion results, and takes the standard parameters for a completer ([seen here](/powershell/2019/09/23/ClassyParameters.html)). The body of the method creates the collection of results as its first line and returns that collection as its last line, in-between it calls the `CompleteFileName` method I mentioned earlier, filtering the results to containers. The final version uses the `CommandName` parameter to filter results for some commands and return everything for others. Between initializing $CompletionResults and the foreach loop is something to convert the `WordToComplete`  parameter into the $wtc argument passed to CompleteFileName ...
The initial idea was to expand 3, 4, or more dots. But I found ..\[tab\] .\[tab\] and ~\[tab\] do not expand – they all need a trailing \ or /.  “I can fix that” I thought ... and then I thought

-  “Wouldn’t it be could if I could find a directory somewhere on my current path” so if I’m in a sub-sub-sub-folder of Documents  \*doc \[tab\] will expand to documents.
-  What about getting back to the PowerShell directory ? I decided ^[tab] should get me there.
-  Previously pushed locations on the stack? It would be nice if I could tab expand "-" but PowerShell takes that to be the start of a parameter name, not a value so I use = instead `=[tab]` will cycle through locations `== [tab]` gives 2nd entry on the stack `===[tab]` the third and so on.
There aren’t many characters to choose from; “.” and all the alphanumerics are used in file names; #$@-><;,| and all the quote and bracket characters tell PowerShell about what comes next. \ and / both mean “root directory”, ? and * are wild cards, ~ is the home directory. Which leaves !£%^_+ and = as available (on a UK keyboard), and = has the advantage of not needing shift. And I’m sure some people use ^ and or = at the start of file names  – they’d need to change my selections.

All the new things to be handled go into one **regular-expression based switch statement** as seen below; the regexes are not the easiest to read because so many of characters need to be escaped. “\\\*” translates as \ followed by * and “^\^” means “beginning with a ^”  and the result looks like some weird ascii art.

{% highlight powerShell %}
$dots    = [regex]"^\.\.(\.*)(\\|$|/)"
$sep     = [system.io.path]::DirectorySeparatorChar
$wtc     = ""
switch -regex ($wordToComplete) {
    $dots{$newPath = "..$Sep" * (1 + $dots.Matches($wordToComplete)[0].Groups[1].Length)
    $wtc = $dots.Replace($wordtocomplete,$newPath) ; continue }
    "^=$"{ foreach ($stackPath in (Get-Location -Stack).ToArray().Path) {
      if ($stackpath -match "[ ']") {$stackpath = '"' + $stackPath + '"'}
      $results.Add([System.Management.Automation.CompletionResult]::new($stackPath))
      }
      return $results ; continue
  }
    "^=+$"      {$wtc = (Get-Location -Stack).ToArray()[$wordToComplete.Length -1].Path  ; continue }
    "^\\\*|/\*" {$wtc = $pwd.path -replace "^(.*$($WordToComplete.substring(2)).*?)[/\\].*$",'$1' ; continue }
    "^~$"{$wtc = $env:USERPROFILE  ; continue }
    "^\^$"      {$wtc = $PSScriptRoot     ; continue }
    "^\.$"      {$wtc = ""  ; continue }
    default     {$wtc = $wordToComplete}
}
{% endhighlight %}

Working up from the bottom,

-  The default is to use the parameter as passed in CompleteFileName. Every other branch of the switch uses continue to jump out without looking at the remaining options.
-  If the parameter is “.”, ”^” or “~” `CompleteFileName` will be told use an empty string, the script directory or the user’s home directory respectively.
`($env:userProfile` is only set on Windows by default. Earlier in my profile I have something to set it to `[Environment]::GetFolderPath([Environment+SpecialFolder]::UserProfile)` if it is missing, and this will return the home directory regardless of OS)
-  If the parameter begins with \* or begins with /* the script takes the current directory, and selects from the beginning to whatever comes after the * in the parameter, and continues selecting up to the next / or \ and discards the rest. The result is passed into completeFileName
-  If the parameter contains a sequence of = signs and nothing else, a result is returned which from the stack, = is position 0, == is position 1 using the length of the parameter
- If the parameter is a single = sign the function returns without calling Completefilename . It looks at each item on the stack in turn, those which contain either a space or a single quote, are wrapped in double quotes before being added to $results, which is returned at the end is returned.
- And the first section of the switch uses an existing regex object as the regular expression. The regex object will get the sequence of dots before the last two, and repeats “..\”  as many times as there are dots, and drops that into `$WordToComplete` . PowerShell is quite happy to use / on Windows where \ would be normal, and to use \ on Linux where / would be normal. Instead of hard coding one I get the “normal” one as `$sep` and insert that with the two dots.

Adding support for = and ^ meant going back to the argument transformer and adding the option so that `cd ^ [Enter]` and `cd = [Enter]` work

I’ve put the [code here](https://gist.github.com/jhoneill/47f5151b22a1dabb4ddc79c083162f77) and a summary of what I’ve enabled appears below.

<table cellspacing="0" cellpadding="2" border="1"><tbody>
<tr><td valign="top" width="240"><p><strong>Keys</strong></p></td><td valign="top" width="320"><p><strong>Before</strong></p></td><td valign="top" width="600"><p><strong>After</strong></p></td></tr>
<tr><td valign="top" width="240"><p>cd ~[Tab]            </p></td><td valign="top" width="320"><p>- (needs ~\)</p></td><td valign="top" width="600"><p>Expands &lt;home&gt;</p></td></tr>
<tr><td valign="top" width="240"><p>cd ~[Enter]          </p></td><td valign="top" width="320"><p>Set-Location &lt;home&gt;</p></td><td valign="top" width="600"><p>Push-Location &lt;Home&gt;</p></td></tr>
<tr><td valign="top" width="240"><p>cd ..[Tab]           </p></td><td valign="top" width="320"><p>- (needs ..\)</p></td><td valign="top" width="600"><p>Expands &lt;Parent<em>&gt;</em> </p></td></tr>
<tr><td valign="top" width="240"><p>cd ..[Enter]         </p></td><td valign="top" width="320"><p>Set-Location &lt;parent&gt;</p></td><td valign="top" width="600"><p>Push-Location &lt;parent&gt;</p></td></tr>
<tr><td valign="top" width="240"><p>cd ...[Tab]          </p></td><td valign="top" width="320"><p>-    </p></td><td valign="top" width="600"><p>Expands &lt;grandparent&gt;  <br />and higher levels with each extra “.”</p></td></tr>
<tr><td valign="top" width="240"><p>cd ...[Enter]        </p></td><td valign="top" width="320"><p>ERROR</p></td><td valign="top" width="600"><p>Push-Location &lt;grandparent&gt;<br />&amp; beyond with each extra “.”</p></td></tr>
<tr><td valign="top" width="240"><p>cd /*pow [Tab]       </p></td><td valign="top" width="320"><p>-    </p></td><td valign="top" width="600"><p>Expand directory/ directories above containing “pow”</p></td></tr>
<tr><td valign="top" width="240"><p>cd /*pow [Enter]     </p></td><td valign="top" width="320"><p>ERROR</p></td><td valign="top" width="600"><p>Push-location to directory containing “pow”<br />(if unique; error if not unique)</p></td></tr>
<tr><td valign="top" width="240"><p>cd ^[Tab]            </p></td><td valign="top" width="320"><p>-    </p></td><td valign="top" width="600"><p>Expands PS Profile directory </p></td></tr>
<tr><td valign="top" width="240"><p>cd ^[Enter]          </p></td><td valign="top" width="320"><p>ERROR</p></td><td valign="top" width="600"><p>Push-Location PS Profile directory</p></td></tr>
<tr><td valign="top" width="240"><p>cd =[Tab]            </p></td><td valign="top" width="320"><p>-    </p></td><td valign="top" width="600"><p>Cycle through location stack</p></td></tr>
<tr><td valign="top" width="240"><p>cd =[Enter]          </p></td><td valign="top" width="320"><p>ERROR</p></td><td valign="top" width="600"><p>Push-location to <em>n</em>th on stack:<br /> = is 1st, == 2<sup>nd</sup>etc      <br />(and <strong>allow</strong> ‘Pop’ back to current location) </p></td></tr>
<tr><td valign="top" width="240"><p>cd -[Enter]          </p></td><td valign="top" width="320"><p>ERROR</p></td><td valign="top" width="600"><p>Pop-location (repeats Pop for each extra -)     <br /><strong>Does not allow</strong> Pop back to current location</p></td></tr>
<tr><td valign="top" width="240"><p>cd- [Enter]          </p></td><td valign="top" width="320"><p>ERROR</p></td><td valign="top" width="600"><p>Pop-location</p></td></tr>
<tr><td valign="top" width="240"><p>cd\ [Enter]          </p></td><td valign="top" width="320"><p>Set-Location \</p></td><td valign="top" width="600"><p>Push Location \</p></td></tr>
<tr><td valign="top" width="240"><p>cd.. [Enter]         </p></td><td valign="top" width="320"><p>Set-Location ..</p></td><td valign="top" width="600"><p>Push-location ..</p></td></tr>
<tr><td valign="top" width="240"><p>cd~ [Enter]          </p></td><td valign="top" width="320"><p>ERROR</p></td><td valign="top" width="600"><p>Push-Location ~</p></td></tr>
s</tbody></table>
