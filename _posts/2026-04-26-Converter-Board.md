## DC:DC Converter and MPPT Power Stage

This article will go over the main converter board portion of our design, describing topics like the converter topology, operation, and parts as well as the additional peripherals included on the PCB to make the board function with the rest of the system. 

### Converter Design Choices

As mentioned in the overview article, our design implements a Buck Converter. The panel capacity we designed for was of such capacity that during the assumed operating period (aka hours when we actually get sun), the voltage on the panel input will always be greater than the battery charge voltage. When outside this range, the current drop that would result from boosting up to the battery charge voltage would result in a very slow battery charging rate. Ergo, rather than attacking the complexity of a buck/boost converter, we decided simply a buck converter would be sufficient for our purposes. That being said, our design constraints are as follows:

**Output (Buck) Voltage:** 14.4 V 
 - This is the recommended charging voltage for a 12V LiFe-Po battery.
**Output Voltage Ripple:** +/- 0.2 
 - A measure of how much our output voltage "wiggles," this will affect our choice of switching frequency.
**Max Charge (Through) Current:** 10A
 - The maximum current allowed to flow through the converter. This will mainly appear when picking parts (ensuring they can handle the large current) and when creating the PCB (as traces will need to be large enough to handle these currents).
**Output Current Ripple:** +/- 30%
 - A standard converter measure of how much the output current "wiggles," this will particularly affect our choice of inductor.
**Conversion Efficiency:** 95%
 - Power out/Power in. Switch Mode Converters can be incredibly efficient if they are designed well, and, if we are anywhere below this number, the MPPT won't really be worth the effort of adding. 

### A Good Place to Start

...would be to look at the structure of a Buck converter. Below are the "canonical" circuits for different voltage converters:
![Voltage Converter Circuits]({{ "/_assets/voltage-Converters.png" | relative_url }})

Now, these are all well and good, but there are a couple edits that we'll make. First, the switch. For the frequency that we'll need too actually switch at, we certainly can't use a physical switch. Instead, we'll use an electronic switch, specifically a MOSFET. These can then be driven by a pulse from our microcontroller. For that same purpose, we'll also swap out the diode for another electronic switch, driven in alternating time with our other switch (we'll call this pair High-Side and Low-Side, respectively). This will help manage the amount of charge flowing through the system and give an easy path to ground during the Low-Side half. 

One problem that may seem to arrise at this point is the timing; if we are driving both of these switches from the same microcontroller, we have to be *very* careful about synchronizing the two signals. Otherwise, we could end up with periods of shorting the output straight from the input. 

Ergo, rather than deal with the ickiness of synchronizing two clocking PWM signals, we will instead use a gate driver with built-in High-Side/Low-Side outputs. That way, all we need do is to feed a PWM into the chip and it'll split it into two alternating signals. Our specific part also includes built in "dead-time" to prevent overlapping "ON" signals that would cause the aforementioned shorting problem. The part in particular that we chose was the DGD1504 from Diodes Incorporated. For the purposes of a power converter, this chip also boasts operation up to 274V on the High-Side supply, allowing us to drive switches at a wider range of inputs (depending on physical limitations of the switches, of course).

#### The Other Bits

Let's take a short detour to talk about some of the other parts involved and the considerations for their choice. First, passive components. These are the resistors, capacitors, and inductors involved. Specific models don't matter so much as the general design specs (as many of these components are basic in nature). The important things, however, are as follows.

**Resistors:** These are pretty simple, stupid, easy, and we recommend you keep it simple, stupid, easy. Resistors have been around for a while and their manufacturing processes have reached incredible standards. Find a low tolerance of the right value and make sure the power dissipation can survive the rest of your system and you should be good. Remember, your system is only as strong and stable as the lowest power rating in your circuit!

**Capacitors:** When possible (as per values available), make these ceramic caps. These have much better performance characteristics for a variety of frequency ranges (important for a SMPS) and, more importantly, are not polarized. Where you might find trouble is your input and output capacitances, which, depending on your other design choices, may be of sufficient capacitances that you cannot find reasonable ceramic caps (this is in the 100uF+ range). This is when you may resort to using an aluminum electrolytic cap (colloquially known as "trash-can" caps, both for their looks and performance). These are polarized, which means they can only operate with the proper polarization (orientation of voltage across them). Because we are using a switch-mode device, we must be very careful about the application of polarized caps, particularly looking at the ripple voltage rating. However, if you stick to ceramic caps and watch out for the max voltage ratings, you should be fine.

