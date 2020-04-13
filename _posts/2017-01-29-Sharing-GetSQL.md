---
layout: post
title:  "Sharing GetSQL"
date:   2017-01-29
categories: 
    - PowerShell
    - Databases
tags:
    - SQL
    - ODBC
    - Scripting
    - SQLServer
---



I’ve uploaded a tool named “GetSQL” to the [PowerShell Gallery](https://www.powershellgallery.com/packages/GetSQL/). 
Three of my last four posts (and one upcoming one) are about sharing stuff which I have been using for a long time and GetSQL goes back several years – every now and then I add something to it, so it seems like it might never be _finished_, but eventually I decided that it was fit to be shared.

When I talk about good PowerShell habits, one of the themes is _avoiding massive “do-everything” functions_; it’s also good minimize the number of dependencies on the wider system state – for example by avoiding global variables. Sometimes it is necessary to put these guidelines to one side –  the module exposes a single command, **Get-SQL** which uses argument completers (extra script to fill in parameter values). There was a time when you needed to get [Jason Shirks TabExpansion Plus Plus](https://www.powershellgallery.com/packages/TabExpansionPlusPlus) to do anything with argument completers but, if you’re running a current version of PowerShell, `Register-ArgumentCompleter` is now a built in cmdlet. PowerShell itself helps complete parameter _names_ – and Get-SQL is the only place I’ve made heavy use of parameter sets to strip out parameters which don’t apply (and needed to [overcome some strange behaviour](/powershell/2016/11/30/ParameterPecularities.html) along the way); having completers to fill in the values for the names of connections, databases, tables and column names dramatically speeds up composing commands – not least by removing some silly typos.

One thing about **Get-** commands in PowerShell is that if the PowerShell parser sees a name which doesn’t match a command where a command-name should be it tries putting `Get-` in front of the name so `History` runs `Get-History`, and `SQL` runs `Get-SQL`. That’s one of the reasons why the command didn’t begin life as “Invoke-SQL” and instead became a single command. I also wanted to be able to run lots of commands against the same database without having to reconnect each time. So the connection objects that it uses are global variables, which survive from one call to the next – I can run    
`SQL "Select id, name from customers where name like '%smith' "`    
`SQL "Select * from orders where customerID = 1234"`    
without needing to make and break connections each time – note, these are shortened versions of the command which could be written out in full as:     
`Get-SQL –SQL "Select * from orders where customerID = 1234"`
The first time `Get-SQL` runs it needs to make a connection, so as well as `–SQL` it has a `–Connection parameter` (and extra switches to simplify connections to SQL server, Access and Excel). Once I realized that I needed to talk to multiple databases, I started naming the sessions and creating aliases from which the session  can be inferred (I described how [here](/powershell/2014/06/16/Aliases-Doing-Parameters.html)) so after running    
`Get-SQL -Session f1 -Excel  -Connection C:\Users\James\OneDrive\Public\F1\f1Results.xlsx`    
(which creates a session named F1 with an Excel file containing results of formula one races) I can dump the contents of a table with     
`f1 –Table "[races]"`

Initially, I wanted to _either_ run a chunk of SQL like the first example (and I added a `–Paste` switch to bring in SQL from the Windows Clipboard), or to get the whole of a table (like the example above with a `–Table` parameter), or see what tables had be defined and their structure so I added `–ShowTables` and `–Describe`.  The first argument completer I wrote used the “show tables” functionality to get a list of tables and fill in the name for the `–Table` or` –Describe` parameters. Sticking to the one function per command rule one would mean writing “Connect-SQL”, “Invoke-RawSQL” “Invoke-SQLSelect”, and “Get-SQLTable” as separate commands, but it just felt right to be able to allow    
`Get-SQL -Session f1 -Excel  -Connection C:\Users\James\OneDrive\Public\F1\f1Results.xlsx –showtables`

It also just felt right to allow the end of a SQL statement to be appended to what the command line had built giving a command like the following one – the output of a query can, obviously, be passed through a PowerShell Where-Object command but makes much more sense to filter before sending the data back.    
`f1 –Table "[races]" "where season = 1977"`

`"where season = 1977" ` is the `–SQL` parameter and `–Table "[races]"` builds "Select * from [Races]" and the two get concatenated to    
`"Select * from [Races] where season = 1977"`.    
With argument completers for the table name, it makes sense to fill column names in as well, so there is a `–Where` parameter with a completer which sees the table name and fills in the possible columns. So the same query results from this:    
`f1 –Table "[races]"   -where "season"   "= 1977"`    
I took it a stage further and made the comparison operators work like they do in `Where-Object`, allowing    
`f1 –Table "[races]" -where "season" -eq 1977`

The script will sort out wrapping things in quotes, changing the normal * into SQL’s % for wildcards and so on. With select working nicely (I added –Select to choose columns, –OrderBy, –Distinct and –GroupBy ) I moved on to insert, update (set) and Delete functionality. You can see what I meant when I said I keep adding bits and it might never be “finished”.

Sometimes I just use the module as a quick way to add a SQL query to something I’m working on: need to query a Lync/Skype backend database ? There’s Get-SQL. Line-of-business database ? There’s Get-SQL. Need to [poke round in Adobe Lightroom’s SQL-Lite database](/powershell/photography/2012/08/09/Lightroom-data.html)? I do that with with Get-SQL -  though in both cases its simple query building is left behind, it just goes back to its roots of running a lump of SQL which was created in some other tool.  The easiest way to get data into Excel is usually the EXPORT part of [Doug Finke’s excellent import-excel](https://www.powershellgallery.com/packages/ImportExcel) and sometimes it is easier to do it with insert or update queries using Get-SQL. The list goes on … when your hammer seems like a good one, you find an awful lot of nails.
