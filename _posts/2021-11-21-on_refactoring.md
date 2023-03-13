---
layout: post
title:  "On Refactoring the code of others"
date:   2021-11-20
categories: PowerShell
tags: 
    - PowerShell
    - Development
---
I have a twist on the old proverb about not criticizing someone until you have walked a mile in their moccasins.
> Don't criticize someone until you have spend an hour re-factoring their code.

I find "massaging" code others have written gives me the best sense of what it's doing; imposing my own style stops me fixating on other people's and helps me get through to what's really happening. But it does something else. Often I say "write code to be understood later" or "write it for when you need to fix it, in a crisis situation, having forgotten all about it". The person who writes a piece of code initially isn't optimizing for *anything*, not speed, not size, not readability, getting it to work is their sole concern. Once it works, other priorities often stop it being made more readable.

After a few conversations recently I wanted to share what I do, so I made notes as I worked on the example below; it was publicly shared on Octopus deploy's web site: someone posted helpful code, and I'm worried that it looks ungracious to pull it apart, the examples of code I've fixed for clients are private and this is public so this doesn't have the confidentially worries, but this isn't a pop at anyone just an example for showing *what I do*. Others may want to do something similar to *my* code - I'm sure there is scope to do so. What this code is *for* doesn't matter too much, so don't spend a lot of time reading what appears immediately below, it's here to refer back to.

{% highlight PowerShell %}
$ErrorActionPreference = "Stop";

# Define working variables
$octopusURL = "https://youroctourl"
$octopusAPIKey = "API-YOUR_API_KEY"
$header = @{ "X-Octopus-ApiKey" = $octopusAPIKey }

Function Clear-SensitiveVariables
{
    # Define function variables
    param ($VariableCollection)

    # Loop through variables
    foreach ($variable in $VariableCollection)
    {
        # Check for sensitive
        if ($variable.IsSensitive)
        {
            $variable.Value = [string]::Empty
        }
    }

    # Return collection
    return $VariableCollection
}

# Get space
$space = (Invoke-RestMethod -Method Get -Uri "$octopusURL/api/spaces/all" -Headers $header) | Where-Object {$_.Name -eq $spaceName}

# Get all projects
$projects = Invoke-RestMethod -Method Get -Uri "$octopusURL/api/$($space.Id)/projects/all" -Headers $header

# Loop through projects
foreach ($project in $projects)
{
    # Get variable set
    $variableSet = Invoke-RestMethod -Method Get -Uri "$octopusURL/api/$($space.Id)/variables/$($project.VariableSetId)" -Headers $header

    # Check for variables
    if ($variableSet.Variables.Count -gt 0)
    {
        $variableSet.Variables = Clear-SensitiveVariables -VariableCollection $variableSet.Variables

        # Update set
        Invoke-RestMethod -Method Put -Uri "$octopusURL/api/$($space.Id)/variables/$($project.VariableSetId)" -Body (
                            $variableSet | ConvertTo-Json -Depth 10) -Headers $header
    }
}

# Get all library sets
$variableSets = Invoke-RestMethod -Method Get -Uri "$octopusURL/api/$($space.Id)/LibraryVariableSets/all" -Headers $header

# Loop through variableSets
foreach ($variableSet in $variableSets)
{
    # Get the variableSet
    $variableSet = Invoke-RestMethod -Method Get -Uri "$octopusURL/api/$($space.Id)/libraryvariablesets/$($variableSet.Id)" -Headers $header

    # Check for variables
    if ($variableSet.Variables.Count -gt 0)
    {
        $variableSet.Variables = Clear-SensitiveVariables -VariableCollection $variableSet.Variables

        # Update set
        Invoke-RestMethod -Method Put -Uri "$octopusURL/api/$($space.Id)/libraryvariablesets/$($variableSet.Id)" -Body (
                    $variableSet | ConvertTo-Json -Depth 10) -Headers $header            
    }
}
{% endhighlight  %}

This code:

