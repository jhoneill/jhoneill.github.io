---
layout: post
title:  "Developers disliking PowerShell - part 2, a mock exam"
date:   2021-11-22
categories: PowerShell
tags: 
    - PowerShell
    - C#
    - Developmnet
---
 [Yesterday I wrote](/powershell/2021/11/20/Alex.html) a response to something from twitter about perecieved preference for C# as console apps and not as PowerShell cmdlets. One of the responses to the tweet that spawned that read:
> I'm curious, what is the benefit of writing a cmdlet instead of an ordinary console program (which is automatically more portable across shell) if the language used is c# anyways?

First of all if .NET is available then PowerShell is also avaialable. There's a valid objection "I can't make the my linux customer install .NET" But then you wouldn't be writing in C#. For customers who want to want to live in other shells, it's easy enough to write a 1 line wrapper which calls pwsh to invoke a command. 
 
This rattled round in my head overnight and I had the idea to use the idea to use an exam paper format to look at the benefits.. So if you're interested give the questions a try. Ready ? Your time starts... **now**.

------------------------------------
A performance lab measures the cylinders of engines. They have asked you to write command line utility which will take the diameter (d) and height (h) of a cylinder and calculate its volume and surface area. \[ Radius (r) is d/2; Volume is given by πr²h and surface area by 2(πr²) + 2πrh \]

--------------------------------------

1.  Describe the program you would write calculate cylinder areas and volumes for the lab.    
   *You do not need to write any code, but for the following questions you* do *need to have a solution in mind - quick pseudo code might be helpful. Like the real world, you have been asked to design the program before you have all the requirements. I suggest making notes of your answers to the subsequent questions for the review at the end.*

    > It is a requirement that your program can run interactively, prompting the user for input, and on other occasions run non-interactively taking input as parameters.

2.  If your design from Question 1 relies on the user typing data in response to prompts like  “Please enter the height” **or** if it can only work when all the parameters are on the command line, describe how you would change it to meet this requirement.

    > Some of the lab technicians write down their measurements as height x diameter, others were taught that engine cylinders are expressed as bore x stroke (diameter x height).

3.  Do you need to add anything your program allow to new users to discover whether it requires height first, or diameter first? And if either order is allowed - how to identify the which value is height and which is diameter?  If only one order is allowed by your design what changes would be required to support entry in either order?

    > Older technicians learned always to give units of measurement. Younger ones always give measurements in mm.

4.  How does your original program respond if the input contains non-numeric characters? 
    Do any error messages help users to correct their input?
    What happens if a negative height is specified?

    > Technicians will typically measure all the cylinders of an engine. Motorbike engines may have only one cylinder, engines for aviation may have many. Sizes are sometimes entered into a simple file on a laptop (assume such files can be parsed easily).

5.  How could a technician use your program to calculate the surface area and volume for each set of dimensions in a file ? 
(Describe any modifications your program might need).

    > One of the purposes of the measurements is to check that manufacturing processes produce cylinders within the correct tolerances.

6.  Having measured the cylinders in a batch of engines how would your program be used to find the minimum, maximum and average volume for the batch?

    > Some of the lab’s customers specify the required cylinder size by giving the volume and either the diameter or the height. 
    You were **not** originally asked to calculate the missing value.

7.  How much effort would be required to modify your program to take any two of the three values, ideally in either order? 
    Would the resulting changes require changes to your solutions for questions 5 and 6? 
    Would the modified program allow the process from question 5 to take any two values from a file and calculate the third?

    > Some of the technicians have asked for a different number of decimal places in the output.

8.  What can you suggest to customize the output?

----------------------------------------------
### The answers

*Question 1.*

Below is my C# code to do the job as a powershell cmdlet. It's an example, and not indented to be a perfect solution.     
I used VS Code to write it and when I started I didn’t have the PowerShell cmdlet template on the machine I was using,       
`dotnet new -i Microsoft.PowerShell.Standard.Module.Template`   
Adds it, and    
`dotnet new psmodule`       
gives you a simple C\# project with a dummy cmdlet in it.
  
I had the advantage of knowing what the other questions would be but the sample
code is not going above and beyond what anyone who has done a cmdlet before would write.

