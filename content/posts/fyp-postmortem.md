---
title: "FYP Postmortem"
date: 2020-06-01T12:18:55+01:00
draft: true
menu: "main"
summary: "A postmortem of my University Final Year Project"
---

For my University Final Year Project (FYP), I wrote a sparse voxel octree renderer using
OpenGL and C++. Now that the project is complete, I would like to analyse what
went right, what went wrong and how things could be improved if I were to re-start the
project from scratch.

![Stanford Bunny Voxel](/bunny.png)

## The Good
In general, I was pretty happy with the end result. The software could render and edit
very detailed voxel models in real time, and the code was reasonably well structured. 
As with any software project, there's a million and one things I'd like to improve upon,
but I think I delivered a pretty cool application.

### Performance
I spent a lot of time tuning the performance of this project, with the GLSL render kernel 
probably receiving the heaviest optimisation. At first, I tried using the GPU profiler
from NSight Graphics, but since the frame profiler doesn't support compute shaders
I had to draw data directly from the raw performance counters, which I found very 
difficult to read (the documentation is also pretty sparse w.r.t these as well). In the
end, I took a different approach. Using the `glGetProgramBinary` API, I was able to 
look at the NVidia shader assembly and figure out slow spots from there.

There's a problem with this technique though: the "instructions" you see in the program
binary listing may or may not have any relation to actual hardware operations. Still, I
was able to determine some nice optimisations by looking through the generated assembly
code, mostly stuff to do with branch elimination. One that surprised me a bit was that
GLSL functions will silently convert parameter types from ints to floats, which can 
introduce the overhead of a conversion operation.


### Modularity
I worked hard to keep the code simple and modular. The core program consists of three
modules: the SVO manager, the renderer and the mesh importer. These modules interact
only minimally, I was able to get them pretty well de-coupled.

The project also relies on very few external libraries (basically GLFW/GLAD for windowing, Dear ImGUI for debug output and CGLTF for mesh loading). All of these dependencies are 
compiled into the distribution binary as static libraries, so this meant that I didn't 
have to go through DLL-hell when preparing for distribution. There's a lot to be said 
for a single static binary, and indeed it seems newer languages like Rust and Go are 
converging towards this strategy.

## The Bad

### Nuclear Mode
There is a really nasty bug somewhere deep in the render init code which causes memory
usage to completely spiral out of control. This bug blew up my graphics driver many
times and even crashed the kernel once. The only way to stop this from happening was to
kill the program as soon as you noticed the pathological memory behaviour. Even if you
killed it in time, the disk would go crazy for five to ten minutes (I suspect cleaning
up the page file). I don't know for certain, but I *think* the bug has to do with the
graphics driver allocating space for the octree leaf data sparse texture storage.

This bug only manifests at SVO depth values above 10, so I simply capped the depth of
the scene SVO at 10 before submission. Still, this bug limited the amount of detail with
which I could showcase the project, which was a shame. Looking back, I really should've
just designed the render subsystem to run within a fixed memory arena, and paged SVO
sub-trees in and out of this arena depending on where the camera travels within the
world.


### (Too) Sparse Textures
While the SVO occupancy data is tightly packed within SSBOs, I still needed a way to
store the voxel leaf data (colours, normals, etc.). Since I wanted to do the minimum
work possible per-voxel, I tried to pack the leaf data into a giant sparse texture, with
one texel per highest-resolution voxel. The theory was that this would function as a 
kind of GPU-capable hashmap, giving me constant voxel access times without having to
store huge amounts of useless "air" voxels. This *almost* worked --- my voxel access
times were indeed very fast, but the data was just too sparse. Most of my sparse texture
pages sat at under 10% occupancy.

I'd have loved to do something like [this](https://www.researchgate.net/publication/318780127_Real-Time_Parallel_Hashing_on_the_GPU). A "real" GPU hashmap would allow me to
combine the fast raycasting of SVO data with the fast access times of the hashmap, 
without the need to store useless data.

## Wishlist
If I could go back in time seven months and start re-writing the project from scratch, 
the first thing I'd do is commit to pure C development. Initially, I had written the
project in what is sometimes called "orthodox" C++; a style of C++ with no classes and
minimal STL use. I found, though, that the C++ features I did use really only got in the
way (automatic conversion through constructors when I *did* use the STL, it was to crutch bad
data management practices (for example, storing all the voxel leaf data in a giant 
vector). If I'd written the project in C from the get-go, I would have had much more
motivation to sit down and really think through the program's data flow, and implement
an appropriate data structure.

Speaking of data flow, I would like to investigate how a fixed, arena-style memory model
would stack up to the dynamic heap-based allocation model I originally chose. To support
infinite SVO scenes, I could allocate a large buffer out of system memory and stream
sub-octrees into that depending on where the camera moves in the world. This would also
eliminate the memory nuke bug described above.

Finally, I'd like to implement some kind of "garbage-collector" for modified SVO blocks.
The original SVO paper authors, Laine and Karras, advised readers that SVOs weren't
really intended to be used for dynamic scenes. Editing an SVO requires some way of
efficiently inserting new octree branches into the existing scene data structure. In my
implementation, I solved this by simply creating a copy of the highest ancestor of the 
new branch that already exists in the tree, and then linking this back to the tree. This
method allows for extremely fast insertions, but it does mean that you get "holes" in the
SVO where the old nodes were. Some way of compacting the SVO blocks would be nice to 
avoid these holes building up over time.
