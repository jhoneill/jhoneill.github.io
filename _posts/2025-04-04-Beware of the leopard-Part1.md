---
layout: post
title: Beware of the leopard – part 1 the quest for charge-point data
date:   2025-04-04
categories: Stable4Days
---

At the start of Douglas Adams’ Hitch-Hikers Guide to the Galaxy, someone
explains that to find some information he had to go to a basement which was
lacking both light and stairs, where it was *“In the bottom drawer
of a locked filing cabinet in a disused lavatory with a sign on the door saying,
‘Beware of the leopard’”*. My experience  getting data covered by the UK's *Public Charge-Point
Regulations* leaves me feeling  **he had it easy**.

The regulations arrived in 2023, they specifically excluded private 
or workplace chargers and gave Charge Point Operators (CPOs) a year to meet several criteria:

-   Chargers must display pricing per kWh and above 8kW they must accept contactless payments (below 8KW *can* require an app). From November 2025 they must also support at least one roaming payment provider.
-   Networks must achieve 99% availability, and operators must submit annual up-time reports.
-   Operators needed to confirm by December 2024 that they had a free 24/365 helpline and then report calls quarterly.
-   Operators must “ensure that reference data and availability data is made available to the public free of charge”; it must be stored in a standard format, [OCPI
    2.2](https://evroaming.org/ocpi-downloads/)

I'm interested in the data rather than other compliance issues with the others. 
A lot of it is static *reference* data - in descending order of importance to the driver:
*where* can we go to charge, *when* each site is open, the *power* available (even if
 we can't use the headline speed), *payment methods* and finally *who* operates it.    
*Pricing* may have peak / off-peak rates and *availability* will change as other drivers come and go or faults occur. 

I expected to go to operators’ websites, find a “how to call our API” page, write a
few lines of PowerShell and have the data - **how hard could it be?** Well...

In early 2025 there are lots of apps to tell drivers about chargers, many linked 
to an operator or payment provider. Some seem to have been written without 
asking, “**How is this useful?**” before making another 
version of “*Google maps showing some chargers*” and/or “*A more awkward way to pay*”. Vendors want me to install their 
app when I just want to see a complete list of options in one place. *Completeness* makes or 
breaks some analysis: a sample lets you say “Rapid changers are busiest at *this* time”,
but saying that [A quarter of England’s strategic A-roads have electric car
charging ‘cold spots’](https://www.theguardian.com/environment/2025/mar/09/england-a-roads-electric-car-charging-cold-spots)
could report *data* gaps as charger gaps. But whether the goal is an app or data analysis,
anyone wanting data from many operators needs to aggregate it for themselves

In hindsight this might have flagged a “Beware of the leopard” experience. So might
the absence of a definitive list of CPOs, but I just started making my own. 
One of the first open-data pages I found was [Fastned's](https://www.fastnedcharging.com/en-gb/uk-open-data). It 
walked me through getting and using an API-key and I had their data in a couple of minutes. 
A CPO with fewer than 50 locations making things easy seemed a good start; keys are a minor nuisance, 
but they’re a first line of defence against things like denial-of-service attacks.    
Next came [Mer](https://uk.mer.eco/live-charge-point-data/) whose link didn't need a key. BP pulse had an [access request from](https://www.bppulse.co.uk/help-centre/charging/faq/public-charge-point-regulations/open-data) and after a while I found [one for Shell](https://www.shell.co.uk/electric-vehicle-charging/shell-recharge-open-data-request.html#iframe=L2Zvcm1zL2VuX3VrX3BjcHJfZGF0YV9yZXF1ZXN0).
Both forms went to [eco-movement](https://www.eco-movement.com/), a Dutch company, who
replied with keys and a short how-to guide, all quick, professional and easy to code because
different keys with the same request URL returned different operators’ data. 

After a third form went to them, I asked if eco-movement would tell me who else they provided this service for. No, they said, that would break
client confidentiality. The following isn't verbatim, but the gist of the exchange that followed.   
&nbsp;&nbsp;&nbsp;&nbsp;“*But this data is – you know* – public - *there’s no need to make a secret of meeeting a legal requirement*” I argued.  
&nbsp;&nbsp;&nbsp;&nbsp;“*Not really, it’s available to the public if they ask for it which isn’t quite the same*”  
&nbsp;&nbsp;&nbsp;&nbsp;“*But you’ll give me data if make the right requests?*” I said, trying another tack   
&nbsp;&nbsp;&nbsp;&nbsp;“*Of course, operators pay us to do that!*”  
&nbsp;&nbsp;&nbsp;&nbsp;“*So, what requests can I make?*” - they weren’t so easily outflanked  
&nbsp;&nbsp;&nbsp;&nbsp;“*Sorry, we can’t tell you*”.

I wasn’t sure if I was in that Douglas Adams scene, something more Kafkaesque or
perhaps in the *Yes, Prime Minister*, episode where he complains “*I don’t* know *because I can’t find what questions to* ask *you, and I don’t know what to ask
you* because *I don’t know*”.

One CPO’s form for *eco-movement* had a box to fill in for the operator-name, so I
tried entering other names, I got one success, three failures and a query about what I was playing at. A few more of their customers have emerged since.

Some CPOs don't publish Open-Data access details, so I had to contact them. 
Three admitted missing the deadline and lacking a data-feed. Others replied promptly, I thanked **Osprey Charging**, for 
being quick and they expressed further support, having been involved in the PCP Regs 
consultation they helped my understanding and they have an [interesting white paper on costs](https://www.ospreycharging.co.uk/post/cost-of-public-rapid-ev-charging) too. Sainsbury's Smart Charge had limited contact options, but had me accessing data with 
25 minutes of calling them - not just exceeding expectations but setting the fastest response record. 

**Others were slower** but eventually provided access or a form to request it. Only one,
[Believ](https://www.believ.com/believ-charge-point-live-data/), are **actively
obstructive**, their API is rate-limited to 1 single-point request per second, and point IDs aren't made available.
I had a *Teams* call with three of their senior people, who were firm that they felt sharing what others shared would harm their business. 
I found a workaround but getting points one by one is *slow*.

Some CPOSs even major ones, **simply ignore requests**, those in my bad books 
include **Instavolt**, **Motor Fuel Group**, and **Tesla** - all major players
in rapid DC charging and **char.gy** and **Total**  among AC point operators. 
Calls to **Instavolt** yeilded promises to resolve things which weren't kept.  
A few days before posting this their CEO took an interest via linked-in, but things haven’t changed yet.. 

Some operators see sharing data as boosting the market and beneficial to them,
others seem to see it as beneficial to their competitors or a source of bad PR. 
As with apps, each operator's data is like a jigsaw piece, only interesting when combined with the others, and the whole is compromised if their piece is missing. I tried to replicate the "cold spots" finding even without the CPOs I mentioned
![UK Map Showing major roads and Chargers](/assets/chargePointDistribution.jpg) 
A CPO might wonder “which of those are *ours*”, but a driver thinks “Hey I use *that* road!”. If I say, wrongly, “There are no chargers on the road from A to B” the *error* matters more than *whose* charger is on that route. 
And a charge-point of Believ’s showing there *is* one on the road from Y to Z doesn't hurt them. 

It started to become clear that simply having a regulation didn’t mean the data 
would appear and I would need to talk to some government people about the
process. And that will be the next post, but for now here's my progress list - which includes some of the fixes the data needs,  and those who will end up as a discussion with the regulator.

<center>
<table cellspacing="0" cellpadding="10" border="1"><tbody>
<tr><td><b>AlfaPower</b></td><td>Website broken for weeks ? Link on Zap map goes back to CPO list</td></tr>
<tr><td>Allego</td><td>Need to fix missing power values from Volts and amps</td></tr>
<tr><td>Applegreen Electric</td><td>Via eco-movement leaves party ID blank</td></tr>
<tr><td>Be.EV</td><td>Via Fuuse</td></tr>
<tr><td><b>Believ</b></td><td>They don't think they need to supply  a full list</td></tr>
<tr><td>Blink charging</td><td>Via eco-movement</td></tr>
<tr><td>BP pulse</td><td>Noticed location errors (chepstow) need to fix missing power values from Volts and amps</td></tr>
<tr><td><b>charg.gy</b></td><td>helpdesk say that <i>"All data must be made publicly available free-of-charge"</i> means "that we do not have an obligation to release this information,"</td></tr>
<tr><td>Chargeplace Scotland</td><td>Need to fix missing power values from Volts and amps same server as evolt and Pogo</td></tr>
<tr><td>ChargePoint</td><td>Via eco-movement</td></tr>
<tr><td>Clenergy</td><td>Need to fix missing missing operator and power values from Volts and amps</td></tr>
<tr><td>Community by Shell Recharge</td><td>Via eco-movement leaves Party ID blank need to fix missing power values from Volts and amps</td></tr>
<tr><td><b>ConnectedKerb</b></td><td>Solution delayed - they are talking to the regulator</td></tr>
<tr><td>Dragon</td><td>Noticed website matched cleanergy, used different server and got access</td></tr>
<tr><td><b>Electroad</b></td><td><b>No answer</b></td></tr>
<tr><td><b>ESB energy</b></td><td>OCPI "Pending" supplied CSV format location data</td></tr>
<tr><td><b>EVDot / bmmnetworks</b></td><td>Web site has links but they're broken</td></tr>
<tr><td>EVolt</td><td>Noticed location errors (Stortford) Same server as chargeplace Scotland and Pogo</td></tr>
<tr><td><b>Evyve</b></td><td><b>No Answers</b></td></tr>
<tr><td>EZ-charge</td><td>API Key</td></tr>
<tr><td>Fastned</td><td>Online signup for API Key</td></tr>
<tr><td>ForEV</td><td>Via Fuuse</td></tr>
<tr><td><b>Fuuse</b></td><td><b>No Answer</b></td></tr>
<tr><td>GeniePoint</td><td>Need to fix Missing operator</td></tr>
<tr><td>Gridserve</td><td>Own form for APIKey. Use non standard text with underscores for "OutOfOrder"</td></tr>
<tr><td><b>Instavolt</b></td><td>No reply to mails to , phoned "We don’t know". CEO invovled. <b>Still blocked</b></td></tr>
<tr><td><b>Ionity</b></td><td>Via Eco movement - No Tariff or capability info</td></tr>
<tr><td>Mer</td><td>Single file download, no API Key</td></tr>
<tr><td><b>MFG</b></td><td><b>No public access</b>, peer-peer sharing only</td></tr>
<tr><td>Osprey Charging</td><td>Own API Key on request. Max Power field not present</td></tr>
<tr><td>Pod Point</td><td>Via eco-movement</td></tr>
<tr><td>Pogo</td><td>Same server as ChargePlace Scotland and EVolt</td></tr>
<tr><td><b>RawCharging</b></td><td><b>No answer</b></td></tr>
<tr><td>Roam Charging</td><td>Has versions of OCPI field names need to fix power, operator, party, Tariffs, names</td></tr>
<tr><td>Sainsbury's Smart Charge</td><td>Phone contact only. Tariffs need a fix</td></tr>
<tr><td>Shell Recharge</td><td>Need to fix missing power values from Volts and amps</td></tr>
<tr><td><b>SureCharge</b></td><td><b>No answer</b></td></tr>
<tr><td><b>Tesla</b></td><td><b>No answer</b></td></tr>
<tr><td><b>Total</b></td><td><b>No answer</b></td></tr>
<tr><td>Ubitricity</td><td>Via eco-movement</td></tr>
<tr><td><b>Urban fox</b></td><td>OCPI "Pending" supplied CSV format location data</td></tr>
<tr><td>WattIf</td><td>Tariff data strangely formatted need to fix missing power values from Volts and amps</td></tr>
<tr><td><b>Weev</b></td><td><b>No Answer</b></td></tr>
<tr><td>Wenea</td><td>Supplied as 2 files in a .zip</td></tr>
<tr><td><b>Zest</b></td><td>OCPI "Pending" supplying snapshots on demand</td></tr>
</tbody></table>
</center>