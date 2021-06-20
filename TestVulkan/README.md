# Frist Triangle

## Instacnce

### Create an instance 创建实例

Vulkan中的许多结构要求显式指定sType成员中的类型,将作为`pNext`的成员以指向扩展信息,使用值初始化时将其保留为`nullptr`.

Vulkan中的信息通过`struct`传递信息以代替`function`,通过填充结构体以为创建新实例提供信息

```cpp
    VkApplicationInfo appInfo{};
    appInfo.sType = VK_STRUCTURE_TYPE_APPLICATION_INFO;
    appInfo.pApplicationName = "Triangle Build demo";
    appInfo.applicationVersion = VK_MAKE_VERSION(1,0,0);
    appInfo.pEngineName = "No Engine";
    appInfo.engineVersion = VK_MAKE_VERSION(1,0,0);
    appInfo.apiVersion = VK_API_VERSION_1_0;
```

以`struct`形式创建app信息

```cpp
    VkInstanceCreateInfo createInfo{};
    createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
    createInfo.pApplicationInfo = &appInfo;
```

以上结构不是可选的，用于告诉Vulkan驱动我们选择的全局扩展及验证层,全局意味着适用于整个程序而不是指定设备

```cpp
    uint32_t glfwExtensionCount = 0;
    const char** glfwExtensions;
    glfwExtensions = glfwGetRequiredInstanceExtensions(&glfwExtensionCount);
    // 通过GLFW内置函数获取扩展
    
    createInfo.enabledExtensionCount = glfwExtensionCount;
    createInfo.ppEnabledExtensionNames = glfwExtensions;
```

以上的几行用于指定全局扩展，由于Vulkan是一个平台无关的API，这意味着您需要一个扩展来与系统接口,这里使用GLFW中的内置函数以返回扩展，并传递至结构体

```cpp
    createInfo.enabledLayerCount = 0;
    // 验证层指定，由于暂时没有用到因此置零

    VkResult result = vkCreateInstance(&createInfo,nullptr,&instance);
    // 通过struct 创建Vk实例
```

最终创建VK实例

在Vk中，对象构造函数参数遵循的一般模式为

- 指向具有创建信息的结构的指针
- 指向自定义分配器回调的指针（在此为`nullptr`）
- 指向存储新对象句柄的变量的指针

若构造成功，则实例句柄存储于`Vkinstance`类成员中

几乎所有的Vulkan函数都返回一个`VkResult`型的值，该值要么为`VK_SUCCESS`或一个错误码,通过确认result值来确认创建实例是否成功

在`vkCreateInstance`中存在一个错误码`VK_ERROR_EXTENSION_NOT_PRESENT`,通过这种方式来确认扩展是否支持。

可以通过`vkEnumerateInstanceExtensionProperties`来检索受支持扩展的列表，该函数使用一个存储扩展数的指针与存储扩展详细信息的`VkExtensionProperties`数组，同时还有一个可选的第一个参数，允许我们按照特定的验证层过滤扩展

```cpp
    uint32_t extensionCount = 0;
    vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);
    // 通过留空获取扩展数量

    std::vector<VkExtensionProperties> extensions(extensionCount);
    vkEnumerateInstanceExtensionProperties(nullptr,&extensionCount,extensions.data());
    // 获取扩展详细信息
```

在每个`VkExtensionProperties` 中，包含扩展的名字和版本号，可以通过循环得知这些信息,可以将这些信息与通过`glfwGetRequiredInstanceExtensions`得到的信息相比较

### Cleaning up 清理实例

清理 Vk 实例的唯一机会在程序结束之前，通过`vkDestroyInstance`实现

第一个参数为vk实例，第二个参数是一个可选参数，可以通过传递`nullptr`来忽略，其他的vk资源需要在vk实例清理钱销毁

## Validation layers 验证层

### 前言

由于Vulkan是围绕最小化驱动开销的思想设计的，因此对错误的检查十分有限，所以要求编写者明确自己所做的任何事

当然，较少的错误检查不意味着不检查，通过可选的验证层，来检验错误，常见操作有：

- 根据规范检查参数值以确定是否误用
- 跟踪对象的创建与销毁以查找资源泄露
- 跟踪调用线程以检查线程安全性
- 将调用与参数记录至标准输出
- 跟踪Vulkan调用以分析与重放

