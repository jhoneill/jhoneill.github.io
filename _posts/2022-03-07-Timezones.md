---
layout: post
title:  "PowerShell and .NET timezones"
date:   2022-03-07
tags: 
    - PowerShell
    - Development
    - .NET
    - Formatting
---
>  People assume that time is a strict progression of cause to effect, but *actually* from a non-linear, non-subjective viewpoint - it's more like a big ball of wibbly wobbly timey wimey stuff.

From Doctor Who - Blink, by Stephen Moffat.

## A brief history of Timezones

The line along the earth's surface where it is *noon* - i.e. the sun is directly overhead - travels 1 degree to the West every 4 minutes, completing the whole 360 degrees in 24 hours.
If you know how much time passes between noon occurring at two points, you know difference in longitude between them. In the early 1700s the Royal Navy had the first clocks able to keep time reliably on-board ships, and they selected Greenwich on the Eastern side of London as their reference point for time differences and longitude calculations. British sea power at the time made *that* the standard and longitude (and time) still works from the Greenwich meridian - the word is the latin for "noon". What is properly called "Coordinated Universal Time (or UTC)" is still "Greenwich Mean Time" to old-fashioned Brits. And UTC is used for the name because it is neither the English nor the French word order!

Before railways and telegraph, clocks everywhere were to set read noon as the sun passed overhead; when differences between nearby places started to cause problems (like travelling East and arriving at the station to catch the train home a few minutes late) people started using timezones - although clocks *should* change in 1 hour steps for every 15 degrees of longitude, there were more convenient groupings and boundaries.

In summer the sun rises before it is time to wake up, but sets before we go back to sleep, and clocks are shifted forward to tell us to wake up closer to sunrise, and to send us to bed having used less artificial light, so "Summer time" is properly called "Daylight Savings Time".
Summer in the Northern Hemisphere is winter in the Southern and vice versa, and there is no global standard for applying the change, making a **.NET timezone** an **offset from UTC** combined with a **set of DST rules** (when the clocks go forwards and back, and by how much)  `Get-TimeZone -ListAvailable | measure` counted 141 time zones in .NET6.

In .NET a `[DateTime]` is *either* **universal** time - fixed to zero offset from UTC, without DST *or* it is **local time** - as you'd read from a clock with the local offset and current DST settings applied. Local times are *not* marked "London" or "Paris" or "New York" time, simply "Local". A local time can convert to UTC using the time zone settings in force when the conversion is done.

```powershell-interactive
    ps> (Get-TimeZone).DisplayName
    (UTC+00:00) Dublin, Edinburgh, Lisbon, London

    ps> $t.ToUniversalTime()     
    21 January 2022 15:57:03

    ps> Set-TimeZone "Eastern Standard Time"

    ps> $t.ToUniversalTime()
    21 January 2022 20:57:03

```

In the examples above `$t` holds a local time. When the time is first converted, the timezone is London (winter) time which is UTC +0, so converting to UTC makes no difference.  
When the time zone is changed, `$t` doesn't magically update to say it should be no longer be *London* local time but *Eastern US and Canada* local time. So the "clock" still reads three minutes before four in the afternoon. When we say "Convert the time on this clock to UTC" and local time is UTC -5, the UTC time is three minutes to nine in the evening. This is less of a problem that it might sound.

As a time "15:57:03" doesn't tell us very much. Is that UTC (sometimes marked 'z' or 'Zulu time')? Or is it Paris Time? New York Time? Generally if a time needs to be saved it should either be UTC (and marked as such) or given with it's offset from UTC as follows:

```powershell-interactive

    ps> $now.ToString('o')
    2022-01-21T15:57:03.0993717-05:00
    
    ps> $now.ToUniversalTime().ToString("u")
    2022-01-21 20:57:03Z

```

As was mentioned at the end of the [previous post](/powershell/2022/03/06/DateFormats.html), the "u" format code will add a z *even if the time is a local time* so **always** convert to universal before using it.  The "o" format says "The clock read 15:57, and was set to UTC -5:00" (Most timezones are use whole hour offsets but a few use half hours.)

Just as culture settings determine how a `[dateTime]` should be *formatted*, timezone settings determine how to *convert* a `[datetime]` from local time to UTC, or from UTC time to local. In timezones that use DST a different number of hours needs to be added or subtracted at different times of the year, so *the date has to be considered when adjusting times*.  
Following on from the previous example, by July daylight savings moves clocks in New York from UTC -5 to UTC -4:

```powershell-interactive
    ps> $now.AddDays(180) 
    20 July 2022 15:57:03

    ps> $now.AddDays(180).ToUniversalTime()
    20 July 2022 19:57:03

```

Notice that when we add days (or smaller amounts) to a local time it moves the "clock hands" without thinking about DST changes - a stopwatch would show 1 hour less had passed, and if we convert to universal time before moving forward 180 days the result is different to the one above.

```powershell-interactive
    ps> $now.ToUniversalTime().AddDays(180)
    20 July 2022 20:57:03

```

"Difference in wall time" and "elapsed time" aren't guaranteed to be the same, and it is valid to ask the difference between a local and a UTC time, so it's a good idea ensure that both times are the right kind before doing any calculation, the following example is from the last time the clocks changed in this part of Europe  

```powershell-interactive
    ps> Set-TimeZone "GMT Standard Time"
    ps> $before = [datetime]'2021/10/30 10:00'
    ps> $before.IsDaylightSavingTime()
    True

    ps> $after  = [datetime]'2021/10/31 10:00'
    ps> $after.IsDaylightSavingTime()
    False

    ps> $after.Subtract($before).TotalHours
    24

    ps> $after.ToUniversalTime().Subtract($before.ToUniversalTime()).TotalHours
    25

```

Whether you *want* "elapsed time" or "difference in wall time" depends on the situation.

.NET's `TimezoneInfo` class understands how to shift the clock to from one timezone to another:

```powershell-interactive
    ps> ([System.TimeZoneInfo]::ConvertTimeBySystemTimeZoneId($after, "Eastern Standard Time")) 
    31 October 2021 06:00:00

    ps> $NYTime = ([System.TimeZoneInfo]::ConvertTimeBySystemTimeZoneId($before, "Eastern Standard Time")) 
    ps> $NYTime

    30 October 2021 05:00:00

```

Clocks in London and New York change to and from DST on different dates, so time zone info has converted local time to UTC using London rules, and then applied Eastern US rules to convert to UTC. **But** as a .NET `[DateTime]` object this is "In New York the clock would say" so is does the time think it is a Local or UTC time ?

```powershell-interactive
    ps> $NYTime.Kind
    Unspecified
```

It's neither, so trying to convert it to local or universal *should* cause an error; but it doesn't, converting To Local and To Universal *both* change the time.

```powershell-interactive
    ps> $NYTime
    30 October 2021 05:00:00

    ps> $NYTime.ToLocalTime()
    30 October 2021 06:00:00

    ps> $NYTime.ToUniversalTime()
    30 October 2021 04:00:00

```

The `.ConvertTimeBySystemTimeZoneId()` operation returns what a clock in another timezone would say. As used above with a single timezone, it converts from "here" to "there", but it can take two time zones, the source and the destination like this:

```powershell-interactive
    ps> [System.TimeZoneInfo]::ConvertTimeBySystemTimeZoneId($after, "Eastern Standard Time" , "AUS Eastern Standard Time") 
    01 November 2021 01:00:00

```

So when it is 10AM in New York, it's 1 the following morning in Sydney.
