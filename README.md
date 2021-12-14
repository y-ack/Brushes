# Brushes
repository for brushes (firealpaca/medibang paint lua brush scripts and krita brush presets)

[(notes on developing FireAlpaca brush scripts)](brushscript_notes.md)


## Sumi-WithPressure.bs
this is a modified version of a calligraphy-type brush from medi.
the original brush is clever in that it saves a list of points and then redraws the stroke,
as a post-processing step, to get a very specific thinning effect along the middle of the stroke.

this version uses pen pressure in real time, instead of faking it in post-processing.
the original behavior can be restored by unchecking "Use pressure"
 - very small (1-6) Pressure offsets (i.e. minimum value for the pressure curve) may improve the flavor
 - very small (11-12) Pressure multiplication factors may improve the flavor depending on your natural pressure
 - Pressure smoothing naively averages the last `n` values. small (6-8) values may smooth some jitter. extreme (20+) values may have strange effects.

|![sumi brush demo 2](Sumi_demo_2.png)|![sumi brush demo 1](Sumi_demo_1.png)|
|-|-|

## PixelSpraypaint.kpp
it's like the spraypaint tool of a certain simple builtin art program.
