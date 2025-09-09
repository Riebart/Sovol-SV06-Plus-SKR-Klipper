# DrGhetto's TMC Driver Tuning Guide for Klipper

```
SOURCE: https://github.com/MakerBogans/docs/wiki/TMC-Driver-Tuning
```

This guide is about configuring some values for the chopper in TMC stepper drivers in SpreadCycle mode. This will not worth for StealthChop.
We are going to perform static tuning for specific stepper motors.
Tuning reduces power dissipation in the stepper drivers and the motors and reduces noise. 
They will not perform any differently.

## Brief Background

TMC drivers do a pretty good job of adapting to the properties of a stepper motor but there are some chopper parameters which are static and have safe defaults some ways off the ideal values.
Tuning stepper drivers *properly* requires a test rig and and oscilloscope ([see this PDF](https://www.trinamic.com/fileadmin/assets/Support/AppNotes/AN001-SpreadCycle.pdf)). 
We are not going to do that, we are just going to set some chopper parameters to better values suited for the the motors we use.

## Prerequisites

We will need two things

1. Data sheets for each different motor type in your printer
2. The Trinamic Calculations sheet.

From the data sheet we need two figures. The resistance of the windings (given in Ohms) and the inductive of the windings (given in milli-Henrys mH).
Sometimes these are described as phase resistance and phase inductance. Whatever they are called they will be in ohms and Henries, and that's all you need.

You should get the TMC Calculation sheet for your stepper drivers from Trinamic, e.g. [TMC 2209](https://www.trinamic.com/products/integrated-circuits/details/tmc2209-la/).
The sheet is an XLS but if you don't have Excel, it works fine in Google Sheets. Just upload the file to your Drive and open it.

## Starting Point

Let's first make sure you have your stepper drivers set to sensible values.
There are two config sections in Klipper for each motor:

1. Stepper config: `[stepper_x]` where a value for Microsteps is found
2. TMC config: `[tmc2209 stepper_x]` which has specific settings for the TMC stepper driver

In the past, sensible defaults were `microsteps: 16` in the stepper config, and `interpolate: True` in the TMC config. People often set `hold_current` values in the latter in an effort to reduce power consumption when idle. 
We've since learned [these are bad](https://www.klipper3d.org/TMC_Drivers.html).
Every TMC config block should have `interpolate: False` and should not have a `hold_current` entry.
Typically you would then increase the number of microsteps in the stepper config block for the A/B motors and Z motors to a value between 64 and 256. 
64 seem safe. 
Note: This will make your motors noisier than with interpolate on (which interpolates to 256 microsteps), and TMC driver tuning is a way of reducing the noise again. I have found you can achieve noise levels similar to unoptimised drivers with interpolate on.

The extruder motor is a special case. For reasons covered later, there microsteps don't work particularly well here. This motor is not typically noisy so I would recommend leaving it at 16 microsteps.

A note about stealthchop on Z-drives motors: Z-drive motors can be quite noisy. They also don't move particularly fast, and the motors are considerably over specified for what is needed to move the gantry/bed. So a valid choice is to operate the Z drives in stealthchop. If you do this, make sure stealthchop is on always by placing `stealthchop_threshold: 999` in the TMC config block for each Z-drive, and *do not* perform the optimisation discussed in the next sections for those motors.
You might decide to optimise your motors to see how you go. 
You can always comment out the relevant settings and switched to stealthchop later.

## What we are changing

We are going to insert four config lines into the TMC config

1. `driver_TBL`: Blanking Time
2. `driver_TOFF`: Time to Slow Decay 
3. `driver_HSTRT`: Hysteresis start
3. `driver_HEND`: Hysteresis end

In a nutshell we are going to alter the timing of chopper cycles and the configuration of the chopper's hysteresis values.

For a full explanation of these settings, see the SpreadCycle Settings section of the [TMC 2209 datasheet PDF](https://www.trinamic.com/fileadmin/assets/Products/ICs_Documents/TMC2209_Datasheet_V103.pdf).

## Plugging stuff into the sheet

We are going to use the Chopper Parameters sheet in TMC's calculations sheet. It's the second sheet.
The first thing we do is plug appropriate values into the yellow cells in the sheet. 

1. fCLK: This is probably 12MHz, which is the TMC2209 default. It's not the clock of the board.
2. VM[V]: Put your power supply voltage here, e.g. 24.
2. TBL: Put 1 here, this seems to be a good value for all our steppers.
3. L[H]: The inductance from our motor data sheet. If it is 1.6mH then put in 0.0016
4. Rcoil[Ohm]: You can get this from your motor data sheet but if you're feeling fancy you can measure it with a meter.
5. Icoil(peak)[A]: The current we're going to run the motor at. Note this is peak, not the same as the `current` value in Klipper.
5. toff setting: This is a reactive setting, leave it at 3 to start with
6. CS: The driver sets this automatically based on current (and the sense resistor), we have to change this to simulate this behaviour as explained later

TMC's sheet will pop up some warnings if your values are in the danger zone. What we want are the values that appear in the blue section titled `Register values for CHOPCONF register bits` *not* the two uncoloured cells under `Desired value`. 

Once you have the basic numbers plugged into the sheet, the next thing you do is change CS manually until `RSENSE using VSENSE=1` at the bottom of the sheet is equal to our sense resistor, usually 0.110 Ohms for printer stepsticks.
We also want to keep CS fairly high if we want microstepping to work right, the sheet will warn you if it's too low.

When we're happy, we take the values in the blue section and set the Klipper `driver_HSTRT` and `driver_HEND` values appropriately, along with `driver_TBL` and `driver_TOFF` set to what is in the top of the sheet. Usually 1 and 3 respectively.
We may also have found that we've tweaked the driver current somewhat to hit the magic numbers.
In this case, we take the calculated value in `Icoil (RMS)[A]` and put that into `run_current` in the TMC config section.

## Example for an AB stepper motor

We are going to optimise for an OMC [17HM19-2004S1](https://www.omc-stepperonline.com/nema-17-bipolar-0-9deg-46ncm-65-1oz-in-2a-2-8v-42x42x48mm-4-wires-full-d-cut-shaft.html) motor. 
This is a 48mm 0.9 stepper which I use in a Voron 2.
The appropriate numbers are 1.45 ohm winding resistance and 4 mH inductance.
We will proceed based on 1 amp RMS current.

Let's leave TBL at 1, and TOFF at 3. We find through some trial and error that we can get the VSENSE=1 value to 0.110 (where it needs to be), with CS at 31 and the motor current `ICoil(Peak)[A]` at 1.38 Amps. 
The calculated cell named `Icoil (RMS)[A]` says that is 0.976 amps (we were shooting for around an amp motor current)
The numbers in the blue CHOPCONF register bits section will be 0 for HSTRT and 3 for HEND.

In sum, we now have the appropriate values and so the final TMC config block looks something like this:

```
[tmc2209 stepper_x]
uart_pin: PE7
interpolate: False
stealthchop_threshold: 0
sense_resistor: 0.110
run_current: 0.976
driver_TBL: 1
driver_TOFF: 3
driver_HSTRT: 0
driver_HEND: 3
```

## Example for an extruder motor

The popular LDO 36mm NEMA14 motors are a bit of an edge case because of the unusual properties of the motor.
Let's optimise for the 17mm variant popular in Mini Sherpas, the 36STH17-1004AHG. 
This motor has 10 Ohms of winding resistance and 6mH inductance.
We know this motor needs around 300mA of current but as in the previous example, let's not be wedded to a round number.

Plugging the numbers in we find that CS needs to be quite low, and the sheet will warn about values less than 16 not being good for micro stepping.
That's why there's not really any point driving this motor with a lot of microsteps.

We generally want to keep the chopper frequency at 40Khz and up because it will often run at 50% of this frequency, and 20Khz is the limit of human hearing. Use the opportunity to change the values of TBL and TOFF to see how this changes the chopper frequency. An appropriate solution would be:

```
run_current: 0.336 # results in CS 10
driver_TBL: 1
driver_TOFF: 3 # 41.7Khz max chopper frequency
driver_HSTRT: 0
driver_HEND: 3
```

This is a good time to note the reality of current settings.
The setpoint I use is CS=10 which is arrived at (by making RSENSE=1 value to 0.11) by setting the peak current to 0.475A, and that gives us the oddball figure of 0.336A RMS which is what we put into Klipper.
There's not a lot of resolution to CS here, so what about CS 9 and CS 11?
CS 9 can be achieved at 0.434A peak current (307mA RMS, so that's quite close!).
CS 11 would be 0.52A peak (368mA RMS) which is a bit high for this motor.

## Document your changes!

TMC driver optimisations are not valid for different motors.
We should make this clear by adding comments in the config, and also to show our working if we come back to this later.
I suggest adding a comment above your motor sections stating the model number and the specs which saves you from having to dig out the data sheet again.
For example:

```
#LDO 48mm 1.8 42STH48-2004AC - 1.4Ohm 3mH
[tmc2209 stepper_z]
uart_pin: PD10
uart_address: 0
interpolate: False
run_current: 0.764
sense_resistor: 0.110
stealthchop_threshold: 0
driver_TBL: 1
driver_TOFF: 3
driver_HSTRT: 1
driver_HEND: 3
```


