1. Declares some variables
2. Declares a function which does something to sensitive values.
3. Makes a REST call to find a "space" - we don't need to know about spaces here, except that the code...
4. Uses the space in another API call to get projects - we don't need to know about them either...
5. For each project it makes an API to get an "variable set" (whatever that is), calls the sensitive values function, and makes another API call to write the variable set back.
6. Repeats 4 & 5 with using a "library" instead of "projects"

So basically, get a list, read each thing in the list, transform it, write it back. There must be millions of scripts just like that. But this one is 66 lines and 2300 characters, and I have to stare it quite hard to get the sense. *Giving me the urge to massage the script.*

The first thing I'm going to do **merge lines with only an open brace into the previous line**. People who learned to program in a language with its roots in C (and that includes not just c++ and c# but also java) prefer to put `{` on its own line. This made sense when we looked at our code as print-outs on continuous paper, and it's too late to change it now.  
Because PowerShell uses `{ScriptBlock}` as a parameter in many cmdlets, something like  
`$list | foreach { Invoke-Something $_ }`  
**needs** the open brace (or open quote marks for a string parameter), on the same line as its command, unless we use backtick to tell PowerShell to splice two lines into one, (I'm using `foreach` as an alias for `ForEach-Object` which isn't good practice, but I want to compare with the `foreach` keyword). A *short* script-block on the same line also makes sense, so if I write:

{% highlight PowerShell %}
foreach ($item in @(1,2,3)) {$x * $x}  
  
{% endhighlight  %}

It doesn't make it any more readable to turned into
{% highlight PowerShell %}
foreach ($item in @(1,2,3))
  {
    $x * $x
  }
  
{% endhighlight  %}

To demonstrate with an extreme case it is quite legal to write this:
{% highlight PowerShell %}
foreach
(
$x
in
@(
1
2
3
)
)
{
$x *
$x
}

{% endhighlight  %}

I'm not looking at listings on (portrait format) print-outs but on a landscape format screen. Vertical space is constrained, horizontal space is plentiful.

All this means I tell VSCode to replace the regular expression "`\n\s*\{\n`"  (*New-line, any spaces, open brace, newline*)  
with " `{\n`" (*one space, open brace and a new line*).  
Generally *Closing* braces which aren't round very simple blocks go on *their own line* unless they are followed by a `|` symbol - so I don't write `} else {`

The comments in the example code are not informative - where there's a variable named `$thing` there is no need to write "`Get the thing`" before it, (tell me about the *process of* getting it if necessary, but I can see you *are* getting it) so I can strip those out replacing "`\n\s*#.*\n`" (*New-line, any spaces, #, anything, New-line*) with "`\n`" (*New-line*)

A couple more changes in the first pass are to do a little alignment, and to put powershell language statements (`function`, `foreach`, `if`, `else` and so on) into lower case; finally I'll weed out brackets which aren't needed and don't improve readability and sometimes add some for readability's sake.  

I notice a lot of scripts where `$()` has been put around most of the expressions where it's typically only needed in a few cases,  
and where c habits have compelled the writer to give every line a semi-colon I'll weed those out as I go.  

**Two pieces of advice** here. First if you are contributing to something you don't own, avoid reformatting. There is no sense at all in reformatting 10,000 lines of theirs to contribute 10 of your own, and it won't be welcome. So when I worked on a project which used three spaces for indents - it should be four and not a tab character - I swore under my breath and used three. Some people write the keywords in upper case, it's better to go with the flow on that too. Hundreds or thousands of lines showing as changed because you removed a semi-colon or even spaces is a burden when someone goes to merge your changes. Whoever "owns" the code sets these rules they'll be convinced they are right even if you know they are are wrong. Which leads to the second bit of advice - if you are creating a whole new version -as I am - apply your own rules, mine may or - especially for some things I do to align text - may not suit you.

**After these changes, the text is pretty dense...**
{% highlight PowerShell %}
$ErrorActionPreference = "Stop"
$octopusURL            = "https://youroctourl"
$octopusAPIKey         = "API-YOURAPIKEY"

