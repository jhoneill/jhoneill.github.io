---
layout: post
title:  "PowerShell Numbers and .NET Cultures"
date:   2022-03-05
categories: PowerShell
tags: 
    - PowerShell
    - Development
    - .NET
    - Formatting
---
After multiple false-starts on an all-in-one guide to how PowerShell translates what we *type* into numbers or dates, and how it translates them back into something we can *read*, I've decided to split the content up, so this is the first part of three.

Starting with something well known: we can just write a decimal number and PowerShell processes it as a "numeric literal" when doing that makes sense. For example:

```powershell-interactive
    ps > $x = 2 + 2 ; $x ; $x.GetType()
    4
    
    IsPublic IsSerial Name                                     BaseType 
    -------- -------- ----                                     --------        
    True     True     Int32                                    System.ValueType
    
```

Here, PowerShell sees "2" and decides it looks like an integer, so defaults it to the `[int32]` type, it sees the + operator and a second "2" and decides it is adding two `[Int32]` objects. The result is another `[int32]` with a value of 4.

From the start, Windows PowerShell had Type-Accelerators: `[int]` corresponding to `[system.int32]` - a 32 bit, signed, integer -  and `[long]` for `[system.int64]`; `[bigint]` wasn't there initially but *is* supported in 5.1. PowerShell Core (6) added a lot more, including  `[short]` for `[system.int16]` and unsigned variations.  
If PowerShell sees a floating point number it treats it as a `[system.double]` and there is another long-established type accelerator, `[float]` for single precision floating point.

