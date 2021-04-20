# ArcadeScenecontrol
Tutorial on how to use Lua scripted scenecontrol within ArcadeZero

# Getting started
We will first explore how Arcade0 scenecontrol works in general and start writing a simple script.

For scenecontrol to even work at all, you need to create a folder named "Scenecontrol" (case sensitive) within your project folder (the same place you put your chart files and music). This is where all your scenecontrol related stuff goes.

Let's first examine the syntax of a scenecontrol command:

    #aff file:
    scenecontrol(timing, scenecontrolType, parameter0, parameter1, parameter2, ...);
All parameters must be float (for now. Ability to have string as a parameter will come in the next version). What the parameters mean and what the command do will be determined by scenecontrolType, therefore we need a way to define the types.

This is where Lua comes in. For each scenecontrol type, Arcade will call the .lua file with the same name as that type (e.g: the type "redline" will correspond to the file "redline.lua") in the Scenecontrol folder.

There's also a special lua file: init.lua. This script will be run first every time you load a chart (if it exists). This is useful if you want to, for example, create an object that all command interacts with (take arcahvdebris, arcahvdistort, ...)

Let's try to make something simple first to get the hang of things. We will import an image into the scene, and move it to somewhere else on each scenecontrol command.

**WARNING: I won't be explaining the syntax of Lua, if you're familiar with programming in any language already then you should be able to follow along without too much trouble. If you're new however I recommend checking out the official Lua tutorial: https://www.lua.org/pil/contents.html**

## Example 1: Moving image
Our scenecontrol command will look like this

    #aff file:
    scenecontrol(timing, moveimage, newXposition, newYposition, newZposition);
   
Each time the command is called, the image will move from its previous position to the new position defined in the command. Because the type is "moveimage" we need to create a file called "moveimage.lua" within the folder "Scenecontrol".
 For the function to work you must to define a function called onAffCommand. This will be called exactly once for each command as the name suggest.


    function onAffCommand(timing, parameter0, parameter1, parameter2...)
    end
The timing and parameter in each scenecontrol command will be passed into here for you to use in your function, although we don't actually need it right now. Still let's give the parameters meaningful names. Within moveimage.lua, write:

    --moveimage.lua
    function onAffCommand(timing, newXposition, newYposition, newZposition)
   
  What do you need to do within the file? First of all you need to get the object you actually want to manipulate. Here we want to manipulate an image (or in another name, sprite), but we don't have any image in our scene. So let's try creating one
  

    --moveimage.lua
    function onAffCommand(timing, newXposition, newYposition, newZposition)
	    Scene.createSprite("image", "image.png")
	end

Feel free to place whatever image you like in to the folder Scenecontrol (and also choose whatever image name you like instead of image.png). Alright, let's write a command into the .aff and see how it goes

    #aff file
    AudioOffset:0
    -
    timing(0,100.00,4.00);
    scenecontrol(0, moveimage, 69, 420, 1337);
Now load the chart and you should see the image appearing in the scene. Great!
Let's try adding more command.

    #aff file
    AudioOffset:0
    -
    timing(0,100.00,4.00);
    scenecontrol(0, moveimage, 69, 420, 1337);
    scenecontrol(1000, moveimage, 69, 420, 1337);
    scenecontrol(2000, moveimage, 69, 420, 1337);
Load the chart. Everything should seem okay until you check the log (click the log on the left). It should complain something about adding a key which already exist.
   
The issue here is that every time we run the aff command, it will create a new Sprite named "image", and we can't have 2 objects of the same name. Ideally in this case we'd want to create the Sprite once, and then each command will interact with that Sprite. To do that you must create the Sprite in the file "init.lua" (placed in the Scenecontrol folder, of course), which will be run exactly once every time you load the chart:
   

    --init.lua
    Scene.createSprite("image", "image.png")
   
Now we have a sprite named "image" in the scene. Back to moveimage.lua again now we just need to get that sprite. Use Scene.GetSprite to do it. We also need to store it somewhere so let's create assign it to a variable named "sprite"

    --moveimage.lua
    function onAffCommand(timing, newXposition, newYposition, newZposition)
	    sprite = Scene.getSprite("image")
    end
   
