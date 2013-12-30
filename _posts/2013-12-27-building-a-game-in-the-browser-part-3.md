---
layout: post
title: Building a game in the browser (part 3 - Planets and Lasers Oh My!!)
---

In [part 2](/2013/11/10/building-a-game-in-the-browser-part-2) we laid out the basic structure we neede to make a game. We setup an entity, a ship, keyboard input, a main loop, ... etc. Now lets get to the fun bits.

##Part 3 - Planets and Lasers Oh My
The things we\'ll build today are:

* A better entity manager
* Planets
* Lasers
* Collision Detection
* Working Game!!!!

##A better entity manager
In our last segment, we had a simple array to act hold all of our entities. Unfortunately, that doesn\'t work so well once we start doing more complex things. Let\'s create a better entity manager that will allow us to add/remove entities on the fly.

{% highlight javascript %}
g.entityManager = {
    _entities: {},
    addEntity: function(entity) {
        return this._entities[entity.id()] = entity;
    },
    removeEntity: function(entity) {
        delete this._entities[entity.id()];
        return entity;
    },
    entityIds: function() {
        return Object.keys(this._entities);
    },
    entity: function(id) {
        return this._entities[id];
    }
};
{% endhighlight %}

Our entity manager is just a simple object with an inner store for the entities and some convenience methods to get an entity and iterate over the entities.

Our entities now need to have a unique id, that will be useful to add/remove entities by id. Let\'s add that.

{% highlight javascript %}
function Entity() {
    var _transform = new Transform();
    var _id = g.entityId++;
    
    return {
        transform: function() {
            return _transform;
        },
        id: function() {
            return _id;
        }
    };
}
{% endhighlight %}

We do have to update our code to use this entity manager so everywhere where there was `g.entities.push` we\'ll need to update it to `g.entityManager.addEntity(...)`. Also anywhere where we iterate over our entities, we\'ll need to update that to use the `g.entityManager.entityIds()` call.

##Planets
Our planets are going to be simple, they\'ll basically be entities (like our ship) and they\'ll be drawn as craggy (crappy?) circles. 

This is pretty straight forward, we create an entity (as before), define a bunch of points to create a convex polygon, adjust each point by a random amount and then use those points to draw our planet.

![craggy_asteroid](/assets/images/craggy_asteroid.png "Craggy Asteroid")

As for updating the planet\'s position, we don\'t have to worry about any user interaction so we can just set a random position, a random velocity, and let the planet float around. When the planet reaches the edge of the screen we wrap it around to the other side.


{% highlight javascript %}
function Planet(scale) {
    var _planet = Object.create(Entity());
    scale = scale;

    var _vertexes = [];
    var _vertexCount = Math.round(Math.random() * 4 + 12); // between 8 and 16
    var _radianIncrement = 2 * Math.PI /  _vertexCount;
    var _i, _radiusAdjust, _radians = 0;
    for (_i = 0; _i < _vertexCount; ++_i) {
        _radiusAdjust = scale * (0.25 - (Math.random() * 100) / 100 * 0.5);
        _vertexes.push([
            Math.cos(_radians) * (scale + _radiusAdjust), 
            Math.sin(_radians) * (scale + _radiusAdjust)
        ]);
        _radians += _radianIncrement;
    }

    var _transform = _planet.transform();

    _transform.x((Math.random() * -g.width) + g.halfWidth)
        .y((Math.random() * -g.height) + g.halfHeight)
        .vx((Math.random() * -g.planetSpeed) + g.planetSpeed / 2)
        .vy((Math.random() * -g.planetSpeed) + g.planetSpeed / 2);
        
    _planet.scale = function() {
        return scale;
    };

    _planet.update = function() {
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
    }

    _planet.render = function(dt, ctx) {
        ctx.strokeStyle = "#FF0000";
        ctx.lineWidth = 1;
        ctx.beginPath();
        ctx.moveTo(_vertexes[_vertexCount - 1][0], _vertexes[_vertexCount - 1][1])
        for (var i = 0; i < _vertexCount; ++i) {
            var vertex = _vertexes[i];
            ctx.lineTo(vertex[0], vertex[1]);
        }
        ctx.stroke();
    };

    return _planet;
}
{% endhighlight %}

Pretty simple. Basically the same as our ship entity, just no keyboard input, less variables, and a different render function.

##Lasers
Once again, our laser is just another entity. This time though, we need to set an initial position based on our ship\'s location. The ship\'s location is provided when the laser is created. 

**Note** A laser doesn\'t last forever. It lives for a specified amount of time (in this case it\'s the g.laserLife global variable). Each update frame we decrease the amount of life the laser has, and when we reach 0, we destroy the laser by removing it from the entity manager.

