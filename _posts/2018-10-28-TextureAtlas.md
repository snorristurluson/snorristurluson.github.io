---
title: Texture Atlas
tags: vulkan c++
---
My [Vulkan sprite renderer](../BackToBasics) uses a texture atlas for more efficient
rendering of sprite.

A texture atlas is simply a large texture composed of many smaller textures. This
applies particularly well to sprite rendering, where the textures are generally small
and there is a lot of them.

Rendering a sprite boils down to setting the texture for the sprite, then issuing
the draw command - in Vulkan terms:
```cpp
vkCmdBindDescriptorSets(...)
vkCmdDrawIndexed(...)
```
The vertices for a sprite are the four corners of a rectangle:

![DrawSprite](/images/DrawSprite.svg)

If two consecutive sprites share the same texture then we can issue two draw
commands without setting the texture again, or better yet, combine the index and
vertex buffers for those two sprites. A sprite-based game will use a lot of textures
so the chances of consecutive sprites sharing the same texture are pretty low, but
if we use a texture atlas, all the sprites can share the same texture, simplifying
the command buffer greatly.

Using an atlas texture, or a texture within a texture atlas implies we're only
using a portion of the larger texture, so we simply adjust the texture coordinates
appropriately.

Here's an example of a texture atlas that has 4 individual atlas textures
loaded:

![TextureAtlas](/images/TextureAtlas.png)

Let's say we want to draw the purple pig:

![TextureAtlas](/images/TextureAtlasWithCoords.png)

The texture coordinates now go from (0.25, 0.25) to (0.5, 0.5) instead of
(0, 0) to (1, 1):

![DrawSprite](/images/DrawSpritePurplePig.svg)


## Texture interface
My engine has a *Texture* class that represents a stand-alone texture, and an
*AtlasTexture* class that represents a texture within the *TextureAtlas*.

These classes implement an *ITexture* interface:
```cpp
struct TextureWindow {
    float x0;
    float y0;
    float x1;
    float y1;
};

struct ITexture {
    virtual VkDescriptorSet GetDescriptorSet() = 0;
    virtual TextureWindow GetTextureWindow() = 0;
    virtual int GetWidth() = 0;
    virtual int GetHeight() = 0;
};
```
Each *Texture* has its own descriptor set, whereas the *AtlasTexture* defers
that to the *TextureAtlas*, so all atlas textures coming from the same texture
atlas have the same descriptor set.

A *Texture* has a texture window that represents the full texture - from (0,0) to
(1,1). An *AtlasTexture* has a texture window that represents its area in the texture
atlas - for example, the purple pig has a window from (0.25, 0.25) to (0.50, 0.50).

The *Renderer* has a *SetTexture* method, that sets the texture for subsequent
*DrawSprite* calls. Internally, that stores the current descriptor set and the
current texture window. *DrawSprite* accumulates the vertices that share the
same descriptor set, and uses the texture window to set the texture coordinates
on each vertex of the sprite.

This means that calling *SetTexture* with an *AtlasTexture* will not break up
the draw command while the atlas textures come from the same texture atlas.

## Populating the atlas
The atlas is populated dynamically when the *Add* method is called on the
*TextureAtlas*. There are two overloads for the *Add* method, one that takes
in a path to an image on disk, and one takes a buffer in memory with the pixel
data.

The allocations in the texture atlas are tracked with an *AreaAllocator* - I
will discuss that in a future blog post.

## Updating the atlas on the device
The *Initialize* method in the texture atlas creates an RGBA texture of the
requested size. The image layout is immediately transitioned to
*VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL* without filling the image with any
data.

Each *Add* call transitions the image back to *VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL*,
transfers the pixels via a staging buffer to the image, then transitions the image
back to *VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL*.

This basically ensures that the texture atlas is image is always ready to be used
for rendering. I'm sure I will need to revisit this at some point to make this more
efficient, to reduce unnecessary image layout transitions.

The staging buffer is created in the *Add* call and filled with the pixel data,
that may have come from reading an image from disk, or a buffer in memory. The
latter case is used, for example, in text rendering, by having FreeType generate
the bitmap for a given character (see my previous blog post on
[text rendering](../TextRenderingWithFreetype)).

The staging buffer is copied to the texture atlas with a call to
*vkCmdCopyBufferToImage*. The staging buffer can't be destroyed until we're sure
the copy has actually taken place, so it needs to stay around until the current
command buffer has been processed. I do this with *DestroyBufferLater* method,
that keeps track of buffers that need destroying.

As always, the details are in the 
[source code on GitHub](https://github.com/snorristurluson/vulkan-sprites).