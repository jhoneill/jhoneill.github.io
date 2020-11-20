---
layout: post
title: Technical Debt and the four most dangerous words for any project.
date:   2016-06-26
categories: Misc
tags: 
    - Management
    - Project Management
    - Software Engineering
    - Technical Debt
---

I’ve been thinking about **technical debt**. I might have been trying to avoid
the term when I wrote [Don’t swallow the
cat](/misc/2012/04/22/DontSwallowTheCat.html),
or more likely I hadn’t heard it back then, but I was certainly describing it –
to adapt [Wikipedia’s definition](https://en.wikipedia.org/wiki/Technical_debt) 
it is *“the future work that arises when
something that is easy to implement in the short run is used in preference to
the best overall solution*”. However, it is *not* confined to software
development as Wikipedia suggests.

“Future work” can come from bugs (either known, or yet to be uncovered because
of inadequate testing), design kludges which are carried forward, dependencies
on out of date software, documentation that was left unwritten… and much more
besides.

The cause of technical debt is simple: **People won’t say “I (or we) cannot
deliver what you want, properly, when you expect it”.**

“When you expect it” might be the end of a Scrum Sprint, a promised date or
“right now”. We might be dealing with someone who asks so nicely that you can’t
say “No” or the powerful ogre to whom you dare not say “No”. Or perhaps
admitting “I thought I could deliver, but I was wrong” is too great a loss of
face. There are many variations.

I’ve [written before](/misc/2007/09/28/Wymiwyg.html) about “**What you measure is what you get**” (WYMIWIG) it’s
also a factor. In IT we measure success by what we can see working. Before you
ask, “How else do you judge success?”, **Technical debt is a way to cheat the
measurement** - things are seen to be working *before all the work is done*. To
stretch the financial parallel, if we collect full payment without delivering in
full, our accounts must cover the undelivered part - it is a liability like
borrowing or unpaid invoices.

Imagine you have a deadline to deliver a feature. (*Feature* could be a piece of
code, or an infrastructure service however small). *Unforeseeable* things have
got in the way. You know the kind of things: the fires which apparently only you
know how to extinguish, people who ask “Can I Borrow You”, but should know they
are jeopardizing your ability to meet this deadline, and so on. Then you find 
that doing your piece *properly* means fixing something that’s
already in production. But doing that would make you miss the deadline (as it
is, you’re doing less testing than you’d like, and documentation will have to be
done after delivery). So, you work around the unfixed problem and make the
deadline. Well done!

Experience teaches us that **making the deadline is rewarded**, even if you
leave a nasty surprise for whoever comes next - they must make the fix AND
unpick your workaround. If they are up against a deadline, they will be pushed
to *increase* the debt. You can see how this ends up in a spiral: like all debt,
unless it is paid down, it increases in future cycles.

[The Quiet Crisis unfolding in Software
Development](https://medium.com/@billjordan1/the-quiet-crisis-unfolding-in-software-development-cffbdafbf450#.lg0eka1ra)
has a warning **to beware of high performers**, they may excel at the *measured*
things by cutting corners elsewhere. It also says **watch out for misleading
metrics** – only counting “features delivered” means the highest performers may
be leaving most problems in their wake. Not a good trait to favour when
identifying prospective managers.

Sometimes we *can* say “We MUST fix this before doing anything else.”, but if
that means the whole team (or worse its manager) can’t do the thing that gets
rewarded then **we learn that trying to complete the task properly can be
unpopular, even career limiting**. Which isn’t a call to do the *wrong thing*:
some things *can* be delayed without a bigger cost in the future; and borrowing
can open opportunities that refusing to ever take on any debt (technical or
otherwise) would deny us. But when the culture doesn’t allow delivery plans to
change, even in the face of *excessive* debt, it’s living beyond its means and
debt *will* become a problem.

We praise delivering on-time and on-budget, but **if capacity, deadline and
deliverables are all fixed, only quality is variable**. Project management
methodologies are designed to make sure that *all* these factors can be varied
and give project teams a route to follow if they need to vary by too great a
margin. But a lot of work is undertaken without this kind of governance.
*Capacity* is what can be delivered *properly* in a given time by the
combination of people, skills, equipment and so on, each of which has a cost.
Increasing headcount is only one way to add capacity, but if you accept that
[adding people to a late project makes it
later](https://en.wikipedia.org/wiki/Brooks%E2%80%99_law) then it needs to be
done *early*. When we must demonstrate delivery beyond our capacity, it is
technical debt that covers the gap.

Forecasting is [imprecise](https://en.wikipedia.org/wiki/Hofstadter%27s_law),
but it is rare to *start* with a plan that we don’t have the capacity to
deliver. I think another factor causes deadlines which *were* reasonable to end
up creating technical debt.

The book [The Phoenix
Project](https://www.amazon.co.uk/Phoenix-Project-DevOps-Helping-Business/dp/0988262509/ref=sr_1_1?s=books&ie=UTF8&qid=1464862347&sr=1-1&keywords=phoenix+project)
has a gathered a lot of fans in the last couple of years, and one of its
messages is that **Unplanned work is the enemy of planned work**. [This time
management piece](http://www.bakadesuyo.com/2015/11/how-to-manage-your-time/)
separates **Deep work** (which gives satisfaction and takes thought, energy,
time and concentration) from **Shallow work** (the little stuff). We can do more
of value by eliminating shallow work and the [Quiet Crisis
article](https://medium.com/@billjordan1/the-quiet-crisis-unfolding-in-software-development-cffbdafbf450#.lg0eka1ra)
urges managers *to limit interruptions* and give people *private workspaces*,
but some of each day will always be lost to email, helping colleagues and so on.

But **Unplanned work is more than workplace noise**. Some comes from [Scope
Creep](https://en.wikipedia.org/wiki/Scope_creep), which I usually associate
with poor specification, but unearthing technical debt expands the scope,
forcing us to choose between more debt and late delivery. But if debt is out in
the open then the effort to clear it – even partially – can be in-scope from the
start.

**Major incidents** can’t be planned and leave no choice but to stop work and
attend to them. But some diversions are neither noise, nor emergency. **“Can I
Borrow You?”** came top in [a list of most annoying office
phrases](https://www.reed.co.uk/career-advice/revealed-10-annoying-office-phrases/)
and “CIBY” serves as an acronym for a class of diversions which start
innocuously. These are **the four dangerous words** in the title.

[The Phoenix
Project](https://www.amazon.co.uk/Phoenix-Project-DevOps-Helping-Business/dp/0988262509/ref=sr_1_1?s=books&ie=UTF8&qid=1464862347&sr=1-1&keywords=phoenix+project)
begins with the protagonist being made CIO and briefed “Anything which takes
focus away from Phoenix is unacceptable – that applies to whole company”. For
most of the rest of the book things *are* taking that focus. He gets to contrast
IT with manufacturing where a coordinator accepts or declines new work depending
on whether it would *jeopardize any existing commitments*. Near the end he says
to the CEO 
>   Are we even allowed to say no? Every time I've asked you to prioritize or defer work on a project, you've bitten my head off. …[we have
>   become] compliant order takers, blindly marching down a doomed path

And that resonates. Project steering boards (or similarly named committees) can assign
capacity to some projects and disappoint others. Without one – or if it is easy
to circumvent - we end up trying to deliver everything and please everyone; “No”
and “What should I drop?” are answers that we don’t want to give - especially to
those who’ve achieved their positions by appearing to deliver everything, thanks
to technical debt.

Generally, *strategic* tasks don’t compete to consume all available resources.
People recognise that these should have documents covering
-   What is it meant to do, and for whom? (the specification / high level design)
-   How does it do it? (Low level design, implementation plan, user and admin guides)
-   How do we know it does what it is meant to? (test plan)

But “CIBY” tasks are smaller, tactical things; they often **lack
specifications**: we steal time for them from planned work [assuming we’ll get
them right first time](https://en.wikipedia.org/wiki/Optimism_bias) , 
but change requests are *inevitable*. Without a spec,
there can be **no test plan**: yet we make no allowance for fixing bugs. And the
work **“isn’t worth documenting”**, so questions must come back to the person
who worked on it. These tasks are bound to **create technical debt** of their
own and they **jeopardize existing commitments** pushing us into more debt.

Optimistic assumptions aren’t confined to CIBY tasks. We assume strategic tasks
will stay within their scope: we set completion dates using assumptions about
capacity (the progress for each hour worked) and about the number of hours
focused on the project each day. [Optimism about
capacity](https://en.wikipedia.org/wiki/Optimism_bias) isn’t a new idea, but I
think planning doesn’t allow for shallow / unplanned work – we work to a formula
like this:

**TIME = SCOPE / CAPACITY**

In project outcomes, debt is a fourth variable and time lost to distracting
tasks a fifth. A better formula would look like this

**DELIVERABLES = (TIME \* CAPACITY) – DISTRACTIONS + DEBT**

Usually it is the *successful* projects which get a scope which properly
reflects the work needed, stick to it, allocate enough time and capacity and
hold on to it. It’s simple in theory, and projects which go off the rails don’t
do it in practice and fail to adjust. *The Phoenix Project* told how failing to
deliver “Phoenix” put the company at risk. After the outburst I quoted above,
the CIO proposes putting everything else on hold, and the CEO, who had demanded
100% focus on Phoenix, initially responds “You must be out of your right mind”.    
**Spoiler**: Eventually he agrees and the book gets to make one of its points through the vehicle of a happy ending.

 The book is trying to illustrate many ideas, but one of them boils down to **“the best way
to get people to deliver what you want is to stop asking them to deliver other
things”.**

Businesses seem to struggle to set priorities for IT: I can’t claim to be an
expert in solving this problem, but the following may be helpful

1. **Understanding the nature of the work**. [Jeffrey
Snover](https://social.technet.microsoft.com/profile/Jeffrey%20Snover%20Windows%20Server)
likes to say, “To ship is to choose”. A late project must find an *acceptable
combination* of additional cost, overall delay, feature cuts, and technical
debt. If you build websites, technical debt is more acceptable than if you build
aircraft. If your project is a New Year’s Eve firework display, delivering
without some features is an option, delay is not. Some feature delays incur
cost, but others don’t.

1. **Tracking all work:** Have a view of what is *completed*, what is *in-progress*, what is “*up next*”, and what is *waiting* to be assigned time. The
next few points all relate to tracking.

1. **Work in progress** has *already consumed effort* but we only get credit when it
is *complete*. An increasing number of tasks marked as in-progress may mean people are
passing work to other team members faster than their capacity to complete it or
new tasks are interrupting existing ones.

1. **All work should have a specification** *before* it starts. Writing
specifications takes time, and “Create specification for X” may be a task in
itself. And yes, *I do know* that technical people generally hate tracking work
and writing specifications.

1. **Make technical debt visible**. It’s OK to *split* an item and categorize part
as *completed* and the rest as something else. Adding the *undelivered* part to
the backlog keeps it as *planned* work, and also gives **partial credit for
partial delivery** – rather than credit being all or nothing. It means some
credit goes to the work of clearing debt. And *I also know* that technical folk see
“fixing old stuff” as a chore, but **not counting it just makes matters worse**.

1. **Don’t just track planned work**. Treat jobs which jumped the queue, that
didn’t have a spec or that displaced others like **defects in a manufacturing
process** – keep the score and try to drive it down to zero. Incidents and
“CIBY” jobs might only be recorded as an afterthought but you want to see where
they are coming from and try to eliminate them at source.

1. **Look for process improvements**. if a business is used to lax project
management, it will resist attempts to channel all work through a project
steering board. Getting stakeholders together in a regular “IT projects meeting”
might be easier but still get the key result (managing the flow of work).

And finally **Having grown-up conversations with customers**. Businesses should
understand the consequences of pushing for delivery to exceed capacity, which
means IT (especially those in management) must be able to deliver messages like
these.

*“For this work to jump the queue, we must justify delaying something else”*

*“We are not going be able to deliver [everything] on time”,* perhaps with a
follow up of *“We could call it delivered when there is work remaining but…
have you heard of technical debt?”*
