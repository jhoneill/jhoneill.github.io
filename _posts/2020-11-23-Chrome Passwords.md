---
layout: post
title: Extracting Passwords and other secrets from Google Chrome, Microsoft Edge and other Chromium browsers with PowerShell
date:   2020-11-23
categories: PowerShell
tags: 
    - PowerShell
    - Programming
    - Security
    - Microsoft Edge
    - Google Chrome
    - Chromium
    - SQLite
---


I recently added SQLite support to my [GETSQL PowerShell
module](https://www.powershellgallery.com/packages/GetSQL) – the final push was
wanting to look at data stored by Microsoft's new (Chromium-based) Edge browser – especially
its *collections* feature. 
<a href="/assets/CredMgr.png"><img style="float:right;display:inline;" border="1" alt="Credential Manager" src="/assets/CredMgr.png" width="245" align="right" height="245
" /></a> 

The Chromium *Engine* - not the “*running gear*” provided 
by Chrome/Edge/Brave or the couple of dozen [others based on Chromium](https://en.wikipedia.org/wiki/Chromium_(web_browser)#Browsers_based_on_Chromium) -
uses SQLite and JSON files to hold configuration information. And encrypted **passwords live in a SQLite database**. 


Historically, Internet Explorer used the *Windows Credential Store*, and the following
three lines of *Windows* PowerShell will show *Web credential*s from the store (selected on the left in the picture). Note that this code 
*doesn’t* work in PowerShell 6 or 7. 

{% highlight PowerShell %}
[void][Windows.Security.Credentials.PasswordVault,Windows.Security.Credentials, 
                                                        ContentType=WindowsRuntime]
$vault = New-Object Windows.Security.Credentials.PasswordVault
$vault.RetrieveAll() | ForEach-Object {$_*.retrievePassword(); $_} | Out-GridView
  
{% endhighlight %}
I've been poking about with credentials stored in the "Windows Credentials" part of the store which need a different access method - I'm using [this PowerShell script I found](https://gist.github.com/toburger/2947424) for that.

There is nothing especially worrying about being able to get your stored
passwords back – the store’s job is to let anything store and retrieve
credentials and ensure that *other accounts cannot access them*. In the same way you
can export a `Credential` object or a `Secure-String` from PowerShell to a file and
*only your account* can decrypt it. 

The world has moved on, and now SQLite is used to hold encrypted passwords. The browser 
locks the database files when its running, but they can be copied, and
it’s safer to work from a copy anyway. 
Most searches for “decrypt Chrome passwords” give instructions which only work 
for **data as it was stored in versions up to 79:** with a row of data from
the logins table, in `$row`, these two lines (in any version of PowerShell) will decode it.
{% highlight PowerShell %}
$output = [System.Security.Cryptography.ProtectedData]::Unprotect($row.password_value, 
                                                                   $null, 'CurrentUser')
$password = [string]::new($output)
  
{% endhighlight %}

The process is similar in its low-friction, *you-and-only-you-can-decode* nature to the using the credential store. Before moving 
on to what has changed in version 80 and beyond, it's worth noting *first*
that an "xcopy" backup (which doesn't preserve the user key) isn't useful - copying 
to a machine with different accounts won't allow the content to be decoded. And 
*second* that **one database can use old and new methods** depending on when 
the password was added. 

The new method uses a key which, for storage, is itself encrypted, prefixed with the 
5 characters “DPAPI”, converted to BASE 64 and saved in a JSON file. The first 
step is to get the protected version back and the same `[ProtectedData]::Unprotect()` 
which was previously used for passwords gets the “OS key” back. 
{% highlight PowerShell %}
$localStateInfo      = Get-Content -Raw "<<path>>\local State")  | ConvertFrom-Json
if ($localStateInfo) {$encryptedkey = [convert]::FromBase64String($localStateInfo.os_crypt.encrypted_key)}
if ($encryptedkey -and [string]::new($encryptedkey[0..4]) -eq 'DPAPI') {
    $masterKey       = [ProtectedData]::Unprotect(($encryptedkey | Select-Object -Skip 5), $null, 'CurrentUser')
}
  
{% endhighlight %}

*Saving* the master key allows encrypted data to move between machines (potentially a benefit). But,
put another way, if the key is obtained once, (however that is done) whoever has it has  everlasting access to 
whatever it secured (potentially a risk). So, the new encryption isn't all positive or all 
negative. The same is true for other things Chromium 
stores - sometimes encrypted, sometimes not. In places it looks like reconnaissance work
to help sell targeted advertising - which is Google's business after all. Microsoft and others 
(like Brave) have removed some of this but not all of it, so *history* goes beyond helping you get back to 
a page or suggesting the right URL as you type; it logs the date and time of every visit, the previous 
page and how long visits last. Auto-completing forms *sounds* good, until you see everything you ever
typed in the search box at eBay has been encrypted and remembered, just in case. 

You can read elsewhere about the details of the AESGCM encryption employed, 
but the short version is that it uses a key and an additional component to encipher the data. 
That "extra" is sometimes called an “Initialization vector” (IV) or a “nonce”
(here in Britain “nonce” is slang for a sex offender – researchers here wouldn’t use
[nonce words](https://en.wikipedia.org/wiki/Nonce_word) for exercises in child language
testing). The former may be reused, and the latter should only be used once with any given 
key. The plain-text, the nonce/IV, and the key all go into the *encryption* process and 
an *authentication tag* comes back with cipher-text. The cipher-text is the same length 
as the plain-text and the authentication tag is a fixed length. *Decryption* doesn't just 
need the key and cipher-text but also the nonce/IV, and the authentication tag. 
Newer versions of .NET provide [an AesGcm class](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.aesgcm)
which it terms a “key” with `encrypt` and `decrypt` methods. In **PowerShell 7** one of
these objects can be created with the master key extracted from the JSON.  I haven't
tried adding AesGCM support to older versions of PowerShell, which using older versions of .NET don't support it directly.
{% highlight PowerShell %}
$GCMKey= [System.Security.Cryptography.AesGcm]::new($masterKey)
  
{% endhighlight %}
The three parameters passed into `$GCMKey.decrypt()`  are concatenated
together in a database field which looks like this:   
`V10[IV/nonce – 12 bytes][Cipher text][Tag - 16 bytes]`   
A script needs to discard the "V10" at the start and split the rest into IV, 
Cipher-Text and Tag. Removing 15 bytes from the start and 16 from the end means the 
output length will be (total_length – 31) and `Decrypt()` needs a buffer of that size to hold the plain-text.
{% highlight PowerShell %}
$encrypted = $row.password_value
[byte[]]$output = 1..($encrypted.length - 31)
$GCMKey.Decrypt($encrypted[3..14] ,
                $encrypted[15..($encrypted.Length-17)],
                $encrypted[-16..-1], $output, $null)
$password = [string]::new($output)
  
{% endhighlight %}

This gives the building blocks for a script to extract passwords
-   Get the Master-key back from the JSON file and create an AESGCM object with it.
-   Copy the SQLite file, query the `logins` table from it.
-   For each row of data, if it begins with “V10” use the `Decrypt` method to get
    the password , otherwise use the `Unprotect` method to get it, and output it
    with other useful parts of the row, like `username`, `site` and `date created`.

*Edge* seems to have imported the passwords that IE left in the Windows Credential Store - judging by
the forgotten junk I can see in my database. If I can export the master-key and keep it secure the 
passwords can remain encrypted in a backed-up SQLite file - there is no need to export the passwords as (insecure) plain text .

When I turned this into a PowerShell script, I decided the best design was to setup the AESGCM key, 
and then use it in an `unprotect` function. Although the original intention was just to dump passwords,
I saw I could make the SQLite *file* and *table* into parameters and get other data and then send it through
`Select-Object` get it in form I want, so *properties* also became a parameter: I set the default value for 
`Table` parameter to be "logins" and default to using the file `Login Data` in its normal location to specify
a database file to copy to `temp` and read with `Get-SQL`.
{% highlight PowerShell %} 
$savedRows = Get-SQL -Lite -Connection $SQLiteFile -Table $Table -Close
  
{% endhighlight %}
`Get-SQL` closes the file at the end, but it can be prevented from doing so if something like `| Select -first 10` stops it, so
I store the results where normally I'd pipe output directly into the next command - `Select-Object` - which passes on
fields that are ready to use, and calls `unprotect` or does other processing (like date conversion) for others.
{% highlight PowerShell %} 
$savedRows | Select-object -Property @{n='User'; e='username_value'}, @{n='Password';e={Unprotect $_.password_value}}...

{% endhighlight %}
By making `Property` a parameter, I can use something like the following to get saved form-filling information.
{% highlight PowerShell %}
Read-Chromium.ps1 '.\AppData\Local\Microsoft\Edge\User Data\Default\Web Data' -Table "autofill" -Property count,
 @{n='Used';e={[datetime]::UnixEpoch.AddSeconds($_.date_last_used)}}, name,  @{n='Value';e={unprotect $_.value}}
  
{% endhighlight %}
Sometimes *dates* are stored as numbers in the 1,600,000,000 range - making them seconds since 1st Jan 1970; as is the case above.    
In other case, like passwords, they are in the 13,000,000,000,000,000 range making them  millionths of a second since 1st Jan 1601.

I can use same script to get saved credit-card information.
{% highlight PowerShell %}
Read-Chromium.ps1 '.\AppData\Local\Microsoft\Edge\User Data\Default\Web Data' -Table "credit_cards"`
     -Property @{n='Name';     e={unprotect $_.name_on_card}} ,  
               @{n='Exp_month';e={unprotect $_.expiration_month}},
               @{n='Exp_year'; e={unprotect $_.expiration_year}},
               @{n='Number';   e={unprotect $_.card_number_encrypted}}
  
{% endhighlight %}

You can get [Read-Chromium from The PowerShell Gallery](https://www.powershellgallery.com/packages?q=read-chromium) - 
There is plenty to explore, and I've shared [some of my early exploration in a GIST](https://gist.github.com/jhoneill/e585bae781f3efa7ac1992b79e037713).