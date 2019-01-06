---
title:        Linear Advance
description:  Controllo della pressione dell'ugello per migliorare la qualità di stampa

author: Sebastianv650, Sineos
category: [ features, developer ]
---

<!-- ## Background -->

In condizioni predefinite, il movimento dell'asse dell'estrusore viene trattato allo stesso modo degli assi XYZ. Il motore dell'estrusore si muove in proporzione lineare rispetto a tutti gli altri motori, mantenendo esattamente lo stesso profilo di accelerazione e i punti di avvio / arresto. Ma un estrusore non è un sistema lineare, quindi questo approccio porta, molto ovviamente, al materiale extra che viene estruso alla fine di ogni movimento lineare.

Prendi come esempio il comune cubo di prova. Anche con la migliore calibrazione, gli angoli di solito non sono nitidi, ma sembrano "goffi". Il riempimento solido superiore mostra la rugosità in cui la direzione di stampa cambia sui perimetri. Questi problemi sono minori o addirittura impercettibili a basse velocità di stampa, ma diventano più evidenti e problematici all'aumentare della velocità di stampa.

La calibrazione del flusso può essere d'aiuto, ma ciò può portare a un'estrusione insufficiente quando si creano nuove linee. Alcuni slicer includono un'opzione per terminare l'estrusione prima di aver completato il percorso, ma questo aggiunge più complessità al codice G e deve essere ricalibrato per temperature e materiali diversi.

Siccome il problema è pressione, `LIN_ADVANCE` elimina l'estrusione dagli altri assi per produrre la pressione corretta all'interno dell'ugello, adattandosi alla velocità di stampa. Una volta che il Linear Advance è regolato correttamente, i bordi goffi e il riempimento solido ruvido dovrebbero essere quasi eliminati.
<br>
# Vantaggi

- Migliore precisione dimensionale grazie alla riduzione dei bordi goffi.
- Sono possibili velocità di stampa più elevate senza alcuna perdita di qualità di stampa, a condizione che l'estrusore sia in grado di gestire i cambiamenti di velocità necessari.
- La qualità di stampa visibile e tangibile viene aumentata anche a velocità di stampa inferiori.
- Nessuna necessità di valori di accelerazione e jerk elevati per ottenere spigoli vivi.
<br>

# Note speciali per v1.5

## Changelog

- K è ora un valore significativo con l'unità [mm di compressione del filamento necessaria per 1 mm/s di velocità d'estrusione] o [mm/mm/s].
- Carico ridotti per gli stepper l'ISR,in quanto non vi sono più necessari calcoli. Invece, l'estrusore funziona con uno spostamento di velocità fisso durante la regolazione della pressione. Pertanto questa versione viene eseguita più velocemente.
- LIN_ADVANCE ora rispetta le limitazioni hardware impostate in Configuration.h, ovvero jerk dell'estrusore. Se le correzioni della pressione richiedono regolazioni più rapide di quelle consentite dal limite dello strappo dell'estrusore, l'accelerazione per questo spostamento di stampa è limitata a un valore che consente di utilizzare la velocità dello strappo dell'estrusore come limite superiore.
- I movimenti di regolazione della pressione non portano a un estrusore rumoroso come nella v1.0: l'estrusore ora funziona a velocità regolare invece di oscillare tra i multipli della velocità di stampa dell'estrusore.
- Questa operazione dell'estrusore scorrevole e il rispetto dei limiti di jerk garantiscono che non vengano saltati passi dell'estrusore.

## Nuovo valore K richiesto

Poiché l'unità di K è cambiata, è necessario ripetere la procedura di calibrazione K. Vedi il prossimo capitolo per i dettagli. Mentre i vecchi valori v1 per PLA potrebbero essere tra 30-130, ora puoi aspettarti che K sia intorno a 0,1-2,0.

## LIN_ADVANCE può ridurre l'accelerazione di stampa

Nella v1, se K era impostato su un valore elevato che non poteva essere gestito dalla stampante, la stampante stava perdendo passi e/o usando tutta la sua potenza di elaborazione per eseguire i passi dell'estrusore. Nella v1.5, questo è gestito molto più intelligentemente. `LIN_ADVANCE` ora controllerà se è in grado di eseguire i passi secondo necessità. Se la velocità dell'estrusore necessaria supera il limite del jerk dell'estrusore, riduce l'accelerazione di stampa per la linea stampata su un valore che mantiene la velocità dell'estrusore entro il limite.