There is more intelligence: "2/2" returns an `int32` but "2/3" returns a `double`; and if something needs a specific type PowerShell usually converts automatically.  
On rare occasions we need to say "Don't make 2 an int32", and all versions of PowerShell have been able to *tag* a number with a suffix: "L" says "make this a long integer", and "D" says make it a [decimal](https://docs.microsoft.com/en-us/dotnet/api/system.decimal?view=net-6.0), so:

```powershell-interactive
    ps > (25d + 25).GetType()

    IsPublic IsSerial Name                                     BaseType
    -------- -------- ----                                     --------
    True     True     Decimal                                  System.ValueType 

```

Having been told one of the values is a decimal, the result is also a decimal.

All versions have also supported `KB`, `MB`, `GB`, `TB` suffixes which multiply by 2<sup>10</sup>, 2<sup>20</sup>, 2<sup>30</sup> and 2<sup>40</sup> (we'll leave aside arguments whether these should be powers of 10 or powers of 2.)  `2dGB` is the 2 as a decimal multiplied by 2<sup>30</sup>, but `2GBd` isn't valid.  
[The help topic about_numeric_literals](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_numeric_literals?view=powershell-7.2) has the complete information on the type accelerators, prefixes and suffixes.

## Hex, Binary and Octal numbers

Everything so far has assumed that input and output are expressed in base 10: `25` becomes the 8 bits `00011001` (padded out to 32 bits); when those bits are converted back for output they are known to represent an integer (not, say, a character) and in decimal notation that integer is written "25". We can prefix a number with `0x` to say it is a *hexadecimal* number; `0x25` is `37` in decimal, and `0x19` is decimal 25. We can also provide a number in binary with `0b` as a prefix, so `0b111101101` is 493 in decimal. This might seem a odd but, for example, converting a unix file mode we might want to turn something into binary and then into a number.

```powershell-interactive
    ps> [int]("0b"  + ("rwxr-xr-x" -replace "-","0" -replace "[^0]","1" ))
    493

```

To *output* as hex we can use one of **.NET's standard formatting strings** - we'll see these in more detail shortly - `X` with an optional number of digits says "render this in hex notation".

```powershell-interactive
    PS > (25).ToString("x4")
    0019

```

And the `[convert]` class - which we will also see more of - can do other bases: `[convert]::ToString(25,16)` says use *base 16* and returns `19` That unix file mode wouldn't be expressed as the decimal number 493, nor as the hex `01ed`, but as **octal**, and `convert` can be told to use base 8:

```powershell-interactive
    PS > [convert]::ToString(493,8)
    755 

```

This is the form used in unix commands like chmod. No discussion of octal would be complete without a nod to the joke about programmers muddling up Christmas day and Halloween: 

```powershell-interactive
    ps> $x = 25 ; "Dec $x = Oct $([convert]::ToString($x,8))"
    Dec 25 = Oct 31
    
```

Only certain numbers are valid as the _base_: 2 for binary, 8 for octal, 10 for decimal and 16 for hexadecimal - you can't [multiply six by nine and render the answer in base 13](https://www.h2g2.com/entry/A4288584).

## Separated by different cultures

For binary, hex, and basic integers there are no punctuation marks between the digits. English speaking countries generally use "." for the decimal point in floating point numbers, but it is written as "," in other places. The job of *cultures* in .NET is to specify such things, living in England I have my culture set like this:

```powershell-interactive
    ps > Get-Culture
    LCID             Name             DisplayName
    ----             ----             -----------
    2057             en-GB            English (United Kingdom)
    
```

Culture *names* are a language code (`en` in this case) and a country code (and for now we'll ignore the subtle distinction between "Great Britain" and "United Kingdom"); in the culture object's `NumberFormat` we can see how *currency* values are written - for example comparing my default (en-GB) settings with what they do in France (fr-FR) (Just as there are multiple cultures with the English language, French-Canada and French-Switzerland exist with different currencies.)

```powershell-interactive
    ps > (get-culture).NumberFormat | ft currency* 
    Currency      Currency         Currency   Currency       Currency  Currency        Currency  
    DecimalDigits DecimalSeparator GroupSizes GroupSeparator Symbol    NegativePattern PositivePattern
    ------------- ---------------- ---------- -------------- --------  --------------- ---------------
                2                .        {3}              ,        £                1               0
    
    ps > (get-culture fr-FR).NumberFormat | ft currency*
    Currency      Currency         Currency   Currency       Currency  Currency        Currency  
    DecimalDigits DecimalSeparator GroupSizes GroupSeparator Symbol    NegativePattern PositivePattern
    ------------- ---------------- ---------- -------------- --------  --------------- ---------------
                2                ,       {3}                        €                8               3
    
```

Where a sum of money in Britain might be written £1,234.56, in France it would be 1 234,56 €. It's impossible to see above, but the group separator ("," in GB) is a *space* in France, and the positive and negative patterns say where the symbols go (before or after the number).

The differences in separators means that some cultures use 123,456 for a large integer, and in others that would be the way to write a floating point number 1/1000th of the size.  
I could use the PowerShell command `Set-culture fr-FR` and start a fresh PowerShell session, but to avoid switching everything into French it's possible to change the culture for the current thread only, and see what happens when we try to convert that string to a number in that culture:

```powershell-interactive
    ps> [System.Threading.Thread]::CurrentThread.CurrentCulture = Get-Culture fr-FR
    
    ps> (Get-Culture).DisplayName
    French (France)
    
    ps> [double]"123,456"   
    123456
    
    ps > ([double]"123,456").ToString("C2")
    123 456,00 €

    ps> ([double]"123 456,00")
    InvalidArgument: Cannot convert value "123 456,00" to type "System.Double".
    Error: "Input string was not in a correct format."

```

## Does one input *always* give the same output?

There is a concept of **invariant culture**; that a program's results for any given input(s) remain the same regardless of any change in culture. The example above shows that `[double]"123,456"` uses *invariant* culture and treats "," as a *thousand separator* despite the culture saying "," is the *decimal point*.  
If we ask `.ToString()` to show us a number to 2 decimal places, *local culture* cuts in - so in the example above the French conventions are used for thousand and decimal separators.  
The last step of the example shows that casting with `[type]` syntax needs strings to use the *Invariant* format. Parsing *local* format requires either the `parse()` static method of the `double` class or use of the `[convert]` class as follows:

```powershell-interactive
    ps> (Get-Culture).DisplayName
    French (France)
    
    ps > [double]::parse("123 456,00")
    123456
    
    ps > [convert]::ToDouble("123 456,00")
    123456

```

But there is a **trap**. These two examples use the *current* culture, there is a risk of *different results for the same inputs*. When I change my machine's culture back to English `[double]::parse("123 456,00")` will give an error. Both `Convert()` and `Parse()` allow the *format provider* to be specified, so for portability it is better to be explicit about the culture:

```powershell-interactive
    ps > [double]::parse("123 456,00",[cultureinfo]::GetCultureInfo('fr-Fr'))
    123456
    
    ps > $French = Get-Culture fr-FR
    ps > [convert]::ToDouble("123 456,00",$french)
    123456

```

Whatever culture is currently selected the last two examples will treat the input as French. (You need to know that the input will be French-formatted, of course). I phrased the idea of invariant culture as the result being the same for the same input, if the result has been converted to a string for printing, the same result may be output in different ways.

## Formatting codes available for numbers

Above I used `.ToString("C2")` for "Local currency, 2 places of decimals" and before that I used `.ToString("x4")` for "Hex, 4 digits" I hinted that there were several of these shorthands - the number part changes its meaning from one to another. The full set of things that can be written in a number formatting string are:

| Code | Explanation                                                                        |
|------|------------------------------------------------------------------------------------|
| .    | Replaced by the **local** decimal separator                                        |
| ,    | Replaced by the **local** thousand separator                                       |
| #    | Digit if required                                                                  |
| 0    | Pad with zeros if no digit present at this space                                   |
| %    | Convert to percentage (i.e multiply by 100 and add the % sign)                     |
| C    | Currency (with optional number of decimal places)                                  |
| D    | Digits, no decimals (with optional number of digits to left pad to)                |
| E    | Exponent notation (with optional number of decimal places)                         |
| F    | Fixed decimals (with optional number decimal places, 2 by default)                 |
| G    | General number format (with optional number of digits before using exponent form)  |
| N    | Number (with thousand separators, and optional number of decimal places)           |
| P    | Percentage (with optional number of decimal places)                                |
| R    | Round trip - floating point numbers only                                           |

Formatting can be used instead of an arithmetic operation when displaying numbers: there is no need to use `[math]::Round`, or convert to an integer, you can format with `0` for "and no decimals" and the number will be rounded up or down as required. In the same way, if you want to show thousands or millions "#,###" is thousands with separators - and will repeat the comma every three digits. But you can omit the final group(s) of 3 digits by writing the comma(s) without the 0 or # place holders which would normally follow, the example below shaves off 4 such groups and adds literal text to indicate trillions - some literal text needs to be enclosed in quotes (single or double both work), but Tn won't be mistaken for anything else.

```powershell-interactive
    ps> (314159265358979).ToString("#,,,,Tn")
    314Tn
```

I said at the start there had been false starts to writing this and one was to publish [a notebook as a gist](https://gist.github.com/jhoneill/fe72f1bf6efc4f28d195c288250c5667): showing how each of the different formats processes different numbers.  

## PowerShell's -f operator

In addition to their use in `.ToString()` these formatting codes can be used in the `.Format()` method of a .NET String, which in PowerShell is normally accessed through the `-f` operator; for example:

```powershell-interactive
    ps> "{0:000} {0,-12:f} {0:x4} {0,10:c} " -f 22
    022 22.00        0016     £22.00
     
```

The first item inside each set of braces is the *index* of the item, 0 represents the first value after -f and it has been used four times in the example;  `,10` says "right aligned padded to 10 spaces" and `,-12` says "Left aligned padded to 12 spaces,  and `:000`, `:f`, `:x4`, and `:c` are the formatting strings. As you can see alignment and formatting can *both* be given, alignment must be written first.

That's it for *numbers*, the next part will look at **Date** formats and conversions.
