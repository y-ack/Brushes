## == drawing engine ==

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

`bs_debug_log(_VERSION)`
`Lua 5.1`\
list of members exposed to lua context: https://gist.github.com/y-ack/a0a8517cf8e450c86d101e87614e26b2
##### undocumented functions?
 - `bs_circle_average` :: ?
 - `bs_rect_average` :: ?
 - `bs_left` :: maybe constant for canvas leftmost (0)
 - `bs_top` :: maybe constant for canvas topmost (0)

### canvas border behavior
![drawing past X=0 and Y=0 clamps?](https://user-images.githubusercontent.com/12588017/147865348-d0f54663-70ed-452b-bd66-81a597f86c23.png)\
drawing to x<0 and y<0 may call main but draw clamped position?

## === per-stroke randomness ===

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

## === performance ===
### dab frequency
`main()` gets called **100 times** between each input coordinate.
for simple brushes that want to update each pixel this means 100x the processing.
it's thus recommended to check the distance from the last draw before doing work:
```lua
local lastDrawX = 0
local lastDrawY = 0
local firstDraw = true
function main( x, y, p )
  if not firstDraw then
    local distance = bs_distance(lastDrawX - x, lastDrawY - y)
    if distance < 1.0 then return 0 end
  end
  
  -- ... brush work ...
  
  lastDrawX, lastDrawY, firstDraw = x, y, false
  return 1
end
```
a distance threshold of 1.0 *will* introduce error, however (light red pixels): ![image](https://user-images.githubusercontent.com/12588017/147887810-b72f9714-60ef-407d-800f-88a3f0db57b8.png)\
2 pixels in the region still differed from truth at the 0.5 distance threshold.
at 0.3 distance threshold, no pixels in the region were affected. one pixel visible at the end of the stroke differed.

### redundant drawing
when looping over individual pixels and drawing consistent dabs, large parts of the draw region will overlap.
let's skip the overlap region for efficiency:
```lua
  -- WARNING: subtly incorrect (because it uses old subpixel/fractional part)
  for j = y1, y2 do
    -- does this row may overlap previous drawing!
    if j >= oy1 and j <= oy2 then 
      if x1 > ox1 then     --   :#[#####: ]
        x1 = ox2 -- START at old end
      elseif x1 < ox1 then -- [ :#####]#:
        x2 = ox1 -- END at old start
      else -- same bounds as before, SKIP row entirely
        x2 = x1 
      end
    end 
    for i = x1, x2 do
      -- (per-pixel processing here)
      bs_pixel_set(i, j, 255, 255, 255, 128)
    end
  end
```
in testing, this cut drawing an 80x80 pixel dab from 3500ms preview time to around 1840ms (preview time).

however, it is subtly incorrect, and some pixels at the edge of the stroke will differ from true rect drawing behavior.
this is because `x1 = ox2` copies the old subpixel value as well, resulting in rounding error at the next location (compounded by using a distance threshold for dabs)

instead, let's use the old bound in whole pixels and copy the subpixel value:
```lua
local function copy_frac(a,b)
  return math.floor(a) + (b % 1)
end
```
```lua
  for j = y1, y2 do
    if j>=oy1 and j<=oy2 then if x1>ox1 then x1 = copy_frac(ox2,x1) elseif x1<ox1 then x2 = copy_frac(ox1,x2) else x2 = x1 end end 
    for i = x1, x2 do
      -- (per-pixel processing here)
      bs_pixel_set(i, j, 255, 255, 255, 128)
    end
  end
```
a demonstration can be found at https://github.com/y-ack/Brushes/blob/main/RedundantDrawSample.bs

dab distance threshold only:\
![distance threshold only](https://user-images.githubusercontent.com/12588017/147891780-6bf37c4d-a5c1-4225-a74d-1d8aaf33c854.png)\
skipping overlap regions:\
![skipping overlap avoids a lot of redundant drawing](https://user-images.githubusercontent.com/12588017/147891766-23f08401-982a-4941-882d-16d09c842b66.png)\
(more opaque indicates redundant draws)\
**TODO**: this isn't perfect either? there are still redundant draws. why? rounding? there also vertical 'bands' of redundant drawing when `i==x1`, which is a very scary bug-like symptom.


### lua performance
general lua performance tips also apply:\
https://springrts.com/wiki/Lua_Performance

 - use `local` for all variables
 - localizing library functions can also help: 
 ```lua
 local floor = math.floor
local function int_mod(a,b)
  return floor(a % b)
end
```
  - however, y noticed little effect when localizing e.g. `bs_pixel_set`
 - in my tests, attempting to cache simple calculations was slower due to introducing a branch for the cache check. caching more complex results may prove fruitful, but avoid it for simple parameter retrieval and a few mul/div operations.
