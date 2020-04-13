---
layout: post
title:  "The classy way to complete and validate PowerShell Parameters"
date:   2019-09-23
categories: PowerShell
---
Do you ever wonder why PowerShell parameters are written the way they are? For example, when saying a parameter may have a value of null why does the attribute need to be written `[AllowNull()]`  with an empty `()` ?

A simple answer would be that `[AllowNull]` alone would be setting the type for parameter’s content, but other attributes have things inside the brackets, and these vary: some just have the argument values, for example `[ValidateRange(0,5)]`
And others have `Name=Value`, like `[Parameter(ParameterSetName='Another Parameter Set')]`


These ‘tags’ , more properly called ‘attributes’ are actually _types_, and we can see what happens when an instance of them is created; here’s the `New` method for `ValidateRange:`

```
>[ValidateRange]::new

  OverloadDefinitions
  -------------------
  ValidateRange new(System.Object minRange, System.Object maxRange)
```
The constructor for a new _ValidateRange_ object needs the min and max values for the range; if you create one with the `New-Object` cmdlet you need to put these in the `-ArgumentList` parameter.
Often you see New-Object written as `New-object ValidateRange(0,5)` which looks like the “New” statements in other languages.
PowerShell parses that line as `New-object -TypeName ValidateRange –Argumentlist (0,5)`.

Looking at the constructor for the  Parameter attribute, shows that it takes no arguments:
```
>[Parameter]::new

  OverloadDefinitions
  -------------------
  Parameter new()
```
If “ParameterSetName='Another Parameter Set'” in the example above is not an argument for the constructor, what is it?
The best way to find out is to create one of these objects and look inside:

```
>[Parameter]::new() | gm -MemberType Property

      TypeName: System.Management.Automation.ParameterAttribute
  Name                            MemberType Definition
  ----                            ---------- ----------
  DontShow                        Property   bool DontShow {get;set;}
  HelpMessage                     Property   string HelpMessage {get;set;}
  HelpMessageBaseName             Property   string HelpMessageBaseName {get;set;}
  HelpMessageResourceId           Property   string HelpMessageResourceId {get;set;}
  Mandatory                       Property   bool Mandatory {get;set;}
  ParameterSetName                Property   string ParameterSetName {get;set;}
  Position                        Property   int Position {get;set;}
  TypeId                          Property   System.Object TypeId {get;}
  ValueFromPipeline               Property   bool ValueFromPipeline {get;set;}
  ValueFromPipelineByPropertyName Property   bool ValueFromPipelineByPropertyName {get;set;}
  ValueFromRemainingArguments     Property   bool ValueFromRemainingArguments {get;set;}
```
Notice that the type name is “ParameterAttribute” – all these types have a suffix of “attribute” which is added automatically. The properties are all valid names in a `[Parameter()]` declaration, so
`[Parameter(ParameterSetName='Another Parameter Set')]`  means create a new ParameterAttribute object and set its “ParameterSetName” property. Much like setting properties in the `New-Object` command with the `-Property` parameter.


## Argument completion and validation.

For a long time now I have been writing _Argument Completers_, for example to allow the name of a printer to be completed by pressing `[Tab]`. Usually these are written as functions and registered like this:
`Register-ArgumentCompleter -CommandName Out-Printer -ParameterName PrinterName -ScriptBlock $Function:PrinterCompletion  `

PowerShell 5 added a new parameter attribute to specify an Argument Completer. Its constructor looks like this:
```
>[ArgumentCompleter]::new

  OverloadDefinitions
  -------------------
  ArgumentCompleter new(scriptblock scriptBlock)
  ArgumentCompleter new(type type)
```
The new attribute can contain the whole of the script block (instead of saving it as a function) or use a small script block as a wrapper to call a function like this:
`{PrinterCompletion $args}`

I saw the ArgumentCompleter attribute used with script block in a script someone had shared on-line (I’d like to credit them here but I can’t recall who it was), initially I thought it was something which had been in PowerShell as long as all the other parameter attributes, the `about_functions_advanced_Parameter`s help was only updated to include it in V6 but the first script where I used failed on PowerShell 4 and more checking showed it was only added in V5. So I mentally filed it as “one to go back to”.