例:

```cpp
VkResult vkCreateInstance(
    const VkInstanceCreateInfo* pCreateInfo,
    const VkAllocationCallbacks* pAllocator,
    VkInstance* instance) {

    if (pCreateInfo == nullptr || instance == nullptr) {
        log("Null pointer passed to required parameter!");
        return VK_ERROR_INITIALIZATION_FAILED;
    }

    return real_vkCreateInstance(pCreateInfo, pAllocator, instance);
}
```

Vulkan没有任意内置的验证层，LunarG Vulkan SDK 提供了一组很好的层，可以检测常见错误，验证层只有在安装在系统时才能使用

以前 Vulkan 中有两种不同类型的验证层: 实例验证层和设备验证层。这个想法是，实例层只检查与全局 Vulkan 对象相关的调用，比如实例，而设备特定层只检查与特定 GPU 相关的调用。设备特定的层现在已被弃用，这意味着实例验证层应用于所有 Vulkan 调用。规范文档仍然建议您在设备级别启用验证层以实现兼容性，这是某些实现所要求的。我们只需在逻辑设备级别将相同的层指定为实例，稍后我们将看到这一点。

### 使用验证层

这节的目的在于了解如何启用 Vulkan SDK 提供的标准诊断层，需要通过指定验证层的名称来启用验证层，所有标准验证捆绑在一个包含在SDK中的层中，称为`VK_LAYER_KHRONOS_validation`

首先向程序添加两个配置变量以确定启用验证层的目标与是否启用验证层，使用宏来实现确定当前模式

```cpp
    const std::vector<const char*> validationLayers = {
        "VK_LAYER_KHRONOS_validation"
    };
    // 启用 Vulkan SDK 标准诊断层

    #ifdef NDEBUG
        const bool enableValidationLayers = false;
    #else
        const bool enableValidationLayers = true;
    #endif
```

其次添加一个新的函数`checkValidationLayerSupport`,以确定所有的请求层是否可用，使用`vkEnumerateInstanceExtensionProperties`列出所有可用层，用法与实例与`vkEnumerateInstanceExtensionProperties `相同

```cpp
    bool checkValidationLayerSupport(){
        uint32_t layerCount ;
        vkEnumerateInstanceLayerProperties(&layerCount,nullptr);

        std::vector<VkLayerProperties> availableLayers(layerCount);
        vkEnumerateInstanceLayerProperties(&layerCount,availableLayers.data());
        return false;
    }
    // 确定所有的验证请求层是否可用
```

接下来验证`validationLayers`中的所有层是否存在于`availableelayers` 列表中

```cpp
    bool checkValidationLayerSupport(){
        uint32_t layerCount ;
        vkEnumerateInstanceLayerProperties(&layerCount,nullptr);

        std::vector<VkLayerProperties> availableLayers(layerCount);
        vkEnumerateInstanceLayerProperties(&layerCount,availableLayers.data());

        for(const char* layerName : validationLayers){
            bool layerFound = false;

            for(const auto& layerProperties : availableLayers){
                if(strcmp(layerName,layerProperties.layerName) == 0){
                    layerFound = true;
                    break;
                }
            }

            if(!layerFound){
                return false;
            }
        }

        return true;
    }
```

在`createInstance`中使用

```cpp
    void createInstance(){

        if(enableValidationLayers && !checkValidationLayerSupport()){
            throw std::runtime_error("validation layers requested, but not available!");
        }
        ...
    }
```

接下来在debug模式下运行程序,若没有发生错误，就对`VkInstanceCreateInfo`进行修改，以在允许启用验证层时启用验证层

```cpp
    if(enableValidationLayers){
        createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
        createInfo.ppEnabledLayerNames = validationLayers.data();
        // 启用验证层
    }
    else {
        createInfo.enabledLayerCount = 0;
        // 禁用下置零
    }
```

若检查成功，则`vkCreateInstance`不会返回`VK_ERROR_LAYER_NOT_PRESENT`错误

### Message callback 消息回调

在标准情况下，验证层会将调试信息输出至标准输出，但我们也可以通过显示的回调来自定义处理方式，这种方式下同样允许自定义是否显示某些指定的调试信息，因为不是所有的调试消息都是必要的

通过使用`VK_EXT_debug_utils`扩展来设置一个带有回调的调试管理程序，用于处理消息和相关的细节

