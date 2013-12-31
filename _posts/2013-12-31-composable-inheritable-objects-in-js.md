---
layout: post
title: Composable inheritable objects in js
---

Many times in Javascript, we can ignore the whole prototypal inheritance thing and blindly go creating one off objects. Other times, we do need inheritance. Fortunately, Javascript supports some nice features to enable this sort programming. Unfortunately, it can be a bit hard to wrap one\'s head around. Here are some examples of how to build composable, inheritable objects with Javascript.

##Prototypal inheritance
I\'m punting here. Essentially this is the crux of how everything works. If you aren\'t familiar with what is actually going on here, check this out [http://javascriptweblog.wordpress.com/2010/06/07/understanding-javascript-prototypes/](http://javascriptweblog.wordpress.com/2010/06/07/understanding-javascript-prototypes/) or this [http://dmitrysoshnikov.com/ecmascript/chapter-7-2-oop-ecmascript-implementation/](http://dmitrysoshnikov.com/ecmascript/chapter-7-2-oop-ecmascript-implementation/).

##Classical way to do prototypal inheritance
The classic way to do prototypal inheritance, is using `new` and setting things in the object\'s prototype. So something like this: 

{% highlight javascript %}
function Foo(x, y) {
    this.x = x;
    this.y = y;
}

Foo.prototype.getX = function() {
    return this.x;
}

Foo.prototype.getY = function() {
    return this.y;
}

function Bar(x, y, z) {
    Foo.call(this, x, y);
    this.z = z;
}

Bar.prototype = Object.create(Foo.prototype);

Bar.prototype.getZ = function() {
    return this.z
}

var f = new Foo(1,2);
var b = new Bar(3,4,5);

console.log("f x,y: " + f.getX() + "," + f.getY()); //1,2
console.log("b x,y,z: " + b.getX() + "," + b.getY() + "," + b.getZ()); //3,4,5
console.log("b is Foo? " + (b instanceof Foo)); //true
{% endhighlight %}

##Working with prototype objects instead of function prototypes
In the classic version, we\'re constantly referencing the object\'s prototype and splitting out the definition of our methods.

Let\'s try to make this a bit more concise by wrapping up the method definitions in an anonymous function.

{% highlight javascript %}
var Foo = {
    init: function (x, y) {
        this.x = x;
        this.y = y;
        return this;
    },
    getX: function() {
        return this.x;
    },
    getY: function() {
        return this.y;
    }
};

var Bar = (function(){
    var _bar = Object.create(Foo);
    _bar.init = function(x,y,z) {
        Foo.init.apply(this,x,y);
        this.z = z;
        return this;
    };
    _bar.getZ = function() {
        return this.z;
    };
    return _bar;
})();

var f = Object.create(Foo).init(1,2);
var b = Object.create(Bar).init(3,4,5);

console.log("f x,y: " + f.getX() + "," + f.getY()); //1,2
console.log("b x,y,z: " + b.getX() + "," + b.getY() + "," + b.getZ()); //3,4,5
console.log("b is Foo? " + Foo.isPrototypeOf(b)); //true
{% endhighlight %}

####What is this madness!?
You may be wondering what just happened\... Basically what we did was get rid the middle man and work with our object prototype directly. This does have the side effect of getting rid of our object\'s constuctor, but that is easily overcome with an init method. 

##Inheritence with property descriptors
That\'s still a bit cumbersome, lets clean up our code with some property descriptors.

{% highlight javascript %}
var Foo = Object.create({}, {
    x: {writable: true, value: 0},
    y: {writable: true, value: 0},
    init: {
        value: function(x, y){
            this.x = x;
            this.y = y;
            return this;
        }
    }
});

var Bar = Object.create(Foo, {
    z: {writable: true, value: 0},
    init: {
        value: function(x, y, z) {
            Foo.init.apply(this, [x, y]);
            this.z = z;
            return this;
        }
    }
});

var f = Object.create(Foo).init(1,2);
var b = Object.create(Bar).init(3,4,5);

console.log("f x,y: " + f.x + "," + f.y); //1,2
console.log("b x,y,z: " + b.x + "," + b.y + "," + b.z); //3,4,5
console.log("b is Foo? " + Foo.isPrototypeOf(b)); //true
{% endhighlight %}

####Property Descriptors?
Yup, you can specify the prototype you\'re inheriting from and specify properties on the object\'s own prototype at the same time. Details are here: [https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global\_Objects/Object/create#Using\_&lt;propertiesObject&gt;\_argument\_with_Object.create](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/create#Using_<propertiesObject>_argument_with_Object.create) [https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global\_Objects/Object/defineProperty](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)
    
This is pretty much just syntactic sugar at this point but makes things a little more concise.

