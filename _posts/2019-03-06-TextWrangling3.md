---
layout: post
title:  "PowerShell formatting [not just] Part 3 of the Graph API series"
date:   2019-03-06
categories: PowerShell
---
Many of us learnt to program at school and lesson 1 was writing something like
```
CLEARSCREEN
PRINT "Enter a number"
INPUT X
Xsqrd = X * X
PRINT “"he Square of " + STR(X) + “Is ” + STR(Xsqrd)
```
So I know I should not be surprised when I read scripts and see someone has
started with CLS (or Clear-Host) and then has a script peppered with `Read-Host`
and `Write-Host`, or perhaps `echo` – and what is echoed is a carefully built up
string. And I find myself saying “STOP”

-   **CLS:** I might have hundreds or thousands of lines in the scroll back
    buffer of my shell. Who gave you permission to throw them away ?
-   **Let me run your script with parameters.** Only use commands like `Read-Host`
    and `Get-Credential` if I didn’t (or couldn’t) provide the parameter when I
    started it
-   **Never print** your output

And quite quickly most of us learn about `Write-Verbose` and `Write-Progress` and
the proper way to do “What’s happening messages” ; we also learn to **output
objects**, not formatted text. However, this can have a sting in the tail: the [previous post](powershell/2019/03/03/GraphAPI2.html) showed this little snipped of calling the graph API.    
`Invoke-Restmethod -Uri "https://graph.microsoft.com/v1.0/me" -Headers $DefaultHeader`

```
@odata.context    : https://graph.microsoft.com/v1.0/$metadata\#users/$entity
businessPhones    : {}
displayName       : James O'Neill
givenName         : James
jobTitle          :
mail              : xxxxx@xxxxxx.com
mobilePhone       : +447890101010
officeLocation    :
preferredLanguage : en-GB
surname           : O'Neill
userPrincipalName : xxxxx@xxxxxx.com
id                : 12345678-abcd-6789-ab12-345678912345
   
```
`Invoke-RestMethod` automates the conversion of JSON into a PowerShell object; so
I have something _rich_ to output but I don’t want all of this information, I want
a function which works like this:
```
> get-graphuser
Display Name Job Title   Mail  Mobile Phones  UPN
------------ ---------   ----  -------------  ---
James O'Neill Consultant jxxx  +447890101010  Jxxx
   
```

If no user is specified, my function selects the current user. If I want a
different user I’ll give it a `–UserID` parameter, if I want something *about a
user* I’ll give it other parameters and switches, but if it just outputs a user
I want a few fields displayed as a table. (That’s not a real phone number by the
way). This is much more _the PowerShell way_, think about what it does, what goes
in and what comes out, but be vaguer about the visuals of that output.

A simple, but effective way get this style of output would be to give
`Get-GraphUser` a `–Raw` switch and pipe the object through `Format-Table`, unless raw
output is needed; but I need to repeat this anywhere that I get a user, and it
only works for immediate output. If I do
```
$U = Get-GraphUser
<some operation with $U>>

```
The formatted output (if I forget `–RAW`) will mean `$U` won’t be valid input for the next step… If I make `-Formatted` the option instead of raw, I can't display with the format I want after the `Get` operation has completed.  **There is a better way** which is to tell
PowerShell “When you see a Graph user, format it as a table: like this” ; that’s
done with a format.ps1xml file – it’s easiest to plagiarize the ones in Windows
PowerShell’s $PSHOME directory – don’t modify them, they’re digitally signed –
you get an XML file which looks like this
{% highlight Xml %}
<Configuration>
    <ViewDefinitions>
        < View>
            <Name>Graph Users</Name>
            <ViewSelectedBy><TypeName>GraphUser</TypeName></ViewSelectedBy>
            <TableControl>
            ...
            </TableControl>
        </View>
    </ViewDefinitions>
< /Configuration>
   
