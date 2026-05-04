## External Sensor Board

When we set out to create the OS-MPPT system, we asked ourselves: what might a club-scale MPPT be missing in the current environment? What design space was there to innovate on? What land has not yet been treaded? What time in the morning is reasonable to still call night? 

Well, we don't claim anything we do here is really first class in innovation, but hopefully we can crack open the design space for more users like us. For that purpose, we looked into creating an external sensor circuit. See, when a solar system utilizes multiple solar panels, these panels are typically connected into a single power delivery network that goes into a single MPPT (or multiple MPPTs across multiple small panel networks). What this does, however, is obfuscate important individual panel data from the user, as the MPPT can only really see the input as a collective. Ergo, since we are already shooting for some wireless networking between our devices, why not take a shot at a simple sensor circuit as well?

### The Pieces of the Pie

For this design, we wanted to shoot for minimal complexity, minimal size, and minimal difficulty in operation. Thus, rather than the hall-effect sensor/Voltage divider/ADC setup we have on the main board, we instead sought a more simple solution: One chip.

**INA228:** This incredible little thing can do everything from the I/V sensing we want to even temperature and charge estimation as well as integrated power calculation. We'll only utilize the I/V sensing, but this leaves open room to really add on complexity if you want *all* the readings. 

'What else do we need?' you may ask. Well, a lot of it is familiar. We'll use the same ESP32 module since it has integrated wireless comms, we'll have a USB_C port for programming and serial access, and we'll have a buck converter to pull power from the line in order to power the ESP32 & INA228, allowing us to avoid an external source like a battery. Notably, we will just use a buck converter going to 3.3V and no LDO. For one, we don't need a 5V rail, and, for two, a well designed buck will be stable enough to power our ESP32. 

Together, our board schematic became:

![External Sensor Board Schematic](/assets/schematicExternalBoard.png)

Which, taken to a layout became:

![External Sensor Board Layout](/assets/layoutExternalBoard.png)