我们将首先创建一个 `getRequiredExtensions` 函数，它将根据验证层是否启用来返回所需的扩展列表:

```cpp
    std::vector<const char*> getRequiredExtensions(){
        uint32_t glfwExtensionCount = 0;
        const char ** glfwExtensions;
        glfwExtensions = glfwGetRequiredInstanceExtensions(&glfwExtensionCount);

        std::vector<const char*>extensions(glfwExtensions,glfwExtensions + glfwExtensionCount);

        if(enableValidationLayers){
            extensions.push_back(VK_EXT_DEBUG_UTILS_EXTENSION_NAME);
        }
        
        return extensions;
    }
    // 根据验证层是否启用来返回所需扩展列表
```

扩展列表始终需要GLFW的扩展，但是调试消息扩展是有条件添加的，以上函数使用`VK_EXT_DEBUG_UTILS_EXTENSION_NAME`宏，等价为`VK_EXT_debug_utils`这个字符串

接下来在`createInstance`中使用这个函数

```cpp
    auto extensions = getRequiredExtensions();
    createInfo.enabledExtensionCount = static_cast<uint32_t>(extensions.size());
    createInfo.ppEnabledExtensionNames = extensions.data();
```

只要没有出现`VK_ERROR_EXTENSION_NOT_PRESENT`即可，同时只要验证层可用，即说明扩展存在

使用 `PFN_vkDebugUtilsMessengerCallbackEXT` 的原型添加一个新的静态成员函数，`VKAPI_ATTR` 与 `VKAPI_CALL` 确保这个函数具有Vulkan调用它的正确签名 

```cpp
static VKAPI_ATTR VkBool32 VKAPI_CALL degbugCallback(
    VkDebugUtilsMessageSeverityFlagBitsEXT messageSeverity,
    VkDebugUtilsMessageTypeFlagsEXT messageType,
    const VkDebugUtilsMessengerCallbackDataEXT* pCallbackData,
    void* pUserData){
        std::cerr << "validation layer: "<< pCallbackData->pMessage << std::endl;
        return VK_FALSE;
}

```

第一个参数指定消息的严重性,包括以下标志

- `VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT` 诊断信息
- `VK_DEBUG_UTILS_MESSAGE_SEVERITY_INFO_BIT_EXT` 消息信息，如创建资源
- `VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT` 一个不一定时错误但很有可能时bug的行为信息
- `VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT` 无效但是有可能导致崩溃的信息

可以通过比较操作检查消息的重要程度是否大于某类信息

```cpp
if(messageSeverity >= VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT){
    // Message is important enough to show
}
```

`messageType` 参数有以下值:

- `VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT` 发生了与规范或性能无关的事件
- `VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT` 当前位置发生了违反规范或表明可能存在错误的事情
- `VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT` 对 Vulkan 的潜在非最佳利用

`pCallbackData` 参数引用一个包含消息本身的详细信息的 `VkDebugUtilsMessengerCallbackDataEXT` 结构，其中最重要的成员是:

- `pMessage` 一个以空为结尾的调试信息字符串
- `pObjects` 与消息相关的 Vulkan 对象句柄数组
- `objectCount` 数组中的对象数

`pUserData` 参数包含一个设置回调时指定的指针，允许将自己的数据传递给它

回调将会返回一个bool值以指示是否应该触发验证层消息的Vulkan调用，如果回调返回`true`,将会抛出一个`VK_ERROR_VALIDATION_FAILED_EXT`错误并终止调用，这只用于测试验证层本身，因此因总是返回`VK_FALSE`

接下来就是将自定义的回调函数告诉Vulkan，Vulkan需要一个需要显示创建和销毁的句柄来管理debug回调，这样回调是debug消息管理器的一部分同时你可以拥有任意数量的debug消息管理器

在`instance`下创建类成员`VkDebugUtilsMessengerEXT`

```cpp
 VkDebugUtilsMessengerEXT debugMessenger;
```

添加一个函数`setupDebugMessenger`，并在`initVulkan`中的`createInstance`之后调用它

```cpp
    void initVulkan() {
        createInstance();
        setupDebugMessenger();
    }

    void setupDebugMessenger(){
        if(!enableValidationLayers) return;
    }
```

