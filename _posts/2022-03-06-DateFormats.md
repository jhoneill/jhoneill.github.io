---
layout: post
title:  "PowerShell Dates, Formatting and .NET Cultures"
date:   2022-03-06
categories: PowerShell
tags: 
    - PowerShell
    - Development
    - .NET
    - Formatting
---
## PowerShell extends dates

 PowerShell has a `Get-Date` cmdlet or you can go direct to the .NET `[System.DateTime]` class and ask for the current time. PowerShell's way of outputting a date gives a different result from simply converting it to a string - this is with *culture set to UK English* :

```powershell-interactive
    ps >[datetime]::now.ToString()
    29/12/2021 11:50:27 
    
    ps > [datetime]::now
    30 December 2021 11:01:33

```

Something simple-looking like deciding how to print a date has a surprising number of 'moving parts'.  
We can discover how PowerShell came to that format by starting with `Get-FormatData` and drilling deeper and deeper into its response: after a few iterations of selecting the interesting property we get:

```powershell-interactive
    ps>  (Get-FormatData System.datetime).FormatViewDefinition.Control.Entries.CustomItems.Expression

    ValueType Value
    --------- -----
     Property DateTime
    
```

So: the way to display a `[System.DateTime]` *object* is to display its `.DateTime` *property*, so let's examine *that*:

```powershell-interactive
    ps> [datetime]::Now | Get-Member DateTime 
    
       TypeName: System.DateTime
    Name     MemberType     Definition
    ----     ----------     ----------
    DateTime ScriptProperty System.Object DateTime {get=if ((& {…
    
```

`DateTime` is a *PowerShell script property*, so PowerShell added it to the standard .NET `[System.DateTime]` object; drilling into the *TypeData* shows how `DateTime` is defined

{% highlight PowerShell %}
    ps > (Get-TypeData -TypeName system.datetime).members['DateTime'].GetScriptBlock
          if ((& { Set-StrictMode -Version 1; $this.DisplayHint }) -ieq  "Date")
          {
          "{0}" -f $this.ToLongDateString()
          }
          elseif ((& { Set-StrictMode -Version 1; $this.DisplayHint }) -ieq "Time")
          {
          "{0}" -f  $this.ToLongTimeString()
          }
          else
          {
          "{0} {1}" -f $this.ToLongDateString(), $this.ToLongTimeString()
          }

{% endhighlight %}

So:

* If there is a `DisplayHint` property and it is set to "Date", the output comes from `.ToLongDateString()`,  
* If there is a `DisplayHint` property and it is set to "Time", the output comes from `.ToLongTimeString()`,  
* Otherwise the output is the long date *and* the long time.  

But the standard .NET object doesn't have a `DisplayHint` property: it isn't added by the *type data* but by the `Get-Date` cmdlet:

```powershell-interactive
    ps > (Get-Date).DisplayHint
    DateTime
    
    ps > Get-Date -DisplayHint Date
    30 December 2021
    
    ps >  Get-Date -DisplayHint time
    11:04:33

```

## The effects of culture settings on dates

The [previous post](/powershell/2022/03/05/NumberFormats.html) talked about cultures and how they affect the way numbers and currencies are converted for display, so what does selecting a different culture, like French, do to a `[DateTime]` object? As in the previous post, the example will change the culture for the current thread only.

```powershell-interactive
    ps> [System.Threading.Thread]::CurrentThread.CurrentCulture = Get-Culture fr-FR
    
    ps> (Get-Culture).DisplayName
    French (France)
    
    ps > [datetime]::now.ToLongDateString()
    jeudi 30 décembre 2021

```

The string representation of the date has French names for weekdays and months. Can we convert that string back to a `[datetime]`?

```powershell-interactive
    ps > [datetime]"jeudi 30 décembre 2021"
    InvalidArgument: Cannot convert value "jeudi 30 décembre 2021" to type "System.DateTime".
    Error: "The string 'jeudi 30 décembre 2021' was not recognized as a valid DateTime. There is an unknown word starting at index '0'."

```

The *cast* operation doesn't like the word "jeudi" at index 0, it would be quite happy with "Thursday", and in the next example you can see it is "décembre" at index 12 which is the problem.

```powershell-interactive
    ps > [datetime]"Thursday 30 décembre 2021"
    InvalidArgument: Cannot convert value "Thursday 30 décembre 2021" to type "System.DateTime".
    Error: "The string 'Thursday 30 décembre 2021' was not recognized as a valid DateTime. There is an unknown word starting at index '12'."

```

