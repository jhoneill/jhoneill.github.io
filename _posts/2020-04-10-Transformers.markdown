---
layout: post
title:  "Transformers for parameters which take secrets."
date:   2020-04-10
categories: PowerShell
---

PowerShell Functions often need to accept secure strings as or PowerShell credentials (which contain secure strings) as parameters. There are people who will advise avoiding use of secure string completely, which isn’t really practical, but one shouldn’t credit Secure String with magic powers it doesn’t have.
1.  It is encoded so that your account on the current machine can decode it at will – any process running under your account can get to the secret.
2.  Another account, or your account on another machine can’t decode it. The main use is **securing something in a file on one machine.**
3.  In PowerShell 7 (but not PowerShell Core 6 or Windows PowerShell 5) the `ConvertFrom-SecureString` cmdlet has an `-AsPlainText` switch which gets the secret back. With older version the standard process is to create a credential object and then ask for the plaintext “network password”.
4.  **It is only securing the credential at rest** in the file. At some stage the **plain text will be in memory(()) (and malware can find it).

A script which looks like the following is obviously bad
{% highlight powerShell %}
$MyUserName = "James"
$MyPassword = ConvertTo-SecureString -force -AsPlainText "NotVerySecret"
$Credential = New-Object PSCredential $myUserName,$MyPassword
{% endhighlight %}
Better to do
{% highlight powerShell %}
if (Test-path mycred.xml)  {
       $Credential = Import-Clixml myCred.xml
}
else {
       $Credential = Get-Credential
       $Credential | export-CliXML my.cred.xml
}
{% endhighlight %}
In the xml file you might see something like this
{% highlight xml %}
<Props>
<S N="UserName">james</S>
<SS N="Password">01000000d08c9ddf011 … 395d4060</SS>
</Props>
{% endhighlight %}
`<S>` Denotes a normal string and `<SS>` a long sequence of digits which become a secure string. This simple method uses a personal, local key; there are ways ro use different keys, and DSC uses one of them to encrypt passwords when making putting them in a .MOF file, which a service can then use to decode a password
Credentials and secure strings are also useful for things like personal access tokens or API Keys where you might prefer not to save the plain text, but the commands which use them expect plain text here’s a simple case:
{% highlight powerShell %}
Function Demo {
    [cmdletBinding()]
    param(
        [string]$ApiKey
    )
    "Your API key is $ApiKey"
}
{% endhighlight %}
Because `[string]` acts as “Convert what you are given to a string”, the function won’t refuse a secure string or a credential - it outputs
Y`our API key is System.Security.SecureString` or `Your API key is System.Management.Automation.PSCredential`

My usual answer to this that `$APIkey` shouldn’t be typed and the body of the function should include a check on the type of `$APIKey` and convert it as required; this approach has worked well for me as long as I have been doing PowerShell but there is an option to do it more elegantly.

One Question which came up on GitHub recently went like this:
_Some commands take an API key use it as a plain text string so they have a string-typed parameter, but it would be useful to have a parameter attribute which allowed a string declared as a string to accept and convert credential or securestring objects._

PowerShell 5 and upwards can support **argument transformers**, and and the one below is named “UnSecureString”, it’s declared as an argument transformation attribute, with a single method, transform, which receives the parameter as “Input data” and returns an object. To work on version 5, it converts a credential to a password, and if it is given a secure string, it converts to it to a credential first.
{% highlight powerShell %}
class UnSecureString : System.Management.Automation.ArgumentTransformationAttribute  {
    [object] Transform([System.Management.Automation.EngineIntrinsics]$EngineIntrinsics, [object] $InputData) {
        if ($InputData -is [securestring]) {
            $InputData =  New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList 'Placeholder', $InputData
        }
        if ($InputData -is  [pscredential]) {
            $InputData =  $InputData.GetNetworkCredential().password
        }
        return ($InputData)
    }
}
{% endhighlight %}
With this in place the parameter declaration in the function changes:
{% highlight powerShell %}
Function Demo {
    [cmdletBinding()]
    param(
        [UnsecureString()]
        [string]$ApiKey
    )
    "Your API key is $ApiKey"
}
{% endhighlight %}
And now it will take string, secure String, or credential
```
>demo "api12345key"
 Your API key is api12345key

>$ss = ConvertTo-SecureString -AsPlainText "api34567key" -Force
>demo $ss
 Your API key is api34567key

>$cred = New-Object pscredential "NoRealName", $ss
>demo $cred
 Your API key is api34567key
```
It’s a debatable whether the parameter should still be typed in this case. In another github discussion, someone said they didn’t like to leave the untyped because on-line help shows parameter types and leaving them as the default, [object] skipped a chance to guide the user to provide the right thing. Since PowerShell lets us specify full paths and relative paths as strings, path info objects, or file info objects to identify a file we want to work on, I’ve always tried to accept make a “-Widget” parameter accept a unique widget ID, a Widget Name (possibly with wildcards), or one or more Object(s), the user shouldn’t need to care and I will write help explaining the three options. Without the help I’d be making the user guess.  The help for this example steers the user towards providing a plain text string
```
NAME
    Demo

 SYNTAX
    Demo [[-ApiKey] <string>]
```
My preference is more towards removing the type and assuming the user will read more of the help, but I don’t think these things are absolutes. As supporting older versions of PowerShell becomes less of an issue, using classes and shifting the “first make sure this is the right kind of thing” code can simplify the function body, which is a plus.
There is a case for changing the parameter type to secure string and having a transformer which turns plain strings to secure ones. But that can lead to writing
`Demo –apikey (convertTo-SecureString …)` , which is the horrible insecure method I showed at the start but more annoying to a user – especially if they look inside the function and see the first thing that happens is the secure string is converted back an placed in a request body. Examples should never show that, but should be based on the secure string coming from a file.