{% highlight javascript %}
unction Laser(transform) {
    var _laser = Object.create(Entity());

    var _transform = _laser.transform();
    _transform.copy(transform)
        .x(_transform.x() - Math.cos(_transform.angle()) * 7.5)
        .y(_transform.y() + Math.sin(_transform.angle()) * 7.5)
        .vx(_transform.vx() - Math.cos(_transform.angle()) * g.laserSpeed)
        .vy(_transform.vy() + Math.sin(_transform.angle()) * g.laserSpeed)
        .angle(0)
        .va(0)
        .sx(2)
        .sy(2);

    var _life = g.laserLife;

    _laser.update = function() {
        if (_life-- < 0) {
            g.entityManager.removeEntity(this);
        }
    };
    
    _laser.destroy = function() {
        _life = -1;
    }

    _laser.render = function(dt, ctx) {
        ctx.strokeStyle = "#FFFFFF";
        ctx.beginPath();
        ctx.moveTo(0, 0);
        ctx.lineTo(-1, 0);
        ctx.stroke();
    }

    return _laser;
}
{% endhighlight %}

Now we just need to be able to create a laser when the fire key is pressed. Let\'s add the space bar to the input manager as our fire key. `LASER: 32, //space`

Then in our ship code, we just need to create a new laser when the fire key is being pressed.

{% highlight javascript %}
_ship.update = function() {
    ...
    if (input.isActive(input.LASER)) {
        g.entityManager.addEntity(Object.create(Laser(_transform)));
    }
    ...
};
{% endhighlight %}

Note that we didn\'t do anything to our laser other than insert it into the game, the laser itself handles it\'s behaviour once it\'s been created.

##Collision Detection 
For collision detection we need to define the bounds of our objects (so we can check if they collide). One of the simplest ways to do this is to use circles to define the bounds around our objects. By doing this, we get a quick way to check if our objects collided or not (quick is good when it comes to collision checking).

###Bounding boxes
Here\'s what the bounds would look like (if we drew them).

![collision_asteroid](/assets/images/collision_bounds_asteroid.png "Collision Bounds Asteroid")

To start, we need to define a bounds object. It will simply hold the radius of our bounding object.

{% highlight javascript %}
function BoundingCircle(radius) {
    var _radius = radius;
    
    return {
        radius: function(_) {
            if (_ != undefined) {
                _radius = _;
                return this;
            } else {
                return _radius;
            }
        }
    };
}
{% endhighlight %}

We also need to add a bounding circle to our entity definition. While we\'re at it, lets also add an type field for our entity. Don\'t worry, you\'ll see later.

{% highlight javascript %}
function Entity() {
    var _transform = new Transform();
    var _id = g.entityId++;
    var _boundingCircle;
    var _type = "Entity";
    
    return {
        transform: function() {
            return _transform;
        },
        id: function() {
            return _id;
        },
        boundingCircle: function(_) {
            if (_ != undefined) {
                _boundingCircle = _;
                return this;
            } else {
                return _boundingCircle;
            }
        },
        type: function(_) {
            if (_ != undefined) {
                _type = _;
                return this;
            } else {
                return _type;
            }
        }
    };
}
{% endhighlight %}

Now that our base entity has a place to store a bounding circle, we can define bounding circles for each or our entity types.

####Ship
{% highlight javascript %}
function Ship() {
    ...
    _ship.boundingCircle(Object.create(BoundingCircle(7.5)));
    ...
}
{% endhighlight %}

####Laser
{% highlight javascript %}
function Laser() {
    ...
    _laser.boundingCircle(Object.create(BoundingCircle(1)));
    ...
}
{% endhighlight %}

####Planet
{% highlight javascript %}
function Planet() {
    ...
    _planet.boundingCircle(Object.create(BoundingCircle(scale)));
    ...
}
{% endhighlight %}

###Checking for Collisions
All we need to do now is check if there\'s been any collisions. This is going to be brute force (so n^2 time) but will get the job done here, though you wouldn\'t want to do this in a larger game.

{% highlight javascript %}
function update(dt) {
    var collisionCheckedEntities = {};
    g.entityManager.entityIds().forEach(function (id){
        /*
            update entity code here
        */
        
        collisionCheckedEntities[id] = [];
        //check for collisions
        g.entityManager.entityIds().forEach(function (collisionId){
            var collisionEntity = g.entityManager.entity(collisionId);
            collisionCheckedEntities[id][collisionId] = true;
            if (id == collisionId 
                || (collisionCheckedEntities[collisionId] && collisionCheckedEntities[collisionId][id])
                || !entity.boundingCircle()
                || !collisionEntity.boundingCircle()) {
                return;
            }
            
            var combinedRadius = collisionEntity.boundingCircle().radius()
                + entity.boundingCircle().radius();
            if (transform.distance(collisionEntity.transform()) < combinedRadius) {
                //collision!!!
                if (entity.handleCollision) {   
                    entity.handleCollision(collisionEntity);
                }
                if (collisionEntity.handleCollision) {
                    collisionEntity.handleCollision(entity);
                }
            }
        });
    });
}
{% endhighlight %}

Pretty straightforward. Run through all of our entities, check if there are any collision, and for each collision try calling handleCollision for each entity.


##Abrupt segue to  Working Game!!!!

[Demo ->](/static/planetoids-part-3/index.html) 

