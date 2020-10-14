# Cubloc CT1720 PLC Shutter actuator

This code is created as a part of my own home automation system which is currently being replaced with a KNX shutter actuator.  
This system was running about more than one year as my primary source to control 11 shutters.

## Some highlights
* Each shutter up & down timings can be programmed.
* There is a "dead" zone between going UP & DOWN to avoid blowing out your relay with EMK. (500 ms). For EACH shutter!
* Event driven with ladder interrupts & serial interrupts.
* Touch screen support to open or close each individual shutter or open / close ALL shutters.
* RS232 support to open or close each individual shutter or open / close ALL shutters.
* RS232 serial interrupt feedback that shows the status of a individual shutter (with offset of 1%) depending on timings.
* RS232 & physical push buttons can be used simultaneously.
* Touchscreen will be dimmed after 1 minute. First touch will light up the backlight, second touch will control the elements on screen.

This code mimics a full blown Shutter Actuator like you would buy for a home automation system.

# Hardware

The hardware that is used for this shutter actuator is a `CT1720` from comfiletech. A `CT1721` also works.
Comfiletech has a complete datasheet that contains the information for this PLC. This can be found starting at Chapter 12: Cutouch  

* [PLC hardware](http://comfiletech.com/embedded-controller/controller-with-touch/cutouch/ct1721c-mono-lcd-touch-cubloc-i-o/)
* [Datasheet & Basic code](http://comfiletech.com/content/cubloc/cublocmanual.pdf)

The PLC including the relays (Schneider 12V relays) where powered through a 2A - 12V DC power supply.  
For the output wiring, it is not required to use a buffer IC as the `CT1720` can deliver enough power to control a 12V relay. 
Even driving 11 at the same time.

## Hardware schematic for the shutter actuator

The full system was powered through a 12V - 2A schneider power supply. This includes the PLC, push buttons & all relays.

## Real life picture of the hardware

### Input

All inputs are described in the file: [shutterIO.inc](ShutterIO.inc)  
For each room two buttons were located to control the shutter for that particular room.  
The event inputs (56, 58, ...) where used for `UP`. The uneven inputs (57, 59, ...) where used for `DOWN`

Example:

```
Usepin 56,In,OmhoogAchterkamer ' Used for `UP` in bedroom 1 
Usepin 57,In,OmlaagAchterkamer ' Used for `DOWN` in bedroom 1
Usepin 58,In,OmhoogSlaapkamer  ' Used for `UP` in bedroom 2
Usepin 59,In,OmlaagSlaapkamer  ' Used for `DOWN` in bedroom 2
Usepin 60,In,OmhoogKeuken ' Used for `UP` in kitchen
Usepin 61,In,OmlaagKeuken ' Used for `DOWN` in kitchen
```

### Output

All outputs were wired like the same according to the inputs.
All outputs are described in the file: [shutterIO.inc](ShutterIO.inc)  
It is following the same format as the inputs.
The event inputs (24, 26, ...) where used for `UP`. The uneven inputs (25, 27, ...) where used for `DOWN`

Example:

```
Usepin 24,Out,RelaisOmhoogAchterkamer ' Used for `UP` in bedroom 1 (Relay)
Usepin 25,Out,RelaisOmlaagAchterkamer ' Used for `DOWN` in bedroom 1 (Relay)
Usepin 26,Out,RelaisOmhoogSlaapkamer ' Used for `UP` in bedroom 2 (Relay)
Usepin 27,Out,RelaisOmlaagSlaapkamer ' Used for `DOWN` in bedroom 2 (Relay)
Usepin 28,Out,RelaisOmhoogKeuken ' Used for `UP` in kitchen (Relay)
Usepin 29,Out,RelaisOmlaagKeuken ' Used for `DOWN` in kitchen (Relay)
```

# Software

The editor to upload & edit the basic & ladder code is Cubloc Studio which can be found here: [Support & software](http://comfiletech.com/pages/support.html).  
The code exists out of two different programming languages: `Basic` & `Ladder`.  
The most critical code is located in the Ladder diagram which is basically controlling the inputs, the output relays as well the EMK logic for each shutter.

The `ShutterIO.inc` allows you to name certain inputs which can be found in the ladder diagram.


## Ladder software

### Ladder input logic.

The ladder input logic is sometimes hard to read. Especially the status calculation which uses WMOV, WINC & WADD on different data objects. The funny thing is, it is just to calculate a shutter percentage!
This section will explain on what does what.

You have basically three sections in the ladder screen.

* Push button input logic
* Relay output logic
* Shutter status calculation (This is done in ladder because why not. In the basic code it was also possible)


#### Push button input logic.

For each room the same ladder code is used. This means for Ladder, duplication! 

![Bedroom 1](images/ladder_input.PNG)

Every row contains the logic for one specific button. The one starting with `P56` is the one for the UP. The one starting with `P57` is the one for down.  
Lets start with the first row. 

* P56 = Push button for Shutter `UP` & P57 = Push button for Shutter `Down`. This is explained in section: [Output](#output) & [Input](#input)
** Due they are put serial, it is never possible to push `UP` & `DOWN` at the same time. Basically if the shutter goes up & you press down. The shutter will stop & the EMK deadzone will kick in which disables the buttons for 500 ms.


* T0 : This is the Timer 0 which represents the status of the `UP` part of the shutter.  
** T0 is required when the shutter is fully up (or down) it disconnects itself. Else the shutter actuator stays under power.    
* T1 : This is the Timer 1 which represents the status of the `DOWN` part of the shutter.  
** T1 is more of a guard check to disconnect the relay if the opposite relay is turned anyway. (To avoid damage to the shutter motor)    
* T2 : This is the Timer 2 which represents the EMK. If the relay is turned from "ON" to "OFF", this gets activated. Meaning it is not possible for the input to be sent to T0.  
** T2 actives when the relay goes "OFF"

* M56 is the simulated push button through RS232 AND for the touchscreen. This acts as a physical button.

> This logic is also duplicated for the "DOWN" but reversed as the UP button. For each room, the timers gets increased by `3`. All the info can be found under: [shutter-timers.info](shutter-timers.info) on what they mean.

This logic has been fully tested for over one year, with simultaneous up & down presses etc, ... So its pretty robust. 

#### Output logic

For each room the same ladder code is used for the relays as well. So again duplication.

![Bedroom 1](images/ladder_output.PNG)

Once again, every row is used for one typical relay. This is for the same room as we mentioned on the Push button input logic part.  
Lets start with the first row.

* T0 : This is the timer that is set on the inputs, when this actives `P24` will be attracted & the shutter relay for `UP` will activate.  
* P57 : In case the push button is pressed to go down, `P24` will turn off. This is the same for the duplicated `M57` merker.  

* TAOFF T2, D200. In the D200 variable, the EMK current status is reset on negative flank (OFF to ON).
* RSTOUT T0 -> When the relay turns off, the timer status gets reset of T0 as well for M56 & M57. 
* M56 & M57, this resets the merker for the RS232 & the touchscreen interface. (On the basic code)

This output code is straight forward. If input is actived, enable relay. When the relay gets turned off, it resets some parameters.

#### Status calculation logic.


## Basic software

 







