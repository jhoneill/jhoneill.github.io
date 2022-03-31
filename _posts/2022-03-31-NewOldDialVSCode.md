---
layout: post
title:  "New and old Hardware, Windows 11, Visual Studio, Code and the Surface Dial"
date:   2022-03-31
categories: Misc
---

In one of those “doesn’t time fly” moments I realized I’ve had my [Surface dial](https://www.microsoft.com/en-gb/d/surface-dial/925r551sktgn) for four years. (I’m finishing this on my birthday, which makes me feel **old**, and
remembering old things as *innovations* doesn't help). When the dial *was* new I wrote [Using the Surface Dial With Adobe
LightRoom](/misc/2018/01/08/SurfaceDial.html) which is still valid today - although late in 2021
[Adobe posted that from V23 Photoshop would no longer support the surface dial.](https://helpx.adobe.com/photoshop/using/microsoft-dial.html%20from%20V23)

My aging MK I Surface Book had a GPU which no longer meets Photoshop's demands, its CPU won't support Windows 11 either, and the limitations of 512GB of storage made my rejection of the 1TB model look penny-pinching and daft in hindsight. So, on the auspicious-looking UK launch date of 22/2/22, a **new** Surface Laptop Studio arrived with a 2TB drive (lesson learned) and Windows 11 pre-installed.    
Its Pen - which was free with pre-orders - had some kind of shipping mishap, and getting Windows *Pro*, a stack of software and 400GB of photos onto it was a troublesome process and it was 10 days after delivery that I went to apply the same Surface Dial settings I'd used on the Surface Book...

**In Windows 11 there has been a bug known since at least October 2021 that the Settings
app crashes when adding an app in the Wheel settings.** [I found a workaround](https://techcommunity.microsoft.com/t5/windows-11/crash-when-trying-to-quot-add-an-app-quot-in-quot-wheel-quot/m-p/2856345)

>  From [Sonia Cuff](https://techcommunity.microsoft.com/t5/user/viewprofilepage/user-id/170596)    
>  This has been reported as a bug, but the following workaround should help:
>  -   Open **Settings** and browse to **Network & Internet**, then **Advanced network settings**, then **Data Usage**.
>  -   From the Data usage screen, type "wheel" in the **Find a setting box** (under your profile pic) and choose **Wheel settings**
>  -   Wheel settings should now be stable for this session. If you exit the settings app, you need to repeat these steps

The change from Surface Book to Surface Laptop studio hasn't changed what I wrote in [that 2018 post](/misc/2018/01/08/SurfaceDial.html). I don’t use it for many things, but **the Surface Dial is great
combined with the Pen**; and Adobe apps could *still* benefit from a way to use the pens 'thumb'
button where they demand Alt + click.    
The old Surface Book came with the Surface Pen in the box, but the new Slim Pen2 is (outside of pre-orders)
a £120 extra with a £2879 Laptop Studio, which seems penny-pinching and daft - the
machine has a wireless charging Dock specifically for the pen, which will go unused by those
who baulk at the extra cost. Whether wireless charging is the reason the pen is *so* light and
plasticky is another matter, it lacks the quality feel that the £89 Surface Dial has.

This week I saw [a tweet](https://twitter.com/Spatacoli/status/1508888893118029825) that someone had
written a [Surface Dial extension for Visual Studio](
https://marketplace.visualstudio.com/items?itemName=MadsKristensen.SurfaceDialToolsforVisualStudio) - a great idea - the tweet asked *"Is this the best Surface Dial integration ever?"* , and possibly Adobe have ceded that title.

These days it is so rare that I need the full version of **Visual studio** that
I only have Visual Studio **Code** on the new machine, I don’t know if *Code* will
support an equivalent extension but for now I *do* know how to add support for
anything that works through key strokes. Below is what I have added for Visual Studio Code in
Wheel settings – just one menu for now.

![Wheel Settings for debug in VS CODE](/assets/VSCodeWheel.png)

The odd thing is that when combined with the pen I use the dial in my left hand, and with VS code debugging it takes the place of the mouse in my right. With a brush, the wheel does *larger* / *smaller* with no action on click, elsewhere in Lightroom click does a *select* of some kind. What is so clever about the dial is "Advance"/"Retreat", (whether that's zooming, scrolling, or stepping through code) and "More"/"Less" (whether that is audio volume, screen brightness, brush size, or slider value) both naturally translate into rotary motions with some action (like "mute" or "process with the what is selected") as the click action.
I find debugging actions are mostly step-over, step-into and step-back-out, which translate well into the dial's into *forward* / *take- action-on* and *go back*. I can see uses in things like search, where the three might be either "Go to Next match", "change", "go to previous match" Or "Change and go to next", "Skip" and "Undo and go back". As with other things in Code, so many choices to explore.