---
title: Test Driven Vulkan
tags: c++ tdd vulkan
---
I really like the concept of 
[test driven development](https://en.wikipedia.org/wiki/Test-driven_development), 
but I've often struggled with
applying it to the codebase I'm working in. It's easy to do simple test projects from
scratch with TDD, and I've used with success for new projects that go well beyond simple
test projects.

It's hard to retrofit a 
[large codebase](https://www.reddit.com/r/Eve/comments/2stgua/motherfucking_legacy_code/) 
with unit tests, especially when you're not 
experienced with TDD to begin with, and I find it doubly so for graphics code, or 
user interfaces.

One of my goals for my 
[Vulkan sprite renderer](https://github.com/snorristurluson/vulkan-sprites) 
was to apply test driven development to
a graphics engine - at least it's a new codebase, and I was hoping to figure out how
to apply it to graphics.

## What to test?
How do you unit test a renderer? When I run something I see if a triangle is drawn on
the screen, but how do I test it? This is where I've usually thrown my hands in the air
and given up on unit tests for a renderer. Once you have an established renderer you
can do regression tests where you capture the output of some operation and do image
comparison to a reference image, but that doesn't help much early on, with the low
level operations.

I've opted for a combination of simple sample programs that I run to see some visual
results and unit tests that check for the details that may not be readily visible.
So, basically, stop worrying about unit tests not telling me whether that first triangle
made it to the screen - I'll check that the old fashioned way.

## Vulkan validation layers
The Vulkan API does very limited error checking to reduce the driver overhead. It does,
however, have the notion of validation layers that you can optionally enable. This
validation can be very thorough, and it's easy to hook into for unit tests.

Using this, I can write unit tests that verify that the renderer operations did not
trigger any errors or warnings from the validation layers.

## Capturing validation messages
I have a class called DebugMessenger, with a static function that can be used as the
callback in the *VkDebugUtilsMessengerCreateInfoEXT* structure passed to 
*CreateDebugUtilsMessengerEXT*. The static function just forwards it to an instance
function, with the instance passed in the *pUserData* field.

```cpp
void Renderer::setupDebugCallback() {
    m_debugMessenger = std::make_unique<DebugMessenger>();

    VkDebugUtilsMessengerCreateInfoEXT createInfo = {};
    createInfo.sType = VK_STRUCTURE_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT;
    createInfo.messageSeverity =
            VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT |
            VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT;
    createInfo.messageType =
            VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT |
            VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT;
    createInfo.pfnUserCallback = DebugMessenger::Log;
    createInfo.pUserData = reinterpret_cast<void*>(m_debugMessenger.get());

    if (CreateDebugUtilsMessengerEXT(m_instance, &createInfo, nullptr, &m_callback) != VK_SUCCESS) {
        throw std::runtime_error("failed to set up debug callback!");
    }
}
```

The DebugMessenger simply counts the messages it gets of each severity, as well as
outputting the message to stderr.

```cpp
class DebugMessenger {
public:
    DebugMessenger();

    int GetErrorAndWarningCount();
    int GetErrorCount();
    int GetWarningCount();
    int GetInfoCount();

    static VKAPI_ATTR VkBool32 VKAPI_CALL
    Log(VkDebugUtilsMessageSeverityFlagBitsEXT messageSeverity, VkDebugUtilsMessageTypeFlagsEXT messageType,
                  const VkDebugUtilsMessengerCallbackDataEXT *pCallbackData, void *pUserData);
protected:
    int m_errorCount;
    int m_warningCount;
    int m_infoCount;

    VkBool32
    Log(const VkDebugUtilsMessageSeverityFlagBitsEXT &messageSeverity, VkDebugUtilsMessageTypeFlagsEXT messageType,
        const VkDebugUtilsMessengerCallbackDataEXT *pCallbackData);
};

VkBool32
DebugMessenger::Log(VkDebugUtilsMessageSeverityFlagBitsEXT messageSeverity, VkDebugUtilsMessageTypeFlagsEXT messageType,
                    const VkDebugUtilsMessengerCallbackDataEXT *pCallbackData, void *pUserData) {
    auto pThis = static_cast<DebugMessenger*>(pUserData);
    return pThis->Log(messageSeverity, messageType, pCallbackData);

}

VkBool32 DebugMessenger::Log(const VkDebugUtilsMessageSeverityFlagBitsEXT &messageSeverity,
                             VkDebugUtilsMessageTypeFlagsEXT messageType,
                             const VkDebugUtilsMessengerCallbackDataEXT *pCallbackData) {
    UNUSED(messageType);

    std::string severity;
    if(messageSeverity & VK_DEBUG_UTILS_MESSAGE_SEVERITY_INFO_BIT_EXT) {
        severity = "info";
        m_infoCount++;
    }
    if(messageSeverity & VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT) {
        severity = "warning";
        m_warningCount++;
    }
    if(messageSeverity & VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT) {
        severity = "error";
        m_errorCount++;
    }

    if (!severity.empty()) {
        std::cerr << severity << ": " << pCallbackData->pMessage << std::endl;
    }

    return VK_FALSE;
}

DebugMessenger::DebugMessenger() : m_errorCount(0), m_warningCount(0), m_infoCount(0) {
}

int DebugMessenger::GetErrorAndWarningCount() {
    return m_errorCount + m_warningCount;
}

int DebugMessenger::GetErrorCount() {
    return m_errorCount;
}

int DebugMessenger::GetWarningCount() {
    return m_warningCount;
}

int DebugMessenger::GetInfoCount() {
    return m_infoCount;
}
```

## The tests
I'm using 
[Google Test](https://github.com/google/googletest), 
and instantiating my renderer requires a 
[GLFW](http://www.glfw.org/) window, so I've got a 
[test fixture](https://github.com/google/googletest/blob/master/googletest/docs/primer.md#test-fixtures-using-the-same-data-configuration-for-multiple-tests)
like this:

```cpp
class RendererTest : public ::testing::Test {
protected:
    void SetUp() override {
        glfwInit();
        glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
        m_window = glfwCreateWindow(800, 600, "RendererTest", nullptr, nullptr);
    }

    void TearDown() override {
        glfwDestroyWindow(m_window);
    }

    GLFWwindow *m_window;
};
```
As a side note, when I'm about to write a new class, I usually start with a test
like this:
```cpp
TEST_F(RendererTest, CanCreate) {
    Renderer r;
}
```
It may seem pointless, but this helps keeping me in the mindset of writing tests first.
It also focuses the work of creating the necessary files, adding the includes needed, etc.

Now I can start writing tests like this:
```cpp
TEST_F(RendererTest, Initialize_EnableValidation) {
    Renderer r;
    r.Initialize(m_window, Renderer::ENABLE_VALIDATION);
    EXPECT_TRUE(r.IsInitialized());
    ASSERT_NE(r.GetDebugMessenger(), nullptr);
    EXPECT_EQ(r.GetDebugMessenger()->GetErrorAndWarningCount(), 0);
}

TEST_F(RendererTest, CreateTexture) {
    Renderer r;
    r.Initialize(m_window, Renderer::ENABLE_VALIDATION);
    auto t = r.CreateTexture("resources/texture.jpg");
    ASSERT_NE(t.get(), nullptr);
    EXPECT_EQ(t->GetWidth(), 512);
    EXPECT_EQ(t->GetHeight(), 512);
    EXPECT_EQ(r.GetDebugMessenger()->GetErrorAndWarningCount(), 0);
}
```

I can also write tests that actually render something:
```cpp
TEST_F(RendererTest, SetTexture) {
    Renderer r;
    r.Initialize(m_window, Renderer::ENABLE_VALIDATION);

    auto t = r.CreateTexture("resources/texture.jpg");

    r.StartFrame();
    r.SetTexture(t);
    r.DrawSprite(0, 0, 100, 100);
    r.EndFrame();
    r.WaitUntilDeviceIdle();

    EXPECT_EQ(r.GetDebugMessenger()->GetErrorAndWarningCount(), 0);
}
```
Of course, this test doesn't really tell me that the renderer is really doing anything,
so I have a sample program like this:
```cpp
void FirstApp::Run()
{
    Renderer r;
    r.Initialize(m_window, Renderer::ENABLE_VALIDATION);
    r.SetClearColor({0.0f, 0.0f, 1.0f, 1.0f});
    auto t = r.CreateTexture("resources/texture.jpg");
    while (!glfwWindowShouldClose(m_window)) {
        glfwPollEvents();
        if(r.StartFrame()) {
            r.SetTexture(t);
            r.DrawSprite(32, 32, t->GetWidth(), t->GetHeight());
            r.EndFrame();
        }
    }
    r.WaitUntilDeviceIdle();
}
```
![FirstAppScreenshot](/images/FirstAppScreenshot.png)

Coupling the test with a sample program like above, the test gives me some confidence
that I'm not overlooking errors. It also means that I can do refactoring on the code
with some confidence that the sample program will still work.

