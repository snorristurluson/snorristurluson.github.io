---
title: Alpha blending using pre-multiplied alpha
tags: vulkan
---
My sprite renderer needs to support alpha blending, and I've opted for a scheme using
pre-multiplied alpha. I'm not going as far as pre-multiplying alpha into the textures,
but rather performing the multiplication in the fragment shader. As far as the blending
stage in the graphics pipeline is concerned, the alpha has been pre-multiplied, however.

Why do it like this? For my purposes, the biggest benefit
is that I can represent 4 different blend modes with one blend setting in the graphics
pipeline. For more detail, see
[Shawn Hargreaves' blog](https://blogs.msdn.microsoft.com/shawnhar/2009/11/06/premultiplied-alpha/)
explaining the benefits of pre-multiplied alpha.

This approach allows me to render the following image with one draw command:

![Blend modes sample](/images/BlendModesSample.png)

Top left is no blending, top right is conventional alpha blending, bottom left is
additive blending and bottom right is 2x additive blending. Each quadrant draws
4 sprites, with a white circle texture colored in 4 different colors.

The source for this is in GitHub: https://github.com/snorristurluson/vulkan-sprites 

This is all rendered with the same pipeline color blend attachment state, set up with
the blend op as *VK_BLEND_OP_ADD*, source color blend factor as *VK_BLEND_FACTOR_ONE* and
destination color blend factor as *VK_BLEND_FACTOR_ONE_MINUS_SRC_ALPHA*.

```cpp
    VkPipelineColorBlendAttachmentState colorBlendAttachment = {};
    colorBlendAttachment.colorWriteMask =
            VK_COLOR_COMPONENT_R_BIT | VK_COLOR_COMPONENT_G_BIT | VK_COLOR_COMPONENT_B_BIT |
            VK_COLOR_COMPONENT_A_BIT;
    colorBlendAttachment.blendEnable = VK_TRUE;
    colorBlendAttachment.colorBlendOp = VK_BLEND_OP_ADD;
    colorBlendAttachment.srcColorBlendFactor = VK_BLEND_FACTOR_ONE;
    colorBlendAttachment.dstColorBlendFactor = VK_BLEND_FACTOR_ONE_MINUS_SRC_ALPHA;
    colorBlendAttachment.alphaBlendOp = VK_BLEND_OP_ADD;
    colorBlendAttachment.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE;
    colorBlendAttachment.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO;
```

The vertex format has a field for storing the blend mode that is filled in by the *DrawSprite*
method of the *Renderer*. The vertex shader uses that to adjust the vertex color before
passing it on to the fragment shader, as well as setting an opacity value.

```glsl
#version 450
#extension GL_ARB_separate_shader_objects : enable

const int BM_NONE = 0;
const int BM_BLEND = 1;
const int BM_ADD = 2;
const int BM_ADDX2 = 3;

layout(binding = 0) uniform UniformBufferObject {
    vec2 extent;
} ubo;

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec4 inColor;
layout(location = 2) in vec2 inTexCoord;
layout(location = 3) in ivec4 inData;

layout(location = 0) out vec4 fragColor;
layout(location = 1) out vec2 fragTexCoord;
layout(location = 2) out float fragOpacity;

out gl_PerVertex {
    vec4 gl_Position;
};

void main() {
    int blendMode = inData[0];
    float opacity = 0.0f;
    vec4 color = inColor;

    if(blendMode == BM_NONE) {
        opacity = 1.0f;
        color.a = 1.0f;
    } else if(blendMode == BM_BLEND) {
        opacity = color.a;
    } else if(blendMode == BM_ADD) {
        opacity = 0.0f;
    } else if(blendMode == BM_ADDX2) {
        opacity = 0.0f;
        color *= 2.0f;
    }

    float halfWidth = ubo.extent.x / 2.0f;
    float halfHeight = ubo.extent.y / 2.0f;
    gl_Position = vec4(inPosition.x / halfWidth - 1.0f, inPosition.y / halfHeight - 1.0f, 0.0, 1.0);

    fragColor = color;
    fragTexCoord = inTexCoord;
    fragOpacity = opacity;
}
```

The fragment shader simply multiplies the outgoing alpha value with the opacity:
```glsl
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(set = 1, binding = 0) uniform sampler2D texSampler;

layout(location = 0) in vec4 fragColor;
layout(location = 1) in vec2 fragTexCoord;
layout(location = 2) in float fragOpacity;

layout(location = 0) out vec4 outColor;

void main() {
    vec4 texel = texture(texSampler, fragTexCoord);
    outColor.rgb = texel.rgb * fragColor.rgb * fragColor.a * texel.a;
    outColor.a = texel.a * fragColor.a * fragOpacity;
}
```

Once the shaders are correctly set up, switching blend modes does not require changing
anything in the graphics pipeline.