{% highlight c# %}
using System;
using System.Management.Automation;

namespace cmdlet
{
    public class cylinder
    {
        public double Diameter { get; set; }
        public double Height { get; set; }
        public double Volume { get; set; }
        public double SurfaceArea { get; set; }
    }

    [Cmdlet(VerbsCommon.Get,"CylinderSize")]
    [OutputType(typeof(cylinder))]
    [Alias("cyl")]
    public class GetCylinderSizeCommand : PSCmdlet
    {
        [Parameter(Mandatory = true, Position = 0, ValueFromPipelineByPropertyName =true)]
        [ValidateRange(0,1e32)]
        public double Height { get; set; }

        [Parameter(Mandatory = true, Position = 1, ValueFromPipelineByPropertyName =true)]
        [ValidateRange(0,1e32)]
        public double Diameter { get; set; }

        protected override void BeginProcessing() {}
        protected override void ProcessRecord()
        {
            double radius   = Diameter / 2 ;
            double endArea  = Math.PI * radius * radius ;
            WriteObject(new cylinder {
                Diameter    = Diameter,
                Height      = Height,
                Volume      = (endArea * Height) ,
                SurfaceArea = (Math.PI * Diameter * Height ) + 2 * endArea
            });
        }
        protected override void EndProcessing() {}
    }
}
{% endhighlight %}
The command has an alias of `cyl`, because the its full name `Get-CylinderSize`, and  PowerShell's automatic contraction to allow `CylinderSize` are both on the long side.    
Parameter declarations have done a lot of the work, including some rudimentary validation; users can omit the parameter names if they are familiar with the command or specify just enough of the name
to be unambiguous (-D and -H will work).    

Output is an object that leaves PowerShell to decide how it should presented: when there are more than four properties it will use a list view, four or fewer and the result is printed as a table like this:

```
> cyl -Bore 30 -Stroke 15

Diameter Height Volume            SurfaceArea
-------- ------ ------            -----------
30      15      10602.8752058656  2827.43338823081

```


**Question 2 “If your design relies on prompting the user to type something…”**    
This  doesn’t. Its alias and PowerShell's treatment of the Get- part of a
command as option it can be run as
* `cyl 10 20` (it will assume height first) or as  
* `CylinderSize -d 10 20` (specifying the first parameter is diameter, so the second is assumed to be height) or as  
*  `Get-Cylindersize -Height 10 -Diameter 20` (to be absolutely explicit).    
As the example above shows, `Bore` and `Stroke` can be used as aliases for `Diameter` and `Height` respectively.

When cmdlet parameters are marked as *mandatory*, as these are, if any is missing PowerShell will handle prompting the user, like this
```  
Supply values for the following parameters:
Diameter:

```  
**Question 3 is about help and discoverability**

* Parameter *names* tab complete (including the aliases to allow “old school” technicians to enter -bore or -stroke) 
* Running the command with -? produces the following help without needing anything in the C\#

```
 NAME
  Get-CylinderSize

 SYNTAX
  Get-CylinderSize [-Height] <double> [-Diameter] <double> [<CommonParameters>]

 ALIASES
  cyl
  
```

**Question 4 is about input validation and error messages.**   
Many unsuccessful programs go off the rails with input the programmer never thought would be given (Like a surname of “O’Neill”)

When input contains letters, the default error message reads     
`Cannot process argument transformation on parameter 'Height'. Cannot convert value "20mm" to type "System.Double". Error: "Input string was not in a correct format.`    
This can be customized for a particular parameter. For a negative number the message reads   
`Cannot validate argument on parameter 'Height'. The -10 argument is less than the minimum allowed range of 0. Supply an argument that is greater than or equal to 0 and then try the command again.`  

You can give yourself bonus marks if your solution will accept 20mm and 50.8mm
and remove the units (and double bonus if mm is displayed in the result).
However if someone can input "20mm" and "2 inches" and your program doesn’t convert, it is generating nonsense output, and you lose all your marks!

**Question 5. “No program is an island, entire of itself”… part 1, accepting inputs.**

My sample was 40 lines in total. Extra file-processing logic could easily add 20 or 30 lines and it may need rewriting at Question 7. By contrast, a cmdlet just needs a couple of parameter attributes.

Given a CSV file with headings “Diameter” and “height” a user could run     
`Import-CSV <path>`     
which will generate objects with properties that match the parameters of `Get-CylinderSize`.    
Piping those objects into the cmdlet will put property-values into parameters with a matching name or alias, if those parameters are tagged `ValueFromPipelineByPropertyName`, so the task in question 5  can be done with      
`Import-CSV <path> | cyl`  

**Question 6. “No program is an island” part 2. Usable outputs**    
If you *print* output, no one is going do the *prayer-based parsing* needed to extract individual
fields from it. But if you output *objects* the properies are simply *there*.

The cylinder objects that cmdlet outputs can be processed by PowerShell's built-in `Measure-Object` cmdlet, like this:     
`Import-CSV test.csv | cyl | Measure-Object -Property volume -Maximum -Minimum -Average`     
Because the command-line knows that `Get-CylinderSize` commands will output `cylinder` objects, names for the property in `Measure-Object` will tab complete.

**Question 7. The evolving customer requirement part 1**
It took me about 2 minutes to modify the program - (and rather longer to get it into the blog post) the new version looks like this  
{% highlight c# %}
using System;
using System.Management.Automation;

namespace cmdlet
{
    public class cylinder
    {
        public double Diameter { get; set; }
        public double Height { get; set; }
        public double Volume { get; set; }
        public double SurfaceArea { get; set; }
    }

    [Cmdlet(VerbsCommon.Get,"CylinderSize",DefaultParameterSetName ="HandD")]
    [OutputType(typeof(cylinder))]
    [Alias("cyl")]
    public class GetCylinderSizeCommand : PSCmdlet
    {
        [Parameter(Mandatory = true, Position = 0, ValueFromPipelineByPropertyName = true, ParameterSetName ="HandD")]
        [Parameter(Mandatory = true, Position = 0, ValueFromPipelineByPropertyName = true, ParameterSetName ="HandV")]
        [Alias("Stroke")]
        [ValidateRange(0,1e32)]
        public double Height { get; set; }

        [Parameter(Mandatory = true, Position = 1, ValueFromPipelineByPropertyName = true,  ParameterSetName ="HandD")]
        [Parameter(Mandatory = true, Position = 1, ValueFromPipelineByPropertyName = true, ParameterSetName ="DandV")]
        [Alias("Bore")]
        [ValidateRange(0,1e32)]
        public double Diameter { get; set; }

        [Parameter(Mandatory = true, ValueFromPipelineByPropertyName = true, ParameterSetName ="HandV")]
        [Parameter(Mandatory = true, ValueFromPipelineByPropertyName = true, ParameterSetName ="DandV")]
        [ValidateRange(0,1e32)]
        public double Volume { get; set; }

        protected override void BeginProcessing() {}

        protected override void ProcessRecord()
        {
            double radius;
            double endArea;
            if (MyInvocation.BoundParameters.ContainsKey("Diameter")) {
                radius   = Diameter / 2 ;
                endArea  = Math.PI * radius * radius ;
            }
            else {
                endArea  = Volume / Height;
                radius   = Math.Sqrt((endArea / Math.PI));
                Diameter = 2 * radius;
            }
            if (! MyInvocation.BoundParameters.ContainsKey("Height")) {
                Height   = Volume / endArea ;
            }
            if (! MyInvocation.BoundParameters.ContainsKey("Volume")) {
                Volume   = Height * endArea ;
            }
            WriteObject(new cylinder {
                Diameter = Diameter,
                Height   = Height,
                Volume   = Volume ,
                SurfaceArea = (Math.PI * Diameter * Height ) + 2 \* endArea
            });
        }
        protected override void EndProcessing() {}
    }
}
{% endhighlight %}

The cmdlet isn’t absolutely guaranteed to be future-proof against everything,
but it’s more capable of evolving than a console app would be. Everything which worked
before still works but the user can now use, for example  
`cyl -d 10 -v 1570.8`

It took three lines to define the `volume` parameter and the help automatically updated.  
Allowing Height & Diameter, Volume & Diameter, or Volume or Height, but not all
three of them together meant adding three further lines. 
Output hasn’t changed – it’s still a cylinder object, and if input is piped in with a `volume` column that will now be processed.

**Question 8 evolving requirements part 2**

For one time use technicans can format output by piping into the Format-table command    
`| ft Diameter, Height, @{f="n2";e="volume"}, @{f="n2";e="SurfaceArea"}`

This specifies 2 places of decimals; .NET copies Excel's number formatting so if users are 
familiar with Excel format strings, they can be used in place of “n2”, for example:
 “`#,###.00`” would display with two places of decimals and thousand separators.

If everyone wants the same format, a format specification for cylinder objects could override PowerShell’s default formatting: the screen shot below shows the results before and after applying  formatting.

![Default formatting compared with custom formatting](/assets/bb78ff6b580f367529b3fc06e4ddff99.png)

Because the formatting is in an XML file there is **no need to change the C\# to change how the content is presented ** Potentially users could have their own column-order, labels, or
number format(s). The example changes all three from the defaults.

### Conclusion.
Would a console app have done *anything* better? Would it take less effort to create and maintain? And this is just a simple app, there would be more benefit in an app one with many command line options, or multiple kinds of output.   
I don't know how anyone writing in .NET for the command-line could spend more than a few minutes looking at cmdlets and say "Nah, I'll stick with console apps".     