{% endhighlight %}
There is a `<view>` section for each type of object and a `<tableControl>` or
`<listControl>` defines how it should be displayed. For `OneDrive` objects I
copied the way headers work for files, but everything else just has a table or
list. The XML says the view is selected by an object with a type name of
*GraphUser*, and we can add any name to the list of types on an object. The core
of the `Get-GraphUser` function looks like this:

{% highlight powershell %}
    $webparams = @{
       Method  = "Get"
       Headers = $Script:DefaultHeader
    }
    if ($UserID) {$userID = "users/$userID"} else {$userid = "me"}
    $uri = "https://graph.microsoft.com/v1.0/$userID"
    # Other URIs may be defined
    $results = Invoke-RestMethod -Uri $uri @webparams
    foreach ($r in $results) {
        if ($r.'@odata.type' -match 'user$') {
                $r.pstypenames.Add('GraphUser')
        }
        ...
    }
    $results
    
{% endhighlight %}
The “common” web parameters are defined, then the URI is determined, then a call
to `Invoke-RestMethod`, which might get one item, or an array of many (usually in
a *values* property). Then the results have the name “GraphUser” added to their
list of types, and the result(s) are returned.

This pattern repeats again and again, with a couple of common modifications ; I
can use `Get-GraphUser <id> –Calendar` to get a user’s calendar, but the
calendar that comes back doesn’t contain the details needed to fetch its events.
So ,going through the foreach loop, when the result is a calendar it is better
for the function to add a property that will help navigation later
{% highlight powershell %}
    $uri = https://graph.microsoft.com/v1.0/$userID/Calendars
    $r.pstypenames.Add('GraphCalendar')
    Add-Member -InputObject $r -MemberType NoteProperty -Name CalendarPath -Value "$userID/Calendars/$($r.id)"
    
{% endhighlight %}
As well as navigation, I **don’t like functions which return things that need to
be translated,** so when an API returns dates as text strings, I’ll provide an
extra property which presents them as a `datetime` object. I also create some
properties for **display use only**, which comes into its own for the second
variation on the pattern. Sometimes it is simpler to just tell PowerShell –
“Show these properties” - when there is no formatting XML PowerShell has one last
check – does the object have a `PSStandardMembers` property with a
`DefaultDisplayPropertySet` child property ? For events in the calendar, the
definition of “standard members” might look like this:
{% highlight powershell %}
[string[]]$defaultProperties = @('Subject','When','Reminder')
    $defaultDisplayPropertySet = New-Object System.Management.Automation.PSPropertySet `
                                        -ArgumentList 'DefaultDisplayPropertySet',$defaultProperties
    $psStandardMembers = [System.Management.Automation.PSMemberInfo[]]@($defaultDisplayPropertySet)
    
{% endhighlight %}
Then, as the function loops through the returned events instead of adding a type
name it adds a property named PSStandardMembers

{% highlight powershell %}
    Add-Member -InputObject $r -MemberType MemberSet -Name PSStandardMembers -Value $PSStandardMembers
    
{% endhighlight %}
PowerShell has an automatic variable `$FormatEnumerationLimit` which says “up to
some number of properties display a table, and for more than that display a
list” – the default is 4. So, this method suits a list of reminders in the
calendar where the ideal output is a table with 3 columns, and there is only one
place which gets reminders. If the **same type of data is fetched in multiple
places** it is easier to maintain a definition in an XML file.

As I said before working on the graph module the same pattern is repeated a lot:
discover a URI which can get the data, then write a PowerShell function which:

-   Builds the URI from the function’s parameters
-   Calls Invoke-RestMethod
-   Adds properties and/or a type name to the returned object(s)
-   Returns those objects

The _first working version_ of a new function helps to decide how the objects will
be formatted which refines the function and adds to the formatting XML as
required. Similarly, the need for extra properties might only become apparent
when other functions are written; so development is an iterative process.

The [next post](/powershell/2019/03/20/TextWrangling4.html) will look at another area which the module uses but applies more
widely which I’ve taken to calling “Text wrangling”: how we build up JSON and
other text that we need to send in a web request.
