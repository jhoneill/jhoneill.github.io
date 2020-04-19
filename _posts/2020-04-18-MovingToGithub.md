---
layout: post
title:  "Moving My blog to github"
date:   2020-04-18
categories: Misc
tags: 
    - Blogging
    - GitHub
    - Jekyll
    - Visual Studio Code
---
Self-evidently this blog is now at github.io; and I may point one of the
domains I own at it in due course. I've moved some recent posts and things that they
link to over to the new platform and will continue to backfill with interesting
posts which seem worth keeping. The the old blog address
[https://jamesone111.wordpress.com](https://jamesone111.wordpress.com) **will** stay
online. Microsoft have kept the old posts I wrote while I was there,
although they have been rehomed, so content, however badly proof-read, sticks
around forever; there's a special kind of embarrassment in finding a spelling or
editing error which has been on-line for a decade.

**Better proofing is one reason** for moving. I’m typing in **Word**, with the
[Writage Extension](http://www.writage.com/), so, I get all the benefits of
working in my everyday word processor. Then I save-as **markdown**. Somewhere
markdown has just become the easy way to put basic pages together.
Twenty-something years of doing raw HTML when I need to hasn’t gone to waste,
because, when I have what I want to say, I move over to **Visual Studio Code** to
finish the markdown. Code **drives git for me** and **Uploading to GitHub was
the second reason** for moving. I have been using *Live writer* for ages, and it felt
 clunkier as time passed, but finally logon to WordPress broke, I think
Writer uses obsolete levels of security which WordPress have now turned off.
Inserting formatted PowerShell into Writer was one of the last things where I still needed Windows PowerShell's ISE,
I **was using a 2009 version of IsePack,** to copy highlighted text as HTML it’s
cumbersome - needing a switch from a WYSIWYG editor to raw HTML and back again; now I can **work on a post and a script in the same editor**, and
just mark it to be _code_ or better, to be **highlighted as PowerShell** (or whatever else I need).
And the last reason for moving was I got **tired of the junk WordPress adds** to my pages;
use their platform and accept their junk, I guess. Or move. I like the new look so
far.

**GitHub automatically publishes from a repository to github.io.** All I needed to do was **add a
repo** named \<\<mygithubUsername\>\>.github.io and, out of sight, **Jekyll**
does the rest. There is plenty to read on-line about Jekyll: installing it
_locally_ means you can build the same site locally that will get built on GitHub,
test it with a simple local web-server and upload when you’re happy with how it
looks.    
I followed 
[the instructions I found here](https://martinbuberl.com/blog/setup-jekyll-on-windows-and-host-it-on-github-pages/)
including his steps for setting up Ruby (you don’t need the full the development
environment), and editing the `\_config.yml file` –**filling in obvious
placeholders** in there.

Editing in VS Code, it made sense to be able call Jeyyll to process everything and start the web-site. I don't often use the "Terminal/Build" menu option in VS code but I setup a command for this with a [tasks.JSON file](https://go.microsoft.com/fwlink/?LinkId=733558):   
{% highlight json%}
  {
    "version": "2.0.0",
    "tasks": [
        {
            "label": "jekyll serve",
            "type": "process",
            "command": "C:\\Ruby27-x64\\bin\\ruby.exe",
            "args": [
                {"value": "C:\\Ruby27-x64\\bin\\jekyll"},
                {"value": "serve"}
            ],
            "problemMatcher": [],
            "group": {
                "kind": "build",
                "isDefault": true
            }
        }
    ]
  }
  
{% endhighlight %}
And as you can see, Jekyll can do **syntax formatting for many file types**. And I will need to wrap a lot of my samples in     
<code>&#123;% highlight powershell %&#125; ... &#123;% endhighlight %&#125;</code> so I added **snippets** to VS code for **H**igh **L**ight **P**owershell and **E**nd **H**igh **L**ight,

<code>
{<br/>
&#160;&#160;&#160;&#160;"highlight PS": {<br/>
&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;"scope": "markdown",<br/>
&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;"prefix": "hlp",<br/>
&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;"body": [<br/>
&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;"&#123;% highlight powershell %&#125;"<br/>
&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;]<br/>
&#160;&#160;&#160;&#160;},<br/>
&#160;&#160;&#160;&#160;"end highlight": {<br/>
&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;"scope": "markdown",<br/>
&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;"prefix": "ehl",<br/>
&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;"body": [<br/>
&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;"&#123;% endhighlight %&#125;"<br/>
&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;]<br/>
&#160;&#160;&#160;&#160;}<br/>
}
</code>

Next, I had a browse through <http://jekyllthemes.org/> for a **theme** and
settled on [Monochrome](https://dyutibarma.github.io/monochrome). I’ve made some
small customizations to it, changing how links render and changing the `syntax.scss` to **colour
different elements of PowerShell correctly;** much back-and-forth was needed
between the developer tools in the browser finding which *class* had been
selected for an element, a paint program to pick up the *colour* used in VSCode
with the ISE colour scheme, and the CSS file to put that colour in for that *class*.

Monochrome has its own **shortcut icon** and I decided I’d use the little avatar
I drew years ago at <https://www.sp-studio.de/> so I dropped that in and edited
the `head.html` file to point to it. SP Studio is still on-line and is being
ported off Flash, and it served as a reminder that I should make a donation to
keep it going.

I didn’t yet have the **sharing links** at the bottom of the page and a bit more
searching brought me to another [great set of
instructions.](https://sourabhbajaj.com/blog/2017/10/29/adding-social-media-share-icons-to-jekyll/)
I added some extra links to the resylting include file, using a couple of tips from [a rather older
page](http://erjjones.github.io/blog/How-I-built-my-blog-in-one-day), including
the code for Previous page / Next page and the idea of using **GitHub Issues for
blog feedback.**

As I was building these, I was moving about three dozen posts over, converting
about 45,000 words to markdown (maybe I need to write shorter ones) and figuring out where actual HTML was needed. Jekyll can support
‘ordinary’ html, so I could have kept the original files, but it was a quick way
to understand what the platform could do, and how bits worked. Whatever the format the last step is to give each one a header to tell Jekyll how to process the page including the go-live date, which is used in the final URL – the markdown for _this_ page begins (and ends) with this:
```
---
layout: post
title:  "Moving My blog to github"
date:   2020-04-18
categories: Misc
tags: 
    - Blogging
    - GitHub
    - Jekyll
    - Visual Studio Code
---
```
