---
layout: post
title: Building a game in the browser (part 2 - Getting things setup)
---

In [part 1](/2013/11/10/building-a-game-in-the-browser-part-1) we laid out the ideas for Planetoids and the order in which to tackle building the game. Let\'s get our hands dirty.

##Part 2 - Getting things setup
The things we\'ll build today are:

* file setup
* global variables
* canvas
* main loop
* entity definition
* ship definition
* keyboard input
* init and start game

##The Files
Nothing fancy here, just a simple html page and the js for our game. 

####index.html

{% highlight html %}
<!DOCTYPE html>
<html>
<body>
    <script src="planetoids.js"></script>
</body>
</html>
{% endhighlight %}
    
####planetoids.js
This will be where all of our javascript will go. (unless otherwise indicated, any javascript used in the game will go in this file)

##Global Variables
Simple object to store all of our global state in.

{% highlight javascript %}
var g = {};
{% endhighlight %}

##The Canvas
We need something to draw on. Canvas is the tool of choice here.

What we\'ll do is create function that creates a canvas element, set the with and the height (stored in our global variable), append it to our body element, and grab and return the 2d drawing context.

{% highlight javascript %}
var g = {
    width: 800,
    height: 600,
    halfWidth: 400, //precalculate for later
    halfHeight: 300 //precalculate for later
};

function createCanvas() {
    var canvas = document.createElement('canvas');
    canvas.width = g.width;
    canvas.height = g.height;
    document.body.appendChild(canvas);

    var ctx = canvas.getContext('2d');
    return ctx;
}
{% endhighlight %}

##Main Loop
Every game (well most) have some sort of main loop. The purpose of the main loop is to coordinate drawing and updating objects in a game. The type of game loop that we\'ll be using is special in the fact that it makes every attempt to update the game logic at a constant interval while letting the rendering occur as often as possible. This is good for 2 reasons, 1. it simplifies our game logic 2. it allows the game logic to become deterministic (always good for a game). For more info see this [article on game loops](http://www.koonsolo.com/news/dewitters-gameloop/).

So the code looks a bit strange, but basically what is happening here is we\'re creating a _loop function that will be called via requestAnimationFrame(). The loop uses the requested updates per second (g.ups) to coordinate calling the update function and render functions.

{% highlight javascript %}
var g = {
    ...
    ups: 30,
    ...
};

function update(dt) {
    //dt is the interval between each update (in milliseconds)
}

function render(dt, ctx) {
    //dt is the interpolation factor between frames (between 0 and 1)
}

function startMainLoop(ctx) {
    var loops = 0, 
        skipTicks = 1000 / g.ups,
        maxFrameSkip = 10,
        nextGameTick = new Date().getTime()
        _ctx = ctx;
  
    var _loop = function() {
        loops = 0;
        var currentTime = new Date().getTime();

        while (currentTime > nextGameTick && loops < maxFrameSkip) {
            update(skipTicks);
            nextGameTick += skipTicks;
            loops++;
        }

        render((nextGameTick - currentTime) / skipTicks, _ctx);
        window.requestAnimationFrame(_loop);
    };
    _loop();
};
{% endhighlight %}

Things to note here. The update function will be called at a constant rate (in this case 30 updates per seconds). The render function will be called as often as possible. Passed into the render function will be the interpolation factor between the current update frame and the last frame. By using this interpolation factor, we can produce smooth animation even if the update rate is lower than our fps.

##Entity Definition
In our game, an entity is the base class of all game objects. An entity should have a position and should be able to be updated and rendered.

####Transforms
Let\'s create a common way to store position information. We\'ll call it a Transform.

Transforms should have:

* x (position in x direction)
* y (position in y direction)
* vx (velocity in x direction)
* vy (velocity in y direction)
* sx (scale in x direction)
* sy (scale in y direction)
* angle (rotational angle in radians)
* va (velocity of rotational angle)

Let\'s define a Transform object (or in this case, a function that creates an object).

{% highlight javascript %}
function Transform() {
    var _x = 0, 
        _y = 0, 
        _vx = 0, 
        _vy = 0, 
        _sx = 1, 
        _sy = 1,
        _angle = 0,
        _va = 0;

    return {
        x: function(_) {
            if (_ != undefined) {
                _x = _;
            } else {
                return _x;
            }
        },
        y: function(_) {
            if (_ != undefined) {
                _y = _;
            } else {
                return _y;
            }
        },
        vx: function(_) {
            if (_ != undefined) {
                _vx = _;
            } else {
                return _vx;
            }
        },
        vy: function(_) {
            if (_ != undefined) {
                _vy = _;
            } else {
                return _vy;
            }
        },
        sx: function(_) {
            if (_ != undefined) {
                _sx = _;
            } else {
                return _sx;
            }
        },
        sy: function(_) {
            if (_ != undefined) {
                _sy = _;
            } else {
                return _sy;
            }
        },
        angle: function(_) {
            if (_ != undefined) {
                _angle = _;
            } else {
                return _angle;
            }
        },
        va: function(_) {
            if (_ != undefined) {
                _va = _;
            } else {
                return _va;
            }
        }
    };
}
{% endhighlight %}

