My [Vulkan sprite renderer](https://github.com/snorristurluson/vulkan-sprites)
so far is quite simple - it just pumps sprites, made up from a pair of triangles each
through a very straight forward vertex shader and pixel shader pipeline. In Vulkan terms
it only has one pipeline.

Now I want to add high dynamic range support, which implies adding another pipeline,
which in turn implies I need to abstract pipelines, finally.

## Pipelines
What is a pipeline, anyway? A Vulkan pipeline is basically the whole setup of things
that happen from the moment you issue a draw command until a pixel shows up in a
frame buffer. This includes all the shaders along the way.

Adam Sawicki posted a [great overview](https://gpuopen.com/understanding-vulkan-objects/)
of Vulkan objects explaining all of this in a lot more detail.


Don't try to solve the general case - stick to a fairly rigid setup. This allows
for a simpler application interface for the Renderer.