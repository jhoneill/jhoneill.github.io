---
layout: post
title:  "Transformers for parameters which take secrets."
date:   2020-04-10
categories: PowerShell
---

<p style="margin-bottom:0">PowerShell Functions often need to accept Secure-Strings or PowerShell credentials (which contain Secure-Strings) as parameters. There are people who will advise avoiding use of Secure-String completely, which isn’t really practical, but one shouldn’t credit Secure-String with magic powers it doesn’t have.</p>
1.  It is encoded so that your account on the current machine can decode it at will – any process running under your account can get to the secret.
2.  Another account, or your account on another machine can’t decode it. The main use is **securing something in a file on one machine.**
3.  In PowerShell 7 (but not PowerShell Core 6 or Windows PowerShell 5) the `ConvertFrom-SecureString` cmdlet has an <code>&#8209;AsPlainText</code> switch which gets the secret back. With older version the standard process is to create a credential object and then ask for the plaintext “network password”.
4.  **It is only securing the credential at rest** in the file. At some stage the **plain text will be in memory(()) (and malware can find it).

<p style="margin-bottom:0">A script which looks like the following is <b>obviously bad</b></p>
{% highlight powerShell %}
$MyUserName = "James"
$MyPassword = ConvertTo-SecureString -force -AsPlainText "NotVerySecret"
$Credential = New-Object PSCredential $myUserName,$MyPassword
{% endhighlight %}
<p style="margin-bottom:0;margin-top:1.3334em"><b>Better to do</b></p>
{% highlight powerShell %}
if (Test-path mycred.xml)  {
       $Credential = Import-Clixml myCred.xml
}
else {
       $Credential = Get-Credential
       $Credential | Export-CliXML my.cred.xml
}
{% endhighlight %}
<p style="margin-bottom:0;margin-top:1.3334em">In the resulting XML file you might see something like this</p>
{% highlight xml %}
<Props>
<S N="UserName">james</S>
<SS N="Password">01000000d08c9ddf011 … 395d4060</SS>
</Props>
{% endhighlight %}
`<S>` Denotes a normal string and `<SS>` a long sequence of digits which become a Secure-String. This simple method uses a personal, local key; there are ways ro use different keys, and DSC uses one of them to encrypt passwords when making them in a .MOF file, which a service can then use to decode a password.    
Credentials and Secure-Strings are also useful for things like personal access tokens or API Keys where you might prefer not to save the plain text, but the commands which use them expect plain text; here’s a simple case:
{% highlight powerShell %}
Function Demo {
    [cmdletBinding()]
    param(
        [string]$ApiKey
    )
    "Your API key is $ApiKey"
}
{% endhighlight %}
<br/>Because `[string]` acts as “Convert what you are given to a string”, the function won’t refuse a Secure-String or a credential - it outputs
Y`our API key is System.Security.SecureString` or `Your API key is System.Management.Automation.PSCredential`    
My usual answer to this that `$APIkey` shouldn’t be _typed_ and the body of the function should include a check on the type of `$APIKey` and convert it as required; this approach has worked well for me as long as I have been doing PowerShell but there is an option to do it more elegantly.

One Question which came up on GitHub recently went like this:    
_Some commands take an API key use it as a plain text string so they have a string-typed parameter, but it would be useful to have a parameter attribute which allowed a string declared as a string to accept and convert credential or securestring objects._

<p style="margin-bottom:0">PowerShell 5 and upwards can support <b>argument transformers</b>, and and the one below is named “UnSecureString”, it’s declared as an argument transformation attribute, with a single method, transform, which receives the parameter as “Input data” and returns an object. To work on version 5, it converts a credential to a password, and if it is given a Secure-String, it converts to it to a credential first.</p>
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
<p style="margin-bottom:0;margin-top:1.3334em">With this in place the parameter declaration in the function changes:</p>
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
<p style="margin-bottom:0;margin-top:1.3334em">And now it will take string, Secure-String, or credential</p>
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
<p style="margin-bottom:0;margin-top:1.3334em">It’s a debatable whether the parameter should <i>still</i> be typed in this case. In another github discussion, someone said they didn’t like to leave the untyped because on-line help shows parameter types and leaving them as the default, [object] skipped a chance to guide the user to provide the right thing. Since PowerShell lets us specify full paths and relative paths as strings, path info objects, or file info objects to identify a file we want to work on, I’ve always tried to accept make a “-Widget” parameter accept a unique widget ID, a Widget Name (possibly with wildcards), or one or more Object(s), the user shouldn’t need to care and I will write help explaining the three options. Without the help I’d be making the user guess.  The help for this example steers the user towards providing a plain text string:</p>
```
 NAME
    Demo
 SYNTAX
    Demo [[-ApiKey] <string>]
```
My preference is more towards removing the type and assuming the user will read _more_ of the help, but I don’t think these things are absolutes. As supporting older versions of PowerShell becomes less of an issue, using classes and shifting the “first make sure this is the right kind of thing” code can simplify the function body, which is a plus.
There is a case for changing the parameter type to Secure-String and having a transformer which turns plain strings to secure ones. But that can lead to writing
`Demo –apikey (convertTo-SecureString …)` , which is the horrible insecure method I showed at the start but more annoying to a user – especially if they look inside the function and see the first thing that happens is the Secure-String is converted back an placed in a request body. Examples should never show that, but should be based on the Secure-String coming from a file.