### Invariant, not inflexible

That [previous post](/powershell/2022/01/15/NumberFormats.html) showed how **invariant culture** worked for number separators and something similar happens for dates; if we use the English name for the month, casting a string to a date will convert it using the *invariant culture* information, and the process can handle different orderings of year month-name and day. Weekdays are optional but must be correct if specified. And because the culture is still set to French for these examples, the date goes in in English and comes out in French.

```powershell-interactive
    ps >  [datetime]"Thursday 30 December 2021"
    jeudi 30 décembre 2021 00:00:00
    
    ps >  [datetime]"Thursday  December 30 2021"
    jeudi 30 décembre 2021 00:00:00
    
    ps >  [datetime]"December 30 2021"
    jeudi 30 décembre 2021 00:00:00
    
    ps >  [datetime]"2021 December 30 "
    jeudi 30 décembre 2021 00:00:00
    
    ps > [datetime]"Monday  December 30 2021"
    InvalidArgument: Cannot convert value "Monday  December 30 2021" to type "System.DateTime". 
    Error: "String 'Monday  December 30 2021' was not recognized as a valid DateTime because the day of week was incorrect."

```

### Ambiguity

The date formats in the example above are all **unambiguous**. In a cast operation like `[datetime]"2022/01/20"` it is assumed that four digits *must* be a year, and year-first implies year-month-day format.  In `[datetime]"20 jan 22"` there is a *possibility* that "20" is the year, but the assumption is year-last. A date like `11/12/13` *could* be year first, "11/12/2021" might be month first or day first depending on whether the person writing it is in the US or not because, as I always tell my Americans friends, "least significant in the middle" isn't used anywhere else.

Invariant dates need to select *something* as their standard, the biggest IT companies are American, and so are the default rules for processing ambiguously formatted dates like "aa Month-name bb", "aa/bb/cc" or "aa/bb/cccc".

Conversion rules can't be specified in Cast operations, so `[datetime]11/12/21` returns "November 12th", *regardless of culture settings*. But casting isn't the only way to  create a date from a string, `.Parse()` and the methods of the `[convert]` class allow a culture to be specified. Similarly when date is transformed to a string a cast uses invariant culture,`.ToString()` can be given culture as a parameter. And wherever a culture *may* be specified, local culture is usually assumed.

Although much of the above is concerned with casting strings to dates, when casting in the other direction, from date to string, the use of invariant rules can be an issue:

```powershell-interactive
    ps> [datetime]::now.ToString()
    30/12/2021 14:41:05
    
    ps > [string][datetime]::now
    12/30/2021 14:41:13

    ps > "it is now " + $now
    it is now 12/30/2021 14:49:03
    
    ps > "it is now {0}" -f $now
    it is now 30/12/2021 14:49:03

```

As you can see the `-f` operator converts one way and the `+` operator converts the other. US authors are insulated from this and in the worst case, can mix these forms, so the rest of the world doesn't know if 02/03/2022 is their local notation for the 2nd day of March, or American for February 3rd.

### Converting from local format

`[convert]::ToDateTime()` or `[DateTime]::Parse()` will process ambiguously formatted dates with **local rules** (in a similar way to parsing / converting local number formats). PowerShell's `-As` operator isn't consistent: floating point numbers use invariant, dates use local. In the examples below the culture is still set to French so French months and days (and their short forms) are understood.

```powershell-interactive
    ps> (Get-Culture).DisplayName       
    French (France)
    
    ps >  [datetime]::parse("jeudi 30 décembre 2021")
    jeudi 30 décembre 2021 00:00:00
    
    ps >  [convert]::ToDateTime("jeudi 30 décembre 2021")
    jeudi 30 décembre 2021 00:00:00

```

Parsing "jeudi 30 décembre 2021" works if the culture is a French language one, but it breaks with a French date and an English culture. You can either trap the error, or use the `TryParse` method, as in the examples below.  
I changed my culture back to English to show one of the pitfalls: the first one fails with because it is picking up the *current* culture and second succeeds because it been told to use French:

```powershell-interactive
    ps> [System.Threading.Thread]::CurrentThread.CurrentCulture = Get-Culture en-gb
    ps> (Get-Culture).DisplayName
    English (United Kingdom)
    
    ps>  $x = [datetime]::now
    ps> [datetime]::TryParse('jeudi 30 décembre 2021 00:00:00', ([ref]$x))
    False
    
    ps > $french = Get-Culture fr-fr
    ps >[datetime]::TryParse('jeudi 30 décembre 2021 00:00:00', $french ,'None', ([ref]$x))
    True

    ps > $x
    30 December 2021 00:00:00

```

