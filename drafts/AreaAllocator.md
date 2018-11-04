The texture atlas relies on an *AreaAllocator* class to keep track of areas
allocated to the individual atlas textures.

The *AreaAllocator* keeps a list of free areas that it pulls from when
*Allocate* is called. The area is not likely to fit the allocation
perfectly so the allocator looks for an area that is equal to or larger
than the requested area. If an area is found, it is removed from the free
list, the requested area cut from it and the remaining pieces added back
to the free list.

![Simple allocation](/images/AreaAllocatorSimpleAllocation.svg)

When an area is freed it is simply added to the list.

What happens if an area isn't found? We may have plenty of space
available even though no single are is large enough to accommodate the
request.