---
title: "GPU Hash Tables For Fast Voxel Retrieval"
date: 2020-07-24T08:49:14+01:00
draft: false
summary: Augmenting Sparse Voxel Octrees with GPU-friendly hashtables to give constant lookup times for voxel attributes.
---


<figure>
  <img src="/gallery_scene.jpg" alt="Image of voxelised art gallery">
  <figcaption>
    The Hallwyl Museum in Stockholm, Sweden. Voxelised to a resolution of 2<sup>-13</sup>
  </figcaption>
</figure>

## The Problem
A Sparse Voxel Octree (abbreviated SVO) is a neat data structure that allows you to
store massive voxel worlds efficiently by compressing away large regions of empty
space. Since GPUs don't have hardware support for SVO encoded scenes, rendering SVOs
takes a little bit more work than regular polygon-based geometry. You can render SVOs by
shooting a ray through the octree for each pixel in the viewport and colouring in any pixel whose ray intersects a voxel.

<figure>
  <img src="/sibenik_scene.jpg" alt="Image of voxelised Sibenik cathedral">
  <figcaption>
    Sibenik Cathedral. Voxelised to a resolution of 2<sup>-13</sup>. This scene contains several
    million leaf voxels that all need to have their colours and normals retrieved each frame in
    real time.
  </figcaption>
</figure>

There is a problem with this approach though: raycasting, by itself, can only tell you
whether or not a particular point in space contains a voxel, you know nothing about what
kind of data is inside that voxel. This makes it impossible to have voxels with varying
*attribute data* --- things like colours and normals. To solve this problem, we need a
way to compactly store voxel attribute data so that we can quickly look up the colour/
normal/etc. for a given voxel in the scene. Any solution needs to be fast, since these
lookups are taking place in the middle of a very tight render loop.

## Hash Tables
A hash table (also called a dictionary or a map) is a kind of data structure 
that allows certain *keys* to be associated with certain *values*. Given a key, you can
then quickly find its corresponding value in the table. Hash tables work by using a *hash
function* to transform a key into a slot in some backing store (e.g. an array), so 
superficially hash table lookups have *O(1)* time complexity. So, hash tables sound like
a good solution to our problem: we can use a voxel's position as a key, and store the 
colour/normal/etc. data as the value associated with that key. Then, when we need that 
data to colour a pixel, we can just look up the attribute data for the pixel's world-space
position.

In practice, things are not quite this simple. If your hash table's backing store only has
100 slots, and you've got 1000 keys, then some keys are necessarily going to get assigned
the same slot. When multiple keys get assigned to the same table slot, this is called a
*collision*. Most of the hard research that goes into hash tables revolves around finding ways
to efficiently and correctly handle collisions.

A simple way of handling collisions is to use an array of linked lists for the backing
store instead of just a simple array. Then, when you get a collision, just append a new
item to the linked list associated with the contested slot. This is called *chaining*.
Unfortunately, chaining is not suitable for our use case; for starters, you can never be sure
how many entries might go into a particular slot, so you need dynamic memory allocations.
Traversing a linked list for every voxel lookup could also slow down our renderer.

## Cuckoo Hashing
Cuckoo hashing is a technique which attempts to resolve collisions by
computing multiple hashes for each key using <em>n</em> different
hash functions (denoted <em>H<sub>0</sub></em> to <em>H<sub>n-1</sub></em>). Keys are first hashed
into their <em>H<sub>0</sub></em> bucket. If  this 
bucket happens to be already occupied, we write the current key in anyway, and
kick the previously resident key out, sending it to its <em>H<sub>1</sub></em>
bucket, and so on. This is where the name "cuckoo" hashing comes from.

