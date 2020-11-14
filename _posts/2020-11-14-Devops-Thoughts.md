---
layout: post
title: Some DevOps thoughts for the weekend
date:   2020-11-14
categories: Misc
tags: 
    - Programming
    - DevOps
---

A little while ago one someone on an internal discussion asked a question "How
do you explain and sell DevOps to Operations people in your part of the
business". And someone else said "...to anyone". I talked about how I saw it
they said, "Can you write it down?" and when I shared it the response was "that
looks like a blog post". So cleaned up a bit, and without internal specifics
here are those thoughts.

Developer-envy
--------------

IT operations people often have a problem unknown to developers: lack of
“situational awareness” it might be any combination of:
-   A lack of clarity what we are delivering, to whom and why.
-   Uncertainty over the components (machines, networking, application services)
    needed to deliver it.
-   Lack of understanding of what state those components are in or how they got
    to be in that state.

When your car is serviced, there is a checklist of what should be done, and you
get a copy with the items marked as done and any comments. But IT infrastructure
can be handed over with no record of how it was put together. Without an audit
trail, if things go wrong, we need to establish *what* was requested and whether
the configuration matches it, before determining what needs to change to put
things right. Often that fixes the immediate problem, but we can’t tell where
else a latent problem remains. One-off changes cause *configuration-drift*, and
we get the frustrating “This works here, but fails there” class of problem.

**Developers have a set of advantages.**
-   What they are asked to do is more **clearly specified.**
-   Their code [usually] **states its dependencies** (all developers learn about
    “it works on my machine” situations where they didn’t know something *was* a
    dependency)
-   **Past versions are saved**, so it is possible to see how code evolved and
    who changed it, and to get back to the way it was a given point in time.

*Versioning* means code has the **audit trail** that infrastructure lacks. If a
developer finds an error in one place, a text search of the code can find other
places it occurs. Specifications tend be more explicit, and it's normal to have
tests which verify that the code complies with them. When errors are found the
specification and the tests may be refined to make the correct behaviour more
explicit.

The DevOps call to *operations people* is simple. **We can do infrastructure
better if we steal from the developers’ toolbox!** We can:
-   Have a better understanding of the deliverable.
-   Record the configuration steps / values to get to the desired state.
-   Follow the steps and confirm they have achieved that state.

Specifications and the steps to get there can (and should) be *subject to
version control, just like code is*. When we change the configuration, we also
update our specification and steps to reflect the change. This is the vital
piece: *every change gets recorded*, we capture *what* changed, *when* and
(hopefully) why / on whose say-so.

Define, automate, and audit.
----------------------------

We could use pen and paper to record the requirement, the steps to meet it, and
progress against those steps. The goal is being able to show that “A” is as
requested, that “B” matches “A” and later when we add “C” it should match both.
Even as a paper process it should
-   Remove the dependency on “the one person who knows” which:
    -   is more efficient (anyone can find out what the configuration should be)
    -   reduces disruption if a person is lost (even to sickness or holiday)
-   Have fewer errors (everyone follows a set of steps which work)

-   Be easier to change (if a change to “A” works, we can be confident about
    applying it to “B” and “C”)

Recording it and using version control means *treating infrastructure in the
same way as you treat code* but when people talk of “infrastructure as code”
they mean something more: recorded configuration instructions lend themselves to
**automated configuration**. Any infrastructure group’s culture must **prefer
automation,** *no-one* should default to connecting to servers and making individual
changes by hand (whether on Linux via SSH or on Windows typing into remote a
PowerShell session, using RDP to the console and running GUI management tools,
or running those same tools locally against a remote server). Again, this is
something where infrastructure can look enviously at development: hand fetching
components, compiling files individually, running “link” or “make” tasks on the
object code and massaging the executables into something which can be taken
somewhere else to run have all become steps in a *build;* the developer just builds
what they have written. Automation translates to *build infrastructure in the
same way as you build code.*

On Culture
----------

Since I mention culture, a culture of “State what is to be done, do it, show it
has been done” should ensoure that the work *done* is the work *expected*. 
**Automated processes eliminates a whole class of errors** by embodying the first 
two steps, and their output shouldn’t simply be “A task ran”; a process can *check* 
as well as *set* items, so **automated processes should incorporate testing** / 
validation of the result - the third of those steps. There should always be an 
audit trail whether it comes from a human ticking off a check list or a machine 
writing a file listing the states of various parts.

