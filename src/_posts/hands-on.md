---
title: Hands on! Building interactive training with A-Frame
author: twitter|salvadelapuente|Salvador de la Puente
date: 2017-07-26 12:00:00
layout: blog
image:
  src: hands-on.png
---

In an industry where manipulation of complex mechanisms involves dozens of tools, expensive equipment and even safety risks, virtual reality helps to reduce material costs and increase operator safety. WebVR puts everything just one link away.

In this tutorial you will learn about core A-Frame concepts, its ecosystem, the entity-component-system architecture, how to use the [A-Frame physics system](https://hacks.mozilla.org/2017/05/having-fun-with-physics-and-a-frame/), common errors, workarounds and good practices. All you need to confront a real project.

This is an advanced virtual reality experience that requires a head-mounted display supporting room-scale VR and [6DoF](https://en.wikipedia.org/wiki/Six_degrees_of_freedom) controllers like those included wih the [HTC VIVE](https://www.vive.com/us/product/#controller-intro) or the [Oculus Rift](https://www.oculus.com/rift). This tutorial was developed using an HTC Vive.

## Project Setup

Use git to clone the [repository of the simulation](https://github.com/delapuente/aframe-interactive-training). Enter the repository and run `npm install` then `npm start` to install the dependencies and start the development server at port `8080`.

Open the browser and enter the URL `localhost:8080` to find the index of steps. Append `step1` at the end of the URL path to experience the project at this point in the series (or select step 1 in the list).

The project is based on the [A-Frame SPA skeleton](https://github.com/delapuente/aframe-spa-skeleton); you can use this to quickly set up the barebones of a virtual reality single-page application with auto reload and publishing features. Read the [a-frame-skeleton README file](https://github.com/delapuente/aframe-spa-skeleton#a-frame-spa-skeleton) for further information.

## Project Goals

The very first thing you need to start implementing a simulation experience is a clear reference of the real procedure. Some clues? Watch videos, take descriptive notes and diagrams, and maintain a fluid conversation with an expert.

In this tuorial we will simulate the process of backing up the contents of a chip from an in-car key reader device, which includes loosening the solder and removing the chip from a circuit board. We used a YouTube video showing the [complete in-real-life procedure](https://youtu.be/qFGX3AHVLqA?t=90) as a reference. The video comprises several steps involving around eight different tools. For this tutorial, we will focus on the specific step on removing the chip from the circuit board. See a detailed sequence here:

<iframe width="560" height="315" src="https://www.youtube.com/embed/tHP2kX6aAZM?rel=0&amp;showinfo=0" frameborder="0" allowfullscreen></iframe>

These are the steps of the process we will simulate in VR:

  1. With a hot air gun, soften the soldering around the chip.
  2. With the help of a suction pad, separate the chip from the circuit board.
  3. Carefully place the chip on the table, to avoid ruining the chip pins.

We started with low fidelity models to focus on the interaction and once the whole process is recreated we can polish the visual realism. This is a common practice when prototyping game environments called [**greyboxing**](http://jackw-gamedesign.tumblr.com/post/139960850160/what-is-greyboxing).

## Setting up the scene

The first thing we need is a scene containing all the elements we want to work with: the air gun, the suction pad, the chip, the soldering around the chip, and the circuit board. We need a representation of the user's hands and some environmental elements such as a sky, a floor and of course a workbench!

This is the initial HTML code for the scene:

```html
<a-scene>
  <!-- World -->
  <a-sky color="#333"></a-sky>
  <a-plane rotation="-90 0 0"
           width="15" height="15" color="#777">
  </a-plane>
  <!-- Workspace -->
  <a-entity id="workspace" position="0 0.8 -0.8">
    <a-box id="workbench" position="0 0 0"
           width="2" depth="1" height="0.02" color="#97b7d5">
    </a-box>
    <a-box id="circuitboard" position="0 0.0115 0.4"
           width="0.05" depth="0.05" height="0.003" color="#008000">
      <a-box id="chip" position="0 0.002 0.005"
             width="0.02" depth="0.02" height="0.001" color="#000">
      </a-box>
    </a-box>
    <a-cylinder id="suction-pad"
                position="-0.2 0.015 0.286" rotation="-90 0 0"
                height="0.1" radius="0.005" color="red">
    </a-cylinder>
    <a-cylinder id="hot-air-gun"
                position="0.3 0.02 0.286" rotation="-90 0 0"
                height="0.15" radius="0.01" color="blue">
    </a-cylinder>
  </a-entity>
  <!-- Operator -->
  <a-camera position="0 0 0" user-height="1.3"></a-camera>
  <a-entity id="left-hand" hand-controls="left"></a-entity>
  <a-entity id="right-hand" hand-controls="right"></a-entity>
</a-scene>
```

When working with A-Frame, all the scene objects are contained inside the `a-scene` element. A-Frame provides some special tags to represent common elements in the scene such as a sky dome, a camera or geometric primitives like boxes, planes or cylinders.

The HTML below sets up the environment with a sky dome, and a plane with a couple of sad grey tones.

```html
<a-sky color="#333"></a-sky>
<a-plane rotation="-90 0 0"
         width="15" height="15" color="#777">
</a-plane>
```

In A-Frame, length properties like position or physical dimensions like width or height are represented in meters following the ([**international system**](https://en.wikipedia.org/wiki/International_System_of_Units)). Angles, are expressed in sexagesimal degrees (0º to 360º). In our scene the floor has a surface of _15m x 15m_, the workbench is _2cm_ thick and has a surface of _2m x 1m_, and the circuit board is _3mm thick_ and measures _5cm x 5cm_. Having a realistic scale is important they **improve the feeling of immersion** and provide **realistic physics**.

When placing elements, take into account that they are placed according to their central point in the coordinate system of the parent element. If they are direct children of the `a-scene` element, then they are positioned according to the world coordinates system. In the case of the chip, **it is positioned relative to the central point of the circuit board** because the circuit board is the parent node of the chip.

```html
<a-box id="circuitboard" position="0 0.0115 0.4"
       width="0.05" depth="0.05" height="0.003" color="#008000">
  <a-box id="chip" position="0 0.002 0.005"
         width="0.02" depth="0.02" height="0.001" color="#000">
  </a-box>
</a-box>
```

A-Frame provides a way of adding empty nodes to the scene or, in A-Frame jargon, an **entity**. [Entities](/docs/0.6.0/core/entity.html) are represented by the `a-entity` element and they have nothing associated with them: no geometry, model, or material. They do have implicit `rotation`, `position`, `scale` and `visible` attributes, which are automatically injected by the framework for all the elements within the scene.

Adding the [`hand-controls`](/docs/0.6.0/components/hand-controls.html) attribute to an entity displays a hand model following the position and orientation of one of the tracked controllers of an HTC Vive or Oculus.

```html
<a-entity id="left-hand" hand-controls="left"></a-entity>
<a-entity id="right-hand" hand-controls="right"></a-entity>
```

This is the resulting scene:

[![Initial scene](/images/blog/hands-on/step1-initial-scene.png)](/images/blog/hands-on/step1-initial-scene.png)

### One workspace to rule them all

When developing the simulation we placed all the elements as if they were resting on the physical desk only 5cm higher up. This allowed us to avoid the hands colliding with the physical keyboard when in VR mode. You can move the whole workspace at the same time by changing the `position` attribute of the `workspace` entity. Having the real and virtual table size and position matching helps to improve the feeling of immersion.

A simpler alternative would have been to make the circuit board and tools children of the table. Unfortunately, this configuration does not play well when adding physics. The physics engine would consider the nested elements to be compounding parts of the root element and we didn’t want the tools and circuit board to be considered part of the table but separated elements.

Also notice that if you are experiencing room-scale VR, you will also need to position the workspace and camera relative to the center of your configured stage, making them match with your real-life table and yourself respectively.

Finally, while developing we spent most of the time sitting in front of the computer. As a final touch we changed the height correction of the camera, the `user-height` attribute, to match the distance of the eyes from the floor when sitting down:

```html
<a-camera position="0 0 0" user-height="1.3"></a-camera>
```

This correction is [automatically disabled when entering VR](/docs/0.6.0/components/camera.html#vr-behavior) mode since the position of the camera is provided by the VR head-mounted display (HMD).

## Your hands in Virtual Reality

An accurate correspondence between virtual and real objects is vital to increase the sense of presence. The problem with the hands models in the scene is that they are not oriented in the same way than your real hands hold the controllers.

Take a look at the following video, focus on how the real fingertips and the virtual hand's don't match. The grab axis, around which the fingers close, is also misaligned.

<video controls src="/images/blog/hands-on/step2-the-holding-problem.mp4"></video>

The simulation is more effective when using the 3D models of the VIVE controllers since the tools you see in VR will match the real tools that you hold in your hands. Using the `vive-controls` instead of `hand-controls` attribute also [allow us to change the model](/docs/0.6.0/components/vive-controls.html#value_model) in the future so we can provide a better representation of the hands.

```html
<a-entity id="left-hand"
          vive-controls="hand: left">
</a-entity>

<a-entity id="right-hand"
          vive-controls="hand: right">
</a-entity>
```

## Adding Physics

The [`aframe-physics-system`](https://github.com/donmccurdy/aframe-physics-system) extension is a wrapper around the [Cannonjs physics engine](http://www.cannonjs.org/) developed by [Don McCurdy](https://twitter.com/donrmccurdy). Mozilla Hacks blog has a good introduction to this extension in [Having fun with physics and A-Frame](https://hacks.mozilla.org/2017/05/having-fun-with-physics-and-a-frame/).

Enabling physics in A-Frame consists on importing the proper module after importing A-Frame (see `/js/step2/index.js`):

```js
import * as AFRAME from 'aframe';
import * as physics from 'aframe-physics-system';
```

Next, we need to set some attributes in the elements of the scene to define how they will be affected by the physics engine. We made the floor, the workbench, and the operator's hands _static bodies_ by adding the `static-body` attribute to these elements. A _static body_ is still a physical object but its properties are not controlled by the physics engine &mdash;they are not affected by gravity or other forces, which prevents the floor and table from falling indefinitely.

```html
<a-plane rotation="-90 0 0"
         static-body
         width="15" height="15" color="#777">
</a-plane>

<a-box id="workbench" position="0 0 0"
       static-body
       width="2" depth="1" height="0.02" color="#97b7d5">
</a-box>

<a-entity id="left-hand"
          static-body="shape: box"
          vive-controls="hand: left">
</a-entity>

<a-entity id="right-hand"
          static-body="shape: box"
          vive-controls="hand: right">
</a-entity>
```

On the contrary, a _dynamic body_ is an object fully controlled by the physics engine. We turned the circuit board containing the chip into a _dynamic body_ by setting the attribute `dynamic-body` on it. The value of the property establishes the mass of the object &mdash; We set it to weigh 10g (which equates to 0.01, since Cannon.js uses [S.I.](https://en.wikipedia.org/wiki/International_System_of_Units) units, so kilograms for mass).

```html
<a-box id="circuitboard" position="0 0.0215 0.4"
       dynamic-body="mass: 0.01"
       width="0.05" depth="0.05" height="0.003" color="#008000">
  <a-box id="chip" position="0 0.002 0.005"
         width="0.02" depth="0.02" height="0.001" color="#000">
  </a-box>
</a-box>
```

Dynamic bodies collide with other dynamic bodies **and with static bodies**. This is what the scene looks like after enabling physics:

[![Red boxes around the floor and table. The circuit board is missing.](/images/blog/hands-on/step2-initial-scene.png)](/images/blog/hands-on/step2-initial-scene.png)

Red boxes are a result of the physics engine debug mode being enabled. You can enable it by setting the `physics` attribute of the `scene` tag to `debug: true`. You'll see a red box around the workbench, floor and circuit board... wait a moment! Where the heck is the circuit board!?

[![The circuit board is under the table. It fell through the table somehow.](/images/blog/hands-on/step2-where-is-the-chip.gif)](/images/blog/hands-on/step2-where-is-the-chip.gif)

Oh! There it is.

### Limitations of the physics system

It turns out that the chip is not resting on the table but on the floor. The chip fell through the table because of the thickness of their physical bodies.

Indeed, Cannon.js [lacks from continuous collision detection](https://github.com/schteppe/cannon.js/issues/50#issuecomment-13730383): it only checks if an object is colliding another from frame to frame. With the Earth gravity (which is the default in Cannon.js), an [object falls around 3cm after the first 80ms](https://en.wikipedia.org/wiki/Free_fall#Uniform_gravitational_field_without_air_resistance) (~5 frames). Assuming we were probably missing some frames during the scene setup, a quick workardound was to increase the thickness (the `height` attribute of a box) of the workbench by a couple of centimetres to prevent the circuit board from passing through the table. This forced me to relocate the elements resting on the table, but this was preferable to changing the gravity.

```html
<a-box id="workbench" position="0 0 0"
       static-body
       width="2" depth="1" height="0.04" color="#97b7d5">
</a-box>
```

Another problem regarding the physics engine is related to the collision bounding shapes used for rotated models such as the VIVE controllers. They are miscalculated in terms of dimensions and positioned out of place for certain initial configurations.

[![Miscalculated and misplaced bounding box.](/images/blog/hands-on/step2-miscalculated-collision-box.png)](/images/blog/hands-on/step2-miscalculated-collision-box.png)

Nevertheless, this is something We will fix later in this tutorial. For now, you can experiment with the scene using the bounding box to move the circuit board around. Since the controllers can pass through the table, a funny interaction is trying to raise the chip placing your controller below it:

[![The VIVE controls interact with the circuit board realistically.](/images/blog/hands-on/step2-raising-the-circuit-board.gif)](/images/blog/hands-on/step2-raising-the-circuit-board.gif)

## The entity-component-system architecture

The [entity-component-system pattern](https://en.wikipedia.org/wiki/Entity%E2%80%93component%E2%80%93system) (ECS) is an architectural pattern in which objects or entities are composed of multiple aspects. A component captures an aspect of an entity and a system orchestrates the global interactions among the entities possessing a component of the same aspect. The A-Frame documentation includes a [complete chapter dedicated to ECS](/docs/0.6.0/introduction/entity-component-system.html).

In A-Frame, the entities are the `a-entity` elements and all the primitives (`a-box`, `a-plane`, `a-cylinder`, `a-camera`...) included in the scene. Components are represented by the elements' attributes so, hereinafter, we will stop using the word _attribute_ and start using _component_ when applicable. For instance, consider the operator's hands:

```html
<a-entity id="left-hand"
          static-body="shape: box"
          vive-controls="hand: left">
</a-entity>
<a-entity id="right-hand"
          static-body="shape: box"
          vive-controls="hand: right">
</a-entity>
```

Each hand is actually an entity which has two components `static-body` and `vive-controls`. The first is required by the physics engine to detect when it is colliding with other bodies. The second shows the specified controller making it automatically match the position and rotation of the tracked control. Components can be one-value such as the former `hand-controls` or multi-value like `static-body` or `vive-controls` (i.e., comprised of several named values like the _mass_ or the colliding _shape_). The different values of a multi-value component are called properties.

Finally, systems are pure behaviour extensions and therefore, they are usually invisible. Take the physics engine for instance. It is a system and it orchestrates the interactions between those entities with `static-body` and `dynamic-body` components (among others).

### A note on primitives

[Primitives are entities under the hood](/docs/0.6.0/introduction/html-and-primitives.html#primitives). Primitives have specific attributes mapped to certain properties of components implied by the primitive. For instance:

```html
<a-cylinder id="suction-pad"
            position="-0.2 0.025 0.286" rotation="-90 0 0"
            height="0.1" radius="0.005" color="red">
</a-cylinder>
```

This `a-cylinder` primitive implies [`geometry`](/docs/0.6.0/components/geometry.html) and [`material`](/docs/0.6.0/components/material.html) components. The `primitive` property of the `geometry` component is automatically set to `cylinder`. Then, `height`, `radius`, and `color` are attributes mapped to `geometry.height`, `geometry.radius`, and `material.color` respectively, while `position` and `rotation` are components on their own.

The equivalent entity looks like:

```html
<a-entity id="suction-pad"
          position="-0.2 0.025 0.286" rotation="-90 0 0"
          geometry="primitive: cylinder; heigh: 0.1; radius: 0.005"
          material="color: red">
</a-entity>
```

## The A-Frame registry

The [A-Frame registry](/aframe-registry/) is a centralized source of components. There you can find a collection of components, download the libraries and visit their home pages. Look at the [`aframe-auto-detect-controllers`](https://www.npmjs.com/package/aframe-auto-detect-controllers-component) component for instance. Given that we replaced the hand controls with the HTC VIVE controllers, we could have used this component to automatically detect which particular tracked controller to use, e.g. HTC Vive or Oculus Touch.

## Grabbing things

There are several ways of grabbing things. The most important thing after grabbing something is to ensure that the grabbed item follows the position and orientation of the tracked hand. We accomplished this in two different ways:

  1. For tools like the hot-air gun and the suction pad, we didn’t rely on physics; we instead performed a quick proximity test and attached the tools directly to the tracked controls.

  2. For other entities such as the circuit board or the chip (once it is separated), We relied on the physics system, setting a [_constraint_](https://github.com/donmccurdy/aframe-physics-system#components--constraint) between the circuit board and the tracked controller. A constraint is a way of binding the physical behavior of a body to a different one.

### The operator system

According to ECS theory, coordinating interactions between the operator’s hands and other elements in the scene sounds like something a system should be in charge of. This system will be the `operator` system.

In A-Frame, a system can be configured using the component notation. For instance, when setting `physics` component to `debug: true`, I’m telling the physics system to render a representation of the collision bounding boxes. The following code will configure the `operator` system that I’ll implement in a while:

```html
<a-scene physics="debug: true"
         operator="hands: [vive-controls]; items: #circuit-board,[tool]">
</a-scene>
```

The `operator` system is parametrized by two properties: `hands` and `items`. The `items` property represents all the items that the hands can grab. I’ve used CSS attribute selectors and they will be translated into lists of elements as if they were looked up using the [`querySelectorAll()`](https://developer.mozilla.org/en-US/docs/Web/API/Document/querySelectorAll) method. The barebones of the `operator` system looks like this:

```js
AFRAME.registerSystem('operator', {
  schema: {
    hands: { type: 'selectorAll' },
    items: { type: 'selectorAll' }
  },
  _grabbedItems: {},
  init() {
    const hands = this.data.hands;
    hands.forEach(hand => {
        hand.addEventListener('gripdown', () => console.log('Grab!'));
        hand.addEventListener('gripup', () => console.log('Drop!'));
    });
  }
});
```

The A-Frame documentation has a chapter explaining in detail — see [how to create and register new systems](/docs/0.6.0/core/systems.html).

The [`gripdown` and `gripup` events](https://aframe.io/docs/0.5.0/components/vive-controls.html#events) are emitted by the entities with the `vive-controls` component when buttons on the grip are pressed or released respectively.

### Grabbing physical bodies

Grabbing a physical body will establish a constraint between the item being grabbed and the hand. The code looks like:

```js
_grab(hand, item) {
  if (item) {
    const isGrabbed = item.hasAttribute('constraint');
    if (!isGrabbed) {
      item.setAttribute('constraint', `target: #${hand.id}`);
      this._grabbedItems[hand.id] = item;
    }
  }
}
```

We keep track of which item is grabbed by each hand to simplify dropping:

```js
_drop(hand) {
  const grabbedItem = this._grabbedItems[hand.id];
  if (grabbedItem) {
    grabbedItem.removeAttribute('constraint');
    this._grabbedItems[hand.id] = null;
  }
}
```

With those in place, the final `init` method looks like this:

```js
init() {
  const hands = this.data.hands;
  const items = this.data.items;
  hands.forEach(hand => {
    hand.addEventListener('gripdown', () => {
      const item = this._findNearby(hand, items);
      this._grab(hand, item);
    });
    hand.addEventListener('gripup', () => {
      this._dropBody(hand);
    });
  });
}
```

The proximity test used to find which of the items is reachable by the operator’s hand can be implemented in several ways. For simplicity, We decided to compute the distance between the controller origin and item origins. If this distance is less than _20cm_, then it is reachable:

```js
_findNearby(hand, items) {
  for (let i = 0, l = items.length; i < l; i++) {
    const item = items[i];
    if (distance(item) < 0.2) {
      return item;
    }
  }
  return null;

  function distance(item) {
    const handPosition = hand.getObject3D('mesh').getWorldPosition();
    const itemPosition = item.getObject3D('mesh').getWorldPosition();
    return handPosition.distanceTo(itemPosition);
  }
}
```

With A-Frame you can use well known HTML APIs to create VR interactions. The only thing new for developers to learn is the API for computing distances that we will cover in this tutorial.

[![Grabbing physical objects](/images/blog/hands-on/step3-grabbing-physical-bodies-thumb.png)](/images/blog/hands-on/step3-grabbing-physical-bodies.gif)

### Grabbing tools

Grabbing tools is a different thing. It usually involves complex handling not only to hold the tool but to hold it properly. For this simulation, We chose to automatically correct the position and rotation of the tool in the hand once the operator grabbed it.

When dropping, instead of letting the tool fall from its current position, We opted for returning it to its original position. To store the proper position and rotation of the tool while being held or resting, We created the `tool` component:

```js
AFRAME.registerComponent('tool', {
  dependencies: [‘tool-original-location’],
  schema: {
    position: { type: 'vec3' },
    rotation: { type: 'vec3' }
  },
  grab() {
    this._rememberLocation();
    this.setAttribute('position', this.data.position);
    this.setAttribute('rotation', this.data.rotation);
    this.isGrabbed = true;
  },
  drop() {
    this._restoreLocation();
    this.isGrabbed = false;
  },
  _rememberLocation() {
    this._originalParent = this.el.parentNode;
    this._originalPosition = this.el.getAttribute('position');
    this._originalRotation = this.el.getAttribute('rotation');
  },
  _restoreLocation() {
    this.el._originalParent.appendChild(this.el);
    this.el.setAttribute('position', this._originalPosition);
    this.el.setAttribute('rotation', this._originalRotation);
  }
});
```

The [`schema`](/docs/0.6.0/core/component.html#schema) key allows us to define multiple properties describing the `component`. A [complete list of all property types](/docs/0.6.0/core/component.html#property-types) can be found in the A-Frame documentation, which also includes a chapter explaining [how to create and register new components](/docs/0.6.0/core/component.html) with additional detail.

We applied the component `tool` to the tools in the scene with the correct values for position and rotation while being held:

```html
<a-cylinder id="suction-pad"
            tool="position: 0 -0.04 -0.035; rotation: -90 0 0"
            position="-0.2 0.025 0.286" rotation="-90 0 0"
            height="0.1" radius="0.005" color="red">
</a-cylinder>
<a-cylinder id="hot-air-gun"
            tool="position: 0 0 0.1; rotation: -90 0 0"
            position="0.3 0.03 0.286" rotation="-90 0 0"
            height="0.15" radius="0.01" color="blue">
</a-cylinder>
```

The following functions implement the code for grabbing and dropping tools. The return value indicates the result of grabbing or dropping and it is used by the callee of the functions to mark the items as grabbed or released:

```js
function _grabTool(element) {
  const tool = element.components.tool;
  if (!tool.isGrabbed) {
    grabbedItem = element;
    tool.grab();
    hand.appendChild(element);
    return true; // signal the success
  }
  return false;
}

function _dropTool() {
  const tool = grabbedItem.components.tool;
  tool.drop();
  return true;
}
```

Before continuing you should be aware that the above code does not function correctly. There is an aspect in which **A-Frame and normal HTML differ**, and it is subtle and counterintuitive.

As you can see, grabbing a tool implies making it a direct child of the operator’s hand. This way the tool will respond to the hand’s movements in a realistic way. But detaching and re-attaching the tool from the DOM **causes the components to be reset**. More precisely: when re-attaching, the components are completely new. By the time `_dropTool()` is called, the component has been detached from the workspace node and re-attached as a child of the hand so it won’t preserve the values in `_originalParent`,  `_originalPosition` and `_originalRotation` set during the `tool.grab()` call.

A solution would have been to declare those values as part of the schema but it does not work out of the box. It requires a call to the entity’s [`flushToDOM()`](https://aframe.io/docs/0.5.0/core/entity.html#flushtodom-recursive) method because A-Frame avoids serializing data for performance reasons. Furthermore, it is currently impossible to properly serialize the parent node if that node doesn’t have an `id`.

One way of preserving this data, without changing the way the element is used, is to store it inside the HTML node. The important piece is in the `init` method where the component creates or reuses a storage object to hold the state.

```js
AFRAME.registerComponent('tool', {
  schema: {
    position: { type: 'vec3' },
    rotation: { type: 'vec3' }
  },
  init() {
    this.el._tool = this.el._tool || {};
  },
  grab() {
    this._rememberLocation();
    this.el.setAttribute('position', this.data.position);
    this.el.setAttribute('rotation', this.data.rotation);
    this.el._tool._isGrabbed = true;
  },
  drop() {
    this._restoreLocation();
    this.el._tool._isGrabbed = false;
  },
  _rememberLocation() {
    this.el._tool._originalParent = this.el.parentNode;
    this.el._tool._originalPosition = this.el.getAttribute('position');
    this.el._tool._originalRotation = this.el.getAttribute('rotation');
  },
  _restoreLocation() {
    this.el._tool._originalParent.appendChild(this.el);
    this.el.setAttribute('position', this.el._tool._originalPosition);
    this.el.setAttribute('rotation', this.el._tool._originalRotation);
  },
  isGrabbed() {
    return this.el._tool._isGrabbed;
  }
});
```

Now we can grab every element in the scene:

[![Grabbing tools](/images/blog/hands-on/step3-grabbing-tools-thumb.png)](/images/blog/hands-on/step3-grabbing-tools.gif)

### Fixing the collision bounding shapes

Before, we mentioned that the physic engine was miscalculating the collision bounding box of the VIVE controllers. Making the bounding box to match the controller model is especially important when trying to grab physical objects, after disabling the debug mode of physics. If the bounding box, and the model position and dimensions don’t match, the user real interactions won't match what the user sees in VR resulting in a frustrating experience.

The problem lies in the communication between A-Frame and the pyhsics component: on one side, the physics component is not considering the current rotation of the controller properly so, unless the controller is resting in its default position, the bounding box dimensions would be miscalculated. On the A-Frame side, the `vive-controls` component is [applying a correction in the position of the model after loading the model](https://github.com/aframevr/aframe/blob/master/src/components/vive-controls.js#L202). This correction is not advertised outside the component so the physics component can not realize when the model is fully placed. Since the bounding box calculation is done before applying the correction, the bounding box will be misplaced.

To fix this problem, I’ve created a `delayed-static-body` component which solves both problems. To deal with the positioning problem, We delayed setting the real `static-component` until the next frame, once we know the browser has recovered the control of the execution and so, the correction must to be applied:

```js
import * as AFRAME from 'aframe';
import * as physics from 'aframe-physics-system';

const Component = AFRAME.registerComponent('delayed-static-body', {
  schema: AFRAME.components['static-body'].schema,
  init() {
    this.el.addEventListener('model-loaded', () => {
      window.requestAnimationFrame(this._resetBoundingBox.bind(this));
    });
  },
  _resetBoundingBox() { /* ... */ }
});
```

Notice how we borrow the schema from the `static-body` component to be the same.

To fix the miscalculation of dimensions, we store the current rotation of the entity, reset the rotation to 0, set the `static-body` component to the value of this component and restore the original rotation so the bounding box calculations happen when the entity lies in its default orientation.

```js
_resetBoundingBox() {
  var currentRotation = this.el.getAttribute('rotation');
  this.el.setAttribute('rotation', { x: 0, y: 0, z: 0 });
  this.el.removeAttribute('static-body');
  this.el.setAttribute('static-body', this.data);
  this.el.setAttribute('rotation', currentRotation);
}
```

The complete component is in the [`js/step3/components/delayed-static-body.js`]() file. To use it, we replaced the name part of the `static-body` component of the hands with `delayed-static-body` leaving the value parts untouched.

```html
<a-entity id="left-hand"
          delayed-static-body="shape: box"
          vive-controls="hand: left">
</a-entity>
<a-entity id="right-hand"
          delayed-static-body="shape: box"
          vive-controls="hand: right">
</a-entity>
```

## A-Frame and Three.js

In the same way the `aframe-physics-system` is a wrapper around the Cannon.js library, which exposes part of the API in the form of convenient `components` for the A-Frame nodes, A-Frame itself is a wrapper for the [Three.js 3D library](https://threejs.org/).

Three.js is a powerful and popular 3D library built on top of WebGL, which exposes a high-level API focused on manipulating a scene graph instead of working directly with 3D primitives.

One final function that we needed for the demo was to be able to calculate distances between objects. To do this we used some of the native Three.js functionality in A-Frame with the help of the [`getObject3d`](/docs/0.6.0/core/entity.html#getobject3d-type) entities’ method. We used this API to access the Three.js [Object3D API](https://threejs.org/docs/index.html) and compute the [distance](https://threejs.org/docs/index.html#api/math/Vector3.distanceTo) between two Three.js [Vector3](https://threejs.org/docs/index.html#api/math/Vector3) objects according to [their world coordinates](https://threejs.org/docs/index.html#api/core/Object3D.getWorldPosition) in the `distance` auxiliary function:

```js
function distance(item) {
  const handPosition = hand.getObject3D('mesh').getWorldPosition();
  const itemPosition = item.getObject3D('mesh').getWorldPosition();
  return handPosition.distanceTo(itemPosition);
}
```

## Conclusion

In this tutorial you have learned how to prototype a complex and semantic VR scene, enable physics, workaround some limitations, and add basic interactions. You’ve discovered the [A-Frame registry](https://aframe.io/aframe-registry/) and now you know how to make the most of the [ECS pattern](/docs/0.6.0/introduction/entity-component-system.html) and create your own [customised components](/docs/0.6.0/core/component.html) and [systems](/docs/0.6.0/core/systems.html).

Stay tuned for new articles of the Hands-on! series and you will learn how to enrich the experience by adding textures, effects, 3D models, sound and UI. If you liked these articles, [mention us on Twitter](https://twitter.com/aframevr) and don't hesitate asking your questions in the [A-Frame slack channel](https://aframevr.slack.com/).