This variable will allow us to modify the image in whatever way we like. Now if you load the chart and check the log no error should appear.
   
Having it sit in that spot is kind of boring, so let's move it somewhere. To move the sprite use the method SetTranslation(x, y, z)

    --moveimage.lua
    function onAffCommand(timing, newXposition, newYposition, newZposition)
	    sprite = Scene.getSprite("image")
	    sprite.setTranslation(newXposition, newYposition, newZposition)
    end

Each aff command will pass its argument into this function, which we defined to just set the sprite's position to whatever that value is. Because the four function are executed one after each other you should find that the image will sit at whatever coordinate you wrote in the last command in the .aff file. Try changing it around and check.

This is nice and all but ideally we would want to move this over time. That's not possible within onAffCommand because it's run only once per command. For this a very important function come into play: Register. Let's spend some time explaining this one.

## Detour: Register()

So at the very heart, what scenecontrol does it that it manipulates object. It's also not a one-to-one relationship either, one scenecontrol can manipulate multiple objects, and multiple scenecontrol can manipulate one object.

What this mean is that we need a way to tell at any moment which scenecontrol event to actually run for each object. Register() does exactly this, it asks for a portion of time on each object where during that time, a function will be run.

Syntax:

    register(object, timing, duration, function_to_run)
What this does, is for object, the portion of time between timing and timing+duration will be given to function_to_run,

Please note that if the registration overlap, say an event register at time 1000 and duration 1000, and another evet register at time 1500 and duration 1000, then the event that starts earlier will be cut short to make way for the new one, in this case time 1000-1500 will be given to event 1 and 1500-2500 will be given to event 2.

If an event is completely surround another however, say event 1 registers at time 1500 and duration 5000, and event 2 registers at time 2000 and duration 100, then after event 2 is completed event 1 will continue running.

Details of event selecting algorithm will come in a later section. Let's return to our example

## Back to moveimage
Okay, so now we know we need to register the sprite to a function. Let's define our function first. It will also be passed the parameter in the scenecontrol command, however the first parameter (timing) will not be the timing of the command but the current timing that the song is at. You can name the parameter itself whatever you want but I'd advise to keep it consistent. Same goes for function name, choose something that makes sense:

    --moveimage.lua
    function MoveImage(charttiming, newXPosition, newYPosition, newZPosition)
    end
 
Now register it in onAffCommand by adding this to the function. Let's give a duration of 1 second.

	register(sprite, timing, 1000, "MoveImage")

The last "MoveImage" refers to the function name you chose earlier. If you named your function something else like function FooBar(timing, parameters, parameters,...) then you'd pass in "FooBar" into register.

Our update function is doing nothing at the moment, so let's have it more the sprite around. Same as before we will use SetTranslation(x,y,z)

    function MoveImage(charttiming, newXPosition, newYPosition, newZPosition)
	    local x = (charttiming - BaseTiming) / 1000 * newXPosition
	    local y = (charttiming - BaseTiming) / 1000 * newYPosition
	    local z = (charttiming - BaseTiming) / 1000 * newZPosition
	    sprite.setTranslation(x, y, z)
    end
Note the local keyword, this variable will not be saved and transferred into other functions which keep things fast. Try to use it whenever you can. Also note the variable "BaseTiming". This variable is automatically provided and can be used in any function, it refers to the timing value in the aff command.

Let's look at our script as a whole now

    function onAffCommand(timing, newXposition, newYposition, newZposition)
	    sprite = Scene.getSprite("image")
	    register(sprite, timing, 1000, "MoveImage")
    end
    
    function MoveImage(charttiming, newXPosition, newYPosition, newZPosition)
	    local x = (charttiming - BaseTiming) / 1000 * newXPosition
	    local y = (charttiming - BaseTiming) / 1000 * newYPosition
	    local z = (charttiming - BaseTiming) / 1000 * newZPosition
	    sprite.setTranslation(x, y, z)
    end

Load the chart and you should see things moving around. Nice!

