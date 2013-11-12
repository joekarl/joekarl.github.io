---
layout: post
title: Building a game in the browser (part 1 - Intro)
---

Despite the fact that I\'ve done fairly little gaming (at least compared to some of my peers), I\'ve always wanted to get into game programming. Graphics, sound, and interactivity have always fascinated me, and the challenge of putting them together into a game (and completing it!) seems like a worthy goal. This series of posts will (hopefully) follow my progress in creating a game.

##Why the browser?
Basically because I felt like it, plus my other endeavours into game dev have been in Java which just isn\'t that hip these days. Also I can keep intermediate states of game in working order that will be \"playable\" in the posts.

##What is this game going to be?
Basically I\'m shooting for an Asteroids clone. Let\'s call it \"Planetoids\" for now (legal reasons as expected). 

![asteroids](/assets/images/asteroids_large.png "Asteroids")

The basic things are as follows:

* Basic gameplay (ship, planets, lives, score, aliens)
* Playability (should be fun)
* Code quality (should be clean, should have some reusable parts for future games)
* Open code (code will be available on [github](https://github.com/joekarl/planetoids))


##Parts
I\'m going to tackle this in a few parts:

* Intro - this
* [Getting things setup](/2013/11/12/building-a-game-in-the-browser-part-2) - canvas, game loop, a ship, keyboard input
* Shooting planets - planet creation, ship lasers, collision detection, game over
* Sounds - sounds using howler.js
* Backtracking - revamping with state machines, entity components
* Making it look good - sprites, menus
* Extras - alien AI, particle effects

##All right! Let\'s get started!
Ok, not so fast. It\'s getting late but part 2 - Getting things setup is coming soon!