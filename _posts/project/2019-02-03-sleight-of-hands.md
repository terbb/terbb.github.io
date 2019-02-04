---
title: "Sleight of Hands"
excerpt: "Play fun minigames using Leap Motion! Try to survive as long as possible."
github: https://github.com/terbb/nwhacks2018/tree/master/sample_video
categories: Project
image_sliders:
  - sleightofhands_slider
tags:
- Three.js
- Leap Motion SDK
---
#### Code:
<a href="https://github.com/terbb/nwhacks2018">View the code here!</a>

#### Summary:

Sleight of Hands is a collection of minigames designed to work with the Leap Motion. You use your hand to beat punching bags, play Simon Says, inflate balloons and rub pet boxes!

If you don't complete the minigame within the time limit, you lose a life. Try and survive for as long as you can!

Made in 24 hours at **nwHacks2018**.

Built using Three.js, an API that uses WebGL, and the Leap Motion SDK.

*Unfortunately, there is no executable. You also require the Leap Motion to play.*

{% include slider.html selector="sleightofhands_slider" %}

#### Notable Achievements:
* Led a team of 4 to create the game, delegating each member to create a minigame of their own.
* Set up the initial scene, complete with camera, spotlight and WebGL renderer.
* Created a custom door-closing animation to transition between minigames that would calculate where to create the doors from the camera position.
* Wrote a minigame prototype class for members to use, along with a minigame loader that would handle loading random minigames as well as teardown
* Added lives to the player.
* Added a timer that would subtract the player's life when it reached 0.
* Loaded a custom font and created a custom mesh and geometry out of it to display the current time.


#### Version Changes:

**01/14/2018 -** The game was created at nwHacks2018.

#### Credits:

Created by Trevin Wong, Alison Wu, Sean Allen and Christine Cheng.