Nah it kinda sucks though. For once it just keeps snapping back to the center and moving out to the coordinate. We need a way to retrieve what its previous position is and then gradually move from there.

This job only needs to be run once so it makes sense we include it in onAffCommand. Use the GetTranslationAt(timing) to retrieve the position, then assign it to a variable

    function onAffCommand(timing, newXposition, newYposition, newZposition)
	    sprite = Scene.getSprite("image")
	    Register(sprite, timing, 1000, "MoveImage")
	    oldPos = sprite.getTranslationAt(timing - 1)
    end
    
oldPos holds 3 value x y z which you can access with oldPos.x, oldPos,y, oldPos.z respectively. Let's do just that:
 

    function MoveImage(charttiming, newXPosition, newYPosition, newZPosition)
	    local x = oldPos.x + (charttiming - BaseTiming) / 1000 * (newXPosition - oldPos.x)
	    local y = oldPos.y + (charttiming - BaseTiming) / 1000 * (newYPosition - oldPos.y)
	    local z = oldPos.z + (charttiming - BaseTiming) / 1000 * (newZPosition - oldPos.z)
	    sprite.setTranslation(x, y, z)
    end
Now everything should work as intended, and the object will move from the old position to new within 1 second.

You might be complaining about the ugly mess of math in there. Don't worry, you do not actually have to write this every single time you want to move something. I've prepared some functions that'll aid you in animating as well. Let's see what it is: the whole function above can be condensed to:

    function MoveImage(charttiming, newX, newY, newZ)
	    local percentage = (charttiming - BaseTiming) / 1000
	    sprite.setTranslation(Ease.Linear(oldPos.x, newX, percentage), Ease.Linear(oldPos.y, newY, percentage), Ease.Linear(oldPos.z, newZ, percentage))
    end

If you're familiar with easing then this won't need explaining. But this is basically how you'd animate things. All ease function takes a lower bound, higher bound, and percentage. It will return a value inbetween the bounds based on the percentage you pass in (and the easing type). There will be a listing of all easing functions you can use as well.
~~(also I shortened the variable name because it got too long. You can do that as well)~~

Our final script looks like this

    function onAffCommand(timing, newXposition, newYposition, newZposition)
	    sprite = Scene.getSprite("image")
	    register(sprite, timing, 1000, "MoveImage")
	    oldPos = sprite.getTranslationAt(timing - 1)
    end
    
    function MoveImage(charttiming, newX, newY, newZ)
	    local percentage = (charttiming - BaseTiming) / 1000
	    sprite.setTranslation(Ease.Linear(oldPos.x, newX, percentage), Ease.Linear(oldPos.y, newY, percentage), Ease.Linear(oldPos.z, newZ, percentage))
    end

That concludes our first example! It might have been a lot to take in at first because there's quite a bit of fundamental concept, but if you got through this you're pretty much in the clear.

## Example 2: redlines
Remember what I said earlier about creating objects every command? In some cases that's exactly what you want to happen, so we need a way to do it without duplicate names.
Solution I came up with is to append an index into the object's name, and you can get the index with the variable EventID (much like BaseTiming this is always available).

Let's use this technique to recreate the red line effect used in Arcahv. First get the red line image (in whatever way you want to, or just get a random image to practice). We'll define the type in redlines.lua, and here will be the aff command syntax:

    scenecontrol(timing, redlines, duration);
It's not the exact syntax used in Arcahv but who cares. Anyway, in redlines.lua:
 

    --redlines.lua
    function onAffCommand(timing, duration)
	    redline = Scene.createSprite("redline", "redline.png")
	    register(redline, 0, 1, "Hide")
	    register(redline, timing, 1, "Show")
	    register(redline, timing+duration, 1, "Hide")
    end
    
    function Hide(timing, duration)
    end
    
    function Show(timing, duration)
    end

Note how we only registered 1ms for each function. That's all we need in this case

We haven't done anything to avoid the problem yet. Let's pass in the EventID now by modifying the createSprite line:

    line = Scene.createSprite("redline"..EventID, "redline.png")
The ".." thing is string concatenation. Kind of weird but hey it's handy. Now every line will have an unique name.

