---
layout: post
title:  "Clear, Simple and Wrong. How KPIs can be damaging."
date:   2020-11-21
categories: PowerShell
tags: 
    - Management
    - Software Engineering
    - KPI
---

There’s an old joke with a new extension. The original runs like this:    
*A lost balloonist sees a golfer and shouts down “Where am I?”; the golfer replies
“Halfway down the fourth fairway!”. The balloonist, non-plussed, shouts back
“You work in IT don’t you?” when the golfer asks how he could tell, the
balloonist says “because you gave an answer that was both totally accurate and totally
useless!”*    
I heard my Dad tell this when I was still at school, and he told it about
accountants, but many professions fit and it probably pre-dates both hot-air balloons *and* golf. In the extension the golfer shouts back:     
*“And you manage IT activities, don’t you?”. Now the 
balloonist is surprised and asks what gave that away. “Well…” says the golfer. “You don’t 
know where you are, or where you’re heading. You don’t know how to ask for help and
complain when you get it. And now…” he says, “You think it’s all my fault!”.*

Like most jokes there’s something we can recognise here, good managers of people
and projects have **situational awareness**, take **responsibility**, and make
**effective requests** of others.

*Experience* is one of the few consolations of age, and my experience is that
successful projects have had certainty about what needed to be done and
understood their progress. Poor grasp of those things and shifting goals have
been signs of  approaching failure. Lack of situational awareness didn’t come 
on its own, the balloonist’s other faults were usually in attendance.

**There shouldn’t be any massive insight in this.** 

Different places I’ve worked in have tried lots of approaches to say informed 
and what works in one may fail elsewhere – otherwise 
we’d all learn and apply a method which was easy and reliable. **Measuring** things in 
an effort to understand what is happening sometimes **changes behaviour** - one of the 
wiser managers I had in my time at Microsoft told me [“**What you measure is what you get (WYMIWYG)**”](/misc/2007/09/28/Wymiwyg.html), 
and measurement sometimes gives an **incentive to do the wrong thing**. 

I'd seen WYMIWYG and the harm done by **encouragement to game the system** 
early in my career, when I worked on a support help line. A junior
colleague was given the objective of handling a given number of calls per day.
If she could pass a call to someone else that counted. If she sent the caller 
away with a task to do (whether it helped solve the problem or not) that
counted. If she gave a wrong answer, it counted. If she just cut the call 
off… I think *that* counted too. Measures like “how many customers were happy 
a week later” were impractical, although there is a modern vogue
for following up every interaction with an organization with “Based on this
would you recommend us” questions - as if we all go round recommending gas
suppliers or software vendors to our friends and relations. Vendors would do
well to remember that being asked makes some of us less likely to recommend them - 
another case where measuring something changes it and not always for the better.