Mentre molto probabilmente non si imbatteranno nelle stampanti direct drive con filamenti come il PLA, ciò avverrà molto probabilmente sulle stampanti bowden, in quanto hanno bisogno di valori K più alti e quindi di adattamenti della velocità più rapidi. Se questo accade in un modo che non vuoi accettare, hai le seguenti opzioni:
- Controllare l'impostazione del jerk dell'estrusore. Se hai la sensazione che sia impostato su un valore molto conservativo, prova ad aumentarlo.
- Mantenere l'accelerazione dell'estrusore bassa. Questo può essere ottenuto abbassando l'altezza del layer o la larghezza della linea, ad esempio.
- Tieni K il più basso possibile. Forse puoi accorciare il tubo Bowden?

## A note on bowden printers vs. LIN_ADVANCE

A quite common note during development of `LIN_ADVANCE` was that bowden systems (and especially delta printers) are meant to be faster due to the lower moving mass. So lowering the print acceleration as described above would be an inadequate solution. On the other hand, bowden systems need a pressure advance feature the most as they usually have the most problems with speed changes.

Well, `LIN_ADVANCE` was developed because I (Sebastianv650) wasn't satisfied by the print quality I got out of my direct drive printer. While it's a possible point of view to say that lowering acceleration on a bowden printer (which is meant to be fast) is a bad thing, I see it differently: if you would be satisfied with the print quality your bowden setup gives you, you wouldn't be reading about a way to do pressure corrections, isn't it? So maybe in this case the highest possible top speed is quite useless.. Bowdens are an option to keep moving mass low and therefore allowing higher movement speeds, but don't expect them to give the same precise filament laydown ability than a direct drive one.
It's like painting a picture: try to paint with a 1m long brush, grabing the rear end of the handle which is made from rubber. Even if you try to compensate for the wobbly brush tip (which is basically what `LIN_ADVANCE` does), it will never be as good as using a usual brush.
On the other hand, if you only need to have a part printed fast without special needs in terms of quality, there is no reason to enable `LIN_ADVANCE` at all. For those prints, you can just set K to 0.

# Calibration
## Generate Test Pattern
Marlin documentation provides a [K-Factor Calibration Pattern generator](/tools/lin_advance/k-factor.html). This script will generate a G-code file that supports determining a proper K-Factor value.
The generated G-code will print a test pattern as shown in the following illustration:


![basics](/assets/images/features/lin_advance/k-factor_basics.png)


Beginning with the chosen `Start value for K`, individual lines will be printed from left to right. Each line consists of 20 mm extrusion using `Slow Printing Speed` being followed by 40 mm of `Fast Printing Speed` and finally being concluded by another 20 mm of `Slow Printing Speed`.

For each new line the K-Factor will be increased by the `K-Factor Stepping` value,  up to the provided `End Value for K`.

<br>

