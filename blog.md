# Saving the fish with bunnies: how we used pixi.js for temporal mapping

# Saving the 🐟 with 🐰 : how we used Pixi.js for temporal mapping

Context: joint effort with Google, Oceana and Skytruth
- a total 150,000 vessels
- up to 100,000 points rendered at the same time
- 4+ years of data
 our partner Skytruth delivers highly efficient binary vector tiles, where points are cluster both in time and spatially.

Google BigQuery


I want to focus here on how we achieved the animated "heatmap" style of this map, with reasonably good performance, a maintainable, high-level codebase, and still some sanity left.

![](animation.gif)



### Early explorations: Canvas 2D and Torque

Vizzuality already implemented an animated heatmap with Canvas 2D, this time not to save fish, but forests, in project called <a href="http://www.globalforestwatch.org/">GlobalForestWatch</a>. This for instance shows the tree cover loss in the southern Amazonia from 2001 to 2015:

![](forest.gif)

Canvas 2D is an API that allows drawing shapes, text and images into a drawing surface. But really what it does is just, basically, moving pixels around. Even if it only relies on the CPU to render graphics, it can achieve solid performance, at least with that kind of pixellated rendering style. How? Prepare a typed array containing all pixel values (<a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8ClampedArray">`Uint8ClampedArray`</a>) and dump them into your canvas all at once (<a href="https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/putImageData">`putImageData()`</a>). Done.

![](https://cloud.githubusercontent.com/assets/704210/16413800/89181720-3d34-11e6-92ce-a3ffc7786b72.gif)

We started early prototypes of GlobalFishingWatch with that approach, but we were aiming at a different rendering style. While the pixellated style works well for analysis, we were looking for something that would convey the idea of a "pulsating" activity, highlighting fishing seasons variations and their potential impact on the environment.

![](heatmap.gif)

To achieve this effect, we need to blend many many times "brushes strokes" (a tiny picture with a radial gradient gradually transparent) into the canvas, each brush representing here a point in time and space where a vessel caught fish. We can then play with the size and opacity of the brush to encode fishing activity.

Can we achieve that with canvas 2D? Well, there are many techniques to try to make canvas 2d rendering faster: shadow canvas, batch drawing API calls, grouping drawing commands by color/opacity, avoid sub-pixel rendering, staring at your screen with creepy eyes, <a href="https://www.html5rocks.com/en/tutorials/canvas/performance/">etc</a>.

<a href="https://github.com/Vizzuality/GlobalFishingWatch/pull/332">I wish I knew earlier</a>, but the truth is that this battle is basically lost in advance. There's just no way a CPU can handle moving that much brushes above 5 fps on a desktop, let alone a mobile CPU.


### A true "heatmap" style: Torque ?


![](bbva-small.gif)


Naturally an awesome contender when you think of spatiotemporal animations: <a href="https://carto.com/torque/">CARTO's Torque</a> (<a href="http://www.vizzuality.com/projects/BBVA">which we used in the past</a>)

Torque works by mashing SQL tables into preprocessed <a href="https://github.com/CartoDB/torque-tiles">tilecubes</a>, then rendered into a good old canvas 2D. It can deliver relatively good performance with typical datasets, because there is a step of spatial and temporal aggregation done 'offline'. It is an amazingly smart way to tackle the problem but unfortunately it comes "by nature" with two major flaws:
- you can't do any client side changes to the rendering, which makes interaction niceties such as mouse hover effects impossible to achieve;
- interacting with the time attributes is very limited. Changing the time span rendered requires changing your SQL query and doing a roundtrip with the server. But we had high ambitions:

![](time.gif)

Additionally, Torque is not able to render, at the time of writing, anything else than points. We needed lines to render the vessel trajectories, so going with Torque would have forced us to use a separate solution for vessel tracks (pain and suffering ahead).

### All hail WebGL

So after a good deal of hesitations and hair pulling, this day finally happened:

![](./webgl-canvas.png)

We dropped all hope of using canvas 2D, and went with a shiny new WebGL implementation instead.

WebGL is tapping into the raw power of Graphic Processing Units (GPUs). Things a GPU is good at. Heating up the room. Drawing a million times the same thing, insanely fast. Sounds like it could work for us. We want to draw heatmap brushes to represent fishing vessels, a lot of times (they are almost the same, but size, opacity and tint will vary, so we might have to be a bit smart about this).

In terms of programming, WebGL is a wildly different beast than canvas 2D. It allows your puny JS code to talk to your GPU through OpenGL Shading Language (GLSL), a language similar to C or C++. GLSL is very, very terse, difficult to debug, and hard to maintain. _[whispered, sobbing voice] I'm afraid of GLSL. Can I go home now ?_.

It turns out there are a bunch of very smart(er) people out there, willing to do the hard work for us, which is: exposing GLSL functionality to a high level API in JavaScript, typically using some stage hierarchy paradigm, think `container.addChild(sprite); sprite.x = 42;` (oooh the glorious days of Flash, may you rest in peace). Those smart people build what's called <a href="https://html5gameengine.com/">2D rendering libraries for the browser</a>: Phaser, Pixi.js, HaxeFlixel, etc. Those libraries are typically used to develop games, but why not for tracking illegal fishing on a map as well?

So how do you pick a rendering engine? Project's activity on GitHub, quality of the documentation, reputation ? Yeah, sure, but more importantly: BUNNIES !

![](bunnies-small.gif)

Yes, people use bouncing bunnies to measure a 2D graphics engine performance, why? Since Pixi.js can render and animate <a href="http://www.goodboydigital.com/pixijs/bunnymark/">tens of thousands of bunnies</a> on a blank canvas without breaking a sweat, it will surely render and animate tens of thousands of fishing vessels on a map.

So we can stay in the nice High-level-land of JavaScript, while leaving the GLSL logic to tested and proven codebases. We focus on highly maintainable, abstract code that ties well with our application model (Redux) and the rest of the UI rendering logic (React), while all the "dirty work" is done by the rendering engine.

As an added bonus, Pixi.js can fallback to rendering into a canvas 2D, for older setups (you might be surprised for instance, to learn that the Intel HD 3000 GPU, which equips a lot of 2010-2012 macbooks, <a href="https://twitter.com/alteredq/status/783240214584107008">is on Chrome's WebGL blacklist</a>). We even discovered, as a late minute surprise, that the performance with this mode was quite tolerable.

### Tinting and switching brush styles

So WebGL is fast and all, but this doesn't mean that we don't have to be a little bit smart. When talking to a graphics API such as WebGL, what's expensive is a lot of times not what's happening within the GPU realm, but rather on the CPU side.

At the lowest level, we have **draw calls**. A draw call is a set of instructions prepared by the CPU and sent to the GPU, and is usually one of the main bottlenecks when doing accelerated graphics.

One of the ways to reduce the number of draw calls is to limit the number of textures we are going to use. Take for example this animation, showing French and Spanish vessels across the borders of the respective countries <a href="http://www.journaldelenvironnement.net/article/france-et-espagne-se-marchent-sur-la-zee-et-ses-ressources,34897">Exclusive Economic Zones</a> (french). We need different colors to distinguish Spanish and French vessels:

![](tinting.gif)

Instead of using two textures for the two brush colors, we'll have them share the same texture, using a technique called texture atlasing:

![](brushes.png)

So when rendering a **sprite**, instead of setting a texture per rendering style, we'll just crop a portion of that big **spritesheet**, the relevant part for both hue and rendering styles (the solid circles on the right are used at higher zoom levels).

We end up with a single geometry, containing all sprites of the scene, using a single texture, which amount to a single draw call. The only thing left to update are the sprites texture offsets, positions and sizes (in the GPU world, vertices UVs and transforms).

### Compute graphical attributes 'offline'

So how do we actually update those positions and sizes ? We have to get them from the raw tiles data, which contains the latitude and longitude of fishing points, as well as two variables encoding the amount of fishing happening, and convert them to point and positions on the screen.

This is a costly operation, all happening on the CPU:
- latitude and longitude, which represents a position on the surface of a sphere (so, angles), have first to be projected on a 2D plane, aka "world coordinates" (using the Web Mercator projection), then to screen coordinates.
- fishing activity must be translated to a sprite size and opacity, which also depends on the current zoom level.

The strategy here was to precompute these values right after a tile gets loaded, instead of doing it at each step of the animation. There are future plans of doing that calculation step on the backend. Which, by the way, is how standard vector tiles work: rather than meaningful geographical data, they carry tile-relative geometrical data, or drawing instructions if you prefer).

### Map / canvas interaction

Handling projection, use world coordinates

Do NOT render while panning or zooming


### Pooling

One other trick we used is probably almost as old as computer graphics: pooling.

Each distinct point in our fishing map is represented by a sprite, a JavaScript object (before being fed to WebGL). It's tempting to instanciate the number of objects you need at each animation frame, but one quickly realizes that:
- creating those objects is costly: framerate will drop
- deleting those objects, something done automatically by the garbage collector*: a lot of frames will be skipped at unpredictable times

 _here's an <a href="https://medium.com/@_lrlna/garbage-collection-in-v8-an-illustrated-guide-d24a952ee3b8">illustrated guide</a>)_

Creating all those objects, my friends, is the cost of using higher level abstractions, because WebGL has no such concept of objects. You trade a little bit of performance, for code expressiveness.

So to bypass that performance limitation you will:
- make an estimation of how many sprites you will need (here, depending on the time span selected and the viewport size);
- instanciate all those sprites at once, store them into a pool;
- at each frame, position on the canvas as many sprites as you need to render that frame;
- keep the leftover sprites just moving them off-stage instead of deleting them
- whenever time span or viewport size changes, instanciate more sprites if needed

### Rendering tracks

![](tracks.gif)

### Conclusion

alternative to Mapbox GL
- Google Maps
- custom tileset format, with a custom GIS pipeline


Phew, that was a journey. BTW, Vizzuality is looking for talented people to stop illegal fishing and deforestation by working on that kind of challenges.
