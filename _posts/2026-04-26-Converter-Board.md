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
![Voltage Converter Circuits](/assets/voltage%20Converters.png)

Now, these are all well and good, but there are a couple edits that we'll make. First, the switch. For the frequency that we'll need too actually switch at, we certainly can't use a physical switch. Instead, we'll use an electronic switch, specifically a MOSFET. These can then be driven by a pulse from our microcontroller. For that same purpose, we'll also swap out the diode for another electronic switch, driven in alternating time with our other switch (we'll call this pair High-Side and Low-Side, respectively). This will help manage the amount of charge flowing through the system and give an easy path to ground during the Low-Side half. 

One problem that may seem to arrise at this point is the timing; if we are driving both of these switches from the same microcontroller, we have to be *very* careful about synchronizing the two signals. Otherwise, we could end up with periods of shorting the output straight from the input. 

Ergo, rather than deal with the ickiness of synchronizing two clocking PWM signals, we will instead use a gate driver with built-in High-Side/Low-Side outputs. That way, all we need do is to feed a PWM into the chip and it'll split it into two alternating signals. Our specific part also includes built in "dead-time" to prevent overlapping "ON" signals that would cause the aforementioned shorting problem. The part in particular that we chose was the DGD1504 from Diodes Incorporated. For the purposes of a power converter, this chip also boasts operation up to 274V on the High-Side supply, allowing us to drive switches at a wider range of inputs (depending on physical limitations of the switches, of course).

#### The Other Bits

Let's take a short detour to talk about some of the other parts involved and the considerations for their choice. First, passive components. These are the resistors, capacitors, and inductors involved. Specific models don't matter so much as the general design specs (as many of these components are basic in nature). The important things, however, are as follows.

**Resistors:** These are pretty simple, stupid, easy, and we recommend you keep it simple, stupid, easy. Resistors have been around for a while and their manufacturing processes have reached incredible standards. Find a low tolerance of the right value and make sure the power dissipation can survive the rest of your system and you should be good. Remember, your system is only as strong and stable as the lowest power rating in your circuit!

**Capacitors:** When possible (as per values available), make these ceramic caps. These have much better performance characteristics for a variety of frequency ranges (important for a SMPS) and, more importantly, are not polarized. Where you might find trouble is your input and output capacitances, which, depending on your other design choices, may be of sufficient capacitances that you cannot find reasonable ceramic caps (this is in the 100uF+ range). This is when you may resort to using an aluminum electrolytic cap (colloquially known as "trash-can" caps, both for their looks and performance). These are polarized, which means they can only operate with the proper polarization (orientation of voltage across them). Because we are using a switch-mode device, we must be very careful about the application of polarized caps, particularly looking at the ripple voltage rating. However, if you stick to ceramic caps and watch out for the max voltage ratings, you should be fine.

**Inductor:** You're guaranteed to have an inductor in your system, which some say is unfortunate, as inductors are probably the worst passive component to have to account for. Because inductors are just coils of wire, they have some non-trivial series resistance. Since they also have a bunch of parrallel loops, they'll have some non-trival capacitance associated with them as well. Additionally, inductors suffer from frequency dependent operation, so check what frequency you expect your buck converter to operate at and pick a part that has good operation at that frequency range. Finally, as a coil of wire, you need the wire to be thick enough to withstand the current you're trying to pump through your circuit. You optimally want the size of your traces on your PCB to be the limiting factor here, *not* your power converter components.

Next, you have the active components, namely...

**Electronic Switches:** Or in our case, MOSFETs. This are weird, semiconductive, non-linear components with a too-short history but too-detailed engineering behind them. However, you can, for the most part, think of these as electronically controlled switches. The switch only allows current flow if a sufficient voltage is presented at the aptly named Gate pin. We will feed our PWM at the gate pin to turn them on/off according to our MPPT algorithm which will drive the buck converter operation towards the MPP. Things to look out for in a MOSFET is the power rating. Similar to the inductor, you want these to have sufficient power rating so as to not burn out during their active mode section. Additionally, look out for their breakdown voltage. These switches won't serve us very well if they break down and let current pass just because the input was a little highing than expected. Next, watch for their rise/fall time ratings. You want the switch to be able to turn on within the duration of the pulse, otherwise your converter falls in efficiency. An important thing to note for our converter is that we need a decently high voltage to be able to turn on the gate, given our intended input/output range of voltages. The ESP32 alone can't do more than 3.3V logic, which is why we implement a...