While we're at it let's also define Hide and Show. In this case there's two ways to do it: one by setting the transparency, the other by disabling the object.

    --set color method:
    function Hide(timing, duration)
	    redline.setColor(255,255,255,0)
	end
	
	function Show(timing, duration)
		redline.setColor(255,255,255,255)
	end
	
	--enable/disable method:
	function Hide(timing, duration)
		redline.setActive(false)
	end
	function Show(timing, duration)
		redline.setActive(true)
	end
    
I recommend the latter method since it's a bit more efficient. If you have a lot of objects then I'd really recommend disabling it when you don't need it

Let's also randomize the position and also scale it up properly. We only need to do this once per sprite:

    function onAffCommand(timing, duration)
	    redline = Scene.createSprite("redline", "redline.png")
	    redline.setScale(1000, 1, 1)
	    redline.setTranslation(0, math.random(-700, 700), 0)
	    register(redline, 0, 1, "Hide")
	    register(redline, timing, 1, "Show")
	    register(redline, timing+duration, 1, "Hide")
    end

And that should be it!

## Example 3: Text
Yes we can have other things than just images. For this example we define each aff command to spawn a random string and have it flying from left to right.

Syntax:

    scenecontrol(timing, livechat);
    
Note that we can't pass in strings as argument yet as of this version.

Guess how you'd create the text by the way

    function onAffCommand(timing)
	    text = Scene.createText("text"..EventID, "kusa", other parameters)
	    y = math.random(250,450)
	    offsetUntilStart = math.random(0,500)
	    duration = math.random(2500,3500)
		
		register(text, 0, 1, "Hide")
	    register(text, timing + offsetUntilStart, duration, "MoveAcross")
	    register(text, timing + offsetUntilStart + duration, 1, "Hide")
    end
	
	function Hide(timing)
		text.setActive(false)
	end
	function MoveAcross(timing)
		text.setActive(true)
		text.setTranslation(Ease.Linear(-3000, 3000), y, 0)
	end

Really the only thing new here is that we're using math.random and the CreateText function.

It's also worth discussing that we don't have to necessarily have it update right at the timing of the aff command, here we offset the event by a random amount of time to introduce some variety

