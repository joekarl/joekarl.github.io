---
layout: post
title: When string concatenation is faster than you think (d3.js edition)
---
One of my recent projects has involved some spherical mercator projection correct rendering of arbitrary paths onto some arbitrary map backgrounds as an SVG. This seemed like a perfect job for Node.js + d3.js! (well maybe, maybe not, but seemed like a good idea at the time)

Lets walk through doing something similar.

##Naive implementation
For our initial implementation, lets go ahead and threw together the code as one would expect to see it with d3.

    var svg = d3.select("body").append("svg");
    var center = [-97, 35];
    var offset = [400, 300];
    var projection = d3.geo.mercator()
        .scale(1)
        .translate([0, 0]);
	var projectedPath = d3.geo.path().projection(projection);
    var features = [
        { 
            type: "Feature",
            geometry: {
                type: "Polygon",
                coordinates: [[-97,35],...,[-97,35]]
            }
        },
        { 
            type: "Feature",
            geometry: {
                type: "Polygon",
                coordinates: [[-97,35],...,[-97,35]]
            }
        }
    ];
    
With out d3 projection and svg element in hand, lets render some SVG elements into it.  We\'ll let d3 do the heavy lifting in terms of 

    svg.append("path")
    	.data(features)
    	.attr("d", projectedPath)
        .style({stroke:"red"});
        
    ... lots more of this sort of thing ...
    
Now all we need to do is convert the svg object into a string so we can work with it (pass it to the browser, render via librsvg, etc...).

    var svgString = "<svg>" + svg.html() + "</svg>";

Which gives us the SVG as a string.

##Whats wrong with this version?
Unfortunately, while d3 is fairly quick to render the svg to string and is quick when operating on the dom elements to append and manipulate elements, d3 is pretty slow to actually initially create the document model and ends up creating being fairly bloated (at least memory usage wise).

Now I can hear you now, \"What? I\'ve never seen d3 be slow just to create a document!\". And you may well be correct for applications in the browser, but for this case (a web server that generates SVGs and tries to get them out the door as quickly as possible), the extra time involved in creating and destroying the document model is really really bad. 

To put it this way, just by having the line `var svg = d3.select("body").append("svg");` in an endpoint can bring express to it\'s knees (around 50 req/s). And that\'s literally just that line, no other processing except creating the initial parent svg object.

##Faster implementation
So what can we do here. Ultimately our goal isn\'t to generate a fancy document model with fancy data binding, but to generate an SVG string. So we can save a lot of time by not generating an object model and just creating our SVG string directly.

So where to start. First we need to setup d3 again:

    var center = [-97, 35];
    var offset = [400, 300];
    var projection = d3.geo.mercator()
        .scale(1)
        .translate([0, 0]);
    var projectedPath = d3.geo.path().projection(projection);
    var features = ...;
    
Now, rather than using d3 to create our SVG, we\'ll just concatenate our string together.

    var svg = '<svg>';
    features.forEach(function renderFeature(feature){
        svg += '<path d="';
        feature.geometry.coordinates.forEach(function renderProjectedPoint(coordinate, i){
            if (i == 0) {
                svg += 'M';
            } else {
                svg += 'L';
            }
            var projectedPoint = projection(coordinate);
            svg += projectedPoint[0] + ' ' + projectedPoint[1] + ' ';
        });
        svg += 'Z" />';
    });
    svg += '</svg>';
    
This will give us the same svg that we got before with the correct projected paths (essentially what d3 was doing for us except we skip all of the extra object stuff created by d3). After some testing, this dramatically increases throughput (now we\'re up to 1,300 req/s) and we\'re using way less memory as well.

##What have we learned
While nice, general, clean APIs are great when maintenance is the big concern (which it almost always is), sometimes you have to break some rules to get the performance that you need (especially in node). Don\'t be afraid to break the rules, but be sure to validate those changes by profiling.
    
    