### General considerations for the Test Pattern Settings

 - [Bowden](http://reprap.org/wiki/Erik%27s_Bowden_Extruder) extruders need a higher K-Factor than direct extruders. Consider a `Start value for K` of around 0.1 up to an `End Value for K` of around 2.0 for LIN_ADVANCE v1.5 or around 30 up to an `End Value for K` of around 130 for v1.0.
 - The best matching K-Factor to be used in production depends on.
	 - Type of filament. Extremely flexible filaments like Ninjaflex may not work at all.
	 - Printing temperature.
	 - Extruder characteristics: Bowden vs.  direct extruder , bowden length, free filament length in the extruder, etc.
	 - Nozzle size and geometry.
 - The extruder's steps/mm value has to be [calibrated precisely](http://reprap.org/wiki/Triffid_Hunter%27s_Calibration_Guide#E_steps). Calibration is recommended at low speeds to avoid additional influences.
 - Minimize Backlash caused by gears [geared extruder](http://reprap.org/wiki/Wade%27s_Geared_Extruder) or by push fittings. As it will not influence the K-Factor, it can lead to strange noises from the extruder due to the pressure control.

> Repeat the calibration, if any of the above parameters change.

<br>

## Evaluating the Calibration Pattern

The transition between `Slow Printing Speed` phases and `Fast Printing Speed` phases are the points of interest to determine the best matching K-Factor. Following illustration shows a magnified view of a line where the K-Factor is too low:


![low](/assets/images/features/lin_advance/k-factor_low.png)


| Phase | Description |
|:--:|--|
| 1 | `Slow Printing Speed` |
| 2 | Beginning of the `Fast Printing Speed`. The pressure build up in the nozzle (= amount of extruded material) is lagging behind the acceleration of the print-head. This results in too little material and a starved line. Towards the end of this phase, the pressure is catching up to its intended value. |
| 3 | Extrusion and print-head movement are in sync. The nominal line width is extruded |
| 4 | Beginning of the deceleration towards `Slow Printing Speed`. Here the opposite of Phase 2 can be observed: The pressure decrease in the nozzle is lagging behind the deceleration of the print-head. This results in too much material being extruded |
| 5 | Beginning of the `Slow Printing Speed` phase. Still the pressure in the nozzle is not in sync with the intended extrusion amount and the line is suffering from over-extrusion. |
| 6 | `Slow Printing Speed` has stabilized. |

A too high K-Factor essentially reverses the above picture. The extruded amount will overshoot at the start of an acceleration and starve in the deceleration phase.
<br>
> The test line, which has a fluid and barely visible or even invisible 
> transition between the different speed phases represents the best 
> matching K-Factor. 

<br>

# Setting the K-Factor for production

## Considerations before using Linear Advance

 - Some slicers have options to control the nozzle pressure. Common names are: *Pressure advance*, *Coast at end*, *extra restart length after retract*. Disable these options as they will interfere with Linear Advance.
 - Also disable options like *wipe while retract* or *combing*. There should be almost no ooze, once the proper K-Factor is found.
 - Recheck retraction distance, once Linear Advance is calibrated and working well. It may even be as low as 0, since pressure control reduces the material pressure at the end of a line to nearly zero.
 
## The following considerations are no longer a problem with LIN_ADVANCE version 1.5
 - This feature adds extra load to the CPU (and possibly more wear on the extruder). Using a communication speed of 115200 baud or lower to prevent communication errors and "weird" movements is recommended.
 - The print host software should be using line numbers and checksums. (This is disabled by default e.g. in Simplify3D)
 - Theoretically there should be no "extra" movements produced by `LIN_ADVANCE`. If extra movements were produced, this would tend to increase wear on more fragile parts such as the printed gears of a Wade extruder.


## Saving the K-Factor in the Firmware
If only one filament material is used, the best way is to set the K-Factor inside `Configuration_adv.h` and reflash the firmware:

    /**
    * Implementation of linear pressure control
    *
    * Assumption: advance = k * (delta velocity)
    * K=0 means advance disabled.
    * See Marlin documentation for calibration instructions.
    */
    #define LIN_ADVANCE

    #if ENABLED(LIN_ADVANCE)
    #define LIN_ADVANCE_K <your_value_here>

## Adding the K-Factor to the G-code Start Script
[G-code Start Scripts](http://reprap.org/wiki/Start_GCode_routines) are supported by various slicers. The big advantage of setting the K-Factor via this methods is that it can easily be modified, e.g. when switching to a different material.
The K-Factor is defined by adding the command `M900 Kxx` to the end of the start script, where *xx* is the value determined with the above test pattern.

The following chapter briefly describes where to find the relevant setting in popular slicers.

**Note 1:**

With the G-code Start Script method, the feature still needs to be activated in the firmware as described in [Saving the K-Factor in the Firmware](#saving-the-k-factor-in-the-firmware). It is recommended to set `#define LIN_ADVANCE_K` to 0, which effectively disables the hard-coded firmware value. In this case only the K-Factor set via the start script is used.

**Note 2:**

The shown G-code Start Scripts are individual to each printer and personal taste. This is only intended to demonstrate where the K-Factor setting can be applied.

<br>
### Cura
*Settings* ---> *Printer* ---> *Manage Printer* ---> *Machine Settings*

![cura](/assets/images/features/lin_advance/cura.png)
<br><br><br>
### Slic3r
*Settings* ---> *Printer Settings* ---> *Custom G-code*

![slic3r](/assets/images/features/lin_advance/slic3r.png)
<br><br><br>
### Simplify 3D
*Edit Process Settings* ---> *Show Advanced* --> *Scripts* ---> *Custom G-code*

![s3d](/assets/images/features/lin_advance/s3d.png)
<br><br>
# Developer Information

## General Informations

The force required to push the filament through the nozzle aperture depends, in part, on the speed at which the material is being pushed into the nozzle. If the material is pushed faster (=printing fast), the filament needs to be compressed more before the pressure inside the nozzle is high enough to start extruding the material.

For a single, fast printed line, this results in under-extrusion at the line's starting point (the filament gets compressed, but the pressure isn't high enough) and over-extrusion with a blob at the end of the line (the filament is still compressed when the E motor stops, resulting in oozing).

For a complete print, this leads to bleeding edges at corners (corners are stop/end points of lines) and in extreme cases even gaps between perimeters due to under-extrusion at their starting points.

## Version 1.5
Version 1.5 handles the pressure correction in a slightly different way to reach the following goals:
- respect extruder jerk
- ensure a smooth extruder movement without rattling
- reduce load inside stepper ISR

It does this by calculating an extruder speed offset within the planner for the given segment. If we have a true, linear acceleration then this will execute the needed advance steps just in time, so we have reached all our needed advance steps just when the acceleration part is over. As Marlin uses an approximation for the acceleration calculation inside the ISR, this is not perfectly true, we will come to this later.
If the needed speed offset exceeds the allowed extruder jerk, the print acceleration for this segment is reduced to a lower value so extruder jerk is not exceeded any more. This is comparable to the check Marlin does for each axis, for example if we have a print acceleration of 2000mm/s² but X axis max. acceleration is set to 500mm/s², print acceleration is reduced to 500mm/s². During the trapezoid calculation, `LIN_ADVANCE` calculates the needed amount of advance steps at cruising speed and at final speed (=speed at and of the block).

When this block is executed by the stepper ISR, the extruder is set to a frequency which represents the calculated speed offset. The advance step is executed together with normal e steps. During the block runtime, the pressure will be advanced until the precalculated needed extra step amount is reached or the deceleration is started. During deceleration we reduce the advance step amount until we reach the amount for final speed or the block comes to an end.
This checks are necessary as Marlin uses an approximation for acceleration calculation as mentioned above. Therefore, for example, we might not reach the final pressure at end of move perfectly but that's not important as this error will not cumulate.

The next update might include improved handling of variable width moves. On variable width path it's possible that we need to deplete pressure during acceleration, for example when the next line is much narrower than the previous one. Therefore `LIN_ADVANCE` would need to check for the actual extruder direction needed.
Another case to be considered is the gap fill to the tip of a triangle: in this case Marlin will move on with a constant speed, but the extruder speed and therefore needed pressure gets smaller and smaller when we approach the tip of the triangle. We should adapt nozzle pressure with max. possible speed (extruder jerk speed) then.
As gap fill is done at quite low speeds by most slicers, we have to decide if that extra load is worth the effect. On 32bit boards where calculation performance is most likely not a problem, it makes sense in any case.

## Version 1.0
The `LIN_ADVANCE` pressure control handles this free filament length as a spring, where `K` is a spring constant. When the nozzle starts to print a line, it takes the extrusion speed as a reference. Additional to the needed extrusion length for a line segment, which is defined by the slicer, it calculates the needed extra compression of the filament to reach the needed nozzle pressure so that the extrusion length defined by the slicer is really extruded. This is done in every loop of the stepper ISR.

During deceleration, the filament compression is released again by the same formula: `advance_steps = delta_extrusion_speed * K`. During deceleration, `delta_extrusion_speed` is negative, thus the `advance_steps` values is negative which leads to a **retract** (or slowed) movement, relaxing the filament again.

The basic formula (`advance_steps = delta_extrusion_speed * K`) is the same as in the famous JKN pressure control, but with one important difference: JKN calculates the sum of all required advance extruder steps inside the planner loop and distributes them equally over every acceleration and deceleration stepper ISR loop. This leads to the wrong distribution of advance steps, resulting in an imperfect print result. `LIN_ADVANCE` calculates the extra steps on the fly in *every* stepper ISR loop, therefore applying the required steps precisely where needed.

For further details and graphs have a look into [this presentation, slides 7-9](https://drive.google.com/file/d/0B5UvosQgK3adaHVtdUI5OFR3VUU/view).

In Marlin, all the work is done in the `stepper.*` and `planner.*` files. In the planner loop, `LIN_ADVANCE` checks whether a move needs pressure control. This applies only to print moves, not to travel moves and extruder-only moves (like retract and prime).

In the `Stepper::isr` method, `LIN_ADVANCE` calculates the number of extra steps needed to reach the required nozzle pressure. To avoid missing steps, it doesn't execute them all at once, but distributes them over future calls to the interrupt service routine.

In v1.0 there is no check if the extruder acceleration, speed or jerk will exceed the limits set inside Configuration.h!
