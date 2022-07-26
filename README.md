# ArcadeScenecontrol
Tutorial and guide on the new Scenecontrol API for ArcadeZero v4
Document for the old API: https://github.com/Tempestissiman/ArcadeScenecontrol/tree/a30c13efc25b0a4a6e32a820b3dbb5c363564d71
日本語版: Coming soon

# Introduction
This document contains information necessary for both newcomers and those who have experience working with the old Scenecontrol API.

As usual the API requires knowledge of the scripting language Lua, and so this guide will assume some familiarity with it. If you're new or would like a refresher on the language, please refer to the official tutorial at: https://www.lua.org/pil/contents.html

####### Contribution: You can help translate this document to other languages!

# What's new?
The new API was made to address these issues with the old one:
1. Slow: The API relied heavily on user-written Lua code, even during runtime. Not only can this be slow, we risk a chance of it never being able to be ported to other platforms such as iOS.
2. Inflexibility: The register system was overly complex and made simple tasks such as multiple scenecontrols controlling the same object overly complex (which was the opposite of what it was made to do!)
3. Was overall ugly and unintuitive.

With that in mind, here are reasons to prefer the new system:
1. Completely deterministic: The new API rely on the concept of *Channels* (more on this later!). This only require a single execution of lua code, and also can be exported to other format that does not require lua!
2. Transparent: You have complete control over the new system, which allow for faster scripting.
3. Collaboration: It's easier to use other people's scripts!

Also unrelated to the core API, but the new update also bring to table some new features:
1. An editing window for scenecontrol. You can edit your events right inside ArcadeZero!
2. Expanded features, such as post-processing effects.
3. Built-in scenecontrol commands.

If that excites you, then let's waste no more time and get right into the tutorial!

## Warning for people familiar with the old system
It's totally possible to port all of your old scripts to the new system, but please note that things like coordinates and rotations might also need to be changed. Some changes were made to make editing them more intuitive and regular.

