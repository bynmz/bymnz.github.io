---
layout: post
title: Simple UI with Dear Imgui
---

I needed a quick and easy solution to implement a UI for The Nile C++ Engine and came across <a href="https://github.com/ocornut/imgui" target="_blank">Dear Imgui</a>
.
The following steps outline how I integrated Dear Imgui into The Nile Engine. If you're not very familiar with Vulkan and graphics APIs and want to follow along with this tutorial, there are several vulkan specification docs and resources online, such as <a href="https://vulkan-tutorial.com/" target="_blank">this one</a> that can provide a better understanding of the graphics API that I used. Feel free to clone my <a href="https://github.com/bynmz/mygraphicsengine" target="_blank">vulkan renderer</a>, which I will be using here and in future tutorials. You may also modify the renderer to suit your own personal requirements.


![](/images/simple_dear_imgui_ui.png)

First, I copied the Dear imgui library into the ```external/imgui``` directory and added it's path to cmake.

```cmake
include_directories(external)

# If IMGUI_PATH not specified in .env.cmake, try fetching from git repo
if (NOT IMGUI_PATH)
  message(STATUS "IMGUI_PATH not specified in .env.cmake, using external/imgui")
  set(IMGUI_PATH external/imgui)
endif()

file(GLOB_RECURSE SOURCES ${PROJECT_SOURCE_DIR}/src/*.cpp)

add_executable(${PROJECT_NAME} ${SOURCES} ${IMGUI_PATH}/backends/imgui_impl_glfw.cpp ${IMGUI_PATH}/backends/imgui_impl_vulkan.cpp ${IMGUI_PATH}/imgui.cpp ${IMGUI_PATH}/imgui_draw.cpp ${IMGUI_PATH}/imgui_demo.cpp ${IMGUI_PATH}/imgui_tables.cpp ${IMGUI_PATH}/imgui_widgets.cpp)
```
Next, I created a new directory ```framework/ui``` for my SimpleUI class header and implementation files.

Create a standard C++ header file ```simpleUI.hpp``` and initialize the member variables, constructor, destructor (for clean programming :)), and a few private/ public methods that we can call in our app.

```cpp
class SimpleUI
{
private:
void loadFonts();

Device& mDevice; 
VkRenderPass mRenderPass;
size_t mImageCount; 
GLFWwindow* mWindow;

// Data
VkPipelineCache mPipelineCache = VK_NULL_HANDLE;
QueueFamilyIndices indices = mDevice.findPhysicalQueueFamilies();
VkQueue mQueue = mDevice.graphicsQueue();
VkResult err;
VkDescriptorPool imguiPool;
ImGuiIO& io;

// UI State
bool show_demo_window = false;
bool show_another_window = false;
ImVec4 clear_color = ImVec4(0.45f, 0.55f, 0.60f, 1.00f);


public:
SimpleUI(Device& device,
        GLFWwindow* window,
        VkRenderPass renderPass, 
        size_t imageCount);
~SimpleUI();

float getFrameRate() { return io.Framerate; }
float getDeltaTime() { return io.DeltaTime; }

void init();
// delete copy constructors
SimpleUI(const SimpleUI &) = delete;
SimpleUI &operator=(const SimpleUI &) = delete;

void startUI();
void renderUI(VkCommandBuffer commandBuffer, Renderer &renderer);

static void check_vk_result(VkResult err);
};
```
In the SimpleUI class implementation we are setting up a Dear Imgui context when the class constructor is called. We are also making sure to cleanup the context and descriptor pool in the destructor.

