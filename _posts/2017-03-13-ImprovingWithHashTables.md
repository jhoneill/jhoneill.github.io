---
layout: post
title:  "Improving PowerShell performance with hash tables."
date:   2017-03-13
categories: PowerShell
---

Often the tasks which we do with PowerShell scripts aren’t very sensitive to performance – unless we are sitting drumming our fingers on the desk waiting for it to complete there isn’t a lot of value in making it faster.   When [I wrote about Start-Parallel](/powershell/2016/12/06/100TimesFasterStartParallel.html), I showed that some things only become viable if they can be run reasonably quickly; but you might assume that scripts which run as scheduled tasks can take an extra minute if they need to.


That is not always the case.  I’ve been working with a client who has over 100,000 Active directory users enabled for Lync (and obviously they have a lot more AD objects than that). They want to set Lync’s policies based on membership of groups, and users are not just placed directly into the policy groups but nested via other groups. If users have a policy but aren’t in the associated group, the policy needs to be removed. There’s a pretty easy Venn diagram for what we need to do.
![VennDiagram](/assets/venn-diagram.png)
If you’ve worked with LDAP queries against AD, you may know how to find the nested members of the group using `(memberOf:1.2.840.113556.1.4.1941:=<<Group DN>>)`.
The Lync/Skype for Business cmdlets won’t combine an LDAP filter for AD group membership and non-AD property filter for policy – the user objects returned actually contain more than a dozen different policy properties, but for simplicity I’m just going to use ‘policy’ here – as pseduo-code the natural way to find users and change policy – which someone else had already written – looks like this:
{% highlight powerShell %}
Get-CSuser –ldapfiler "(   memberOf <<nested group>> )" | where-object {$_.policy –ne $PolicyName} | Grant-Policy $PolicyName
Get-CSuser –ldapfiler "( ! memberOf <<nested group>>) " | where-object {$_.policy –eq $PolicyName} | Grant-Policy $null
{% endhighlight %}
(Very late in the process I found there was a way to check Lync / Skype policies from AD but it wouldn’t have changed what follows).
You can see that we are going to get every user, those _in_ the group in the first line and those _out of_ it in the second. These “fan out” queries against AD can be slow – [MSDN](https://docs.microsoft.com/en-gb/windows/win32/adsi/search-filter-syntax) has a warning _“Some such queries on subtrees may be more processor intensive, such as chasing links with a high fan-out; that is, listing all the groups that a user is a member of.”_   Getting the two sets of data was taking over an hour.  But so what? This is a script which runs once a week, during quiet hours, provided it can run in the window it is given all will be well.  Unfortunately, because I was changing a production script, I had to show that the correct users are selected, the correct changes made and the right information written to a log. While developing a script which will eventually run as scheduled task, testing requires we run it interactively, step through it, check it, polish it, run it again, and a script with multiple segments which run for over an hour is, effectively, **untestable** (which is why I came to be changing the script in the first place!).

I found I could unpack the nested groups with a script a _lot_ more quickly than using the “natural” 1.2.840.113556.1.4.1941 method; though it feels _wrong_ to do so. I couldn’t find any ready-made code to do the unpack operation – any search for expanding groups comes back to using the OID method which reinforces the idea.

I can also get all 100,000 Lync users – it takes a few minutes, but provided it is only done once per session it is workable, (I stored the users in a global variable, if it was present I didn’t re-fetch them)

So: I had a variable `$users` with users and a variable `$members` which contains all the members of the group; I just had to work out who is each one but not in both. But I had a new problem. Some of the groups contain tens of thousands of users. Lets assume half the users have the policy and half don’t (50,000 in each category). If I run
`$users.where{$_.policy -ne $PolicyName -and $members -contains $_.DistinguishedName}`
and
`$users.where{$_.policy -eq $PolicyName -and $members -notcontains $_.DistinguishedName}`
the `-contains` operation is going to have a LOT of work to do: if everybody has been given the right policy _none_ of the 50,000 users without it are in the group, but we have to look at all 50,000 group-members to be sure – 2,500,000,000 string comparisons. For the 50,000 users who do have the policy, on average \[Not\]contains has to look at half the group  members before finding a match so that’s 1,250,000,000 comparisons. 3.75 Billion comparisons for each policy means it is still too slow for testing. Then I had a flash of inspiration, something which might work.

I learnt about Hash-tables as a computer science undergraduate, and – as the on-line help puts it  – they are “very efficient for finding and retrieving data”. This can be read as _they lead to neat code _(which has been their attraction for me in the past) or as _they minimize CPU use_, which is what I need here.  Microsoft also call hash tables  “associative arrays” and often the boundary between a set of key-value pairs (a dictionary) and a “true” hash table is blurred – with a “true” hash tables the _location_ of data in memory is based on the key value – so an item can be found without scanning the whole list. Some ways to do fast-finds make tables slow to build. Things I’d never considered with PowerShell hash tables might turn out to be important at this scale. So I built a hash table to return a user’s policy given their DN:
`$users | ForEach-Object -Begin {$hash=@{}} -Process {$hash[$_.distinguishedName] = "" + $_.policy}`
About 100,000 users were processed in under 4 seconds, which was a relief; and the right policy came back for `$hash["cn=bob…"]`
– it looked instant compared with a couple of seconds with `$users.where({$_.distinguishedName –eq “cn=bob…”}).policy`

This hash table will return one of 3 things. If Bob isn’t set-up for Lync/Skype for Business I will get NULL; if Bob has no policy I will get an empty string (that’s why I add the policy to an empty string – it also forces the policy object to be a string), and if he has a policy I get the policy name. So it was time to see how many users have the right policy (that’s the magenta bit in the middle of the Venn diagram above)
`$members.where({$hash[$_] -like $policyname}).count`

I’d found ~10,000 members in one of the policy groups, and reckoned if I could get the “find time” down from 20 minutes to 20 seconds that would be OK and … fanfare … it took 0.34 seconds. If we can check can look up 10,000 items in a 100,000 item table in under a second, these must be proper hash tables. I can have the `.where()` method evaluate
`{$hash[$_] –eq $Null}` for the AD users who aren’t Lync users or
`{$hash[$_] –notin @($null,$policyName) }` for users who need the policy to be set.
It works just as well the other way around for setting up the hash table to return “True” for all members of the AD group; non-members will return null, so we can use that to quickly find users with the policy set but who are not members of the group.
```
$members | ForEach-Object -Begin {$MemberHash=@{}} -Process {$MemberHash[$_] = "" + $true}
$users.where({$_.policy -like $policyname -and -not $memberhash[$_.distinguishedName]}).count
```

Applying this to all the different policies slashed the time to do everything in the script from several hours down to a a handful of minutes.  So I could test thoroughly before the ahead of putting the script into production.