Let's also make the string random

    textList = {"kusa", "lol", "ww", "haha"}
    text = Scene.createText("text"..EventID, textList[math.random(1, #textList)], other parameters)
This is just lua stuff, you should check the lua tutorial if you want to find out more about these things. I'll note a few things though:
  

 - Lua start indexing at 1 instead of 0 like most popular language
 - #textList return the length of the list

## Example 4: Superchat

Here we'll see how setting object in a hierarchy can really save our day. Let's just copy paste over the file from the previous example and modify it to also include a nice superchat background

    function onAffCommand(timing)
	    text = Scene.createText("text"..EventID, "kusa", other parameters)
	    y = math.random(250,450)
	    offsetUntilStart = math.random(0,500)
	    duration = math.random(2500,3500)
		
		superchat = Scene.createSprite("superchat"..EventID, "superchat.png")
		superchat.setParent(text)
		superchat.setTranslation( --tinker around with this-- )
		
		register(text, 0, 1, "Hide")
	    register(text, timing + offsetUntilStart, duration, "MoveAcross")
	    register(text, timing + offsetUntilStart + duration, 1, "Hide")
    end
	
	function Hide(timing)
		text.setActive(false)
	end
	function MoveAcross(timing)
		text.setActive(true)
		text.setTranslation(Ease.Linear(-3000, 3000), y, 0)
	end

By setting the parent of superchat to text we can skip updating superchat altogether. superchat will follow text's position, and disabling text also disable superchat.

We have a problem though and that is the text might not be visible through the superchat image. Solution would be to set it's sorting layer with setLayerOrder() and setLayerName().

Let's talk about layer name first. There's 3 names you can choose from: "Background" - behind track, "Foreground" - in front of track and "Topmost" - above everything else (even the UI).

Within each layer you can also choose which objects appear in front of another with layer order. Objects with higher layer order will appear in front. Let's apply this:

    superchat.setLayerOrder(EventID*2)
    text.setLayerOrder(EventID*2+1)
   
We used event id to also make sure that newer event appear in front of older ones. If you want the object to appear on top of Track for example, just include superchat.setLayerName("Foreground") and text.setLayerName("Foreground") as well

 
## Manipulating in game elements
For example, you might want to hide the track, move the sky input line, dye the UI in pink, etc,...

Well you'll be glad to hear that all of those are just the same as sprite so you don't need to learn anything new here, aside from the fact that you don't need to actually create anything to access it. Just use the Scene.getObject(name) method to get it. Here are the list of them:

 - "Track"
 - "CriticalLine"
 - "DivideLine01"
 - "DivideLine12"
 - "DivideLine23"
 - "SingleLineL"
 - "SingleLineR"
 - "SkyInputLine"
 - "SkyInputLabel"
 - "Background"
 - "UIPause"
 - "UIInfo"
 
## Wait is this phigros?
You heard me right. You can rotate and move individual notes with this. It's really hard to work with though so I'll keep it short and leave it to the big brained to figure it out (which doesn't include me)

Essentially, you can have the notes themselves be manipulated as well. The method GetNoteGroup() will give you an object that control all notes within the same timing group as the scenecontrol event.

Details of what you can do with this is in the listing below. But beware: even if you move the notes their judgepoints do not move. So make sure to return them to their original position if you don't want your chart to look weird

## Additional tips:

If for some reason scenecontrol just completely doesn't show up in your chart, start looking at the error log (You can access it by clicking the button on the left panel within Arcade). If it detects any error within your scripts, it'll be reported there.

Additionally, you can also log stuff within your script. Just return whatever you want to be logged

	function onAffCommand(timing, params0, params1):
		return "Hello world"
	end

If you try loading the chart then it'll log "Hello world" for each scenecontrol command in the aff.

You must return a string by the way. If you need to return a number then somehow convert it into string, either through tostring(number) or concatenate it with some other string.

# All objects and method

## Built in functions and variables:
You can call these directly anywhere in your script

Variables:
|Variable|Description|
|-|-|
|EventID|Current id of the scenecontrol event. Event earlier in the script are given lower id. Id counts per scenecontrol type|
|BaseTiming|The first value passed into an aff scenecontrol command: it's timing value|

Functions:
|Method|Description|Output|
|-|-|-|
|getNoteGroup()|Get a reference to the notes within the same timing group as the scenecontrol event|NoteGroupController|
|register(ControllerBase object, number timing, number duration, string function)|See the register section|Nil|
## Built in utility class:

#### XYZ
A data type class that wraps up the 3 coordinate value. Other functions and class methods work with this class to return coordinate and take in coordinate as parameter.

Properties
|Variable|Description|
|-|-|
|x|X coordinate component|
|y|Y coordinate component|
|z|Z coordinate component|

Example usage:

	xcoord = object.getTranslationAt(t).x --get translation return a value of type XYZ
	object2.setTranslation(x, 0, 0)       --match the x position of the two object


#### RGBA
A data type class used to wrap up color data into RGB value plus transparency value. Other functions and class methods primarily work with this class to return color and take in color as parameter.

Properties
|Variable|Description|
|-|-|
|r|Red component of color, ranges from 0 to 255|
|g|Green component of color, ranges from 0 to 255|
|b|Blue component of color, ragnes from 0 to 255|
|a|Alpha (transparency), ranges from 0 to 255|

Example usage:
	
	sprite = Scene.getSprite("examplesprite")
	color = sprite.getColorAt(t)              --this variable has type: RGBA
	return color.r..color.g..color.b..color.a --log the component of color


#### HSVA
Similar to RGBA but holds color data through hue, saturation and value. Mainly provided for convenience for the scripter when working with color.

Properties
|Variable|Description|
|-|-|
|h|Hue component, ranges from 0 to 360|
|s|Saturation component, ranges from 0 to 1|
|v|Value / Brightness component, ranges from 0 to 1|
|a|Alpha (transparency), ranges from 0 to 1|

Example usage:

	sprite = Scene.getSprite("examplesprite")
	color = sprite.getColorAt(t)                -- type: RGBA
	hsvcolor = Color.RGBAToHSVA(color)          -- convert RGBA to HSVA
	hsvcolor.h = (hsvcolor.h + 180) % 360       -- invert the hue
	sprite.setColor(Color.HSVAToRGBA(hsvcolor)) -- convert color back and set value

#### Ease
A static class that provides numerous easing types to the scripter. You can see all the easing methods visually here: https://easings.net

Methods

All methods take in *start*, *end*, *percentage* as parameters, and output a number between *start* and *end*. *percentage* must ranges from 0 to 1.

Method names:
 - Linear
 - InSine
 - OutSine
 - InOutSine
 - InQuad
 - OutQuad
 - InOutQuad
 - InCubic
 - OutCubic
 - InOutCubic
 - InQuart
 - OutQuart
 - InOutQuart
 - InQuit
 - OutQuit
 - InOutQuit
 - InExpo
 - OutExpo
 - InOutExpo
 - InCirc
 - OutCirc
 - InOutCirc
 - InBack
 - OutBack
 - InOutBack
 - InElastic
 - OutElastic
 - InOutElastic
 - InBounce
 - OutBounce
 - InOutBounce

Example usage:

	x = Ease.OutSine(-1000, 1000, (timing - BaseTiming) / duration)
	y = Ease.OutBounce(-300, 300, (timing - BaseTiming) / duration)
	object.setTranslation(x, y, 0)

#### Color
A static class that provide easy conversion between different 3 color type: HSVA, RGBA (see above) and hex string (e.g "#1f1e33ff")

Methods:

|Method|Description|Output|
|-|-|-|
|RGBAToHex(RGBA rgbacolor)|Convert RGBA to hex representation|string|
|RGBAToHex(number r,number g,number b, number a)|Save as above, except passing the component separately. r, g, b, a ranges from 0 to 255|string|
|HSVAToHex(HSVA hsvacolor)|Convert HSVA to hex representation|string|
|HSVAToHex(number h, number s, number v, number a)|Same as above, except passing the component separately. h ranges from 0 to 360, s and v and a ranges from 0 to 1|string|
|HexToRGBA(string hex)|Convert hex string to RGBA. Return black on invalid string|RGBA|
|HexToHSVA(string hex)|Convert hex string to HSVA. Return black on invalid string|HSVA|
|RGBAToHSVA(RGBA rgbacolor)|Convert RGBA color to HSVA color|HSVA|
|RGBAToHSVA(number r, number g, number b, number a)|Same as above, except passing the component separately. r, g, b, a ranges from 0 to 255|HSVA|
|HSVAToRGBA(HSVA hsvacolor)|Convert HSVA color to RGBA color|RGBA|
|HSVAToRGBA(number h, number s, number v, number a)|Same as above, except passing the component separately. h ranges from 0 to 360, s and v and a ranges from 0 to 1|RGBA|

Example usage

	hex = HSVAToHex(Ease.Linear(0, 360, timing), 1, 1)
	text = "<color="..hex..">gaming moment</color>"
	textObject.setText(text)

### Scene
A static class that allow scripter to create objects, and retrieve objects by name

Methods:
|Method|Description|Output|
|-|-|-|
|createSprite(string name, string imagePath[, string blendMode])|Create a sprite in the scene from the image specified by imagePath, naming it name and assign the blending mode blendMode (which can be one of the following: "Add", "Multiply", "Darken", "Lighten", "Screen". Leave it empty for default blending)|SpriteController|
|getSprite(string name)|Get a sprite in the scene with the given name|SpriteController|
|createText(string name, number width, number height, number lineSpacing, string alignment, bool overflow[, string font])|Create a text object in the scene, naming it name. The text box will have the size width * height. lineSpacing define how long the lines are apart form each other. alignment define where the text are anchored to and can be of the following 9 values: "UpperLeft", "UpperCenter", "UpperRight", "MiddleLeft", "MiddleCenter", "MiddleRight", "LowerLeft", "LowerCenter", "LowerRight". If overflow is false then the text will be cut off by the box. The last variable font speficy the font name to use, which must be installed on your OS already|TextController|
|getText(name)|Get a text in the scene with the name|TextController|

## Controllers

Controllers are sandboxes that references actual objects within Unity. Through each controller you can modify the object that it references.

There are multiple type of controllers but they all inherit from ControllerBase: any method here is available to ALL controllers.

#### ControllerBase

All controllers have access to the following methods:

|Method|Description|Output|
|-|-|-|
|setTranslation(XYZ xyzcoord)|Set the translation (or position) of the object|Nil|
|setTranslation(number x, number y, number z)|Same as above except passing in the components separately|Nil|
|getTranslationAt(number t)|Get the translation of the object at the timing t. Note that scenecontrol events that take places after the current ones are not taken into account (this is true for all following "get" function)|XYZ|
|setRotation(XYZ xyzcoord)|Set the rotation of the object|Nil|
|setRotation(number x, number y, number z)|You get the point|Nil|
|getRotationAt(number t)||XYZ|
|setScale(XYZ xyzcoord)||Nil|
|setScale(number x, number y, number z)||Nil|
|getScaleAt(number t)||XYZ|
|setActive(bool state)|If state is true then object is disabled (no longer visible)|Nil|
|getActiveAt(number t)||bool|
|setParent(controller other[, bool worldTransformStays])|Set the parent of the current object. If worldTransformStays is true then the coordinate doesn't change after the fact (default is false)|Nil|

#### SpriteController

Sprite controllers have all methods of ControllerBase and additionally:

|Method|Description|Output|
|-|-|-|
|setColor(RGBA rgbacolor)|Set the color tint of the sprite|Nil|
|setColor(HSVA hsvacolor)||Nil|
|setColor(number r, number g, number b, number a)||Nil|
|getColorAt(number t)|Return the color of the object at time t in RGBA value|RGBA|
|setLayerName(string layer)|Set the layer of the sprite. layer can be of 3 value: "Background", "Foreground", "Topmost" (technically there's more but don't mess with it please)|Nil
|getLayerNameAt(number t)||string|
|setLayerOrder(number order)|Set the order on which sprites are visible within a single layer|Nil|
|getLayerOrderAt(number t)||number|

#### TextController

Text controllers have all methods of ControllerBase and additionally:

|Method|Description|Output|
|-|-|-|
|setText(string text)|Rich text supported. Check here for more info: https://docs.unity3d.com/Packages/com.unity.ugui@1.0/manual/StyledText.html|Nil|
|getTextAt(number t)||string|
|setAlpha(number a)|Set the transparency of the text box|Nil|
|getAlphaAt(number t)||number|
|setLayerName(string layer)|Set the layer of the sprite. layer can be of 3 value: "Background", "Foreground", "Topmost" (technically there's more but don't mess with it please)|Nil
|getLayerNameAt(number t)||string|
|setLayerOrder(number order)|Set the order on which sprites are visible within a single layer|Nil|
|getLayerOrderAt(number t)||number|

#### NoteGroupController

Controll group of note in a timing group. Note group controllers have all methods of ControllerBase and additionally:

|Method|Description|Output|
|-|-|-|
|setRotationIndividual(XYZ xyzcoord)|Set rotation of individual notes along each of their axis|Nil|
|setRotationIndividual(number x, number y, number z)||Nil|
|getRotationIndividualAt(number t)||XYZ|
|setScaleIndividual(XYZ xyzcoord)|Similar but for scale|Nil|
|setScaleIndividual(number x, number y, number z)||Nil|

#### UIPauseController and UIInfoController

These controls the pause button and the Info panel respectively. They both have the following additional methods to the ones inherited from ControllerBase:

|Method|Description|Output|
|-|-|-|
|setColor(RGBA rgbacolor)|Set the color tint of the UI|Nil|
|setColor(HSVA hsvacolor)||Nil|
|setColor(number r, number g, number b, number a)||Nil|
|getColorAt(number t)|Return the color of the UI at time t in RGBA value|RGBA|
