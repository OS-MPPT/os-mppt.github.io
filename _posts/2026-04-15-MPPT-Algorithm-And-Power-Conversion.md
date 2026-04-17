## MPPT Charge Controllers: An Overview

This article's purpose is to serve as an elaboration on some of the details given on our home screen. Here, we will look into some deeper questions on what a physical MPPT Charge Controller looks like and some of the systems and subsystems that make them work.

### System Overview

Our design is broken into essentially three main "components." First, the Main Board, which has the microcontroller that computes the Maximum Power Point and operates the DC:DC converter. Second, the External Sensor Board, which has a current and voltage sensor alongside a microcontroller that relays panel characteristics back to the Raspberry Pi. Finally, the Digital Database, which consists physically of a Raspberry Pi and a Router who together relay information from the External Board and the Main Board to a wirelessly hosted database for the user to access.

Each of these Subsystems will get their own dedicated article. For now, we we'll look at more general terms.

## A Quick Vocab Lesson

You may have a vague idea of some of the terms at play by now, but let's review.

**MPPT** : Maximum Power Point Tracker. It's all in the name, but we'll elaborate on this later.
**Vmpp** : Voltage at Maximum Power Point. A panel statistic describing the optimal operating voltage.
**Impp** : Current at Maximum Power Point. A panel statistic describing the optimal operating current.
**Voc** : Open Circuit Voltage. Essentially, this is the voltage that can be read by measuring the panel's output when it's not connected (hence, open circuit). 
**Isc** : Short Circuit Current. The current that would flow through the wires if you connected your panel positive output directly to the panel negatic output. As a safety note, don't try to test this for yourself. Instead, search for your panel model online and read the datasheet from the manufacturer.
**Pmpp** : The Power at Maximum Power Point. This is equivalent to Vmpp * Impp, and it's this that the MPPT tries to look for.

Some other terms that can be important, and might be referenced in some of these articles:
**DC:DC Converter** : A device that takes in one Direct Current voltage and outputs another Direct Current Voltage
**AC** : Alternating Current, like wall power. Think of a sinusoid signal, where the current *alternates* according to some frequency.
**DC** : Direct Current, like battery power. This is what will be coming off of your solar panel. This doesn't necessarily mean the value is unchanging, just that the *direction* isn't *alternating* (a reversed direction, such as connecting positive to your ground input and your negative to your positive input, is called Reverse Polarity and usually results in broken electronics and expensive smoke).
**Ground (GND)** : A neutral reference point for a circuit. Voltages will be measured as a difference from this reference. This reference is typically assumed to be 0V. While this *can* refer to Earth Ground (like if you just stuck a rod into the soil), it doesn't always. Notably, our design does not feature a Earth Ground reference.

## Solar Club

You may be wondering, how exactly does an MPPT (a.) find the Maximum Power Point and (b.) use that to give me efficient power? Great question! First, we must learn a little about solar panels. 

Solar panels have a defined Voltage and Current range they operate in. This range is defined, at the base, by the chemical makup on individual cells and, at a larger scale, by the combination and construction of multiple cells into one full panel. Recalling Ohm's Law, V = I * R, we can step through different load values and find the respective Voltage and Current at each of these different steps, all while maintaining a constant Solar Irradiance. Doing so will allow us to produce the panel's power curve, a plot of I vs. V that characterizes the panel's operating range (aka, the spectrum of different Voltages and Currents that the panel operates at). This plot will look like a line with an ever-so-slightly descending slope up until it reaches a "knee" where it then rapidly falls off until I = 0. Alternatively, you can plot the Power against Voltage, giving you something that looks more like a ramp that suddenly falls off, although maintaining a similar "knee" in the top right corner. It's this "knee" that we'll be interested in as this is where we find our Maximum Power Point.

![Solar Panel Power Curve](os-mppt.github.io/_assets/IV_curve_solar_cell.png)

Here's where some of our vocab comes in handy: That y-intercept? That's where the Voltage is ~0 and the Current is Isc, the short circuit current (as there is no voltage difference across a short). The x-intercept? That spot I mentioned where I = 0? That's the Voc, the open circuit voltage (as no current flows through an open). Further, the Vmpp and Impp are the X & Y coordinates to the Pmpp on the plot, which exists somewhere on the curve of the "knee." We'll use a mix of mathematical algorithms and electronics to get out circuit as close to this elusive point as we can.