Function Clear-SensitiveVariables{
    param   ($VariableCollection)
    foreach ($variable in $VariableCollection){
        if  ($variable.IsSensitive){$variable.Value = [string]::Empty}
    }
    return   $VariableCollection
}
$space           =  Invoke-RestMethod -Method Get -Uri "$octopusURL/api/spaces/all" -Headers $header | Where-Object {$_.Name -eq $spaceName}
$projects        =  Invoke-RestMethod -Method Get -Uri "$octopusURL/api/$($space.Id)/projects/all" -Headers $header
foreach ($project in $projects){
    $variableSet =  Invoke-RestMethod -Method Get -Uri "$octopusURL/api/$($space.Id)/variables/$($project.VariableSetId)" -Headers $header
    if ($variableSet.Variables.Count -gt 0){
                    $variableSet.Variables = Clear-SensitiveVariables -VariableCollection $variableSet.Variables
                    Invoke-RestMethod -Method Put -Uri "$octopusURL/api/$($space.Id)/variables/$($project.VariableSetId)" -Body (
                        $variableSet | ConvertTo-Json -Depth 10) -Headers $header
    }
}
$variableSets    =  Invoke-RestMethod -Method Get -Uri "$octopusURL/api/$($space.Id)/libraryvariablesets/all" -Headers $header
foreach ($variableSet in $variableSets){
    $variableSet =  Invoke-RestMethod -Method Get -Uri "$octopusURL/api/$($space.Id)/libraryvariablesets/$($variableSet.Id)" -Headers $header
    if ($variableSet.Variables.Count -gt 0){
                    $variableSet.Variables = Clear-SensitiveVariables -VariableCollection $variableSet.Variables
                    Invoke-RestMethod -Method Put -Uri "$octopusURL/api/$($space.Id)/libraryvariablesets/$($variableSet.Id)" -Body (
                            $variableSet | ConvertTo-Json -Depth 10) -Headers $header
    }
}

{% endhighlight  %}

The new form has *one advantage* for me: I can see *everything on screen at once*. So I can see that the original author never set `$spacename`, and that space name, Octopus URL and Octopus API key are all parameters which I can move into a `param()` block.  
 A bit of case-pedantry is to start *local* variables in lower case and capitalize things which come from outside, i.e. parameters  and variables which are used across scopes.

{% highlight PowerShell %}
param (
        $OctopusURL    = "https://youroctourl"
        $OctopusAPIKey = "API-YOURAPIKEY"
        $SpaceName     = 'default'
)
  
{% endhighlight  %}

My early programming experience with LISP put me in the "we don't need no stinking variables" school of coding, and often I'll find a lot of variables used only once, **sometimes this helps readability**. But if I see someone writing  
{% highlight PowerShell %}
    $result = Invoke-FirstApi
    $items = $result.items
    $itemNames = $(foreach ($i in $items) {$i.name})
    foreach ($Name in $item) {Invoke-SecondAPi $name}
  
{% endhighlight  %}

I'll want to change it to:
{% highlight PowerShell %}
    (Invoke-FirstApi).items | ForEach-Object {Invoke-SecondAPi $_.name}
  
{% endhighlight  %}

This example has doesn't have *that* problem. However, the variable `$header` is *only* used as parameter for `Invoke-RestMethod`, so I'm going use *splatting to save space, and improve readability*.  
If I declare a hash-table of variables where each key is a parameter name, I can **splat** that  into `Invoke-RestMethod`, and splatting lets me fix another irritation - *my preference variables*, what was previously on *my* screen, and so on are not *yours* to mess with; the original author wanted things to stop if `Invoke-RestMethod` hit an error, so I'll add that to my hash table of parameters

When I aligned the text in the first revision, all the `Invoke-RestMethod` lined up letting me see how often urls begin  
 `"$octopusURL/api/$($space.Id)/` so let's put that in a variable after we discover the space ID.
Unless I'm emphasizing "*This* call is a `GET` and *that* is `PUT`" I can omit the `-Method Get` from `Invoke-RestMethod` or `Invoke-WebRequest`  
{% highlight PowerShell %}
$restParams      = @{ErrorAction = 'stop'; Header =  @{ "X-Octopus-ApiKey" = $octopusAPIKey } }
$space           = Invoke-RestMethod @restParams -Uri "$octopusURL/api/spaces/all" | Where-Object {$_.Name -eq $SpaceName}
$apiUri          = "$octopusURL/api/$($space.Id)"
  
