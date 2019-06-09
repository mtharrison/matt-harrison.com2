---
title: "Ray Tracing: WebAssembly vs JavaScript"
date: 2018-08-12T00:00:00+01:00
aliases:
    - "/raytracing-webassembly-vs-javascript"
---
I've spent the last couple of weeks learning about the art and science of [Ray Tracing](https://en.wikipedia.org/wiki/Ray_tracing_(graphics)). Ray tracing, for those who aren't familiar is one technique for generating 3d computer graphics. Ray tracing isn't the fastest way to generate 3d images but it's appeal lies in both the realistic effects that can be achieved and in the elegance of simplicity of the technique. This technique is used in movies and for photo-realistic architectural renderings.

<img src="https://mattharrison.s3.amazonaws.com/blog/raytracing.jpg" width=500/>

Without getting too much into the weeds, ray tracing works by going through each pixel of the resulting 2d image and shooting a virtual light ray into the scene and tracing it's path by reflecting it off surfaces to determine which color to set the pixel. It's that simple really. For a simple scene consisting of spheres and planes we can use some basic geometry with vectors to calculate intersection points with objects. To determine shadows, we similarly shoot a ray from an object to our virtual light source and check if there are any intersections in between and therefore the object is in shadow.

Let's take a step back for a moment though and I'll tell you why I've been learning about this. For the past few months I've been toying about with WebAssembly. The examples I've built using WebAssembly were very simple and could easily have been written in JavaScript with perfectly adequate performance. This got me thinking it's about time I make something to really shows where WebAssembly shines. This led me down the path of thinking about very compute-demanding applications. An obvious example is 3d graphics rendering. Even a small scene like the ones I've been creating involve computing millions of vector dot product calculations per second. This kind of CPU-intensive application seemed right up the street of WebAssembly.

I started out with this great example, [Literate Raytracer](https://github.com/tmcw/literate-raytracer) by [Tom MacWright](https://github.com/tmcw). The explanations in the code comments really made sense to me. I extended Tom's code to add some additional features such as support for planes, checkerboard textures and some improved reflections, to get more familiar with the techniques. There's some outstanding content available at [https://www.scratchapixel.com/](https://www.scratchapixel.com/). If you want to learn more about Raytracing or computer graphics in general, it would be a great place to start.

Once I had a JavaScript version of my ray tracer in place with a simple scene, I got to work on creating a WebAssembly version. Rust is my language of choice for writing Wasm modules. I just ported a straight 1:1 copy of my JS ray tracer over to Rust and compiled it using the great tools [wasm-pack](https://github.com/rustwasm/wasm-pack) and [wasm-bindgen](https://github.com/rustwasm/wasm-bindgen).

<iframe style="width:500px; height:450px;border:0;overflow-x:hidden" src="https://mtharrison.github.io/wasm-raytracer/simplified.html"></iframe>

Just to be clear I wasn't out to make the fastest or most fully-featured Ray tracer, for one thing, that's not really my avenue, I'm mainly a web developer. My goal here was to see how much I could demand from my browser and how it would perform compared to JavaScript. There's another more subtle point to make here. I didn't want to spend too much time on optimisation of either the JS or the Wasm version. I understand that's it's often possible to get JS to perform on a par or close to Wasm's execution performance but doing so involves knowing some compiler tricks and optimisation strategies. I'm not that interested in learning those and I don't believe most should have to either. I wanted to compare a naive implementation.

This seems to vary per browser but as a rough, not scientific result, here's what I was getting in Firefox Quantum 61.0.2:

- **WebAssembly: 22.0fps**
- **JavaScript: 2.5fps**

So in conclusion saw around a 9x speedup when I wrote a naive port of a JavaScript application to WebAssembly. I have to say I'm quite impressed with this result.

You can play with my Ray tracer by getting the code on [Github](https://github.com/mtharrison/wasm-raytracer) or there's a [fun interactive example here](https://mtharrison.github.io/wasm-raytracer/) where you can compare the performance on your own machine.

I'm sure I've missed some things out here, so feel free to leave a comment below or ping me on twitter @mt_harrison.