I had to go back to it recently because I was converting a script cmdlet to C# and I didn’t want leave the completers as scripts.
Moving the script block out of the call to `Register-ArgumentCompleter` and into a parameter attribute is simple enough, but it takes a bit more digging to understand using a type; and after looking at all the parameter attributes, I found a similar one which was new to PowerShell 6 (Core)
```
>[validateset]::new

  OverloadDefinitions
  -------------------
  ValidateSet new(Params string[] validValues)
  ValidateSet new(type valuesGeneratorType)
```
For both argument completer and validate set the _type_ parameter is a _class_ that we can define. One class might complete Color parameters used in for Excel formatting; another might validate printer names. The classes must implement methods which follow a specific template, and these templates are usually known as “Interfaces”, so for for a ValidateSet the interface says “I have a Method, `GetValidValues()`” – and for an Argument Completer it says “I have a Method, `CompleteArgument(String, String, String, CommandAst, IDictionary)`” which is the same set of parameters you can use when writing a script block for the attribute or for `Register-ArgumentCompleter`.

Let’s look at converting from using the `Register-ArgumentCompleter` example above to using a class which implements the IArgumentCompleter interface. I wrote a function to give PowerShell 6 (core) the Out-Printer functionality found in Windows PowerShell (5); I wanted Tab-completion of printer names and the function to do that looks like the code below:

{% highlight powerShell %}
Function PrinterCompletion {
    param(
       $commandName,
       $parameterName,
       $wordToComplete,
       $commandAst,
       $fakeBoundParameter
    )
    $wildcard          = ("*" + $wordToComplete + "*")

    [System.Drawing.Printing.PrinterSettings]::InstalledPrinters.where({$_ -like $wildcard }) |
        ForEach-Object {[System.Management.Automation.CompletionResult]::new("'" + $_ + "'")}
}
{% endhighlight  %}

So the change is to implement this as a _method_ of a _class_, and use my new class in the `argumentcompleter` parameter attribute: before going into that there is an one other thing to look at…

## Using “using”

Powershell 5 and later supports a using statement in a similar way to C# and VB to shorten
`System.Management.Automation.CompletionResult` to `CompletionResult`, or
`System.Drawing.Printing.PrinterSettings` to `PrinterSettings`
Sometimes writing things explicitly is good but a `using` statement reduces verbosity without sacrificing clarity.
Classes need be explicit about types where functions can be lazy, so before converting the function I’m going to _type_ all the parameters and that would look horrible - with Using I need to specify 4 namespaces because CommandAst, IDictionary, PrinterSettings and CompletionResult are each in different ones; the revised function looks like the following example.

{% highlight powerShell %}
using namespace System.Collections
using namespace System.Drawing.Printing
using namespace System.Management.Automation
using namespace System.Management.Automation.Language

Function PrinterNameCompleterFunction {
    param(
    [string]      $commandName,
    [string]      $parameterName,
    [string]      $wordToComplete,
    [CommandAst]  $commandAst,
    [IDictionary] $fakeBoundParameter
    )

    $wildcard          = ("*" + $wordToComplete + "*")

    [PrinterSettings]::InstalledPrinters.where({$_ -like $wildcard }) |
        ForEach-Object –Process {[CompletionResult]::new("'" + $_ + "'")}

}
{% endhighlight %}

This version will work with Argument Completer attribute and a simple script block like this :
{% highlight powerShell %}
[ArgumentCompleter({PrinterNameCompleterFunction $args})]
$name
{% endhighlight %}

The class implements IArgumentCompleter: and function morphs into the class’s only method, “CompleteArgument”. As well as being explicit about inputs, methods are more explicit about returning their results and what type they are so the class looks like this:
{% highlight powerShell %}
using namespace System.Collections
using namespace System.Collections.Generic
using namespace System.Drawing.Printing
using namespace System.Management.Automation
using namespace System.Management.Automation.Language

