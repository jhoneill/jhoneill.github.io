---
layout: post
title:  "PowerShell Text wrangling [not just] part 4 of the Graph API series"
date:   2019-03-20
categories: PowerShell
---
Almost the first line of PowerShell I ever saw was combining two strings *like
this*:  
`"{0} {1}" -f $x , $y`    
And my reaction was “*What !! If the syntax to concatenate two strings is so
opaque this might not be for me*”.  
[**Edit** as if to prove the awkwardness of the syntax, the initial post had two
errors in such a short fragment. Thanks Doug.]

The `–f` operator is a wrapper for .NET's `[String]::format` and it **is** useful 
for inserting strings into another string. For example, I might define a SQL
Statement in one place like this:  
`"Insert Into Users [GivenName], [Surname], [endDate] Values ('{0}', '{1}','{2}')"`    
and later I can get a ready-to-run query, using     
`$sqlInsert -f $first,$last,$end`

Doing this lets me arrange a script with long strings placed away from the logic; I’m less happy with this:
```
@"
  Update Users
  Set[endDate] = '{0}'
  where {1} = '{2}'
    And {3} = '{4}'
"@ -f $end,$fieldName1,$value1,$fieldName2,$value  

```
because the string is _there_, my brain automatically goes back and forth filling
in what should be in {0}, {1} and {2}, so I’d prefer either to put $first,
$Last and $end inside one string, or move the string out of sight. The string
format operator is there to _apply formatting_ and going over some downloaded code
`–f` let me change this :
{% highlight powershell %}
  $now = Get-Date
  $d   = $now.Day.ToString()
  if ($d.Length -eq 1) {$d ="0$d"}
  $m = $now.month.ToString()
  if ($m.Length -eq 1) {$m ="0$m"}
  $y = $now.Year.ToString()
  $logMessage = "Run on $d/$m/$y"
  
{% endhighlight %}
To this:
{% highlight powershell %}
    $now = Get-Date
    $logMessage = "Run on {0:d}" –f $now
  
{% endhighlight %}
For years, my OneNote notebook has had a page which I lifted from the
now-defunct blog of Kathy Kam (which you may still find re-posts of) which
explains what the formatting strings are. In this case `:d` is “local short date”
which is better than *hard-coding* the exact date format; formatting strings
used in Excel generally work, but there are some extra single-character formats
like `:g` for general date/time, and `:D` for long date. If you live somewhere that
puts the least significant part of the date in the middle, then you might ask    
“Why not use `"Run on $now"` ?”

The 10th day of March 2019 outputs “Run on 03/10/2019 13:48:32” it is `culture-invariant`– but in most of the
world that means “3rd day of October”. But we could avoid `-f` by using:    
`"Run on $($now.ToString('d'))"`

I guess most people who use PowerShell will have used the `$()` syntax to evaluate
the _property of a variable embedded in a string_. But you can put a lot inside
`$()`, this example will give you a list of days:
{% highlight powershell %}
   "Days are $(0..6 | foreach {"`r`n" + $now.AddDays($_).ToString('dddd')})"
   
{% endhighlight %}
The quote marks inside the `$()` *don’t end the string* and what is being
evaluated can run over multiple lines like this:
{% highlight powershell %}
   "Days are $(0..6 | foreach {
      "`r`n" + $now.AddDays($_).ToString('dddd')
    } )"
    
{% endhighlight %}

Again, there are places where I have found this technique to be useful, but
encountering it in an unfamiliar piece of script means it takes me a few seconds
to see that ``[`r`n]`` is a string made of escape codes, inside a code block, inside a string, in my
script.  
I might use `@" … "@` , which I think was once *required* for multi-line
strings, instead of `"…"` and although that certainly works _now_, it leaves me looking for the
*closing* quote – which isn’t the *next* quote. If the code set the first part of
the string and looped to add days to it, that would be easier to follow. Incidentally
when I talk of “an unfamiliar piece of script” I don’t just mean other peoples’
work, I include work *I did long enough ago that I don’t remember it*.

Code inside a string, concatenating multiple strings, or using `–f` might all
work, so which one is best in a given situation varies (sometimes it is shortest
code that is easiest to understand and other things are clearer spread over a
few lines) and the choice often comes down to personal coding style.

