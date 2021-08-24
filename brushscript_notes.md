=== drawing engine ===

firealpaca drawing seems to work something like this...
 - an individual dab is triggered by unique canvas position (determined by some min distance from prev dab)
 - for each dab, main() is called
  - lua script's main() returns, yielding control to the main application thread
 - processing continues until another dab position is available

this has implications...\
'continuous drawing' (sometimes called 'airbrush') mode is not possible.
looping within main will stall the program,
the api does not expose ways to listen for position changes in order to abort a loop,

but maybe with some very tricky lua something could yet be done...


=== per-stroke randomness ===

the original Sumi.bs used an interesting idiom:
it recorded dabs into a `pos = {}` array
```lua
    pos[point_count] = {}
    pos[point_count].x = x
    pos[point_count].y = y
    pos[point_count].rad = rad
```
while drawing only a rough visual stroke, then in `last()` 
did a post-processing step where it drew all the bristle lines over again.

it's clever, but at the same time it was awkward, 
and didn't necessarily behave the way one would expect from a preview.

each simulated bristle has random offsets and variation in other parameters,
generated at the end of the stroke.
in converting it to dynamically use pen pressure, this lead to another idiom:\
per-stroke random values.

per-dab (i.e. each main() call) randomness is common for brush effects,
but per-stroke noise is also an important idiom, 
and is expressed simply as "random values determined during the first brush call"

this 'first call initialization' is usually expressed as such:
```lua
function main(x, y, p)
  if first_call then
    init(x, y, p)
    first_call = false
  end
```
where `init()` will then go on to seed the rng (`math.randomseed(bs_ms())`)
and do other one-off determination.

here we also allocate the simple but important task of 
'producing randomness that should be coherent throughout the stroke'

thus the random line determination
```lua
for j = 0, bs_param1() do -- bs_param1() is # of lines/bristles to draw, i.e. for each bristle:
  local rnd = math.random(100) / 1000
  local pp = 0.40 + bs_param2() / 500 + rnd
  local p1, p2, p3 = c * pp, c * (1 - pp), 1 / pp
  local ofx, ofy = max_width * (math.random(100) / 100 - 0.5), max_width * (math.random(100) / 100 - 0.5)
```
becomes
```lua
function init(x, y, p)
  local i
  for i = 0, bs_param1() do
    line_rnd[i] = {}
    line_rnd[i].rnd = math.random(100) / 1000
    line_rnd[i].xrnd = math.random(100) / 100 - 0.5
    line_rnd[i].yrnd = math.random(100) / 100 - 0.5
  end
end
```
and the random calls at draw time are changed to refer to these consistent values\
(e.g. `local rnd = line_rnd[j].rnd`)