I was talking to someone recently who was troubled by seeing **simplistic
gameable measures driving undesirable behaviour.**  I've written about 
people who [give an impression of delivery by creating technical debt ](/misc/2016/06/26/TechnicalDebt4DangerousWords.html). 
and when I was writing that post I found ["The quiet crisis unfolding in software development"](https://medium.com/@billjordan1/the-quiet-crisis-unfolding-in-software-development-cffbdafbf450#.lg0eka1ra) which 
I can recommend, if only to see the context of this quote - which *Medium* says is the most highlighted part of the piece
>   Odds are far better than good that your high performers are achieving what appears to be high levels of productivity by building technical debt into the application by taking shortcuts whether intentionally or unintentionally. 

My friend saw that people obsessed on measures designated as “Key Performance 
Indicators” and he labelled the result “KPI-induced problems”. 
I'd call it "Getting what you measure". Simplistic measures – number of work 
items completed by a developer, or the average time items are in their queue,
meant that people were **closing items in any way they could** – just like 
those support calls from years ago. The results included pushing work items 
to others (like madly punching the buttons on a chess clock), declaring them 
done when they were not done *properly* and would give rise to another work 
item and so on. 

Let me try to illustrate the role of KPIs in this…

Developer X has marked 8 work items as done. Of these 4 will eventually come back 
for re-work, but we can't tell which 4 at this stage.  
Half of everything X *starts* is also re-work, so 4 of the items 8 were being reworked, 
and 2 of *those* had already been reworked, and 1 of *those* had been reworked 
*twice* before. So out of the 8 we have
-   2 new items completed at the first attempt,
-   2 new items which look OK now but will require re-work, so aren’t truly *complete*, but are *counted*
-   2 old items completed at this attempt
-   2 old items which will need at least one more attempt (again *counted but not complete*)

Developer Y has marked 6 items as done. One was a rework item, and 1 is going to
going to come back for rework so out of 6 that’s
-   4 done at the first attempt
-   1 old item completed at this attempt
-   1 item which will require re-work (incomplete but counted)

Who is the better developer? Y *finished* 5 items; X *finished* 4 Y is
double-counting 1, but X has managed to double-count 4, **simplistic KPIs have
created a perverse incentive**, doing the work badly and repeating it looks
better than doing it right first time.

I mentioned **technical debt** earlier. So, let's extend the example to developer Z who 
has only checked in 4 items. For one of those, Z encountered a piece of **X’s poor work**,
and had to choose between sending it back to X, fixing it, or bodging around it. 
Creating a new work item to fix it and getting that item scheduled 
(especially getting X to get it right) would delay items in Z’s queue and 
make other KPIs look bad. Really, we’d want to track that work, but again KPIs 
*drive the wrong behaviour*. **Z did the right thing** and fixed the issue, but 
now **X has got credit** for an item which Z’s work got to a proper state. 
Is X a bad person putting in work that others need to finish? Or are 
they **just doing what they’ve been told to do**, marking items as “done” as 
quickly as possible? Which of the three developers would you want on your team? 
Who will come out best when it’s annual review time? And if your instinct is 
to create a second KPI counting the items returned for rework, take a moment 
to think about how people will behave and what assigning blame for rework 
items does to teamworking .

Indicators which don’t tell you the truth are unhelpful and possibly even
dangerous, so why tolerate KPIs which say the worst team member is the best? The
answer, I think, is wanting to **answer difficult questions with easy answers**. If
we want to know “To what extent are X,Y and Z helping our group towards its
goal” even when the goal is clear, there is a flaw in assuming *numbers that can
be readily obtained* are good proxies for *nuanced questions*. The title of this post 
comes from something written by HL Mencken: 
>   **For every complex problem there is  an answer that is clear, simple, and wrong.”, 

we know the answer is wrong but if it’s clear and simple, well, two out of three…    
**Clear simple and wrong answers** *do* seem to be all around. I was recently
committing some code and git told me there were a spectacular number of lines
involved. In fact, the contribution was much more modest than it looked, I had a
test which needed a data file, that file ran to nearly 1000 lines of JSON but was only a
moment’s work to generate. My commit added a PowerShell command to a module and was
accompanied by a markdown help file autogenerated by `PlayPS`, nearly 200 lines,
but only 20 of mine. And even the PowerShell wasn’t especially creative in this
case, mostly *copy and pasted* from two existing commands. I might have put some
effort into making it more efficient, but **if someone is looking at lines of
code committed** (clear and simple) then **why spend more time to commit fewer
lines**? Measuring me on this indicator would push me to do the wrong thing.

If I can *copy and paste* something which already works well enough, why improve it? 
A previous gig produced a system which needed to deliver new information once
per minute and was taking 15 seconds to do it. The project called in an external
expert, who found that instead of joining two database tables in a
query, a piece of code queried `table-1` and for each record returned it made a 
fresh query against `table-2`, replacing one query with hundreds. The owner of the 
code said, “I just copied it” and we found the **harmful pattern** "`For each record in table`" used by multiple
developers in **dozens of places**. It was **not a lack of expertise** 
compared with the external consultant which led team members to do this but **how 
they were told to work**. It's my belief that developers knew this 
should have been fixed but they disincentivized from fixing old technical debt. 

Writing *this* post, I had to unearth the [Wymiwyg article which I wrote *Thirteen* years ago](/misc/2007/09/28/Wymiwyg.html), and it was useful to be reminded of a couple of things I had said in 2007: 

>   [We talk about] X who took action Y because they were given an objective of increasing/reducing metric Z... Action Y made no sense, except to meet a target – one seemingly created by someone who is themselves measured on the number of metrics they create for measuring others.

"Target culture" in the public sector was a hot topic at the time and I mused on whether it meant schools, hospitals or the police did the right thing or chased KPIs and Service Level Agreements I quoted the [Inspector Gadget blog ](https://theinspectorgadget.wordpress.com/)  which was quite withering on SLAs and had this to say on the Police concentrating resources in the wrong places. 

>  Heard of Anti Social Behaviour? or ASB as we call it at the moment? An ASB Hot Spot is NOT a place where there is a lot of ASB. It is a place where loads of people phone the police and SAY there is a lot of ASB, or ONE person phones the police loads of times ...[analysts] get their information from looking at the number of calls the police receive...

Gadget deleted his blog (the link above is a 'best of' site), [his book is still on Amazon](https://www.amazon.co.uk/Perverting-Course-Justice-Hilarious-Shocking/dp/1906308047/) and the WayBack Machine still finds the original. One of that blog's main themes was how an over-simplified view of the world led to policy which made him do the *wrong* things, in this case spending time in \[mis\]designated "Hot Spots" to the detriment of real incidents. Gadget saw a huge difference between what people wanted from the police and what policy caused, and might agree with something I wrote when talking about [the four most dangerous words for any project](/misc/2016/06/26/TechnicalDebt4DangerousWords.html):  *“the best way to get people to deliver what you want is to stop asking them to deliver other things”.*

We know that answers which are too easy to get will probably misinform us, but it's rare that we ask "*Does this indicate what I need to know?*" and "*is it telling people to deliver* "other things", *not what the things that I want?*" - and *that* problem may be older than the balloonist joke.  