Ambiguous dates leave us guessing: will a command  process an argument of "02/03/2022" with invariant or local rules? and does *local* always mean the same thing - if a script runs in London, Paris and New York will it do the same thing everywhere? (Servers might be set to common standard regardless of location, workstations probably aren't.)  
If the output was "02/03/2022" did the program mean March knowing I'm in the UK? Better to show me "2 March 2022" in my local language. Non-English month names won't parse; so although 2022/03/02 might jar for a human reader, if the "reader" will be another program then it is preferable; but cmdlets like `Export-csv` convert dates using *local* culture - unless they are explicitly told which culture to use. Commands which interpret dates written as strings should state their cultural assumptions, but very few do.  
In the next post I will talk about errors that can result from handling times without proper care, and this shapes how time should be written.

### Comparing cultures

Invariant is not the same as US - the **date** format is US style, but **time** isn't: if I change culture again:

```powershell-interactive
    ps> [System.Threading.Thread]::CurrentThread.CurrentCulture = Get-Culture en-us
    ps> (Get-Culture).DisplayName
    English (United States)

    ps > [datetime]::now.ToString()
    12/30/2021 2:44:54 PM
    
    ps> [string][datetime]::now
    12/30/2021 14:45:01

```

English-US culture has a 12 hour clock with AM and PM, but invariant culture uses the 24 Hour clock. Both will parse but there can be ambiguity about the hour after midnight.

```powershell-interactive
    >[string][datetime]"12/30/2021 12:44:54 AM" 
    12/30/2021 00:44:54
    
    ps > [datetime]"12/30/2021 12:44:54"
    Thursday, December 30, 2021 12:44:54 PM

```

You might notice that English-US and French both include the name of the weekday when outputting, but English-GB doesn't. This is another option defined as part of the culture settings - the following shows the date and time formatting strings defined for a selection of cultures.

```powershell-interactive
    ps > $props = (get-culture).DateTimeFormat.psObject.properties.name                                                               
    ps > $inv = (Get-Culture -Name "").DateTimeFormat                                                                                 @
    ps > $us  = (Get-Culture -Name "en-us").DateTimeFormat                                                                            
    ps > $gb  = (Get-Culture -Name "en-GB").DateTimeFormat                                                                            
    ps > $fr  = (Get-Culture -Name "fr-fr").DateTimeFormat                                                                            
    ps > $props | % {[pscustomobject]@{"PropertyName"=$_; "fr-FR"=$fr.$_;"en-GB"=$gb.$_;"en-us"=$us.$_;"Invariant"=$inv.$_ }} | ft -a 
    
    PropertyName                     fr-FR                                  en-GB                                  en-us                                  Invariant
    ------------                     -----                                  -----                                  -----                                  ---------
    AMDesignator                     AM                                     AM                                     AM                                     AM
    Calendar                         System.Globalization.GregorianCalendar System.Globalization.GregorianCalendar System.Globalization.GregorianCalendar System.Globalization.GregorianCalendar
    DateSeparator                    /                                      /                                      /                                      /
    FirstDayOfWeek                   Monday                                 Monday                                 Sunday                                 Sunday
    CalendarWeekRule                 FirstFourDayWeek                       FirstFourDayWeek                       FirstDay                               FirstDay
    FullDateTimePattern              dddd d MMMM yyyy HH:mm:ss              dd MMMM yyyy HH:mm:ss                  dddd, MMMM d, yyyy h:mm:ss tt          dddd, dd MMMM yyyy HH:mm:ss
    LongDatePattern                  dddd d MMMM yyyy                       dd MMMM yyyy                           dddd, MMMM d, yyyy                     dddd, dd MMMM yyyy
    LongTimePattern                  HH:mm:ss                               HH:mm:ss                               h:mm:ss tt                             HH:mm:ss
    MonthDayPattern                  d MMMM                                 d MMMM                                 MMMM d                                 MMMM dd
    PMDesignator                     PM                                     PM                                     PM                                     PM
    RFC1123Pattern                   ddd, dd MMM yyyy HH':'mm':'ss 'GMT'    ddd, dd MMM yyyy HH':'mm':'ss 'GMT'    ddd, dd MMM yyyy HH':'mm':'ss 'GMT'    ddd, dd MMM yyyy HH':'mm':'ss 'GMT'
    ShortDatePattern                 dd/MM/yyyy                             dd/MM/yyyy                             M/d/yyyy                               MM/dd/yyyy
    ShortTimePattern                 HH:mm                                  HH:mm                                  h:mm tt                                HH:mm
    SortableDateTimePattern          yyyy'-'MM'-'dd'T'HH':'mm':'ss          yyyy'-'MM'-'dd'T'HH':'mm':'ss          yyyy'-'MM'-'dd'T'HH':'mm':'ss          yyyy'-'MM'-'dd'T'HH':'mm':'ss
    TimeSeparator                    :                                      :                                      :                                      :
    UniversalSortableDateTimePattern yyyy'-'MM'-'dd HH':'mm':'ss'Z'         yyyy'-'MM'-'dd HH':'mm':'ss'Z'         yyyy'-'MM'-'dd HH':'mm':'ss'Z'         yyyy'-'MM'-'dd HH':'mm':'ss'Z'
    YearMonthPattern                 MMMM yyyy                              MMMM yyyy                              MMMM yyyy                              yyyy MMMM
    AbbreviatedDayNames              {dim., lun., mar., mer.…}              {Sun, Mon, Tue, Wed…}                  {Sun, Mon, Tue, Wed…}                  {Sun, Mon, Tue, Wed…}
    ShortestDayNames                 {D, L, M, M…}                          {S, M, T, W…}                          {S, M, T, W…}                          {Su, Mo, Tu, We…}
    DayNames                         {dimanche, lundi, mardi, mercredi…}    {Sunday, Monday, Tuesday, Wednesday…}  {Sunday, Monday, Tuesday, Wednesday…}  {Sunday, Monday, Tuesday, Wednesday…}
    AbbreviatedMonthNames            {janv., févr., mars, avr.…}            {Jan, Feb, Mar, Apr…}                  {Jan, Feb, Mar, Apr…}                  {Jan, Feb, Mar, Apr…}
    MonthNames                       {janvier, février, mars, avril…}       {January, February, March, April…}     {January, February, March, April…}     {January, February, March, April…}
    IsReadOnly                       True                                   True                                   True                                   True
    NativeCalendarName               calendrier grégorien                   Gregorian Calendar                     Gregorian Calendar                     Gregorian Calendar
    AbbreviatedMonthGenitiveNames    {janv., févr., mars, avr.…}            {Jan, Feb, Mar, Apr…}                  {Jan, Feb, Mar, Apr…}                  {Jan, Feb, Mar, Apr…}
    MonthGenitiveNames               {janvier, février, mars, avril…}       {January, February, March, April…}     {January, February, March, April…}     {January, February, March, April…}

```

## Date formatting codes

The use of y, M, d, h, m and s and quoted characters first appeared in Excel in the late 1980s.  
There are some case sensitivity traps: Upper case M is Month and lower case m is minute. H is 24 hour clock, and h is 12 hour clock.

| Code | Explanation                                                                        |
|------|------------------------------------------------------------------------------------|
| d    | Day of month; dd = with leading zero, d = without                                  |
| ddd  | Day of week; ddd = 3 letter abbreviation, dddd = name in full                      |
| f    | Fraction of second; ff = 2 places, fff = 3 places etc., up to 6 places             |
| h/H  | Hour of day; Lowercase for 12-hour clock: hh = with leading zero, h = without      |
|      | Upper case for 24-hour clock; HH with leading zero, H without                      |
| m    | Minute; Lower case mm = with leading zero, m= without                              |
| M    | Month; Upper case MM = with leading zero; M without                                |
|      | MMM as 3 letter abbreviation; MMMM name in full                                    |
| s    | Seconds; ss = with leading zero, s = without                                       |
| t    | am / pm; t = as "A"/"P" ; tt as "AM"/"PM"                                          |
| y    | Year; yy = with leading zero, y = without. Either yyy or yyyy = as 4 digits        |
| z /K | Time Zone; zz = as +hours with leading zero, z = without                           |
|      | zzz or zzzz or K as HH:mm format                                                   |
| g    | Epoch; ("A.D" or "B.C")                                                            |

When formatting columns of data it is preferable to use leading zeros to keep everything aligned, months, days and hours embedded in a block of text may look better without the leading zero.

You can’t use the letters above singly. So for example, if you want to know if the time is AM or PM, `.ToString("t")` or `Get-Date -format "t"` doesn’t give the right result, it needs to be `"%t"`. The reason is that there are single letter shortcuts, as follows.

| Code      | Description                            | Notes                         |
|---------- |----------------------------------------|-------------------------------|
| "d"       | Short-date pattern                     |                               |
| "D"       | Long-date pattern                      |                               |
| "t"       | Short-time pattern                     |                               |
| "T"       | Long-time pattern                      |                               |
| "f"       | Full date/time pattern (short time)    | Long-Date  + Short-time       |
| "F"       | Full date/time pattern (long time)     | Long-Date  + Long-time        |
| "g"       | General date/time pattern (short time) | Short-date + Short-time       |
| "G"       | General date/time pattern (long time)  | Short-Date + Long-time        |
| "M", "m"  | Month/day pattern                      |                               |
| "O", "o"  | Round-trip date/time pattern           | Not part of culture object    |
|           | yyyy-MM-ddTHH:mm:ss:fffffffzzz         |                               |
| "R", "r"  | RFC1123 pattern                        | See note below                |
| "s"       | Sortable date/time pattern             | yyyy'-'MM'-'dd'T'HH':'mm':'ss |
| "u"       | Universal sortable date/time pattern   | See note below                |
| "U"       | Universal full date/time pattern       |                               |
| "Y", "y"  | Year month pattern                     |                               |

These short codes are especially useful with the `-f` operator, the local codes are preferable to spelling out day, month and year because if a script travels internationally the date will be wrong in some places, but `Today is {0:D}" -f $now`  will display in local long format, by contrast, `"Today is $now"` results in a cast operation and displays date and time in invariant format which might not be what the user expects.  

The third post in this series will look at time zones, and I said above that errors can result from handling times without proper care and this applies to the "R" and "u" patterns in the table...

`Get-Date` gets the current **local time** and if part of the date/time is specified as parameter, then that part is adjusted; the resulting time can be converted to **UTC time**, and formatting can be applied, like this

```powershell-interactive
    Get-Date -month 6 -day 30 -Format "r"
    Thu, 30 Jun 2022 12:12:15 GMT

    Get-Date -month 6 -day 30 -Format "u"
    2022-06-30 12:12:17Z

    Get-Date -month 6 -day 30 -Format "U"
    2022-06-30 11:12:19

    Get-Date -month 6 -day 30 -AsUTC
    30 June 2022 11:12:23

    Get-Date -month 6 -day 30 -AsUTC -Format "u"
    2022-06-30 11:12:30Z

    Get-Date -month 6 -day 30 -Format "o"
    2022-06-30T12:12:34.8117506+01:00
```

On the 30th of June my local time zone will be on British Summer Time - which is UTC+1. So the clock will read "12:12" but the UTC time is 11:12. The format codes "R", "r" and "u" attach a suffix to say "this is UTC Time" without ensuring the time **is** UTC first. "U" (upper and lower case u are different) **will** convert to universal time without adding a suffix to tell you it has done so.  
One built in format "O" or "o" (either case) will output the date and time as the clock would display it, together with the offset from UTC, and this the one to use when writing something which will be read by another program.

And those considerations will be the core of part 3.

## TimeSpans

Before leaving the formatting of Dates and times, I wanted to mention that `[TimeSpan]` objects have similar formatting.  
A timespan is the difference between two times - for example how long did PowerShell wait for my input between  finishing one command and starting the next?

```powershell-interactive
    $waitTime = (get-history -id 533).StartExecutionTime - (get-history -id 532).EndExecutionTime
    $waitTime
    Days              : 0
    Hours             : 0
    Minutes           : 0
    Seconds           : 7
    Milliseconds      : 730
    Ticks             : 77300101
    TotalDays         : 8.94677094907407E-05
    TotalHours        : 0.00214722502777778
    TotalMinutes      : 0.128833501666667
    TotalSeconds      : 7.7300101
    TotalMilliseconds : 7730.0101

```

There are 3 short form formats "c" ,"g" and "G", for "common" "Short General" and "Long General".  
For custom formats dd/d, hh/h, mm/mm ss/s (for days, hours, Minutes and seconds with/without leading zeros ) and f..fffff for fractions have the same meanings as they do for `[datetime]` formatting but there is one important change - any punctuation (or spaces) must be escaped with a `\` character.  
So `$waitTime.ToString("s.fff")` will cause an error that format string was in the wrong format because the "." was not escaped, but the following, with the addition of a `\`, works:

```powershell-interactive
    $waittime.ToString("s\.fff") 
    7.730

```