When working on my [Graph API
module](https://www.powershellgallery.com/packages/MSFTGraph/1.0) I needed to
send JSON like this (from the [Microsoft
Documentation](https://docs.microsoft.com/en-us/graph/api/group-post-groups?view=graph-rest-1.0))
to create a *Group* :
{% highlight JSON%}
{
    "description": "Group with designated owner and members",
    "displayName": "Operations group",
    "groupTypes": [
        "Unified"
    ],
    "mailEnabled": true,
    "mailNickname": "operations2019",
    "securityEnabled": false,
    "owners@odata.bind": [
        "https://graph.microsoft.com/v1.0/users/26be1845-4119-4801-a799-aea79d09f1a2\&quot;
    ],
    "members@odata.bind": [
        "https://graph.microsoft.com/v1.0/users/ff7cb387-6688-423c-8188-3da9532a73cc\&quot;,
        "https://graph.microsoft.com/v1.0/users/69456242-0067-49d3-ba96-9de6f2728e14\&quot;
    ]
}
{% endhighlight %}
This might be done as a large string with embedded variables, and even a couple
of embedded for loops like the previous example, or to I could build the text up
a few lines at a time. Eventually I settled on doing it like this.
{% highlight Powershell %}
$settings = @{  'displayName'     = $Name
                'mailNickname'    = $MailNickName
                'mailEnabled'     = $true
                'securityEnabled' = $false
                'visibility'      = $Visibility.ToLower()
                'groupTypes'      = @('Unified')
}
if ($Description) {$settings['description']        = $Description }
if ($Members)     {$settings['members@odata.bind'] = @() + $Members}
if ($Owners)      {$settings['owners@odata.bind']  = @() + $Owners }

$json   = ConvertTo-Json $settings  ; Write-Debug $json
$group = Invoke-RestMethod @webparams -body $json
   
{% endhighlight %}
`ConvertTo-Json` **only processes two levels of hierarchy by default** so when the
Hash-table has more layers it needs the `–Depth` parameter to translate properly.    
Why do it this way? JSON says ‘here is something (or a collection of things)
with a name’, so **why say that in PowerShell-speak only to translate it?** Partly
it’s keeping to the philosophy of **only translating into text at last moment**;
partly it’s to do **less mental context-switching** – this is script, this
is text with script-like bits. Partly it is to make *getting things right
easier than getting things wrong*: if things are built up a few lines at a time,
I need to remember that ‘Unified’ should be _quoted_ because it is a string value,
but as a Boolean value, ‘false’ _should not be_, I need to track unclosed quotes and
brackets, to make sure commas are where they are needed and nowhere else: in
short, **every edit is a chance to turn valid JSON into something generates a
“Bad Request” message** – so everywhere I generate JSON I have `Write-Debug
$Json`. But any syntax errors in that hash table will be highlighted as I edit
my script.

And partly… When it comes to parsing text, I’ve been there and got the T-Shirts;
better code than mine is available, built-in with PowerShell; I’d like to apply
that same logic to creating such text: I want to **save as much effort as I
can** between “I have these parameters/variables” and “this data came back”.
That was the thinking behind writing my [GetSQL
module](https://www.powershellgallery.com/packages/getsql/1.2.0.1): I know how
to connect to a database and can write fairly sophisticated queries, but **why
keep writing variations** of the same few simple ones? SQL statements need the same “context switch” – if I
type “–eq” instead of “=” in a SQL query it’s not because I’ve forgotten the SQL I
learned decades ago, but because I didn’t shift from PowerShell to SQL
seamlessly. `Get-SQL` lets me keep my brain in PowerShell mode and write:
{% highlight PowerShell %}
    SQL –Update LogTable –Set Progess –Value 100 –Where ID –eq $currentItem
   
{% endhighlight %}
My perspective – centred on the script that calls the API rather than the API or
its transport components – isn’t the *only* way. Some people prize the skill of
handwriting descriptions of things in JSON. A recent project had me using DSC
for bare-metal builds (I need to parse MOF files to construct test scripts, and
I could hand crank MOF files, but why go through that extra pain? ); DSC
configuration functions take a configuration data parameter which is a
hash-table holding all the details of all the machines. This was _huge_. When work
started it was natural to create a PowerShell variable holding the data but when
it became hundreds of lines I moved that data to its own file, but it remained a
series of declarations which could be executed – this is the code which did that
{% highlight PowerShell%}
    Get-ChildItem -Path (Join-Path -Path $scriptPath -ChildPath "\*.config.ps1") |
        ForEach-Object {
            Write-Verbose -Message "Adding config info from $($_.name)"
            $ConfigurationData.allNodes += (& $_.FullName )
    }
    
{% endhighlight %}
There was *no decision* to store data as PowerShell declarations, it just
happened as an accident of how the development unfolded, and there were people
working on that project who found JSON easier to read (we could have used any
format which supports a hierarchy). So, I added something to put files through
`ConvertFrom-JSon` and convert the result from a PSCustomObject to a hash-table so
they could express the data in the way which seemed natural to them.

Does this mean data should always be shifted out of the script ? Even that
answer is “it depends” and is influenced by personal style. The examples which
Doug Finke wrote for the
[ImportExcel](https://www.powershellgallery.com/packages/ImportExce) module
often start like this:
{% highlight PowerShell%}
$data = ConvertFrom-Csv @'
Item,Quantity,Price,Total Cost
Footballs,9,21.95,197.55
...
Baseball Bats,38,159.00,6042.00
'@ 
    
{% endhighlight %}

Which is simultaneously both good and bad. It is placed at the start of the file,
not sandwiched between lumps of code, we can see that it is data for later and
what the columns are, and it is only one per row of data where JSON would be 6
lines. But csv **gives errors a hiding place** – a mistyped price, or an extra comma
is hard to see. But that doesn’t matter in this case. But we wouldn’t mash
together the string being converted from other data… would we ?