The cuckoo hash tables used in this project are based on Dan Alcantara et. al's
technique in *[Building an Efficient Hash Table on the GPU (2011)](https://www.researchgate.net/publication/211178395_Building_an_Efficient_Hash_Table_on_the_GPU)*

Cuckoo hashing is well-suited to GPU programming for a couple of reasons.
First, the table can be constructed using only `atomicExchange` functions,
meaning that no complex synchronisation logic needs to be layered on top
of our hashtable. Second, cuckoo hashing guarantees that a key can be found
in at most <em>n</em> lookups. This is ideal for our use case, where many 
thousands of hash lookups take place each frame.

## The Hash Function
A *hash function* is a mapping from keys to slots in your backing store. The quality of a
hash function can have a very big impact on a hash table implementation's performance: if your
hash function doesn't do a good enough job of generating very dissimilar slots for different
keys, you will end up with a lot of collisions.

It has been said that nearly every single software project contains a TODO
item to find a better hash function<sup>[[1]](#fn-0)</sup>. The one I picked is almost certainly
not optimal, but it does get the job done, and it's fast. Our core problem
is hashing some 3D float vector <em>V</em> into a single 32-bit unsigned
integer. I used a two-step process for my hash function --- first, the
3D vectors are converted into integers with the following formula:

```glsl
uint HashVec3(vec3 V)
{
    const uvec3 C = uvec3(73856093, 19349663,83492791);
    uvec3 H = uvec3(V) * C;
    return H.x ^ H.y ^ H.z;
}
```
(From [this](https://stackoverflow.com/a/5929567) StackOverflow answer)

Next, we must compute the actual hash table keys from these integers. The
[hash prospector](https://github.com/skeeto/hash-prospector) project is a good candidate for our
use case, since it provides a list of constants that can all be used with the
same basic formula to produce distinct hash functions. Since we need multiple
hash functions to do cuckoo hashing, we can simply write the formula once and
use GPU vector programming to compute all of the hashes in parallel, like so:

```glsl
uvec4 ComputeHashes(uint Key)
{
    const uvec4 S0 = uvec4(16, 18, 17, 16);
    const uvec4 S1 = uvec4(15, 16, 15, 16);
    const uvec4 S2 = uvec4(16, 17, 16, 15);

    const uvec4 C0 = uvec4(0x7feb352d, 0xa136aaad, 0x24f4d2cd, 0xe2d0d4cb);
    const uvec4 C1 = uvec4(0x845ca68b, 0x9f6d62d7, 0x1ba3b969, 0x3c6ad939);

    uvec4 K = uvec4(Key);

    K ^= K >> S0; // Perform all 4 shifts in parallel
    K *= C0;      // Same thing for the multiplies
    K ^= K >> S1;
    K *= C1;
    K ^= K >> S2;

    return K % (TableSize - 1);
}
```

## Hash Table Data Format
Our hash table will be stored in a SSBO (shader storage buffer object) and is simply a very large 
array of the following structs:

```glsl
struct htable_entry
{
    uint Key; // Voxel position encoded by HashVec3 above
    uint PackedNormal; // Snorm packed normal (See "normalised integers" below)
    uint PackedColour; // Unorm packed colour
};
```

### Normalised Integers
In our `htable_entry` struct above, the colour and normal data are stored as single
32-bit unsigned integers, even though the full resolution data are actually floating
point `vec3`s. They are packed this way for two reasons: first, it saves space in the
hash table storage; second, it allows the table construction kernel to use only one
`atomicExchange` instruction per attribute, since the GLSL atomic exchange function 
can only work on one 32-bit integer at a time.

This packing is accomplished by first clamping each component of the input vector
into the range `[-1.0, 1.0]` and then multiplying by the value `127.0`. We can then
round to produce a signed 8-bit integer in the range `[-127, 127]`. Finally, each of
these encoded components are combined into a single 32-bit unsigned integer. This
type of packing is known as *snorm* packing (*unorm* packing is the same thing but
does not use negative values).

GLSL has native support for vectors packed into this format via the `unpack4x8{S,U}norm`
functions, so it should be quite fast to decode the voxel attributes in our render loop.

## Constructing The Table
To begin constructing the hash table, we reserve one key value to indicate that a 
table slot is free, such as `0xFFFFFFFF` --- this is the "null" key. Next, we 
allocate a large, fixed-size SSBO to serve as our hash table backing store and
memset that to the null key. Since cuckoo hash tables can be constructed using
only `atomicExchange` memory instructions, we can use a compute shader to
perform the heavy lifting involved in building the hash table, allowing us to compute
many hashes in parallel.

```glsl
void main()
{
    // We only have 1-dimensional data
    uint ThreadID = gl_GlobalInvocationID.x;
    htable_entry InputEntry = InputBuffer[ThreadID];

    // Start by computing the first hash for this key
    uint Slot = ComputeHashes(InputEntry.Key).x;

    // MAX_STEPS is the number of times we try to relocate an evicted key
    // before giving up. Make sure to keep reasonably small
    for (int Try = 0; Try < MAX_STEPS; ++Try)
    {
        // Swap InputEntry.Key with the data already in OutputBuffer[Slot]
        InputEntry.Key = atomicExchange(OutputBuffer[Slot].Key, InputEntry.Key);
        // (same thing for the colours and normals)

        // If the slot we wrote into was unoccupied, we're done
        if (NULL_KEY == InputEntry.Key) return;

        // Otherwise, we need to find a home for the evicted entry
        uvec4 H = ComputeHashes(InputEntry.Key);

        // if H.x == Slot then Slot = H.y
        // if H.y == Slot then Slot = H.z
        // if H.z == Slot then Slot = H.w
        // if H.w == Slot then Slot = H.x
        bvec4 Msk = equal(H, Slot);
        Slot = dot(mix(vec4(0), H.yzwx, Msk), vec4(1));
    }

}
```

Our compute shader kernel is responsible for computing the hashes for each input entry and
writing that entry into its corresponding table slot (in this code, `OutputBuffer` is the output
hash table SSBO). A new home is selected for evicted keys through a simple round-robin approach
(this is the technique used in the Alcantara paper). A select/dot product trick is
used here to avoid branches, though this table builder kernel isn't as performance critical as
the renderer kernel.

Note that we could possibly lose data here: in the unlikely event that a new slot for an 
evicted entry cannot ever be found (perhaps with an unlucky choice of hash functions or too
small of a table size) then we lose that entry. This can in theory be avoided through re-starting
the construction process with a new set of hash functions.


## Voxel Lookups
So, now we have our GPU hash table in an SSBO that maps 3D positions to voxel data attributes.
The last thing left to do is implement voxel atrribute lookups in the render shader. The SVO
raycaster kernel will always know a voxel's 3D coordinates, so when this kernel needs a voxel
data attribute, it can easily determine the key by first encoding the 3D float vector into an
integer using `HashVec3`, then determining four possible locations for this attribute by using
`ComputeHashes`. Finding the correct entry, then, is simply a matter of checking which slot in
the hash table holds the required key.

```glsl
// Sadly, can't have multi-return in GLSL, so we need to use out params
// P is the voxel position ran through HashVec3
void LookupVoxelData(uint P, out vec3 Normal, out vec3 Colour)
{
    uvec4 H = ComputeHashes(P);

    if (HashTable[H.x].Key == P)
    {
        Normal = unpackSnorm4x8(HashTable[H.x].PackedNormal).xyz;
        Colour = unpackSnorm4x8(HashTable[H.x].PackedColour).rgb;
    }
    else if (HashTable[H.y].Key == P)
    {
        Normal = unpackSnorm4x8(HashTable[H.y].PackedNormal).xyz;
        Colour = unpackSnorm4x8(HashTable[H.y].PackedColour).rgb;
    }
    else if (HashTable[H.z].Key == P)
    {
        Normal = unpackSnorm4x8(HashTable[H.z].PackedNormal).xyz;
        Colour = unpackSnorm4x8(HashTable[H.z].PackedColour).rgb;
    }
    else if (HashTable[H.w].Key == P)
    {
        Normal = unpackSnorm4x8(HashTable[H.w].PackedNormal).xyz;
        Colour = unpackSnorm4x8(HashTable[H.w].PackedColour).rgb;
    }
}
```

## Conclusion
This was pretty fun to implement! Replacing my original sparse texture voxel lookup with a more
robust solution let me bump up the voxel resolution by a huge amount, meaning the renderer could
handle bigger and more detailed scenes. This approach is, of course, not perfect. Collisions, 
though comparatively rare, do still occur with this approach (they manifest as voxels with 
obviously "wrong", or discontinuous, colours). In essence, this design is similar to so-called
Monte Carlo algorithms: guaranteed to be fast, probably accurate.

Designs that require guaranteed voxel accuracy can still leverage cuckoo hashing, though the
table will need to be re-constructed with new hash functions if you can't find a slot for an 
evicted entry.


<small id="fn-0">[1] https://youtu.be/WyXBawK1jpE?t=2369</small>