The benefits of DevOps - efficiency, resilience, and responsiveness are easy to
sell to management. The ability to audit is also a benefit and some areas like
implementing security are “applied auditing” – what has been left open (and why)
and are the things which *should be* closed *actually* closed? Some
infrastructure people don’t like writing down what they know (unlike
programmers, whose job is to do exactly that), some see their value as being the
person who carries out the task (even if it mostly clicking “next”). But did you
ever meet a programmer whose pride was in knowing how to fetch dependencies and
compile, rather than write the code? No? I haven't either. Some teams cling to
doing these grotty tasks and can’t get out of *manually making unrecorded changes
to achieve vague goals*. **Culture is the main blocker for DevOps.**

I’ve likened some projects I’ve been brought into to a flight where no-one knows
which passengers boarded the plane, the cabin-crew have said they are going to
London-Gatwick, but the pilots can’t agree if they are landing at
London-Heathrow from the east or the west, and don’t know their position or how
much fuel they have. Culture does that. An aviation culture says (among other
things) you always have manifest for who and what is on-board, you file a flight
plan and check progress against it; the crew has a simple shared goal (get
somewhere, safely, in reasonable comfort, on-time) and knows which of them is
responsible for any aspect. An airline whose culture was the opposite of any of
those would court disaster. **Bringing DevOps to a group sometimes needs a
transformation of culture in order to succeed.**

Teamwork
--------

For now, assume management is working to foster the right culture.  
I have two interesting employers on my CV: the Mercedes Formula One team and
Microsoft; and I sometimes say that *you only know teamwork if you have worked
in a sports team*. Team goals are clear and shared: for Mercedes F1 it was “To
win multiple championships”. Everyone knows how they contribute, directly or
indirectly - Mercedes talk of "Win together or lose together", nobody wins if
the car breaks down; nobody loses if there are trophies in reception on Monday
morning. Microsoft is the opposite - its diverse businesses give it set of goals
which are incomprehensible even to people who work there. Non-cooperating
factions arise, and Microsoft doesn't do "togetherness".

Where one group operates what the other has developed it is worth asking if they
look like Microsoft, lobbing things between ivory towers or do they have the
unity of Mercedes? The latter sees a simple goal of taking some requirement and
meeting it with something in production. Planning, writing software, testing it,
building the infrastructure to run it, deploying to that infrastructure, and
maintaining the result all contribute to that goal – everyone understands
their relationships with the other parts.

A large part of “Doing DevOps” is creating more of a “one team” culture: in a
“siloed” world, developers create something, and curse operations when it isn’t
running – either not deployed or broken. Operations look at what has been
“thrown over the wall” and curse development for expecting them make such a
thing work at all, never mind keep it working. I said problems stem from people
not understanding the deliverable, what’s used to deliver it or how it got to
where it is – these can be symptoms of people working in siloes.

For developers, the pitch for DevOps is simple – less time spent sorting out
deployment problems (either rework or diagnosing operations’ problems), and
shorter time to production. To operations, the pitch is fewer deployment problems
(reconfiguration or diagnosing dev issues) and for management it is speed and
efficiency.

This means some **shared tooling**:
-   Can everyone see what others are working on, and plan their work
    accordingly?
-   Can development see how an environment is built and take that it into
    account or request changes?
-   Can a single pipeline join up the stages of automatically building the
    code, deploying it for production or test, validating what has been done and
    sharing the results?

That single pipeline is the Continuous Integration / Continuous Delivery idea which is 
widely talked about. If deployments are **efficient** (and an automated deployment can run 
in less time than it takes to find the person who knows how to do some arcane piece of 
configuration) and **reliable** – traits management like - then they can be done **more often, 
with smaller changes**. That means shorter time to production (good for developers, and 
for the customer) and easier recovery when things do go wrong (good for operations). 
CI/CD can also give us a world known as *continuous deployment*. Instead of “big bang” 
updates with many changes, each feature or fix *can* go into production as it is ready, 
and the goal is to be able to do that, even if we choose to batch changes at intervals.
I like to talk of **“Deploy at will”** we don’t have to deploy *every* day, but we can
deploy on *any day we choose*.

Microsoft use a definition *“DevOps is the union of people, process, and
products to enable continuous delivery of value to our customers.”*. **Value**
comes in the form of something which **meets a need** being **in production.**
Without the right culture and processes continuous delivery can’t happen and
products exist to support those processes. I’d reverse the wording. “A model
which allows new and updated IT services to be deployed at will needs the
combination of culture, process and enabling products known as ‘DevOps’.”