####Entities
So an entity is just an object with a transform. Easy enough.

{% highlight javascript %}
function Entity() {
    var _transform = new Transform();
    return {
        transform: function() {
            return _transform;
        }
    };
}
{% endhighlight %}

Awesome, now we have an entity class that we can extend later.

##Our Ship
What our game needs now is a Ship. Since everything in our game is an entity, we\'ll define our ship to extend entity.

{% highlight javascript %}
function Ship() {
    var _ship = Object.create(Entity());
    
    return _ship;
}
{% endhighlight %}

Now we can create a ship which is an entity which has a transform by simply calling `Object.create(Ship())`. Nice.

####How does this work?
Object.create was added in ES5 to help manage the minefield that is prototypal inheritance. Essentially it creates a new object, assigns the new object\'s prototype to the passed in object\'s prototype and gives you back your fully constructed object. Confused? Check out this talk for more in depth info [https://speakerdeck.com/getify/new-rules-for-javascript](https://speakerdeck.com/getify/new-rules-for-javascript).

####Drawing our ship
So if we want to actually draw our ship, we\'ll need to add some rendering code to our Ship.

{% highlight javascript %}
function Ship() {
    var _ship = Object.create(Entity());
    
    _ship.render = function(dt, ctx) {
        ctx.strokeStyle = "#FFFFFF";
        ctx.beginPath();
        ctx.moveTo(0, 0);
        ctx.lineTo(1, 1);
        ctx.lineTo(-1.5, 0);
        ctx.lineTo(1, -1);
        ctx.lineTo(0, 0);
        ctx.stroke();
    }
    
    return _ship;
}
{% endhighlight %}

Pretty straight forward, but that looks a little small (-1px by 1px). Let\'s change our transform to make it bigger (the render function will take care of handling the transforms for us but we need to specify what they are here). While we\'re at it, let\'s rotate our ship so it\'s facing the correct direction.

{% highlight javascript %}
function Ship() {
    var _ship = Object.create(Entity());
    
    var _transform = _ship.transform();
    _transform.sx(5);
    _transform.sy(5);
    _transform.angle( -Math.PI / 2);
    
    _ship.render = function(dt, ctx) {
        ctx.strokeStyle = "#FFFFFF";
        ctx.beginPath();
        ctx.moveTo(0, 0);
        ctx.lineTo(1, 1);
        ctx.lineTo(-1.5, 0);
        ctx.lineTo(1, -1);
        ctx.lineTo(0, 0);
        ctx.stroke();
    }
    
    return _ship;
}
{% endhighlight %}

Cool, now we can draw our ship, we just need to hook up our mainloop with our ship.

####Rendering our game\'s entities
To do that we need to add a place to store the entities that are currently in the game. We\'ll add this to our global object.

{% highlight javascript %}
var g = {
    ...
    entities = []
    ...
};
{% endhighlight %}

We also need to modify our render function to clear our canvas and actually draw our entities. To draw our entities, we need to get their transform, translate to the entity\'s location, scale the entity, rotate it, and then draw it.

{% highlight javascript %}
function clearScreen(ctx) {
    ctx.translate(0,0);
    ctx.scale(1,1);
    ctx.rotate(0);
    ctx.fillStyle = "rgb(0,0,0)";
    ctx.fillRect (0, 0, g.width, g.height);
}

function render(dt, ctx) {
    clearScreen(ctx);
    g.entities.forEach(function renderEntity(entity){
        if (entity.render) {
            var transform = entity.transform();
            ctx.save();
            ctx.translate(transform.x() + g.halfWidth - dt * transform.vx(), transform.y() + g.halfHeight - dt * transform.vy()); 
            ctx.scale(transform.sx(), -transform.sy()); //flip axis
            ctx.rotate(transform.angle());
            entity.render(dt, ctx);
            ctx.restore();
        }
    });
}
{% endhighlight %}