class printerNameCompleterPSClass : IArgumentCompleter {
    [IEnumerable[CompletionResult]] CompleteArgument(
        [string]      $CommandName ,
        [string]      $ParameterName,
        [string]      $WordToComplete,
        [CommandAst]  $CommandAst,
        [IDictionary] $FakeBoundParameters
    )
    {
        $wildcard          = ("*" + $wordToComplete + "*")
        $CompletionResults = [List[CompletionResult]]::new()
        [PrinterSettings]::InstalledPrinters.where({$_ -like $wildcard } |
            ForEach-Object {$CompletionResults.Add([CompletionResult]::new("'" + $_ + "'")}
        return $CompletionResults
    }
}
{% endhighlight  %}
With the class in place it can be used in the Argument Completer attribute like this:
{% highlight powerShell %}
    [ArgumentCompleter([printerNameCompleterPSClass])]
    $name
{% endhighlight %}
If/when you write cmdlets in C#, classes are the way to embed the completers and we can also write the class in C# and load it with Add-Type, in a PowerShell script like the following:

{% highlight powerShell %}
Add-type  -ReferencedAssemblies "System.Drawing.Common", "System.Linq",
                   "System.Collections", "System.Management.Automation"  -TypeDefinition @"
using System.Collections;
using System.Collections.Generic;
using System.Drawing.Printing;
using System.Linq;
using System.Management.Automation;
using System.Management.Automation.Language;
public class printerNameCompleterCSharpClass: IArgumentCompleter {
    IEnumerable<CompletionResult> IArgumentCompleter.CompleteArgument(
        string      commandName,
        string      parameterName,
        string      wordToComplete,
        CommandAst  commandAst,
        IDictionary fakeBoundParameters
    )
    {
        WildcardPattern wildcard = new WildcardPattern("*" + wordToComplete + "*", WildcardOptions.IgnoreCase);
        return PrinterSettings.InstalledPrinters.Cast<string>().ToArray().
            Where(wildcard.IsMatch).Select(s => new CompletionResult("'" + s + "'"));
    }

}
"@ -WarningAction SilentlyContinue
{% endhighlight %}

The list of referenced assemblies may need to change on different versions of PowerShell, this one was PowerShell 7 Preview 4.
Note that the class needs to be a Public class, and because it has no public methods, Add-Type generates a warning (which is supressed in the example above).
I can see reasons for using any of the ways

-  For compatiblity with PowerShell before V5, stick with Register-ArgumentCompleter, this has the disadvantage that you can’t see there is a completer when you are looking at the code, which is solved if you …
-  Use the argument completer attribute with a PowerShell function or Class. If you won’t target older versions. The function is probably more natural to write.
-  If you are prototyping a cmdlet to which will eventually be implemented in C#, then using a C# class from the start saves changing it later; and if you have code that you can borrow from C# it saves re-writing, just ensure the class is public and you list the right assemblies to for the version of PowerShell.

![Completing one parameter based on the value of another]()
**Completers and ValidateSets drive Intellisense***, but the behaviour is different. Completers suggest  what the full argument _could_ be, returning a list based on what has been typed so far, they can use everything on the command line to make a suggestion, so when I wrote [Get-Sql](https://www.powershellgallery.com/packages/getsql) the completer for column names looks at the –Table parameter and gets the columns for that database table.
The completer decides which of the “possibles”  are valid suggestions – and completer can become sluggish if the logic in it is to0 complicated.
In the printer names example above I wanted “PDF” to suggest “Microsoft Print to PDF” so the filter matches `"*$wordToComplete*"`.
The user is **not constrained to the the values suggested** by the completer – for example it might suggest, “Red” or “Green” but #0000ff might a valid way to specify Blue. The validation inside the function decides that “Gray” is valid and “Grey” is not  – even the names of colors/colours change their spellings in different flavours/flavors of English.

**ValidateSets define allowed choices**,  if the value entered is not in the set, PowerShell will throw an error saying “valid values are …”.  The set is passed to the shell which filters the list to valid options (this only works against the start of the text, so “PDF” doesn’t match “Microsoft Print to PDF”). PowerShell will also use Enum types to produce a set of of choices, but an invalid value causes a different error when PowerShell tries to convert it to the Enum type.

**Hard coding the valid values will fail for some things**, like Printers or Fonts which vary between machines ; V6 add support for using types which implement the `IValidateSetValuesGenerator` interface; the interface specifies one method “GetValidValues” which takes no arguments and returns an array of strings, a ValidateSet for printer names can be created at runtime, with a class like this:
{% highlight powerShell %}
using namespace System.Management.Automation
using namespace System.Drawing.Printing
class ValidPrinterSetGenerator : IValidateSetValuesGenerator {
    [string[]] GetValidValues() {
        return [string[]][PrinterSettings]::InstalledPrinters
    }
}
{% endhighlight %}
and which can be used like this
{% highlight powerShell %}
    [ValidateSet([ValidPrinterSetGenerator])]
    $name
{% endhighlight %}

As with the argument completer, this class could be written in C#,  and loaded with `Add-Type`; the following example is written for PowerShell 7 preview 4:

{% highlight powerShell %}
Add-type  -ReferencedAssemblies "System.Drawing.Common", "System.Linq",
              "System.Management.Automation"  -TypeDefinition @"
using System.Drawing.Printing;
using System.Linq;
using System.Management.Automation;
public class PrinterNameValidator : IValidateSetValuesGenerator {
      public string[] GetValidValues() {
        return PrinterSettings.InstalledPrinters.Cast<string>().ToArray();
      }
}
"@
{% endhighlight %}
## Adding custom parameter attributes

Additional special attribute classes are available in PowerShell 5 onwards, and they are used in slightly different way. You still declare a class, but now that class says it implements one of two classes rather than an interface. One of these does validation, and its job is to throw an error when the argument is not valid; here is an example.
{% highlight powerShell %}
using namespace System.Management.Automation
using namespace System.Collections.Generic
using namespace System.Drawing.Printing

class ValidatePrinterExistsAttribute : ValidateArgumentsAttribute {
    [void] Validate([object]$Argument, [EngineIntrinsics]$EngineIntrinsics) {
        if(-not ($Argument -in [PrinterSettings]::InstalledPrinters)) {
          Throw [ParameterBindingException]::new("'$Argument' is not a valid printer name.")
        }
    }
}
{% endhighlight %}

This creates a class whose name ends with “Attribute” which implements the `ValidateArgumentsAttribute` class; it inherits the properties and methods of that class but replaces the `Validate()` method with its own code. `Validate` doesn’t return a value, it either completes or it throws an exception, and it takes two arguments, the argument being validated and “Engine Intrinsics” which is what we can see as $ExecutionContext in a script. This has some advantages over using `[ValidationScript{}]`:

-  It is easier to read than embedding a long script in an attribute.
-  It removes duplication when same validation applies to multiple parameters (for example if we have to apply the same Printer name check in more than one command)
-  We control the error message. This :
<code><font color="#c0504d">'Wibble' is not a valid printer name</font></code>
is more helpful than
<code><font color="#c0504d">Cannot validate argument on parameter 'name'. The "$_ -in [System.drawing.Printing.PrinterSettings]::InstalledPrinters " validation script for the argument with value "wibble" did not return a result of True.
Determine why the validation script failed, and then try the command again.</font></code>
- It’s how things are done in C# – as before , the class above could be written in C# and loaded using `Add-Type`.

When we tag a parameter with this class we can omit the “Attribute” part of the Class name and need to include the () to say we are creating a new object of this type as an attribute, so it is written:
{% highlight powerShell %}
    [ValidatePrinterExists()]
    $name
{% endhighlight %}
The other class that works in this way is the Argument Transformation Attribute. Again we have the option to use Add-Type and write the class in C# but if we do it PowerShell the declaration looks like this
{% highlight powerShell %}
using namespace System.Management.Automation
using namespace System.Collections.Generic
using namespace System.Drawing.Printing

class PrinterTransformAttribute : ArgumentTransformationAttribute  {
    [object] Transform([EngineIntrinsics]$EngineIntrinsics, [object] $InputData) {


       ## transform $inputdata to $something

        return $something
    }
}
{% endhighlight %}
This,too can throw if the input is invalid, so I could look for a printer which matches InputData and if I find exactly one, return it. If I find none, or more than one, I can throw an error. This might be better than using the custom validate set: I have these printers on my Laptop:
```
Brother HL-1110 series
EPSON Stylus Photo R2880
Fax
Microsoft Print to PDF
Microsoft XPS Document Writer
OneNote
Send To OneNote 2016
```
Notice I have two OneNote versions, each with their own driver. So a transformation attribute would need to check for a perfect match and then check for a partial match. If I combine this with the completer I can:-

-  Keep pressing tab until I get “Microsoft Print to PDF”
-  Type PD [tab] to fill in “Microsoft Print to PDF”
-  Type PDF and let the transformation attribute change it to “Microsoft Print to PDF”
-  Use “Brother”, “Epson”, “PDF”, “XPS”, or “OneNote” as printer short names.
-  Reject names which are wrong like “PFD” or ambiguous like “Microsoft”

More than one combination of validation, completion and transformation may be right, and different ones might be optimal in different cases. If you need backwards compatibility your choices are more limited, but knowing what is available, and where, lets you pick the one best suited for the task at hand. I like to tell people that job of validation is to help users put in good input, not to save you from catching bad input, intellisense, transformation, and custom validations help.
A message like  <code><font color="#c0504d">Supply an argument that matches "\d{2}-\d{2}-\d{2}[a-z]?"</font></code>  is unhelpful;  but a custom validator takes only a little longer to write and can tell the user <code><font color="#c0504d">'1234' is not a valid Part number. Part numbers are formatted as '11-22-33' or '99-88-77C'</font></code>; it can can be reused if part numbers are parameters in multiple places, and it also makes the script easier to read later, because `[ValidatePattern("\d{2}-\d{2}-\d{2}[a-z]?")]` means we mentally parse the regular expression and then say, “ah, yes, that’s describing a part number”.  `[ValidAsPartNumber()]` tells us what is being done, if we need to know _how_ we look somewhere else for the answer. They don’t support early versions of Windows PowerShell (4 and below), but I expect to use them where that is not an issue.