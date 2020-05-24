# PA-Calibration with LA-Test
This is a fast calibration test for Pressure Advanced. The idea is from [FHeilmann](https://github.com/FHeilmann).
![Screenshot Printed Pattern](screenshots/printed-pattern.png)

## Requirement
At first we have to add a gcode-macro to convert the M900 LA gcode to set the Pressure Advance value. Please add the following lines in your `printer.cfg`.
```
[gcode_macro M900]
default_parameter_K=0
gcode:
  SET_PRESSURE_ADVANCE ADVANCE={K}
```

## Generate Gcode
Now we use the [Marlin k-factor tool](https://marlinfw.org/tools/lin_advance/k-factor.html) to generate the gcode.
Here are my parameters I used to generate the pattern in the picture above.

![Screenshot Printer](screenshots/printer.png)
Change Hotend, Bed-Temperature and Retraction-Distance.

![Screenshot Print Bed](screenshots/print-bed.png)
Resize the bed to your size.

![Screenshot Speed](screenshots/speed.png)
Change the Retract Speed to your Retract-Speed.

![Screenshot Pattern](screenshots/pattern.png)
Ending Value max is 1 (Bowden). If you use a Direct-Drive printer, you can set the "Ending Value for K" to 0.25 and "K-faktor Stepping" to 0.01. If you like to print the Numbers per Line, check-in the "Line Numbering" (the numbers are horrible to remove from your buildplate).

![Screenshot Advanced](screenshots/advanced.png)
Change the "Extrusion Multiplier" for you filament.

Fill in a Filename in the field and click on "Generate G-code". Modify the Start-Gcode on the right side and click on "Download as file".

Now print the pattern and look at which line is most constant. Count the lines from below and multiply with the "K-factor Stepping" value. This is your perfekt Pressure Advance value.