You might notice that we\'re transforming all of our graphics context with a negative y scale. This allows us to work in a more traditional coordinate space (x increases going to the right, y increases going up). You might also notice that all positions are added half of the canvas width/height. This allows us to put the origin of our coordinate space at the center of our canvas. Not a lot of changes, but makes things much easier to work with.

##Putting it together
Putting that together we can actually get our canvas up and running, our ship on the screen, and have everything drawing.

{% highlight javascript %}
/* at the end of our code */
var ctx = createCanvas();
g.entities.push(Object.create(Ship()));
startMainLoop(ctx);
{% endhighlight %}

Cool, everything should be up and running. But it\'s a bit boring, sure our ship is there, but it doesn\'t do anything. Lets fix that.

##Keyboard Input
To make our ship fly around, we need to get some keyboard input from the user and respond to it.

Lets add a simple input object to our global that listens for key events in the window and sets on/off states for those key codes.

{% highlight javascript %}
function initInput() {
    g.input = {
        _pressed: {},
        
        THRUST: 38,
        ROTATE_LEFT: 37,
        ROTATE_RIGHT: 39,

        isActive: function(code) {
            return this._pressed[code];
        },

        onKeydown: function(e) {
            this._pressed[e.keyCode] = true;
        },

        onKeyup: function(e) {
            delete this._pressed[e.keyCode];
        }
    };

    window.addEventListener('keyup', function(event) { g.input.onKeyup(event); }, false);
    window.addEventListener('keydown', function(event) { g.input.onKeydown(event); }, false);
}
{% endhighlight %}

As you can see, I\'ve defined some human readable names for our up/left/right arrow keys. We\'ll use those names instead of the keycodes in our update code.

####Updating the ship\'s position based on keyboard input
Now we just need to react to the keyboard input. Lets add an update method to our ship so handle it.

{% highlight javascript %}
var g = {
    rotationSpeed: 0.07, //affect of rotation
    thrustSpeed: 0.3 //affect of thrust
};

function Ship() {
    var _ship = Object.create(Entity());

    var _transform = _ship.transform();
    ...

    _ship.update = function() {
        if (_transform.x() > g.halfWidth) {
            _transform.x(-g.halfWidth);
        } else if (_transform.x() < -g.halfWidth) {
            _transform.x(g.halfWidth);
        }
        if (_transform.y() > g.halfHeight) {
            _transform.y(-g.halfHeight);
        } else if (_transform.y() < -g.halfHeight) {
            _transform.y(g.halfHeight);
        }

        var input = g.input;
        if (input.isActive(input.ROTATE_LEFT)) {
            _transform.va(g.rotationSpeed); 
        }
        if (input.isActive(input.ROTATE_RIGHT)) {
            _transform.va(-g.rotationSpeed); 
        } 
        if (!input.isActive(input.ROTATE_RIGHT) 
                && !input.isActive(input.ROTATE_LEFT)) {
            _transform.va(0); 
        }
        if (input.isActive(input.THRUST)) {
            _transform.vx(_transform.vx() - Math.cos(_transform.angle()) * g.thrustSpeed);
            _transform.vy(_transform.vy() + Math.sin(_transform.angle()) * g.thrustSpeed);
        }
    }

    _ship.render = function(dt, ctx) {...};

    return _ship;
}
{% endhighlight %}

Basically what we have here are the basic physics for our game. If rotate left/right is active, we set our rotational velocity. If thrust is active we calculate our current angle and update our x/y velocities based on that. Also if we go off of the 

Lastly we need to hook up our mainloop to our entities so that they'll be updated. We\'ll grab each entity, get it\'s transform, update the transform based on the available velocities in it and then call update on the entity.

{% highlight javascript %}
function update(dt) {
    g.entities.forEach(function updateEntity(entity){
        var transform = entity.transform();
        transform.x(transform.x() + transform.vx());
        transform.y(transform.y() + transform.vy());
        transform.angle(transform.angle() + transform.va());
        if (entity.update) {
            entity.update(dt);
        }
    });
}
{% endhighlight %}

##Initializing our game
Let\'s wrap up the pieces of our game with a nice init function to set everything up.

{% highlight javascript %}
function init() {
    var ctx = createCanvas();
    g.entities.push(Object.create(Ship()));
    initInput();
    startMainLoop(ctx);
}
{% endhighlight %}

Then we can just call `init();` to start our game.

##Whew that was a lot
Yup, but this isn\'t trivial stuff, we\'ve built an entity system, a keyboard input system, a basic gameloop/updater/renderer, and hooked it into the browser. So now that we\'ve built it, let\'s try it out (feel free to view page source and dig through it). 

[Demo ->](/static/planetoids-part-2/index.html) 