{% endhighlight  %}
Now I can shrink the rest, I have one more fix which I'll explain soon:
{% highlight PowerShell %}
$projects        =  Invoke-RestMethod @restParams -uri "$apiUri/projects/all"
foreach ($project in $projects){
    $variableSet =  Invoke-RestMethod @restParams -Uri "$apiUri/variables/$($project.VariableSetId)" -Method Get
    if ($variableSet.Variables.Count -gt 0){
                    $variableSet.Variables = Clear-SensitiveVariables -VariableCollection $variableSet.Variables
                    Invoke-RestMethod @restParams -Uri "$apiUri/variables/$($project.VariableSetId)" -Method Put -Body (
                        $variableSet | ConvertTo-Json -Depth 10)
    }
}
$variableSets    =  Invoke-RestMethod @restParams -Uri "$apiUri/libraryvariablesets/all"
foreach ($variableSet in $variableSets){
    $variableSet =  Invoke-RestMethod @restParams -Uri "$apiUri/libraryvariablesets/$($variableSet.Id)" -Method Get
    if ($variableSet.Variables.Count -gt 0){
                    $variableSet.Variables = Clear-SensitiveVariables -VariableCollection $variableSet.Variables
                    Invoke-RestMethod @restParams -Uri "$apiUri/libraryvariablesets/$($variableSet.Id)" -Method Put -Body (
                            $variableSet | ConvertTo-Json -Depth 10)
    }
}
  
{% endhighlight  %}

Now I can turn my attention to the function: something where the pseudo-code looks like this:  
{% highlight PowerShell %}
Foreach thing in Some-collection
    if the-thing meets some-condition
       do some-action with the-thing
  
{% endhighlight  %}
might be better written as `thing.where({condition}) | foreach-Object {act_on $_}`
so I'll change the function like that
{% highlight PowerShell %}
function Clear-SensitiveVariables{
    param ($VariableCollection)
    $VariableCollection.where({$_.IsSensitive}) | foreach-object {$_.Value = ''}
    $VariableCollection
}
  
{% endhighlight  %}
**Functions of this kind are counter-productive**. If they are called thousands of times they have a performance hit, but that's not the case here. It's doing something simple, it's not doing it many places, but we encounter the function call when reading the code we need to jump somewhere to see that it does and then return to where we were reading.  
{% highlight PowerShell %}
$variableSet.Variables = Clear-SensitiveVariables -VariableCollection $variableSet.Variables
  
{% endhighlight  %}
could become
{% highlight PowerShell %}
$variableSet.Variables.where({$_.IsSensitive}) | foreach-object {$_.Value = ''}
  
{% endhighlight  %}
And we can dispense with the function completely... since we call the same URI with `get` and `put` I can move the URI into the hash-table of Rest parameters, which I was hinting at above.
{% highlight PowerShell %}
$variableSets    =  Invoke-RestMethod @restParams -Uri "$apiUri/libraryvariablesets/all"
foreach ($variableSet in $variableSets) {
    $restParams['uri'] =  "$apiUri/libraryvariablesets/$($variableSet.Id)"
    $variableSet =  Invoke-RestMethod @restParams -Method Get
    if ($variableSet.Variables.Count -gt 0){
                    $variableSet.Variables.where({$_.IsSensitive}) | foreach-object {$_.Value = ''}
                    Invoke-RestMethod @restParams  -Method Put -Body ($variableSet | ConvertTo-Json -Depth 10)
    }
}
  
{% endhighlight  %}
I will re-write the value of a loop index - like `$variableSet` here - if it is necessary, but it's often a sign that something else is wrong, and I don't like code in the form:  
`$collection = something; foreach ($item in $collection)`  
both things are easy to rework
{% highlight PowerShell %}
Invoke-RestMethod @restParams -Uri "$apiUri/libraryvariablesets/all" | ForEach-Object {
    $restParams['uri'] =  "$apiUri/libraryvariablesets/$($_.Id)"
    $variableSet =  Invoke-RestMethod @restParams -Method Get
    if ($variableSet.Variables.Count -gt 0){
                    $variableSet.Variables.where({$_.IsSensitive}) | foreach-object {$_.Value = ''}
                    Invoke-RestMethod @restParams  -Method Put -Body ($variableSet | ConvertTo-Json -Depth 10)
    }
}
  