```cpp
#include "simple_ui.hpp"

    SimpleUI::SimpleUI(
      ...
        )
        : 
      ...
    io( [] -> ImGuiIO& {
        // Setup Dear ImGui context
        IMGUI_CHECKVERSION();
        ImGui::CreateContext();
        return ImGui::GetIO();
        }())
    {
        init();
    }

    SimpleUI::~SimpleUI() {
        // Cleanup
        err = vkDeviceWaitIdle(mDevice.device());
        check_vk_result(err);
        ImGui_ImplVulkan_Shutdown();
        ImGui_ImplGlfw_Shutdown();
        ImGui::DestroyContext();
        vkDestroyDescriptorPool(mDevice.device(), imguiPool, nullptr);
    }

    void SimpleUI::init() {
        //1: Create descriptor pool for IMGUI
        VkDescriptorPoolSize pool_sizes[] =
        {
            {VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, 1},
        };

        VkDescriptorPoolCreateInfo pool_info = {};
        pool_info.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO;
        pool_info.flags = VK_DESCRIPTOR_POOL_CREATE_FREE_DESCRIPTOR_SET_BIT; 
        pool_info.maxSets = 1;
        pool_info.poolSizeCount = (uint32_t)IM_ARRAYSIZE(pool_sizes);
        pool_info.pPoolSizes = pool_sizes;
        err = vkCreateDescriptorPool(mDevice.device(), &pool_info, nullptr, &imguiPool);
        check_vk_result(err);

        (void)io;
        // Enable Keyboard Controls
        io.ConfigFlags |= ImGuiConfigFlags_NavEnableKeyboard;    
        // Enable Gamepad Controls 
        io.ConfigFlags |= ImGuiConfigFlags_NavEnableGamepad;       

        // Setup Dear ImGui style
        ImGui::StyleColorsDark();

        // Setup Platform/Renderer backends
        ImGui_ImplGlfw_InitForVulkan(mWindow, true);
        ImGui_ImplVulkan_InitInfo init_info = {};
        init_info.Instance = mDevice.getInstance();
        init_info.PhysicalDevice = mDevice.getPhysicalDevice();
        init_info.Device = mDevice.device();
        init_info.QueueFamily = indices.graphicsFamily;
        init_info.Queue = mQueue;
        init_info.PipelineCache = mPipelineCache;
        init_info.DescriptorPool = imguiPool;
        init_info.Subpass = 0;
        init_info.MinImageCount = 2;
        init_info.ImageCount = mImageCount;
        init_info.MSAASamples = VK_SAMPLE_COUNT_1_BIT;
        init_info.Allocator = nullptr;
        init_info.CheckVkResultFn = check_vk_result;
        ImGui_ImplVulkan_Init(&init_info, mRenderPass);
    }

    void SimpleUI::check_vk_result(VkResult err)
    {
        if (err == 0)
            return;
        fprintf(stderr, "[vulkan] Error: VkResult = %d\n", err);
        if (err < 0)
            abort();
    }
```
The ```startUI``` method initializes ImGui for a new frame.
```cpp
void SimpleUI::startUI() {
    ImGui_ImplVulkan_NewFrame();
    ImGui_ImplGlfw_NewFrame();
    ImGui::NewFrame();

    // 1. Show the big demo window (Most of the sample code is in ImGui::ShowDemoWindow()! You can browse its code to learn more about Dear ImGui!).
    if (show_demo_window)
        ImGui::ShowDemoWindow(&show_demo_window);

    // 2. Show a simple window that we create ourselves. We use a Begin/End pair to create a named window.
    {
        static float f = 0.0f;
        static int counter = 0;
        // Create a window called "Hello, world!" and append into it.
        ImGui::Begin("Hello, world!");                          
        ImGui::Text("Application average %.3f ms/frame (%.1f FPS)", 1000.0f / this->getFrameRate(), this->getFrameRate());
        // Edit bools storing our window open/close state
        ImGui::Checkbox("Settings", &show_demo_window);     
        // Edit 3 floats representing a color 
        ImGui::ColorEdit3("clear color", (float*)&clear_color); 
        ImGui::End();
    }

    // 3. Show another simple window.
    if (show_another_window)
    {
        // Pass a pointer to our bool variable (the window will have a closing button that will clear the bool when clicked)
        ImGui::Begin("Another Window", &show_another_window);   
        ImGui::Text("Hello from another window!");
        if (ImGui::Button("Close Me"))
            show_another_window = false;
        ImGui::End();
    }
}
```
Render ImGui and record ImGui primitives into the Vulkan command buffer. The method also updates the clear values in the renderer with the values obtained from ImGui.
```cpp
void SimpleUI::renderUI(VkCommandBuffer commandBuffer, Renderer& renderer) {
    ImGui::Render();
    ImDrawData* draw_data = ImGui::GetDrawData();
    const bool is_minimized = (draw_data->DisplaySize.x <= 0.0f || draw_data->DisplaySize.y <= 0.0f);
    if (!is_minimized)
    {   
      renderer.clearValues[0].color.float32[0] = clear_color.x * clear_color.w;
      renderer.clearValues[0].color.float32[1] = clear_color.y * clear_color.w;
      renderer.clearValues[0].color.float32[2] = clear_color.z * clear_color.w;
      renderer.clearValues[0].color.float32[3] = clear_color.w;
      // Record dear imgui primitives into command buffer
      ImGui_ImplVulkan_RenderDrawData(draw_data, commandBuffer);
    }
}
```
Initialize a new SimpleUI object from anywhere in our app and start the UI inside the game loop before calling the render method; Make sure you are passing in a valid commandBuffer to the SimpleUI class.
```cpp
SimpleUI simpleUI{
    device,
    window.getGLFWwindow(),
    renderer.getSwapChainRenderPass(),
    renderer.getImageCount()
};

while (!window.shouldClose()) {
    glfwPollEvents();

    // Start the DearImgui frame
    simpleUI.startUI();

    if (auto commandBuffer = renderer.beginFrame()) {
      ...

      // Begin render pass
      renderer.beginSwapChainRenderPass(commandBuffer);

      //Render UI
      simpleUI.renderUI(commandBuffer, renderer);

      renderer.endSwapChainRenderPass(commandBuffer);
      renderer.endFrame();
    }
}

```

![](/images/nile_ui_in_action.png)

The SimpleUI class encapsulates the setup, management, and rendering of ImGui components seamlessly into the application. With ImGui, you can easily create interactive user interfaces for tweaking parameters, visualizing data, and enhancing the debugging process.

Feel free to extend the functionality of the SimpleUI class to include additional ImGui features and cater to the specific needs of your application. Experiment with different ImGui widgets, layouts, and styles to create a user-friendly environment that enhances the overall usability of your graphics application.