---
layout: post
title:  "SQLite in PowerShell - Extending GetSQL"
date:   2020-11-18
categories: PowerShell
---

_**TLDR**. `Install-Module GetSQL` will give you a module which queries **ODBC** (including
Access and Excel files) and queries **SQL-Server** and (now) **SQLite** natively._

When I talk about writing PowerShell - or any code – I stress making it easy to follow. 
I like to say “Don’t make it readable for a random stranger - do it for your future self".  
Faced with altering something - especially fixing it in a crisis, *future-you* can love or hate *present-day-you*.    
And when I go back to old code, I feel pride or pain depending on how well I lived up to that, so I
was quite pleased to change GetSQL for the first time 
in over two years and found it longer to update the help than to add new functionality - the "what was I doing here" time was almost zero.

**Quick history**: the [GetSQL module ](https://www.powershellgallery.com/packages/GetSQL/) has 
one command, `Get-SQL` which PowerShell run can as simply `SQL` (PowerShell's final attempt to resolve an unmatched name is put `Get-` in front of it). The command:
* throws SQL statements at databases. 
* composes `Select`, `Update`, `Insert` and `Delete` queries from the command line with the aid of **intellisense**
* keeps the database-sessions open, to avoid tearing one down only to set it up afresh a few seconds later
* can keep multiple sessions open and makes aliases of itself to call them.

So, I can do this
{% highlight powerShell %}
Get-SQL -Excel -Connection ~\onedrive\public\f1\f1Results.xlsx
Get-SQL -Table "Races" -Where "WinningDriver" -EQ "Lewis Hamilton" -Select "Venue" -Distinct -OrderBy "Venue" | Out-String 
Get-SQL -Close 
   
{% endhighlight %}

The first line makes a default / unnamed session, in this case using the ODBC driver for Excel. If I had specified `-Session "F1"` then I could open another database and use this one either with     
`Get-SQL -session F1 -table...`    
or by using the session name as a command alias:    
`F1 -table...`    
For the second line I typed `sql -t[tab]` to complete the parameter-name and `ra[tab]*` to complete the value; having a database connection allows tab-completion to get a list tables so I don't need to remember if the name is "Race" or "Races" (more commands run first time). Then, knowing the table, it can cycle through field names when I get to `Where`, `Select`, and `OrderBy`. I wrote about and gave talks on how I did the completion way back in 2013, and I still find **intellisense /tab-completion saves a lot of time** . 

 I can give the command a ready-made SQL statement and/or merge the *connect*, *query*, and *close* operations into one command:     
{% highlight powerShell %}
sql "SELECT TOP 20 DriverName, min(Driverage) As Age  
     FROM Results WHERE RacePos  =  '1' 
     GROUP BY Drivername 
     ORDER BY min(Driverage)" -Connection "~\onedrive\public\f1\f1Results.xlsx"  -Excel -close
   
{% endhighlight %}
A parameter named `-SQL` takes the first unnamed  value on the command line and adds it to any statement built by the other parameters  (so if they didn't build *anything*, it just runs); `$SQL` seems the natural variable to hold a SQL statement so the command is sometimes run as `SQL -SQL $SQL`.    
I **added a `-paste` parameter** because I was often copying a query from other code in order to test it. I also found myself doing     
`SQL «some query» | ft` so often that I assigned it the  **¬** symbol unlikely to be used by anything else because it's not on US keyboards - which have 1 key fewer than UK ones and lack a pound (£) sign. (Yes, I know Americans *do* call # "pound" because it was once used for pounds weight). That allowed the following : 
```
¬ "SELECT TOP 5 DriverName, count(RaceDate) as wins
>> FROM Results WHERE RacePos  =  '1'  GROUP BY Drivername 
>> ORDER BY count(raceDate) desc"
```
all composed in six keystrokes `[¬] [space] ["] [ctrl]+[v] ["] [return]`

The **original version** of Get-SQL connected to **ODBC** using either a pre-configured Data Source Name or a connection string. 
When I **added SQL Server Native Client** support I didn't want two parameters beginning "SQL" so a switch `-MSSqlServer` signifies "connect using the SQL native client" and if `-connection` is just be a server name the function will be build connection string around it. That led to **switches for Access and Excel files**, reducing this:    
`-Connection "Driver={Microsoft Excel Driver (*.xls, *.xlsx, *.xlsm, *.xlsb)}; DriverId=790; ReadOnly=0; Dbq=$xlPath;"`    
to this:    
`-Connection  $xlPath -Excel"`

I've been working with **SQLite** using the [ODBC driver](http://www.ch-werner.de/sqliteodbc/) to talk to Adobe's data in **Lightroom** since... well [this blog post](/powershell/photography/2012/08/09/Lightroom-data.html) dates from 2012. But I **don't want make people to download the driver** before than using something and distributing the driver myself would be problem. But there is a native [ADO driver](https://system.data.sqlite.org/index.html/doc/trunk/www/index.wiki) which can be [downloaded from Nuget](https://www.nuget.org/packages/System.Data.SQLite.Core) and added to a module. As with SQL-Server I didn't want to have another switch beginning  with SQL so I added a `-Lite` switch, after doing it I thought "Oh a light-switch. ha-ha.". The thing that made feel I'd designed it right all those years ago was how little work it was. If `-Lite` is specified the function loads the right version of `System.Data.SQLite` and where there was anything driver-specific for SQL server, I duplicate it and change the driver name for SQLite. I've found the SQLite driver doesn't always release file locks, so I added some extra lines on closing a connection which seem to improve things - but not fix it entirely.   

_**Note**: the module works in PowerShell 7.x, 6.x and Windows PowerShell 5.x (32 and 64-bit) and should be cross platform, but the Linux files in the Nuget package do not appear to be complete, and I haven't tested on a Mac. If anyone has any tips for non-windows platforms please put them in [an Issue on github](https://github.com/jhoneill/GetSQL/issues)_ 

So now I can get to my Adobe-Lightroom data in two ways - the old one with the pre-configured DSN:
{% highlight PowerShell %}
sql -Connection  DSN=LR 
sql -Table AgInternedExifCameraModel  -Select "value" -OrderBy  "value"  -Distinct
    
{% endhighlight %}
or the new one with the path to the file and `-Lite`.  
{% highlight PowerShell %}
$lrpath = Join-Path ([environment]::GetFolderPath("MyPictures")) "Lightroom\Catalog-2.lrcat"
sql -Connection  $lrpath  -lite 
sql -Table AgInternedExifCameraModel  -Select "value" -OrderBy  "value"  -Distinct
   
{% endhighlight %}
**Chromium data** pushed me to **support opening SQLite files like Access** or Excel ones.  Chromium-based Microsoft **Edge** became my browser of choice during its beta (and I've kept running the Dev version after release) and I became interested in the information it stores, but **Google Chrome**, and **other Chromium browsers** store the data in the **same way**. For the *Edge-specific collections* I can start exploring like this 

{% highlight PowerShell %}
cd ~
copy '.\AppData\Local\Microsoft\Edge Dev\User Data\Default\Collections\collectionsSQLite' $env:temp
sql -lite -Connection "$env:temp\collectionsSQLite" -ShowTables 
    
{% endhighlight %}
But there are a many other things which seem to use SQLite and **Get-SQL** should help with most or all of them, you can get it from [the PowerShell gallery](https://www.powershellgallery.com/packages/GetSQL) and the source files are on [GitHub](https://github.com/jhoneill/GetSQL). I've included a simple filter that checks to see if a file is a SQLite database, and script to download the files from Nuget and add them to the module (so you don't need to trust the binaries I give you!) I have also [posted a Gist](https://gist.github.com/jhoneill/7f27ff9a7f00945fea900d9a656613de  ) with some examples of using it.   