# Getting started
Tools you will need:
- ArcadeZero v4 or later, or any forks of ArcadeZero v4.
- A sufficiently good IDE / text editor such as VSCode is recommended.
- You should also install the "Vscode Arcaea File Format Support" Extension for VSCode (https://github.com/yojohanshinwataikei/vscode-arcaea-file-format), and the vscode-lua Extension.

# Part 1. Built-in commands
Let's start from the very basic, and use the built-in commands. We'll first familiarize ourselves with the new Scenecontrol editing window.

### The aff command syntax
You might already be used to writing out the scenecontrol commands by hand by now in the .aff chart file. If you aren't, here's the basic syntax:
```
#aff file:
scenecontrol(timing, scenecontrolType, argument0, argument1, argument2, ...);

e.g:
scenecontrol(1000, trackdisplay, 1000, 127);
```
- `timing` (generally) signifies when the effect takes place.
- `scenecontrolType` defines the type of this command.
- `argument0, argument1,...` define all the arguments, which essentially dictate exactly how the command will operate.

It's also possible to nest scenecontrol commands inside a timinggroup, which may or may not make a difference (it depends on the command type, of course!)

### Using the built-in commands
So out of the box, ArcadeZero supports these command types:
```
#aff file:
scenecontrol(timing, trackdisplay, duration, changeToAlpha);
scenecontrol(timing, hidegroup, duration, changeToAlpha);
scenecontrol(timing, enwidenlane, duration, changeToState);
scenecontrol(timing, enwidencamera, duration, changeToState);
```

- `trackdisplay` changes the transparency (or *alpha*) of the main track.
- `hidegroup` changes the transparency of all notes within the same timing group of the scenecontrol command.
- `enwidenlane` adds 2 more lanes to both side of the main track (Used in Pentiment, Arcana Eden and Testify)
- `enwidencamera` moves the camera to support enwidenlane.

Both `timing` and `duration` arguments are in miliseconds. `changeToAlpha` takes in a value from 0 to 255. `changeToState` takes in either 0, which means disable, or 1 which means enable the effect.

> Technically this is different from the official game's specification, but I deliberately chose to depart from it in order to favor consistency.

If you aren't used to these scenecontrol commands then I highly recommend writing these out on your own for a while. It's a good skill to have!

### The editing window
Let's now familiarize ourselves with the new editing window for scenecontrol. Open the chart, then click with the button with a star icon on the right side. You'll be greeted with a small window like the following:

<img src="https://i.imgur.com/oBUABzr.png">

On the top, you have:
- The Scenecontrol Type dropdown. Select the type to work with here.
- The Switch Field button. This toggles between separated fields for each arguments and one combined field. Just press it  a few time to see what happens.
- Refresh button. This refresh the list and reload any included scripts.

In the middle is the list of events. Right now it's probably empty for you, because your chart does not contain any event corresponding to the currently selected scenecontrol type. Let's add one by pressing the "+" button on any row. You should see a new one show up:

<img src="https://i.imgur.com/Im7es5n.png">

From here you can change the arguments to your liking!

Also note that the window only show scenecontrol events included in the currently active timing group. It can be easy to forget this and wonder why an effect is active even though your list shows nothing.

That's it for the basic! Now that we have a rough understand of how scenecontrol works, we'll start diving into the scripting part in the next section.

# Part 2. Scripting

### 2.1. Project structure
So far you've been working with the provided scenecontrol types. But the crux of customizing your Arcaea chart is being able to define your own scenecontrol type. Let's get started on that!

Start by navigating to your project folder (The one that includes the `Arcade` folder, and your music file, jacket art file and such). Create a folder named `Scenecontrol` (case sensitive), and within it create a file named `init.lua`.

`Scenecontrol/init.lua` is the script that's run by default by ArcadeZero upon loading your chart file. The `Scenecontrol` folder is also where you'll put all of your asset files (images and such) to be read by the script(s).

> For advanced users: you can also use `require` to split your lua code into multiple script files. The file names are not important! (except for the default `init.lua` of course)

Let's open the newly created lua file and start coding! However, for now we won't actually be defining our own scenecontrol type just yet, and focus on a concept much more fundamental: *Controllers* and *Channels* (we'll learn how to define your own type in the next part!)

### 2.2. Controllers & Channels - Level 1

>The Level 1 of Controllers and Channels will prime you to take anything that already exists within the scene and manipulate it to your liking!

Controllers are where your custom lua code will interact with in order to control objects in the scene. For example, you'll interact with a track controller to control the track, a sprite controller to control a 2D image sprite object, and so on. Interact here just means modifying it's properties, such as position, rotation, scale, color, etc.

So far it's the same as the old system! However instead of modifying the properties *directly* like the old system, you'll be doing it a bit more indirectly this time. Introducing: **Channels**.

So what is a channel? All channels have one job in common: they answer the question:
###### "It's now x miliseconds into the song. What's the value right now?"

Each property of a controller will have a channel attached to it, which will determine how the property will change at any point in time.

For now it might still be unclear on what this means. Let's look into a concrete example of actual lua code to illustrate this:

We'll begin by defining a controller. Let's just use an internal controller right now, and we'll worry about importing your own controllers later. We'll grab one by using `Scene`, which is where you'll grab most of your controllers (and also where you'll create most of your controllers):

```lua
-- init.lua
local controller = Scene.track --Grabs the track controller
```

Set the controller's position to somewhere else:
```lua
-- init.lua
...
controller.translationX = Channel.constant(1)
controller.translationY = Channel.constant(2)
controller.translationZ = Channel.constant(3)
```

Click the `Refresh` button on the scenecontrol edit window. This will reload your script, which told Arcade to move the track to the new position located at (1, 2, 3). Your track should be sitting at a new location which looks like this:

<img src="https://i.imgur.com/JK7SyQ5.png">

Here's a detailed explanation of what just happened:
1. `Channel.constant(value)` define a channel that will always return the value you passed in, regardless of time.
2. You took 3 of those channel and assigned it to 3 properties of a controller.
3. Now the controller will update its values based on the set channels, which we defined to be constant values of 1, 2 and 3.

Your script doesn't do anything interesting right now, but what you just wrote is incredibly fundamental to understanding the whole system. If you understand working with channels, you understand the new scenecontrol. So take your time here!

But if you're confident you got it, then let's move on and spice things up, by making the object move over time. For that `Channel.constant()` simply won't do. Entering: **Keyframe channels**.

A keyframe channel is defined by a series of **Keys**. If you have ever animated anything before then this concept will be familiar to you. For those that haven't, let's look at an example:

<img src="https://i.imgur.com/t7w0ezs.png">

This channel has 3 keys.
- The first key is at timing 0, and have value of 0.
- The second key is at timing 1000, and have value of 1.
- The third key is at timing 2000, and have value of 0.
You then "connect the dots" and that will define the output value at any point in time.

It's pretty simple to work with keyframe channels, actually. Let's code the channel described above in lua code:
```lua
--init.lua
local channel = Channel.keyframe()
channel.addKey(0, 0)
channel.addKey(1000, 1)
channel.addKey(2000, 0)
controller.translationX = channel
```

>That was just one way to write it, and also one that's the clearest to illustrate what's going on. You might want to write it this way to shorten it:
```lua
controller.translationX = Channel.keyframe().addKey(0, 0).addKey(1000, 1).addKey(2000, 0)
```

> You can reuse channels! The same channel can be assigned to multiple property, and changing one channel will change all properties reading from it. This is a good practice in my opinion as it means you only have to keyframe once to change multiple properties.
```lua
local channel = Channel.keyframe().addKey(0, 0).addKey(1000, 1).addKey(2000, 0)
controller.translationX = channel
controller.translationY = channel
controller.translationZ = channel
-- even completely unrelated property is fine
controller.colorR = channel
```

Refresh and see the result. The track should move in a linearly to another position at from 0ms to 1000ms, then back from 1000ms to 2000ms.

But if you don't want it to be move linearly? You'll want to use easings! Simply pass an additional string like this:
```lua
local channel = Channel.keyframe()
channel.addKey(0, 0, 'so')
channel.addKey(1000, 1, 'so')
channel.addKey(2000, 0, 'so')
```

>WARNING: 'so' here actually curves similar to si arcs (fast at first then slow in the end). It's the official specification that's wrong. Just one of those small things you'll have to be used with

You can also change the default easing when creating the channel. This is equivalent to the code block above:
```lua
local channel = Channel.keyframe().setDefaultEasing('so')
channel.addKey(0, 0)
channel.addKey(1000, 1)
channel.addKey(2000, 0)
```

Here's the list of all the easings supported. Each easing type can be written multiple ways.
| Name | Aliases | Shape |
| - | - | - |
| linear | l | <img src="https://i.imgur.com/RW6npU1.png" width=120em> |
| inconstant | inconst, cnsti | <img src="https://i.imgur.com/OtPvAGb.png" width=120em> |
| outconstant | outconst, cnsto | <img src="https://i.imgur.com/UYKymRq.png" width=120em> |
| inoutconstant | inoutconst, cnstb | <img src="https://i.imgur.com/D916zO3.png" width=120em> |
| insine | si | <img src="https://i.imgur.com/6FWfL7v.png" width=120em> |
| outsine | so | <img src="https://i.imgur.com/VDB347V.png" width=120em> |
| inoutsine | b | <img src="https://i.imgur.com/uprRIh9.png" width=120em> |
| inquadratic | inquad, 2i | <img src="https://i.imgur.com/qafhmH3.png" width=120em> |
| outquadratic | outquad, 2o | <img src="https://i.imgur.com/lPvQZiq.png" width=120em> |
| inoutquadratic | inoutquad, 2b | <img src="https://i.imgur.com/KrMCfGe.png" width=120em> |
| incubic | incube, 3i | <img src="https://i.imgur.com/pBtlUPe.png" width=120em> |
| outcubic | outcube, 3o | <img src="https://i.imgur.com/pBtlUPe.png" width=120em> |
| inoutcubic | inoutcube, 3b | <img src="https://i.imgur.com/2vsaAfH.png" width=120em> |
| inquartic | inquart, 4i | <img src="https://i.imgur.com/1gkdwj3.png" width=120em> |
| outquartic | outquart, 4o | <img src="https://i.imgur.com/1gkdwj3.png" width=120em> |
| inoutquartic | inoutquart, 4b | <img src="https://i.imgur.com/DpLpSKE.png" width=120em> |
| inquintic | inquint, 5i | <img src="https://i.imgur.com/lO6CS1R.png" width=120em> |
| outquintic | outquint, 5o | <img src="https://i.imgur.com/cjNq1sq.png" width=120em> |
| inoutquintic | inoutquint, 5b | <img src="https://i.imgur.com/0ElZdrQ.png" width=120em> |
| inexponential | inexpo, exi | <img src="https://i.imgur.com/CxM7iHY.png" width=120em> |
| outexponential | outexpo, exo | <img src="https://i.imgur.com/zSi35WU.png" width=120em> |
| inoutexponential | inoutexpo, exb | <img src="https://i.imgur.com/bYFbAOK.png" width=120em> |
| incircle | incirc, ci | <img src="https://i.imgur.com/XlLATi4.png" width=120em> |
| outcircle | outcirc, co | <img src="https://i.imgur.com/V7jXmBe.png" width=120em> |
| inoutcircle | inoutcirc, cb | <img src="https://i.imgur.com/Nq5JkvD.png" width=120em> |
| inback | bki | <img src="https://i.imgur.com/vL2NZps.png" width=120em> |
| outback | bko | <img src="https://i.imgur.com/YKBd78p.png" width=120em> |
| inoutback | bkb | <img src="https://i.imgur.com/JnSeIih.png" width=120em> |
| inelastic | eli | <img src="https://i.imgur.com/BsUF01a.png" width=120em> |
| outelastic | elo | <img src="https://i.imgur.com/IhlWlew.png" width=120em> |
| inoutelastic | elb | <img src="https://i.imgur.com/oULApFZ.png" width=120em> |
| inbounce | bni | <img src="https://i.imgur.com/fCHgebv.png" width=120em> |
| outbounce | bno | <img src="https://i.imgur.com/58pzeQD.png" width=120em> |
| inoutbounce | bnb | <img src="https://i.imgur.com/dhdjvX0.png" width=120em> |


> For fun! You can also turn extrapolation of a channel on. Extrapolation basically continues the curve beyond the keyframes you have added. I don't know why one would use this but it's here anyway
```lua
local channel = Channel.keyframe()
					   .setDefaultEasing('so')
					   .setIntroExtrapolation(true)
					   .setOuttroExtrapolation(true)
```

Now that you have mastered the basics of controllers and channels, changing any property to behave anyway you like is possible now! But the API provides much more useful functionalities that will simplify the process for you. We'll return to them at the Level 2 of Controllers and Channels.

For now let's actually set out to do what we intended to at the start: *defining a scenecontrol type*.

### 2.3 Adding a scenecontrol type

You can totally ignore the whole scenecontrol type concept, and just key everything in `init.lua`. But for reusability, defining the types is recommended.

Let's remind ourselves of what we have to do to achieve this:
- Specifying the name of our scenecontrol type (Feel free to choose your own here!)
- Specifying the arguments, how many we will use, and their names.
- Define the behaviour of the scenecontrol type.

2/3 of them are covered within a single function. The rest is what we just did in the previous chapter. Let's see it!

We'll make a simple scenecontrol type that will read in 3 numbers, and move the track from its current position to a new one specified by those 3.
```lua
local track = Scene.track
track.translationX = Channel.keyframe()
track.translationY = Channel.keyframe()
track.translationZ = Channel.keyframe()

addScenecontrol("mytypename", 3, function(cmd)
	local timing = cmd.timing
	local x = cmd.args[1] -- lua starts counting from 1 unlike other languages
	local y = cmd.args[2]
	local z = cmd.args[3]
	track.translationX.addKey(timing, track.translationX.valueAt(timing))
	track.translationX.addKey(timing + 500, x)
	track.translationY.addKey(timing, track.translationY.valueAt(timing))
	track.translationY.addKey(timing + 500, y)
	track.translationZ.addKey(timing, track.translationZ.valueAt(timing))
	track.translationZ.addKey(timing + 500, z)
end)
```

So what exactly happened?
1. We first defined the keyframe channels to add our keys into.
2. We defined a scenecontrol type named "mytypename". It has 3 arguments.
	This means our aff command will look like this: `scenecontrol(timing, mytypename, arg1, arg2, arg3`
3. We defined the behaviour of our scenecontrol type through a function. Basically this function is run for every scenecontrol event in the chart. Our function will read that command, and act accordingly, which is to add a keyframe to our channels.

>If you're familiar with Arcade macros, this pattern should seem familiar! The only difference is that we're taking in a argument to read the scenecontrol command's data.

Also note the use of  `channel.valueAt(timing)`. It's a handy function when working with keyframes.

This example also illustrates nicely the basic structure of a scenecontrol script
- Setting up the controllers,
- Assigning the channels,
- Then defining scenecontrol types which will fill in the keys.

Now that we successfully defined our type, open up Arcade, then open the scenecontrol edit window. You should see "mytypename" appear in the dropdown box now, and adding events to it should work like intended.

By the way, you might see small texts above the first row of the scenecontrol edit window. They denote the name of arguments in each column, and yes, you can change it!
```lua
addScenecontrol("mytypename", {"xpos", "ypos", "zpos"}, function(cmd)
	...
end)
```

`{"xpos", "ypos", "zpos"}` creates a list of strings will will be interpreted as names for arguments. Refresh scripts and check the window again!

>Events with zero argument also work. This code below correspond to the following aff command type: `scenecontrol(timing,mytypename);`
```lua
addScenecontrol("mytypename", 0, function(cmd) ... end)
-- or --
addScenecontrol("mytypename", {}, function (cmd) ... end)
```

By now you should be able to replicate a ton of effects already. But let's take it one step further, by knowing the in-and-outs of the `Scene`!

### 2.4. Scene

The `Scene` object serves 2 function: grabbing internal controllers, and creating new controllers. We have done the former already, but only with one type of them so far. It's not practical to describe in details every type of controllers in this guide, so please refer to the documentation for more information.

To get an idea of what you can do, here's the list of all internal controllers
| Path | Type | Description |
| - | - | - |
| Scene.gameplayCamera | CameraController | The main camera |
| Scene.combo | TextController | The combo text |
| Scene.score | TextController | The score text |
| Scene.jacket | ImageController | The jacket art |
| Scene.title | TextController | The song's title text |
| Scene.composer | TextController | The song's composer text |
| Scene.difficultyText | TextController | The difficulty display's text component |
| Scene.difficultyBackground | ImageController | The difficulty display's image component |
| Scene.hud | CanvasController | The canvas containing the pause button and info panel |
| Scene.pauseButton | ImageController | The pause button |
| Scene.infoPanel | ImageController | The info panel's background image |
| Scene.background | ImageController | The background image |
| Scene.videoBackground | SpriteController | The video background component |
| Scene.track | TrackController | The main track |
| Scene.track.divideLine01 | SpriteController | Dividing line between lane 0 and 1 |
| Scene.track.divideLine12 | SpriteController | Dividing line between lane 1 and 2 |
| Scene.track.divideLine23 | SpriteController | Dividing line between lane 2 and 3 |
| Scene.track.divideLine34 | SpriteController | Dividing line between lane 3 and 4 |
| Scene.track.divideLine45 | SpriteController | Dividing line between lane 4 and 5 |
| Scene.track.divideLines | Table (of SpriteController) | List of all dividing lines |
| Scene.track.criticalLine0 | SpriteController | Critical line of lane 0 |
| Scene.track.criticalLine1 | SpriteController | Critical line of lane 1 |
| Scene.track.criticalLine2 | SpriteController | Critical line of lane 2 |
| Scene.track.criticalLine3 | SpriteController | Critical line of lane 3 |
| Scene.track.criticalLine4 | SpriteController | Critical line of lane 4 |
| Scene.track.criticalLine5 | SpriteController | Critical line of lane 5 |
| Scene.track.criticalLines | Table (of SpriteController) | List of all critical lines |
| Scene.track.extraL | SpriteController | The left extra lane (lane 0) |
| Scene.track.extraR | SpriteController | The right extra lane (lane 5) |
| Scene.track.edgeExtraL | SpriteController | The edge of the left extra lane |
| Scene.track.edgeExtraR | SpriteController | The edge of the right extra lane |
| Scene.singleLineL | SpriteController | The left Memory Archive line controller |
| Scene.singleLineR | SpriteController | The right Memory Archive line controller |
| Scene.skyInputLine | SpriteController | The sky input line controller |
| Scene.skyInputLabel | SpriteController | The sky input label controller |
| Scene.skyInputLabel | SpriteController | The sky input label controller |
| Scene.darken | SpriteController | The darken sprite controller (used by internal trackdisplay type) |

> You might be wondering what the difference between ImageController and SpriteController is. To put it simply:
> - SpriteController can set their own sorting layer and sorting order, and ImageController can not.
> - You can control ImageController's position more easily with pivots and anchors. ImageController's sorting order and layer can only changed by parenting them to another canvas.

Instead of using internal controllers again however, let's instead try to create our own controllers here, and explore the different types of controllers this way. Again, you will need `Scene` to achieve this

| Method | Return type | Description |
| - | - | - |
| Scene.createSprite(string imgPath, string material = "default", bool newMaterialInstance = false) | SpriteController | Create an sprite from the path, with the specified material type |
| Scene.createCanvas(bool worldSpace = false) | CanvasController | Create a canvas, either displaying in world coordinates or screen coordinates |
| Scene.createImage(string imgPath, string material = "default", bool newMaterialInstance = false) | ImageController | Create an image from the path, with the specified material type |
| Scene.createText(string font = "default", number fontSize = 40, number lineSpacing = 1, string alignment = "middlecenter", string material = "default") | ImageController | Create an image from the path, with the specified material type |
| Scene.createMesh(string objPath, string texturePath) | MeshController | Create a 3d object from a .obj file and a texture image |
| Scene.getNoteGroup(number group) | NoteGroupController | Get the controller associated with a timing group |

Anytime you see a `=` in the argument (for example `material = "default"`), it means the argument has a defaut value and you don't have to specify it.

You can use the `material` argument to change the blending mode. Here's the list of all of them
- default
- add
- colorburn
- darken
- difference
- exclusion
- fastadd
- fastdarken
- fastlighten
- fastmultiply
- fastscreen
- hardlight
- lighten
- linearburn
- lineardodge
- linearlight
- multiply
- overlay
- screen
- softlight
- subtract
- vividlight
`newMaterialInstance` should be left as `false` most of them time. Set it to `true` if you need to change the controller's texture offset or texture scaling.

With that in mind, let's try adding a custom sprite into our scene. By default a sprite is placed in the "Background" layer, which is below what the track belongs to. So we need to change that to see anything at all. A sprite's layer can be changed over time, so we need to use a channel again, but this time it's a `StringChannel`

```lua
local sprite = Scene.createSprite("test.png")
sprite.layer = StringChannel.constant("Foreground") -- Change the layer
sprite.order = Channel.constant(1) -- Change the order within the layer
```

And we should see our image in the scene:
<img src="https://i.imgur.com/tAEbq6Q.png">

Feel free to change "test.img" to whatever the name of your image file is.

Let's try it with an image, and for good measure let's change it's material as well this time. Change the code above to:
```lua
local image = Scene.createImage("test.png", "fastadd")
image.rectW = Channel.constant(300) -- Set the proper width and height
image.rectH = Channel.constant(200)

local canvas = Scene.createCanvas(false)
image.setParent(canvas) -- image now uses the sorting layer of canvas
canvas.layer = StringChannel.constant("Foreground") -- Change the layer
canvas.sort = Channel.constant(1) -- Change the order within the layer
```

Notice how you have to specify the width and height of the image this time. That's one major difference between sprites and images. Also hopefully the Add blend mode didn't make your image too hard to see, I had to change it to conflict side here. But you can always try other blend mode as well.

<img src="https://i.imgur.com/vstHQ7X.png">

Lastly let's try creating some text as well:
```lua
local text = Scene.createText("Forte") -- The font must be installed on the user's OS
text.text = StringChannel.create().addKey(0, "Hello").addKey(1000, "World")

local canvas = Scene.createCanvas(false)
text.setParent(canvas)
canvas.layer = StringChannel.constant("Foreground") -- Change the layer
canvas.sort = Channel.constant(1) -- Change the order within the layer
```

String channels can also be keyframed, as you can see. Here we made it display "Hello" at 0ms and then change itself to "World" at 1000ms.

>You can also use easings on string channels! Try changing it to `addKey(0, "Hello", "so")` and see what happens.

And here's our result:
<img src="https://i.imgur.com/vPq6Zm8.png">

This covers the most important things to keep in mind when creating controllers. If you want to see the available properties of each type of controller, please refer to the documentation.

### 2.5. Controllers & Channels - Level 2

> Level 2 of Controllers & Channels will teach you how to save tremendous amount of time when working with channels. You'll be learning about different types of channels, and how to combine channels together for various effects.

Let's first consider how we'd tackle a very simple effect: moving an object back and forth, indefinitely, in a sine wave pattern. Our channel will look something like this:

(img)

It's totally doable to add each keyframe manually to achieve this effect. And in fact, to get an idea of how much work it'd take, let's take a look at the code to do that:
```lua
-- You don't have to replicate this. This is a bad idea
Channel c = Channel.keyframe()
for timing = 0, Context.songLength, 2000 do
	c.addKey(timing, 0, "so")
	c.addKey(timing + 500, 1, "b")
	c.addKey(timing + 1500, -1, "si")
end
```
This channel will keep oscilating between -1 and 1 in a sine wave form. It works, but we can do so much better.

In fact, the API provides a bunch of *other* types of channels other than keyframe channels to help you out. All of the code above can be shortened to:

```lua
Channel c = Channel.sine(2000, -1, 1, 0)
-- A sine wave, ranging from -1 to 1, with timng offset of 0
-- The wave repeats itself every 2000ms
```

You have a range of other channels available to you as well! Here's the list of them.

| Method | Type | Description |
| - | - | - |
| Channel.keyframe() | KeyChannel | A keyframe channel |
| Channel.constant(value) | ValueChannel | A channel with unchanging value |
| Channel.random(min, max, seed = 0) | ValueChannel | A channel that returns random value every time |
| Channel.noise(frequency, min, max, offset = 0, octave = 1) | ValueChannel | A perlin noise channel |
| Channel.sine(period, min, max, offset = 0) | ValueChannel | A sinusoidal wave channel |
| Channel.saw(string easing, period, min, max, offset = 0) | ValueChannel | A saw wave moving between max to min with the specified easing |
| Channel.fft(freqBandMin, freqBandMax, min, max, smoothness = 0.1, scalar = 1) | ValueChannel | Returns the average loudness of the currently playing audio within the specified frequency range |
| Channel.max(channelA, channelB) | ValueChannel | Take the maximum value of the two channels |
| Channel.min(channelA, channelB) | ValueChannel | Take the minimum value of the two channels |
| Channel.clamp(valueChannel, minChannel, maxChannel) | ValueChannel | Clamp the valueChannel between the value of minChannel and maxChannel|

> FFT stands for Fast Fourier Transform, in case you're curious.
> And by the way, by default FFT channels operate with 256 frequency bands, but you can change this with `Channel.setGlobalFFTResolution(resolution)`. Resolution must be integer in power of 2s (64, 128, 256, 512, ...)

It doesn't get much better than a single line of code, is it? Well it actually does. You can combine different channels together for infinite number of possible effects:

```lua
-- Vibrate between -2 and 2
Channel vibrate = Channel.noise(100, -2, 2) 
-- Every 1000ms: change it's value from 1 to 0, then jump back
Channel dampen = Channel.saw("so", 1000, 1, 0) 
-- How much it vibrates changes over time!
Channel combined = vibrate * dampen
```
By multiplying the `vibrate` channel with `dampen`, we limit the how much the channel vibrates over time. When `dampen` returns 1 it vibrates the most, when `dampen` returns 0, it does not vibrate. Because of the shape of `dampen`, which is a saw wave, we get an interesting pulsating effect.

Of course, you can combine keyframe channels with other channels this way as well. This also opens up possibilty for a very efficient technique, which was actually used internally to implement the built-in scenecontrol types. Let's have a look at one of them, `enwidenlane`.

The scenecontrol has these jobs:
- Change the opacity of two extra track lanes from 0 to 255,
- Change the opacity of the two extra track edges from 0 to 255,
- Change the opacity of the default two track edges from 255 to 0,
- Change the two extra critical lines (which are of lane 0 and 5) from 0 to 255,
- Change the divide line between lane 0-1 and 4-5 from 0 to 255,
- Change the position of the two extra track lanes from -100 to 0.

You can keyframe everything manually, it's just very annoying and take a lot of writing to do so. Instead what happens internally is something along the line of this:

```lua
-- NOTE! Internal c# code translated to lua. This does not operate properly
local track = Scene.track
  
-- The main channel, which is 0 by default
local enwidenLaneFactor = Channel.keyframe().setDefaultEasing("l").addKey(0, 0);
  
-- These objects are disabled by default. Enabling them:
track.extraL.active = Channel.constant(1)
track.extraR.active = Channel.constant(1)
track.criticalLine0.active = Channel.constant(1)
track.criticalLine5.active = Channel.constant(1)
track.divideLine01.active = Channel.constant(1)
track.divideLine45.active = Channel.constant(1)
track.edgeExtraL.active = Channel.constant(1)
track.edgeExtraR.active = Channel.constant(1)
  
-- Assign the channels, which are really just simple transformation of the main channel
track.edgeExtraL.colorA = track.edgeExtraL.colorA * enwidenLaneFactor
track.edgeExtraR.colorA = track.edgeExtraR.colorA * enwidenLaneFactor
track.criticalLine0.colorA = track.criticalLine0.colorA * enwidenLaneFactor
track.divideLine01.colorA = track.divideLine01.colorA * enwidenLaneFactor
track.divideLine45.colorA = track.divideLine45.colorA * enwidenLaneFactor
track.criticalLine5.colorA = track.criticalLine5.colorA * enwidenLaneFactor
track.extraR.colorA = track.extraR.colorA * enwidenLaneFactor
track.extraL.colorA = track.extraL.colorA * enwidenLaneFactor
  
local posY = -100 * (1 - enwidenLaneFactor)
track.extraL.translationY = track.extraR.translationY + posY
track.extraR.translationY = track.extraR.translationY + posY
  
local alpha = (1 - enwidenLaneFactor)
track.edgeLAlpha = track.edgeLAlpha * alpha
track.edgeRAlpha = track.edgeRAlpha * alpha
  
-- All we need to do for each scenecontrol command is:
addScenecontrol("enwidenlane", {"duration", "toggle"}, function(cmd)
    local timing = cmd.timing
    local duration = cmd.args[1]
    local toggle = cmd.args[2]
    enwidenLaneFactor.addKey(timing, enwidenLaneFactor.valueAt(timing))
    enwidenLaneFactor.addKey(timing + duration, toggle)
end)
```

If you're still unclear on what this code is doing, here's a brief explanation:
- The second argument, "toggle", is a value of either 0 or 1. We write this value directly into the `enwidenLaneFactor` channel.
- Every property we need to change simply some variant of the main channel, which were calculated directly on the channels themselves instead of individual keys.

You might also notice that we're performing arithmetic between a channel and a number value with `posY` and `alpha` channels. Internally the numbers actually get converted into constant channels, so this is just a convenient shorthand.

Please note that this script won't actually work, because internally the extra tracks already keyed to set their alpha to 0, so multiplying on top of that won't do anything. So instead you should do this:
```lua
track.extraEdgeL.colorA = track.extraEdgeL.colorA + 255 * enwidenLaneFactor
-- Similar for critical lines, divide lines, extra lanes

local posY = 100 * enwidenLaneFactor
track.extraL.translatonY = Channel.min(track.extraEdgeL.translationY + posY, Channel.constant(0))
-- Similar for extraR
```

What's happening here is we're adding our channel and the internal channel together. For colors the values above 255 are not different so we don't have to worry about that, but for the extra lane's position we don't want it to move beyond 0, just in case one would use both scenecontrol types together. We can use `Channel.min` to ensure this.

Of course if you don't care about using the internal type then:
```lua
track.extraEdgeL.colorA = 255 * enwidenLaneFactor
-- Similar for critical lines, divide lines, extra lanes

track.extraL.translationY = -100 * (1 - enwidenLaneFactor)
-- Similar for extraR
```
is fine as well.

So the actual lua compatible version of this script would be:
```lua
local track = Scene.track
  
-- The main channel, which is 0 by default
local enwidenLaneFactor = Channel.keyframe().setDefaultEasing("l").addKey(0, 0);
  
-- No need to enable anything. Internal types enabled them for us already

track.edgeExtraL.colorA = track.edgeExtraL.colorA + 255 * enwidenLaneFactor
track.edgeExtraR.colorA = track.edgeExtraR.colorA + 255 * enwidenLaneFactor
track.criticalLine0.colorA = track.criticalLine0.colorA + 255 * enwidenLaneFactor
track.divideLine01.colorA = track.divideLine01.colorA + 255 * enwidenLaneFactor
track.divideLine45.colorA = track.divideLine45.colorA + 255 * enwidenLaneFactor
track.criticalLine5.colorA = track.criticalLine5.colorA + 255 * enwidenLaneFactor
track.extraR.colorA = track.extraR.colorA + 255 * enwidenLaneFactor
track.extraL.colorA = track.extraL.colorA + 255 * enwidenLaneFactor

local posY = 100 * enwidenLaneFactor
track.extraL.translationY = Channel.min(track.extraR.translationY + posY, Channel.constant(0))
track.extraR.translationY = Channel.min(track.extraR.translationY + posY, Channel.constant(0))
  
local alpha = (1 - enwidenLaneFactor)
track.edgeLAlpha = track.edgeLAlpha * alpha
track.edgeRAlpha = track.edgeRAlpha * alpha
  
-- Choosing a different type name to avoid conflict
addScenecontrol("enwidenlanelua", {"duration", "toggle"}, function(cmd)
    local timing = cmd.timing
    local duration = cmd.args[1]
    local toggle = cmd.args[2]
    enwidenLaneFactor.addKey(timing, enwidenLaneFactor.valueAt(timing))
    enwidenLaneFactor.addKey(timing + duration, toggle)
end)
```

That concludes it for this section. With this knowledge complex movement are just a couple lines of code away!

#### Extra tip for debugging:
You can view what the channel is composed of by logging it. For example:
```lua
local channel = Channel.saw("si", 1000, 1, 0) * Channel.noise(100, -1, 1)
channel += Channel.keyframe()
channel *= 5
log(channel)
```

You should see the following result when opening the Error Log:
```
((unnamed@saw(si,1000,0,1,0)) * (unnamed@noise(100,-1,1,0,1)) + unnamed@key(0)) * (5)
```
This will come in handy to figure out why the effect isn't working as you intended it to.

### 2.6. Working with note groups

Note groups are a bit odd to work with. The workflow of defining the controllers, then keying them in scenecontrol definition workflow we've been using so far aren't going to work for one simple reason: we don't know which controllers we want. Because we don't know which timing groups our events will be placed in ahead of time.

> You might be wondering why it's called note group instead of timinggroup. To put it simply you aren't interacting with timing events here, but the actual notes themselves. I chose NoteGroup to reflect that a bit better.

The only way to be sure is to query for the controller for every scenecontrol command.

```lua
addScenecontrol("myType", {}, function(cmd)
	local noteGroup = Scene.getNoteGroup(cmd.timingGroup)
end)
```

That's totally fine but we can't really create a new channel for every scenecontrol command. This will override the old channel for every scenecontrol command:
```lua
addScenecontrol("myType", 1, function(cmd)
	local noteGroup = Scene.getNoteGroup(cmd.timingGroup)
	noteGroup.translationX = Channel.keyframe().addKey(0, cmd.args[1])
end)
-- This doesn't work and the note group's properties will only be assigned the last value.
```

Instead we need a way to find out if we have already created a channel for a note group already, and if not, create a new one.  And you can, in fact, do this with lua code. It's just very clunky and tedious to do so for every channel, and every propety out there.

Introducing: named channels. Remember when you were logging your channel and there were a bunch of "unnamed" in there? That's the default name of every channels. You can change your channels' name to your liking as well.

```lua
local channel = Channel.named("myChannelName").keyframe()
```

Then if a channel is combined from multiple channels, you can separate out each one to edit with by using `.find(name)`. Let's see it in action:

```lua
local myChannel = Channel.named("IWantThisBackLater").keyframe()
-- Let's add a few keys so we can be sure that this is the same channel later
myChannel.addKey(0, 0).addKey(1000, 1).addKey(2000, 2)
log(myChannel.keyCount) -- Should output 3 in the Error Log

local otherChannel = Channel.named("OthersCreatedThisAndIDontCareAboutIt").keyframe()
local combinedChannel = myChannel * otherChannel

local myChannelFound = combinedChannel.find("IWantThisBackLater")
log(myChannelFound.keyCount) -- Should also output 3 in the error log
```

The combinedChannel has the ability to search from it's component for a channel with the name you want. This name is also local to said channel, so you don't even have to worry about naming your channel differently for each properties.

What this allows us to do is solving the issue we had earlier with note groups. Let's try to find our channel within the property of the note group, and if it doesn't already exist then we'll create a new one and assign it.

```lua
addScenecontrol("myType", 1, function(cmd)
	local noteGroup = Scene.getNoteGroup(cmd.timingGroup)

	-- Try finding the channel from the property
	local channel = noteGroup.translationX.find("myType")
	if channel == nil then -- If it's not found
		-- Make a new channel
		channel = Channel.named("myType").keyframe()
		-- Assign it
		noteGroup.translationX = channel
	end
		
	channel.addKey(0, cmd.args[1])
end)
```

Always naming your channel is also a very good idea, as it help with debugging. Especially if you intend on sharing your scripts with other people to use.

### 2.7. Post processing

Post processings are effects visible on your entire screen. They range anywhere from: blurring the screen, distorting the screen, adding a vignette effect, making the screen noisy, adding glow effects, etc.

I'll be honest, I kind of just... copied over everything that unity supports and put wrapper over them in the form of Channels. So I don't even know what 80% of the arguments do myself. Unity's documentation on this is also pretty horrible, so you're mostly on your own here.

With that said, If you've mastered Channels, you're 90% of the way there to also be able to use post processing. The only difference is that you have to grab the controllers from the `PostProcessing` object, and you also have to enable each argument individually in order for it to take effects (Because having them on have massive impacts on performance)

Each post processing controller can also have properties that are not changed over time. I won't demonstrate them because none of them seem useful, but do feel free to check the documentation for more details on this if you're curious.

Here's a demonstration on probably the most useful post processing effect of them all, Color Grading:
```lua
local colorGrading = PostProcessing.colorGrading
colorGrading.enableEffects({"temperature", "colorfilter", "saturation"}) -- Enable 3 properties
colorGrading.temperature = Channel.constant(10) -- Increase the temperature
colorGrading.ColorS = Channel.constant(-0.2) -- Decrease the saturation
```

And here's the result:
<img src="https://i.imgur.com/3raINS4.png">

### 2.8. A few extra tips and cautions

These are all small details that you should know but aren't enough to warrant an entire section about.

##### Cautions
1. Camera's coordination in aff camera command and in scenecontrol API are very different!
2. All file path are relative to the *`Scenecontrol` folder*, NOT to the running script
> For example, if within your `init.lua` script, you referenced the script `folder/other_script.lua`, and within `other_script.lua` you tried to create a sprite with the path `image.jpg`, the actual file path should be located at `Scenecontrol/image.jpg` and NOT `Scenecontrol/folder/image.jpg`
3. When choosing materials for sprites, images and texts, be aware that anything that doesn't start with `fast` can hurt performance significantly, especially on lower-end hardware.

#### Tips
1. For those familiar with macros. The `Event` and `Context` object is available. I don't know why one would need `Event` but why not
2. Alternative to `log`, `notify` is also an option, which will display a message on a toast notification.
3. You can always use `log(channel.valueAt(timing))` to something's value at any point in time. This can be useful to figure out the coordinates of objects in the scene as well.
4. You can get the list of all valid installed fonts with `Context.availableFonts`. Only use this for debugging however and don't include this in an actual script.
5. Always use `local` unless you do intend on making a variable global.

# What's next?
Congratulations on finishing this tutorial on the new scenecontrol API!

You should now be aimed with enough knowledge to use the reference document effectively, which will details out everything about the API for you to reference while writing your own scripts. Please check it out at: (coming soon)

Have fun scripting!
