---
layout: post
title:  "Why it is better not to use PowerShell Parameter validation"
date:   2011-01-11
categories: PowerShell
---
I was giving a talk shortly before Christmas and I was giving some advice based
on what I had learned writing my PowerShell library for Hyper-V. I said

-   Don’t force user to use an object as a parameter – convert names to objects in your code
-   Don’t force users to expand arrays – expand in your code
-   Don’t automatically punish users if a parameter is empty

The corollary from this is **Don’t be over-proscriptive with parameter checking**, especially when it comes to types – which kicked off an interesting debate. This is best explained with real world examples, so let’s take a simple case from my Hyper-V world , and all the background you need to know is

-   A server contains zero or more Virtual Machines.
-   Virtual Machines can be “Running” or “Stopped” (and in other states)
-   Virtual Machines are represented by VM objects, which have a state property to indicate whether they are running or stopped.

With that in mind I want to look at 3 commands:  
`Get-VM `which returns VM Objects and must, as a minimum, accept parameters of
`-server` to specify where to look for VMs and `–VMName` to filter the selection by
name (in case you don’t know, if there is no other parameter that starts `–VM`
PowerShell will let you abbreviate this as `–VM`)  
`Start-VM` will change the state to running  
`Stop-VM` will change the state to stopped.

Before implementing the commands, one must decide (among other things):
-   What are valid inputs for the -Server and -VMName parameters in Get-VM ?
-   What inputs should start-VM and Stop-VM take.
-   The Output of Get-VM can become the input of Start-VM and Stop-VM. What should happen if no VMs are found on a server ?

It would be a good idea for you to think about how you’d answer these questions
before reading on because I’m going to set out my view here. My view is *right*,
of course, but other views are *not necessarily* wrong.

<u>To me</u>, **flexibility is key.** `Get-VM`, in my view, must allow the person typing the command to specify multiple servers easily. The most obvious example is    
`Get-VM -Server ClusterNode1, ClusterNode2`

If parameter validation says the server name must be a single string, then you
force the user to do something like this    
`"ClusterName1", "ClusterNode2" | foreach-Object {Get-VM –Server $_}`

Not only is the first way shorter but it can be done by a user who has no
PowerShell background. In the same way it should be possible to get those VMs
whose names indicate they are located in particular cities  
`Get-VM -VM "London*" ,"Paris*"`

Yes, I have just sneaked in support for Wildcards. Not allowing this means
forcing the user into something like  
`Get-VM | Where-Object {($_.name –like "London*") –or ($_.name –like "Paris*") }`

This may mean **more work when we implement** the Command (which we do once) **to save work when it is run ** (which happens many times).

What about the case where we run    
`Get-VM -VM "London-DC01" -Server ClusterNode1`   
but `London-DC01` is running on ClusterNode**2** : Should this command return an error?

My (limited) background in databases says that if the query runs successfully
and finds no matching data, “Nothing” is a perfectly valid output, and more
desirable than an exception stopping a script. This begins to answer the
question of what should the input to `Start-VM` and `Stop-VM` be ?.

1.  It would be illogical if they did not accept the output of Get-VM, so the
    following should be possible  
    `$myVMs = GET-VM ; Start-VM –VM $MyVMs`  
    `Start-VM -VM (Get-VM –VM "London-DC01")`  
    `Get-VM | Start-VM`  
    And _should not_ produce an error if the `GET-VM` command returns no VMs.

2.  Some might think it acceptable to say the `-VM` parameter of `Start-VM` and
    `Stop-VM` must contain VM objects. But if it is possible to Get VMs by passing
    VM name(s) and/or server name(s) then many administrators would say that  
    `Start-VM -VM (Get-VM –VM "London-DC01")`  
    is too like coding, and not enough like the shell command line they would
    expect which would be  
    `Start-VM -VM "London-DC01"`