我们需要在结构中填充有关消息管理器和回调的细节

```cpp
void populateDebugMessengerCreateInfo(VkDebugUtilsMessengerCreateInfoEXT& createInfo){
    createInfo = {};
    createInfo.sType = VK_STRUCTURE_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT;
    createInfo.messageSeverity = VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT;
    createInfo.messageType = VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT;
    createInfo.pfnUserCallback = debugCallback;
    createInfo.pUserData = nullptr;
}
```

`messageSeverity` 字段允许指定希望调用回调函数的所有严重性类型，在这个实例中，调用了除了`VK_DEBUG_UTILS_MESSAGE_SEVERITY_INFO_BIT_EXT`之外的所有类型，以便在忽略冗长的常规调试信息外接受关于可能出现问题的通知

`messageType` 字段允许筛选通知回调的消息类型，这个实例中启用了所有的回调类型

`pfnUserCallback` 字段用于指定指向回调函数的指针，可以选择一个指向`pUserData`字段的指针，这个字段将通过`pUserData`参数传递给回调函数

注意，配置验证层信息和调试回调函数的方法还有很多，更多有关信息，见[扩展规范](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VK_EXT_debug_utils)

以上创建的结构体应该传递给`vkCreateDebugUtilsMessengerEXT`函数，以创建`VkDebugUtilsMessengerEXT`对象。但由于这个函数是一个扩展函数，所以不会自动加载，必须使用`vkGetInstanceProcAddr`函数查找相应的地址，这里通过代理函数来实现

```cpp
VkResult CreateDebugUtilsMessengerEXT(VkInstance instance, const VkDebugUtilsMessengerCreateInfoEXT* pCreatrInfo, const VkAllocationCallbacks* pAllocator, VkDebugUtilsMessengerEXT* pDebugMessenger){
    auto func = (PFN_vkCreateDebugUtilsMessengerEXT)vkGetInstanceProcAddr(instance, "vkCreateDebugUtilsMessengerEXT");
    // 若函数无法加载，则返回nullptr
    if(func != nullptr){
        return func(instance, pCreatrInfo, pAllocator, pDebugMessenger);
    }
    else{
        return VK_ERROR_EXTENSION_NOT_PRESENT;
    }
}
// 用vkGetInstanceProcAddr函数查找相应的地址
```

在`setupDebugMessenger`中实例化对象并调用该函数

```cpp
void setupDebugMessenger(){
    if(!enableValidationLayers) return;
    
    VkDebugUtilsMessengerCreateInfoEXT createInfo;
    
    if(CreateDebugUtilsMessengerEXT(instance, &createInfo, nullptr, &debugMessenger) != VK_SUCCESS){
        throw std::runtime_error("failed to set up debug messenger!");
    }
}
```

倒数第二个参数是可选分配器的回调，我们将其设置为`nullptr`

同时`VkDebugUtilsMessengerEXT`需要调用`vkDestroyDebugUtilsMessengerEXT`摧毁，类似于`vkCreateDebugUtilsMessengerEXT`，同样需要显式加载

```cpp
void DestroyDebugUtilsMessengerEXT(VkInstance instance, VkDebugUtilsMessengerEXT debugMessenger, const VkAllocationCallbacks* pAllocator){
    auto func = (PFN_vkDestroyDebugUtilsMessengerEXT) vkGetInstanceProcAddr(instance, "vkDestroyDebugUtilsMessengerEXT");
    if(func != nullptr){
        func(instance, debugMessenger, pAllocator);
    }
}
```

在`cleanup`函数中调用

```cpp
void cleanup() {
    if(enableValidationLayers){
        DestroyDebugUtilsMessengerEXT(instance, debugMessenger, nullptr);
    }
    
    vkDestroyInstance(instance, nullptr);
    // 清理 vk 实例

    glfwDestroyWindow(window);
    // 清理窗口

    glfwTerminate();
    // 关闭glfw
}
```

### Debugging instance creation and destruction 调试实例的创建与销毁

对`vkCreateDebugUtilsMessengerEXT`调用而言，需要一个有效的实例，且在这实例销毁前通过`vkDestroyDebugUtilsMessengerEXT`销毁

扩展文档中提供了一种方法使得能专门为`vkCreateInstance` 和 `vkDestroyInstance`创建一个单独的调试单元，需要在 `VkInstanceCreateInfo` 的 `pNext` 字段中传递一个指向`VkDebugUtilsMessengerCreateInfoEXT`结构体的指针

