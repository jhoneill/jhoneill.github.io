---
layout: post
title: Don’t Swallow the cat. Doing the right thing with software development and other engineering projects.
date:   2012-04-22
categories: Misc
tags: 
    - Management
    - Project Management
    - Software Engineering
    - Technical Debt
---

In my time at Microsoft I became aware of the saying “**communication occurs
only between equals**.” usually couched in the terms “People would rather lie
than deliver bad news to Steve Ballmer”. Replacing unwelcome truths with
agreeable dishonesty wasn’t confined to the CEO's direct reports, and certainly
isn’t a disease confined to Microsoft. I came across ‘The Hierarchy of Power
Semantics’ more than 30 years ago when I didn’t understand what was meant by the
title; it was written in the 1960s and if you don’t recognise *“In the "beginning
was the plan and the specification, and the plan was without form and the
specification was void and there was darkness on the face of the implementation
team”* the see [here](http://www.wussu.com/humour/semantic.htm) – language warning
for the easily offended.

[Wikipedia](http://en.wikipedia.org/wiki/Celine%27s_laws) says the original form
of “communication occurs only between equals” is **Accurate communication is
possible only in a non-punishing situation**. There are those who (consciously
or not) use the impossibly of saying “No” to extract more from staff and
suppliers; it can produce extraordinary results, but sooner or later it goes
horribly wrong. For example, the Challenger disaster was caused by the failure
of an ‘O’ ring in a solid rocket booster made by Morton Thiokol. [The engineers
responsible for the booster were quite
clear](http://en.wikipedia.org/wiki/Roger_Boisjoly#Challenger_disaster) that in
cold weather the ‘O’ rings were likely to fail with catastrophic results. NASA
asked if a launch was OK after a freezing night and, fearing the consequences of
saying “No”, managers at Morton Thiokol over-ruled the engineers and allowed the
disastrous launch to go ahead. Most people can think of some case where someone
made an impossible promise to a customer, because they were afraid to say no.

Several times recently I have heard people say something to the effect that
‘**We’re so committed to doing this the wrong way that we can’t change to the
right way**.”.Once the person saying it was me, which was the genesis of this
post. Sometimes, in a software project because saying to someone – even to
ourselves – “We’re doing this wrong” is difficult, so we create work rounds. The
odd title of this post comes from a song which was played on the radio a lot
when I was a kid.

>   *There was an old lady, who swallowed a fly, I don’t know why she swallowed a fly.*     
>   *I guess she’ll die.*
>
>   *There was an old lady, who swallowed a spider that wriggled and jiggled and ticked inside her.*     
>   *She Swallowed the spider to catch the fly … I guess she’ll die*    
>
>   *There was an old lady, who swallowed a bird. How absurd to swallow a bird.*    
>   *She swallowed the bird to catch the spider … I guess she’ll die*
>
>   *There was an old lady, who swallowed a cat. Fancy that to swallow a cat.    
>   She swallowed the cat to catch the bird … I guess she’ll die*
>
>   *There was an old lady, who swallowed a dog. What a hog to swallow a dog.   
>   She swallowed the dog to catch the cat … I guess she’ll die*
>
>   *There was an old lady, who swallowed a horse. She’s dead, of course*

In other words, **each cure needs a further, more extreme cure**. In my case the
“fly” was a simple problem I’d inherited. It would take a couple of pages to
explain the context, so for simplicity it concerns database tables, and the
“spider” was to store data de-normalized. If you don’t spend your days working
with databases, imagine you have a list of suppliers, and a list of invoices
from those suppliers. Normally you would store an ID for the supplier in the
*invoice* table, and look up the name from the *supplier* table using the ID.
For what I was doing, it was better to put the supplier name in the invoices
table, and ignore the ID. All the invoices for the supplier can be looked up by
querying for the name. The same technique applied to products supplied by that
supplier: store the supplier name in the *product* table, look up products by
supplier name. This is **not because I didn’t know any better**, I had [database
normal forms](http://en.wikipedia.org/wiki/Database_normalization#Normal_forms)
drummed into me two decades ago. To stick with the metaphor: I know that, *under
normal circumstances*, swallowing spiders is bad, but faced *with this specific
fly* it was demonstrably the best course of action.

At this point someone who *could* have saved me from my folly pointed out that
supplier names had to be editable. I protested that the names don’t change, but
*Amalgamated Widgets* did, in fact, become *Universal Widgets*. This is an issue
because *Amalgamated* not *Universal* raised the invoices in the filing cabinet so
matching them to invoices in the system requires preserving the name *as it was
when the invoice was generated.* “See, I was right name should be stored” –
actually, this exception doesn’t show I was right at all, but on I went. On the
other hand, all of Amalgamated’s products belong to Universal now. Changing
names means doing a cascaded update (update any product with the old company
name to the new name when a name changes) the real case has more than just
products. If you’re filling in the metaphor you’ve guessed I’d reached the point
of figuring out *how to swallow a bird*. Worse, I could see another problem
looming (anyone for Cat ?): changes to products had to be exported to another
system, and the list of changes had their own table requiring cascaded updates
from the cascaded updates.

One of the great quotes in *Macbeth* says *“I am in blood stepped in so far that
should I wade no more, Returning were as tedious as go o’er.”* he knows what he’s
doing is wrong, but it is as hard to go back (and do right) as it is to go on.
But that is not the case - the solution is not to swallow another couple more spiders and
a fly, the solution is to swallow a bird, then a cat and so on. The dilemma is
that the effort for each additional work-round is smaller than the effort to go
back fix the initial problem and unpick all the work-rounds to date. **The easy solution is to choose the course which needs
the least effort now**. The sum of effort required for future work-rounds is
greater, but we can **discount effort that isn’t needed now**. Only in a
non-punishing situation can we tell people that progress must be halted for a
time to fix a problem which has, so far, been mitigated. Persuading people that
such a problem needs to be fixed *at all* isn’t trivial, I heard this quote in a
[Radio programme](http://www.bbc.co.uk/programmes/b00v1qrz) a while back

>   *“Each uneventful day that passes reinforces a steadily growing false sense
>   of confidence that everything is alright: that I, we, my group must be OK
>   because the way we did things today resulted in no adverse consequences.”*

(The quote's author, Scott Snook, was writing about *normalization of deviance*,
that’s how Nasa came to lose a Space Shuttle, and in the case he was talking
about, how the military built up to a “friendly fire” disaster.)  
The problem that spawned this post was being fixed at the time of writing, but in how many
organisations is it a *career limiting move* to tell people that something which
has had no adverse consequences to date must be fixed?
