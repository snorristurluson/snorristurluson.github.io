---
title: Area Allocator
tags: c++ tdd
---
The [texture atlas](/TextureAtlas) in my
[Vulkan sprite renderer](https://github.com/snorristurluson/vulkan-sprites)
relies on an *AreaAllocator* class to keep track of areas
allocated to the individual atlas textures. I have implemented such a class
before, when I was working on EVE Online. This time around I'm building it from
the ground up with unit tests, trying to do it in a proper test driven
development style.

## Allocating
The *AreaAllocator* keeps a list of free areas that it pulls from when
*Allocate* is called. The areas are rectangular, and the allocator starts
out with one area, representing the whole texture atlas.

When allocating, the requested area is not likely to fit the allocation
perfectly so the allocator looks for an area that is equal to or larger
than the requested area. If an area is found, it is removed from the free
list, the requested area cut from it and the remaining pieces added back
to the free list.

![Simple allocation](/images/AreaAllocatorSimpleAllocation.svg)

Here's the source for the *Allocate* method:
```cpp
Area* AreaAllocator::Allocate(int width, int height)
{
    Area *oldArea = getFreeArea(width, height);

    if(!oldArea) {
        collapseFreeAreas();
        oldArea = getFreeArea(width, height);
    }

    if(!oldArea) {
        return nullptr;
    }

    auto newArea = new Area {oldArea->x, oldArea->y, width, height};
    m_allocatedAreas.emplace_front(newArea);

    if(oldArea->width > width) {
        // Add an area to the right of newly allocated area
        m_freeAreas.emplace_back(new Area {oldArea->x + width, oldArea->y, oldArea->width - width, height});
    }
    if(oldArea->height > height) {
        // Add an area below the newly allocated area
        m_freeAreas.emplace_back(new Area {oldArea->x, oldArea->y + height, width, oldArea->height - height});
    }
    if(oldArea->width > width && oldArea->height > height) {
        // Add an area diagonally to the right and below the newly allocated area
        m_freeAreas.emplace_back(new Area {oldArea->x + width, oldArea->y + height, oldArea->width - width, oldArea->height - height});
    }
    return newArea;
}
```
When an area is freed it is simply added to the list:
```cpp
void AreaAllocator::Free(Area *area)
{
    m_allocatedAreas.remove(area);
    m_freeAreas.emplace_back(area);
}
```
Note that keeping track of allocated areas isn't strictly necessary, but it
is useful for testing purposes. I could also use that to validate that an
area being freed really was allocated.

## Collapsing free areas
What happens if an area isn't found? We may have plenty of space
available even though no single area is large enough to accommodate the
request. If no free area large enough is found, the allocator tries to
collapse adjacent areas to produce larger areas and tries again.

```cpp
void AreaAllocator::collapseFreeAreas()
{
    if( m_freeAreas.size() < 2 )
    {
        return;
    }

    int collapsed = 0;
    do {
        collapsed = 0;
        AreaList collapsedAreas;
        while(!m_freeAreas.empty()) {
            auto first = m_freeAreas.front();
            m_freeAreas.pop_front();
            while(!m_freeAreas.empty()) {
                auto other = m_freeAreas.FindAdjacent(*first);
                if(other != m_freeAreas.end()) {
                    first->CombineWith(**other);
                    delete *other;
                    m_freeAreas.erase(other);
                    ++collapsed;
                } else {
                    break;
                }
            }
            collapsedAreas.emplace_back(first);
        }
        m_freeAreas = collapsedAreas;
    } while(collapsed > 0);
}

```

The *FindAdjacent* method lives on the *AreaList* class, that extends
a *std::list*. It simply iterates over all the areas, checking to see
if they are adjacent to the given area:
```cpp
AreaList::const_iterator AreaList::FindAdjacent(const Area &area)
{
    for(auto it = cbegin(); it != cend(); ++it) {
        if(area.IsAdjacent(**it)) {
            return it;
        }
    }
    return cend();
}
```
If an adjacent area is found, it's combined with that area and the entry
for that area is removed from the free areas list. This process is repeated
until no adjacent areas are found.

For completeness sake, the source for *IsAdjacent* looks like this:
```cpp
bool Area::IsAdjacent(const Area &other) const
{
    if (x == other.x && width == other.width && y + height == other.y) {
        // Other is immediately below
        return true;
    }

    if (x == other.x && width == other.width && other.y + other.height == y) {
        // Other is immediately above
        return true;
    }

    if(y == other.y && height == other.height && x + width == other.x) {
        // Other is immediately to the right
        return true;
    }

    if(y == other.y && height == other.height && other.x + other.width == x) {
        // Other is immediately to the left
        return true;
    }

    return false;
}
```

## Simple and stupid
When looking for an area to allocate, I simply iterate over the list, looking
for an area that is large enough:
```cpp
AreaList::const_iterator AreaList::FindArea(int width, int height)
{
    auto foundIt = cend();

    for(auto it = cbegin(); it != cend(); ++it) {
        if((*it)->width >= width && (*it)->height >= height) {
            foundIt = it;
            break;
        }
    }

    return foundIt;
}
```
This simple approach works for my purposes for now. My samples so far load their
textures up front, so there isn't any churn on free areas, and I'm not doing
anything on a scale where the texture atlas space is at a premium.

I know from experience, though, that at some point this simple approach won't cut
it. At the very least, we should try to find an exact fit and prefer that over
the current greedy approach of taking the first area that is large enough.

Other considerations include trying to avoid thin (or tall) slivers that are
unlikely to be usable later, and avoiding fragmentation.

On the other hand, I want to avoid premature and speculative optimizations. Also,
with the TDD approach, I found that sticking to the process of writing a test
first, then implementing the minimal code needed to make the test pass resulted
in quite simple and elegant code. Sure, it doesn't necessarily give me the most
efficient solution, but later on, when that matters, I can revisit the area
allocator. I will then add tests that check for those efficiency requirements
before changing the allocation code itself.