**Gate Driver:** This device will take an incoming PWM and reconstruct high- and low-side PWMs at a sufficient voltage to drive the gates. Some of this I have already mentioned earlier, such as the built-in dead time. Really, simplicity is what's best to look out for here. Our specific gate driver is the DGD1504. 

Putting all these components together in our synchronous buck configuration, this becomes the following:

![Synchronous Buck Converter Block Diagram](/assets/OS-MPPTE_SynchBuck_Block.drawio.png)

### The Microcontroller, Brain of the Beast

Microcontrollers are basically (and really, literally) small computers, which can be used as a low-profile, intelligent operator of an electronic system. You have lots of freedom in your choice in microcontroller, with something as simple as an arduino nano probably having enough power to operate this system. All you need is I/O for an off-board connection and to get readings from your ADC, I/O capable of outputing a PWM, and processing power to compute MPPT (a surprisingly low bar). For our use, we chose an **ESP32** specifically the **ESP32-C3-WROOM-2** model. This particular model has an included antenna for WiFi/MQTT functionality. Although it has a reduced number of total I/O pins, we don't actually use all that many anyway!

#### Peripheral Detour

Something I've referenced and something you may be wondering about is how we are actually getting readings about voltage and current from the input/output of our system to our microcontroller and then off the board. Here's where we get a couple more devices on board. The primary things we need are an (1.) an accurate current reading, (2.) and accurate voltage reading, and (3.) a way to translate those readings into the digital world. We'll use the following:

**TMCS1108 Hall-Effect Sensor:** This TI part uses some fun electrical magic to translate a current going through the change into an analog voltage output. We've got two of these, one for input current and another for the output current.

**Voltage Divider:** Wait a minute, this isn't an actual chip? Which is nice! Keeps the complexity low. All we need to do is to put two resistors in series and pull a reading from the middle to get a voltage reading relative to the voltage on the actual bus. We'll make the resistors high value for two reasons. One, it will reduce the power dissipated through the voltage divider, thus keeping our system efficiency balanced. Two, it will reduce the voltage to one that can be read by the analog inputs of our last device (as it can only take a certain amount of voltage on its pins).

**ADS1015 ADC:** Last but certainly not least, this piece will take the analog readings coming from the voltage divider and hall-effect sensors and translate them to digital readings accessible by our ESP. This particular part is convenient because we can fit all four (voltage in/out, current in/out) readings onto just one copy of the part. We'll have two things to account for when polling values from the ADC: first, adjusting the measured voltage up to the actual voltage by multiplying back the ration of our voltage divider. Second, converting the voltage reading for the hall-effect sensors back into a current value (which we can do with a simple in-code calculation in correspondence with what the TMCS1108 datasheet defines).

### The Algorithm. The MPPT Algorithm

The exiting bit! Or more accurately, the anti-climatic bit. Really all this comes down to is the following:

```c
void MPPT_algorithm() {
  if (Pold > inputPower) {
    DC_direction *= -1;
  }

 
  else {
    if (dutyCycle <= 0) {
    dutyCycle = 1;
    } 
    else if (dutyCycle >=250) {
      dutyCycle = 249;
    }
  }
}

void setPWM() {
    if (dutyCycle > 0 && dutyCycle < 250) {
    dutyCycle += DC_direction;

    Pold = outputPower;
    ledcWrite(PWM_PIN, dutyCycle);
  }
}
```

That's it. That's a basic MPPT algorithm. Now, what's actually going on here? Well, in the main loop, we poll the ADC to get I/V for input & output, we calculate power through P=IV, and then we run the above. If the previous power measured is larger than our current, we know that we are going in the wrong direction and should "turn around." Additionally, we don't want to step "out-of-bounds" of what our PWM can output (255 ish being the limit for Arduino I/O). Otherwise, we can just incrementally step in the direction of increasing power, whether than be increasing PWM or decreasing PWM. As with all things, you can find our full code in our linked Git repo. Do note that ours is much more complex with experimental variable stepping included as well as a bunch of protection checks, like reverse polarity, overcurrent, etc.

### Everything That Is, Was, and Will be

Putting everything we've talked about together, our end-schematic looks like the following:

!(Main Board Schematic)[]