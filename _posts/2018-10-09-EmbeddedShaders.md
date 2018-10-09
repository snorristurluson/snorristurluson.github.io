---
title: Embedded shaders
tags: vulkan c++ cmake
---
My simple Vulkan sprite renderer doesn't have a large collection of complicated shaders.
In fact, it only has one vertex shader and one fragment shader. Rather than loading the
SPIR-V code from files on disk, I've opted for embedding them in the executable.

I found [this](https://gist.github.com/vlsh/a0d191701cb48f157b05be7f74d79396)
GitHub Gist to compile shaders in cmake, and have extended it to encode the
resulting compiled code so that I can include it in a C++ file.

```
file(GLOB_RECURSE GLSL_SOURCE_FILES
        "shaders/*.frag"
        "shaders/*.vert"
        )

foreach(GLSL ${GLSL_SOURCE_FILES})
    get_filename_component(FILE_NAME ${GLSL} NAME)
    set(SPIRV "${PROJECT_BINARY_DIR}/shaders/${FILE_NAME}.spv")
    add_custom_command(
            OUTPUT ${SPIRV}
            COMMAND ${CMAKE_COMMAND} -E make_directory "${PROJECT_BINARY_DIR}/shaders/"
            COMMAND ${GLSL_VALIDATOR} -V ${GLSL} -o ${SPIRV}
            COMMAND cat ${SPIRV} | xxd -i > ${GLSL}.array
            DEPENDS ${GLSL})
    list(APPEND SPIRV_BINARY_FILES ${SPIRV})
endforeach(GLSL)

add_custom_target(
        Shaders
        DEPENDS ${SPIRV_BINARY_FILES}
)

add_dependencies(engine Shaders)

add_custom_command(TARGET engine PRE_BUILD
        COMMAND ${CMAKE_COMMAND} -E make_directory "$<TARGET_FILE_DIR:engine>/shaders/"
        COMMAND ${CMAKE_COMMAND} -E copy_directory
        "${PROJECT_BINARY_DIR}/shaders"
        "$<TARGET_FILE_DIR:engine>/shaders"
        )

```
The [xxd](https://linux.die.net/man/1/xxd) command encodes its input as hexadecimal
characters, and with the -i flag it does so in a format suitable for including in a C
file as a character array.

I have a class called *ShaderLib* that manages the shaders:
```
class ShaderLib {
public:
    static uint8_t* GetVertexShader();
    static size_t GetVertexShaderSize();

    static uint8_t* GetPixelShader();
    static size_t GetPixelShaderSize();
};

namespace {
    uint8_t s_vertexShader[] = {
        #include "../engine/shaders/shader.vert.array"
    };
    uint8_t s_pixelShader[] = {
        #include "../engine/shaders/shader.frag.array"
    };
}

uint8_t *ShaderLib::GetVertexShader() {
    return s_vertexShader;
}

size_t ShaderLib::GetVertexShaderSize() {
    return sizeof(s_vertexShader);
}

uint8_t *ShaderLib::GetPixelShader() {
    return s_pixelShader;
}

size_t ShaderLib::GetPixelShaderSize() {
    return sizeof(s_pixelShader);
}
```

The shaders are built before the engine is built so the files included are always ready when
the ShaderLib files are built, ensuring that the latest shaders are always included in the
engine.

This setup allows me to write simple samples and tests that don't have dependencies on
the shader files - very convenient.