PowerShell parameter declarations can specify how their type and content should
be validated. “Real” programmers who are used to always specifying the type of
everything, tend to grasp this and say “We WILL specify a type (and other
validation) in every parameter declaration”. In C#, for example, if someone
tries to pass your code something of the wrong type, Visual studio will stop
them and tell not to be so silly – their code won’t compile so they never see a
ugly red runtime error. Making parameter types agree makes a little more work,
but their code will be run many times (hopefully) so that’s tolerable. But a
PowerShell user might type a command in the shell once and then it’s gone, that
extra work is less tolerable, and if input which seems logical to them violates
rules you have set, the first they will know is a ugly red runtime error: **any
programmer should worry when normal user behaviour produces runtime errors**
(though a lot will just code to avoid the runtime error, not to adapt their
rules to the way users expect to work).

In PowerShell, in practice, I’ve found I can only get this flexibility by
allowing *anything* to be passed in and doing the validation, longhand, in the
body of the code. In the VM example that means code which says “Is this an array? I’ll deal with each item”; “Is this a string? I’ll treat it as a name which I can turn into an object”; “Is it an Object of the Class I want? Yippee! I can
process it !”; “Is it an object of some other class from which I can get an
object of the class I want? Turn it into the right object.”; “Was it anything
else? If so, do I need to stop execution, or can I return nothing?” Allowing
anything into the function body feels wrong, but I’d ask the question “If the
language did not allow you to specify the parameter type, would you expressly
write code to throw a runtime error if the parameter passed wasn’t of the
expected type? If so, might that error say, ‘If you want to use this as an input, then
do X’ ?”. If the answer is yes to both then your code should do more to cope
with _normal user behaviour_ but if it is yes to the first and no to the second
then Validating Type might be the right way to go.

By way of a second example I came across some code to create a hash from the
content of files, and because PowerShell lets you add properties to objects, the
code returned file objects with an added hash , so you do    
`Get_Some_Files | Add-Hash | something_to_find_Duplicates_using_hashes`    
But the person who wrote `Add-Hash` refused to allow anything but a file object; I
couldn’t do  
`$myFile = Add-Hash "C:\user\James\myFile.stuff"`  
but worse `dir –recurse | Add-Hash` produces an error when it hits directory
objects.

I could insert a `Where-Object` command before the `Add-Hash` to filter down to only
files, but if that is how the `Add-Hash` is going to be used on many occasions,
wouldn’t it be simpler for the command to do that itself? If silently skipping directories bothers you, then catch directories, and use w`rite-verbose` to say
“Ignoring Directory Xyz”, and if someone is trying to add a hash to something
which makes no sense – like a VM object – really bothers you then catch anything
that isn’t a filename, file object or directory object and throw a runtime error
further down the script.

As I was writing this [Shay Levy](https://twitter.com/ShayLevy) retweeted a link
to [the Windows Scripting Guys’ post on Validating
parameters](https://devblogs.microsoft.com/scripting/validate-powershell-parameters-before-running-the-script/);
what’s interesting is they show a function which checks phone number formats. So
let’s put in my phone number formatted as the [ITU says it should
be](https://en.wikipedia.org/wiki/E.164)

<code>test-parameters "+44 (7801) 8 8 10 10" <br/></code>

<code><span style="color:#ff0000;">
Test-Parameters : Cannot validate argument on parameter 'phoneNumber'. The
argument "+44 (7801) 8 8 10 10" does not match the "\d{3}-\d{3}-\d{4}"
pattern. Supply an argument that matches "\d{3}-\d{3}-\d{4}" and try the
command again.</span></code>

What kind of user understands *“Supply an argument that matches
“\d{3}-\d{3}-\d{4}” and try the command again.”* ?

Even if we know that the number is ALWAYS American, if the ITU says we can put
brackets, dashes and spaces into the number to aid readability shouldn’t we
allow (425) 555 1234 or 4255551234 and then clean up the number in the function?

Over-prescriptive (and often plain wrong) validation comes up in plenty of
places: I’ve lost count of web sites which tell me “Credit card numbers must be
entered without spaces.” (with all that computing power you think they could
strip out the spaces, and maybe even identify Visa and Mastercard
automatically). And there are the ones who say names can only contain A-Z and
a-z, tough luck if yours has a hyphen, apostrophe or accented character. (being
an O’Neill this one drives me nuts. So does not checking for apostrophes and
throwing a SQL error). Realistically we’re not going to get rid of it all. Just
don’t add to it, OK?