## Your MPPT And You
Alright, we know what we're looking for now. How do we actually get there? Now is when we should look at the beating electrical heart of a MPPT Charge Controller (Hint: it has to do with controlling the charge). Can you guess it? Here's a hint, it was one of the vocab words. That's right! The DC:DC Converter. Recall, a DC:DC Converter takes one voltage in and gives another out. In order to maintain conservation of energy, the current will additionally change (with some power loss according to the efficiency of the converter).

The class of DC:DC Converter we will be looking at are called Switch Mode Power Supplies. This is because their topology (the arrangement of electrical elements that allow them to operate) involves active switches to manage the flow of current in order to generate the desired output voltage. There are three main types: **buck**, **boost**, and **buck/boost**. **Buck converters** take a higher voltage down to a lower voltage, **Boost converters** take a lower voltage to a higher voltage, and **Buck/Boost converters** will regulate lower voltages up and higher voltages down to a desired voltage. The typical downside of using a SMPS is, of course, the switching noise, which is not desirable in a DC system. This can be mitigated by just designing a good converter. The upside is that SMPSs are high efficiency and, depending on the particular setup, their characteristics can be changed on the fly (a surprise tool we'll use later). As we'll elaborate on in another article, our design chose to use a buck converter. 

Here's where a little electrical engineering comes into play, so bear with me. Recall the power curve and how it's a plot of I vs. V. Recall also Ohm's law, V=IR. Rearranging terms and doing rise-over-run, notice how the slope of the power curve is actually 1/R. This is glossing over some of the finer physics of electricity, but the panel will present a voltage (based on the sunlight hitting it) which then *drives* a current based on the *resistance* of the load attached (more particularly, the **Impedance** of the load). Put all of this together and you find the following:

*We can ride along the power curve by 'presenting' a different impedance to the panel*

How do we present different impedances to the panel? The DC:DC Converter! Whenever we change the operating characteristics on the DC:DC Converter, we are also causing the converter to present a different impedance to the panels. If we then watch the power coming from the panels by simply taking a look at the incoming Current and Voltage (and using P = I*V), we can then change the converter characteristics and watch if the power has increased or decreased. We then rinse and repeat, stepping along the power curve until we can hover around the maximum power point. 

## Putting it all together
We've got a charge controller, which both steps the voltage up or down to our desired voltage and presents an alternating impedance to the panel in order to walk towards the maximum power point. All we have to put it together and fill out some of those loose ends. First, some sensors at the MPPT input and output to take samples of the current and voltage at both ends. Next, lets add a microcontroller (basically a small computer) to take in those values and calculate (using an algorithm) and adjustment that takes us closer to the maximum power point. Finally, we send that adjustment to the converter. Loop it all together and you have a system that tracks incoming and outgoing power, adjusts its conversion to maximum power from the panel, and gives you that power at a voltage level you want. 

The next steps from here and to get into the nittier, grittier details on how our design in particular operates. The next few articles will focus on the following: the main board converter design, the external board sensor design, the MPPT algorithm, and the wireless database design. Hopefully, by the end of it, you too will be able to construct and iterate on our design. Best of luck!


Due to a plugin called `jekyll-titles-from-headings` which is supported by GitHub Pages by default. The above header (in the markdown file) will be automatically used as the pages title.

If the file does not start with a header, then the post title will be derived from the filename.

This is a sample blog post. You can talk about all sorts of fun things here.

---

### This is a header

#### Some T-SQL Code

```tsql
SELECT This, [Is], A, Code, Block -- Using SSMS style syntax highlighting
    , REVERSE('abc')
FROM dbo.SomeTable s
    CROSS JOIN dbo.OtherTable o;
```

#### Some PowerShell Code

```powershell
Write-Host "This is a powershell Code block";

# There are many other languages you can use, but the style has to be loaded first

ForEach ($thing in $things) {
    Write-Output "It highlights it using the GitHub style"
}
```