```cpp
void populateDebugMessengerCreateInfo(VkDebugUtilsMessengerCreateInfoEXT& createInfo){
    createInfo = {};
    createInfo.sType = VK_STRUCTURE_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT;
    createInfo.messageSeverity = VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT;
    createInfo.messageType = VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT;
    createInfo.pfnUserCallback = debugCallback;
}
// 填充有关消息管理器和回调的细节

void setupDebugMessenger(){
    if(!enableValidationLayers) return;
    
    VkDebugUtilsMessengerCreateInfoEXT createInfo;
    populateDebugMessengerCreateInfo(createInfo);
    
    if(CreateDebugUtilsMessengerEXT(instance, &createInfo, nullptr, &debugMessenger) != VK_SUCCESS){
        throw std::runtime_error("failed to set up debug messenger!");
    }
}
```

更新函数`populateDebugMessengerCreateInfo` `setupDebugMessenger`

```cpp
void createInstance(){
    if(enableValidationLayers && !checkValidationLayerSupport()){
        throw std::runtime_error("validation layers requested, but not available!");
    }

    VkApplicationInfo appInfo{};
    appInfo.sType = VK_STRUCTURE_TYPE_APPLICATION_INFO;
    appInfo.pApplicationName = "Triangle Build demo";
    appInfo.applicationVersion = VK_MAKE_VERSION(1,0,0);
    appInfo.pEngineName = "No Engine";
    appInfo.engineVersion = VK_MAKE_VERSION(1,0,0);
    appInfo.apiVersion = VK_API_VERSION_1_0;

    VkInstanceCreateInfo createInfo{};
    createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
    createInfo.pApplicationInfo = &appInfo;
    // 以上结构不是可选的，用于告诉Vulkan驱动我们选择的全局扩展及验证层
    // 全局意味着适用于整个程序而不是指定设备

    // 接下来的几行用于指定全局扩展，由于Vulkan是一个平台无关的API，这意味着您需要一个扩展来与系统接口
    // 这里使用GLFW中的内置函数以返回扩展，并传递至结构体
    
    auto extensions = getRequiredExtensions();
    createInfo.enabledExtensionCount = static_cast<uint32_t>(extensions.size());
    createInfo.ppEnabledExtensionNames = extensions.data();
    
    VkDebugUtilsMessengerCreateInfoEXT debugCreateInfo{};
    
    if(enableValidationLayers){
        createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
        createInfo.ppEnabledLayerNames = validationLayers.data();
        // 启用验证层
        
        populateDebugMessengerCreateInfo(debugCreateInfo);
        createInfo.pNext = (VkDebugUtilsMessengerCreateInfoEXT*) &debugCreateInfo;
    }
    else {
        createInfo.enabledLayerCount = 0;
        // 禁用下置零
        createInfo.pNext = nullptr;
    }

    // 通过struct 创建Vk实例
    if(vkCreateInstance(&createInfo,nullptr,&instance) != VK_SUCCESS){
        throw std::runtime_error("failed to create instance");
        // 若返回result不为VK_SUCCESS 则抛出错误
    }

}
// 创建实例，向驱动程序提供信息以对特定应用优化
```

更新`createInstance`函数

`debugCreateInfo` 变量放在 if 语句之外，以确保它不会在 `vkCreateInstance` 调用之前被销毁。通过这种方式创建一个额外的调试信使，它将在 `vkCreateInstance` 和 `vkDestroyInstance` 期间自动使用，并在此之后清理。

## Physical devices and queue families 物理设备和家庭队列

### Selecting a physical device 选择物理设备

在通过 `VkInstance`初始化Vulkan库后，需要在系统中一个选择支持我们需要特性的图形卡

添加一个函数`pickPhysicalDevice`，并在`initVulkan`中添加对它的调用

```cpp
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    pickPhysicalDevice();
}

void pickPhysicalDevice(){
    
}
```

以`VkPhysicalDevice`作为句柄，建立一个新的类成员，以存储最终选择的图形卡对象，当`VkInstance`被销毁时，这个对象将被隐式销毁

```cpp
VkPhysicalDevice physicalDevice = VK_NULL_HANDLE;
```