##Other cool things, composable objects
Say now we want all Zed objects to share a single Bar object. We can easily achieve this by wrapping our object creation in a self executing function:

{% highlight javascript %}
var Foo = Object.create({}, {
    x: {writable: true, value: 0},
    y: {writable: true, value: 0},
    init: {
        value: function(x, y){
            this.x = x;
            this.y = y;
            return this;
        }
    }
});

var Bar = Object.create(Foo, {
    z: {writable: true, value: 0},
    init: {
        value: function(x, y, z) {
            Foo.init.apply(this, [x, y]);
            this.z = z;
            return this;
        }
    }
});

var Zed = Object.create({}, {
    bar: {value: Object.create(Bar)}
});

var f = Object.create(Foo).init(1,2);
var b = Object.create(Bar).init(3,4,5);
var z = Object.create(Zed);
z.bar.x = 6;
z.bar.y = 7;
z.bar.z = 8;

var z2 = Object.create(Zed); //this will have the same bar value

console.log("f x,y: " + f.x + "," + f.y); //1,2
console.log("b x,y,z: " + b.x + "," + b.y + "," + b.z); //3,4,5
console.log("z bar x,y,z: " + z.bar.x + "," + z.bar.y + "," + z.bar.z); //6,7,8
console.log("z2 bar x,y,z: " + z2.bar.x + "," + z2.bar.y + "," + z2.bar.z); //6,7,8
console.log("b is Foo? " + Foo.isPrototypeOf(b)); //true
{% endhighlight %}

####What about a new Bar object per instance?
We can do this to, just need to setup a getter that uses lazy initialization to create a Bar object as needed and then caches it for later use:

{% highlight javascript %}
var Foo = Object.create({}, {
    x: {writable: true, value: 0},
    y: {writable: true, value: 0},
    init: {
        value: function(x, y){
            this.x = x;
            this.y = y;
            return this;
        }
    }
});

var Bar = Object.create(Foo, {
    z: {writable: true, value: 0},
    init: {
        value: function(x, y, z) {
            Foo.init.apply(this, [x, y]);
            this.z = z;
            return this;
        }
    }
});

var Zed = Object.create({}, {
    bar: {
        get: function() {
            if (this.__bar__) {
                return this.__bar__;
            } else {
                this.__bar__ = Object.create(Bar);
                return this.__bar__;
            }
        }
    }
});

var f = Object.create(Foo).init(1,2);
var b = Object.create(Bar).init(3,4,5);
var z = Object.create(Zed);
z.bar.x = 6;
z.bar.y = 7;
z.bar.z = 8;

var z2 = Object.create(Zed);

console.log("f x,y: " + f.x + "," + f.y); //1,2
console.log("b x,y,z: " + b.x + "," + b.y + "," + b.z); //3,4,5
console.log("z bar x,y,z: " + z.bar.x + "," + z.bar.y + "," + z.bar.z); //6,7,8
console.log("z2 bar x,y,z: " + z2.bar.x + "," + z2.bar.y + "," + z2.bar.z); //0,0,0
console.log("b is Foo? " + Foo.isPrototypeOf(b)); //true
{% endhighlight %}

Of course you could leave that property undefined and define it after construction too: 

{% highlight javascript %}
var Foo = Object.create({}, {
    x: {writable: true, value: 0},
    y: {writable: true, value: 0},
    init: {
        value: function(x, y){
            this.x = x;
            this.y = y;
            return this;
        }
    }
});

var Bar = Object.create(Foo, {
    z: {writable: true, value: 0},
    init: {
        value: function(x, y, z) {
            Foo.init.apply(this, [x, y]);
            this.z = z;
            return this;
        }
    }
});

var Zed = Object.create({}, {
    bar: {writable: true}
});

var f = Object.create(Foo).init(1,2);
var b = Object.create(Bar).init(3,4,5);
var z = Object.create(Zed);
z.bar.x = 6;
z.bar.y = 7;
z.bar.z = 8;

var z2 = Object.create(Zed);
z2.bar = Object.create(Bar);

console.log("f x,y: " + f.x + "," + f.y); //1,2
console.log("b x,y,z: " + b.x + "," + b.y + "," + b.z); //3,4,5
console.log("z bar x,y,z: " + z.bar.x + "," + z.bar.y + "," + z.bar.z); //6,7,8
console.log("z2 bar x,y,z: " + z2.bar.x + "," + z2.bar.y + "," + z2.bar.z); //0,0,0
console.log("b is Foo? " + Foo.isPrototypeOf(b)); //true
{% endhighlight %}

##Things to take away
There\'s a lot of different ways to create objects in Javascript and allow inheritence to still be available. Don\'t shy away from them.