{% endhighlight  %}
This change has shown a flaw in the original logic, which runs *If there are variables, change any sensitive ones, and write them all back*. Wouldn't it be better to have  *change any sensitive variables, and write them back only if there were changes*?
{% highlight PowerShell %}
Invoke-RestMethod @restParams -Uri "$apiUri/libraryvariablesets/all" | ForEach-Object {
    $restParams['uri'] =  "$apiUri/libraryvariablesets/$($_.Id)"
    $variableSet      =  Invoke-RestMethod @restParams -Method Get
    $changes          =  0
    $variableSet.Variables.where({$_.IsSensitive}) | foreach-object {$_.Value = '' ; $changes ++}
    if ($changes)     {  Invoke-RestMethod @restParams  -Method Put -Body ($variableSet | ConvertTo-Json -Depth 10) }
}
  
{% endhighlight  %}
Of course this is a dangerous thing to do so really the script should support confirmation
{% highlight PowerShell %}
Invoke-RestMethod @restParams -Uri "$apiUri/libraryvariablesets/all" | ForEach-Object {
    $restParams['uri'] =  "$apiUri/libraryvariablesets/$($_.Id)"
    $variableSet       =  Invoke-RestMethod @restParams -Method Get
    $changes           =  0
    $variableSet.Variables.where({$_.IsSensitive}) | foreach-object {$_.Value = '' ; $changes ++}
    if ($changes -and ($Force -or $pscmdlet.ShouldProcess($_.name,"Modify $changes variables in Library variable-set") )) {  
                         Invoke-RestMethod @restParams  -Method Put -Body ($variableSet | ConvertTo-Json -Depth 10)
    }
}
  
{% endhighlight  %}
And now the whole block wants a comment, which can be more helpful than ones I removed.
{% highlight PowerShell %}
[cmdletbinding(supportsShouldProcess=$true,ImpactLevel='High')]
param (
    $OctopusURL    = "https://youroctourl"
    $OctopusAPIKey = "API-YOURAPIKEY"
    $SpaceName     = 'default'
)

#region resolve space name to an ID and set variables for remaining rest API calls
$restParams      = @{erroraction = 'stop'; header =  @{ "X-Octopus-ApiKey" = $octopusAPIKey } }
$space           = Invoke-RestMethod @restParams -Uri "$octopusURL/api/spaces/all" | Where-Object {$_.Name -eq $SpaceName}
$apiUri          = "$octopusURL/api/$($space.Id)"
#endregion

#For all projects, get their variable-set, reset any variables flagged "issensitive" and write back if there are changes (confirming first)
Invoke-RestMethod @restParams -Uri "$apiUri/projects/all" | ForEach-Object {
    $restParams['uri'] =  "$apiUri/variables/$($_.VariableSetId)"
    $variableSet       =  Invoke-RestMethod @restParams -Method Get
    $changes           =  0
    $variableSet.Variables.where({$_.IsSensitive}) | foreach-object {$_.Value = '' ; $changes ++}
    if ($changes -and ($Force -or $pscmdlet.ShouldProcess($_.name,"Modify $changes variables in  variable set") ))   {
                         Invoke-RestMethod @restParams  -Method Put -Body ($variableSet | ConvertTo-Json -Depth 10) }
}

#Repeat for library variable sets
Invoke-RestMethod @restParams -Uri "$apiUri/libraryvariablesets/all" | ForEach-Object {
    $restParams['uri'] =  "$apiUri/libraryvariablesets/$($_.Id)"
    $variableSet       =  Invoke-RestMethod @restParams -Method Get
    $changes           =  0
    $variableSet.Variables.where({$_.IsSensitive}) | foreach-object {$_.Value = '' ; $changes ++}
    if ($changes -and ($Force -or $pscmdlet.ShouldProcess($_.name,"Modify $changes variables in  variable set") ))   {
                         Invoke-RestMethod @restParams  -Method Put -Body ($variableSet | ConvertTo-Json -Depth 10) }
}
  
{% endhighlight  %}
Now it's become clear that I'm **calling identical code twice** maybe I *do* want a function
{% highlight PowerShell %}
[cmdletbinding(supportsShouldProcess=$true,ImpactLevel='High')]
param (
    $OctopusURL    = "https://youroctourl"
    $OctopusAPIKey = "API-YOURAPIKEY"
    $OpaceName     = 'default'
    [switch]$Force
)

#region resolve space name to an ID and set variables for remaining rest API calls
$restParams      = @{erroraction = 'stop'; header =  @{ "X-Octopus-ApiKey" = $octopusAPIKey } } 
$space           = Invoke-RestMethod @restParams -Uri "$octopusURL/api/spaces/all" | Where-Object {$_.Name -eq $SpaceName}
$apiUri          = "$octopusURL/api/$($space.Id)"
#endregion 

#define a function to get a variable set from its URI, clear any variables flagged "issensitive" and write back if there are changes (confirming first)
function Update-VariableSet {
    param ($uri , $Psc , [switch]$force)
    $variableSet      =  Invoke-RestMethod @restParams -Method Get -uri $uri
    $changes          =  0
    $variableSet.Variables.where({$_.IsSensitive}) | foreach-object {$_.Value = '' ; $changes ++}
    if ($changes -and ($Force -or $psc.ShouldProcess($_.name,"Modify $changes variables in variable-set") ))   {
                         Invoke-RestMethod @restParams  -Method Put -uri $uri -Body ($variableSet | ConvertTo-Json -Depth 10) }
}

#call the function for all projects and for all library variable-sets,
Invoke-RestMethod -Uri "$apiUri/projects/all"             @restParams | ForEach-Object {
                            Update-VariableSet "$apiUri/variables/$($_.VariableSetId)" $PsCmdlet -force:$force
}
Invoke-RestMethod -Uri "$apiUri/libraryvariablesets/all"  @restParams | ForEach-Object {
                            Update-VariableSet "$apiUri/libraryvariablesets/$($_.Id)"  $PsCmdlet -force:$force
}  
  
{% endhighlight  %}

Using `region`, and moving the bulk of work into the function means when I fold the code it's very easy to read.  
{% highlight PowerShell %}
 [cmdletbinding(supportsShouldProcess=$true,ImpactLevel='High')]
>param (...
  
>#region resolve space name to an ID and set variables for remaining rest API calls
  
 #define a function to get a variable set from its URI, clear any variables flagged "issensitive" and  write back if there are changes (confirming first)
>function Update-VariableSet {...

 #call the function for all project and and all library variable set, 
 >Invoke-RestMethod -Uri "$apiUri/projects/all"             @restParams | ForEach-Object {...
 >Invoke-RestMethod -Uri "$apiUri/libraryvariablesets/all"  @restParams | ForEach-Object {...
  
{% endhighlight  %}

I'm a big fan of using `#region`; Often I find that people who have been taught that their programs should break down into smaller parts want to write
{% highlight PowerShell  %}
Function first_part
Function second_part

first_part
Second_part
  
{% endhighlight  %}
And they would be better with
{% highlight PowerShell %} 
#region first part

#region second part 

{% endhighlight  %}

This example shows that if you rearrange things to flowing in a linear way, it becomes easier to see where you are starting to repeat yourself, and the code *really* belongs in a function compared with a form of [Conways's Law](https://en.wikipedia.org/wiki/Conway%27s_law) where the structure of the software reflects the way whoever wrote it visualized things at the time.  
Where something isn't broken, the process is much the same; 

1. Format for reading.
1. Remove unnecessary jumping about.
1. Optimize selection and looping.
1. Apply language features (where it is smart to do so).
1. Look for duplications and things which *should* be in functions.
1. Identify natural places to put `region` blocks
1. Add `Write-Verbose` or `Write-Progress` to tell the user what is happening and double up as comments.
1. Add / replace comments - explain the need for things, or particular methods but don't restate what the code is doing in the comment
