# Frist Triangle

## Setup

### Instacnce

#### Create an instance 创建实例

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

#### Cleaning up 清理实例

清理 Vk 实例的唯一机会在程序结束之前，通过`vkDestroyInstance`实现

第一个参数为vk实例，第二个参数是一个可选参数，可以通过传递`nullptr`来忽略，其他的vk资源需要在vk实例清理钱销毁

### Validation layers 验证层

#### 前言

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

#### 使用验证层

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

#### Message callback 消息回调

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

#### Debugging instance creation and destruction 调试实例的创建与销毁

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

### Physical devices and queue families 物理设备和队列族

#### Selecting a physical device 选择物理设备

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

检查支持Vulkan的显卡数量，若没有，则无法运行

```cpp
void pickPhysicalDevice(){
    uint32_t deviceCount = 0;
    vkEnumeratePhysicalDevices(instance, &deviceCount, nullptr);
    // 首先检查支持Vulkan的设备数量
   
    if(deviceCount == 0){
        throw std::runtime_error("failed to find GPUs with Vulkan support!");
    }
}
// 选择物理设备
```

现在分配一个数组来保存所有的`VkPhysicalDevice`

```cpp
std::vector<VkPhysicalDevice> devices(deviceCount);
vkEnumeratePhysicalDevices(instance, &deviceCount, devices.data());
```

由于不是每个显卡都是相同的，所以需要引入一个新的函数来判断该显卡是否适合我们要执行的操作

```cpp
bool isDeviceSuitable(VkPhysicalDevice device){
    return true;
}
```

逐个显卡检查是否适合操作，若适合则设为实际使用的物理设备

```cpp
for(const auto& device : devices){
    if(isDeviceSuitable(device)){
        physicalDevice = device;
        break;;
    }
}

if(physicalDevice == VK_NULL_HANDLE){
    throw std::runtime_error("failed to find a suitable GPU!");
}
```

#### Base device suitability checks 基本设备适用性检查

可以通过`VkPhysicalDeviceProperties`查询物理设备的名称、类型与支持的Vulkan版本等基本设备属性

```cpp
VkPhysicalDeviceProperties deviceProperties;
vkGetPhysicalDeviceProperties(device, &deviceProperties);
```

可以通过`VkPhysicalDeviceFeatures`查询物理设备的可选功能支持，比如纹理压缩，64位浮动和多视图渲染(对 VR 很有用) 

在这个示例中，只需要Vulkan支持，即可使用，因此不需要对其进行特定的检查

#### Queue families 队列族

对Vulkan而言，几乎所有的操作都需要向队列提交指令，来自不同的队列族的队列有着不同的类型，每个队列族只允许命令的一个子集。例如，可能存在一个只允许处理计算命令的队列族，或者一个只允许内存传输相关命令的队列族

通过添加一个新函数`findQueueFamilies`以检查设备支持的队列族与队列族支持我们使用的指令

现在我们只需要寻找一个支持图形命令的队列，同时由于下一章需要另一个队列，所以要将索引打包为一个结构体

```cpp
struct QueueFamilyIndices{
    uint32_t graphicsFamily;
};

QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device){
    QueueFamilyIndices indices;
    
    return indices;
}
```

若队列族不可用，则在`findQueueFamilies`中抛出异常，但这个函数并不是一个决定设备适用性的位置，例如在某种情况下，我们更倾向于去选择具有专门传输队列的设备，但是这个并不是必须的，因此我们需要一些方法来表示是否找到了特定的队列族

实际上并没有一种魔法值来表示队列族是否存在，因为uint32的任何值都可以是一个有效的队列族索引，因此这里使用C++17引入的新的数据结构来区分一个值是否存在

`std::optional`是一个包装器，不包含任何值，直到为它分配了一些东西。在任何时候，都可以通过调用`has_value()`函数来确认是否包含值

此时数据结构是这样的

```cpp
struct QueueFamilyIndices{
    std::optional<uint32_t> graphicsFamily;
};
```

使用`vkGetPhysicalDeviceQueueFamilyProperties`来检索队列族，使用`VkQueueFamilyProperties`来存储队列族的一些详细信息，包括支持的操作类型及基于该队列族创建的队列数量。需要至少一个支持`VK_QUEUE_GRAPHICS_BIT`的队列族

```cpp
uint32_t queueFamilyCount = 0;
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, nullptr);

std::vector<VkQueueFamilyProperties> queueFamilies(queueFamilyCount);
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, queueFamilies.data());

int i = 0;
for(const auto& queueFamily : queueFamilies){
    if(queueFamily.queueFlags & VK_QUEUE_GRAPHICS_BIT){
        indices.graphicsFamily = i;
    }
    i++;
}
```

接下来使用`isDeviceSuitable`函数检查设备是否可以处理我们要使用的命令，同时改变`QueueFamilyIndices`结构体使得存在通用检查

```cpp
struct QueueFamilyIndices{
    std::optional<uint32_t> graphicsFamily;
    
    bool isComplete(){
        return graphicsFamily.has_value();
    }
};

bool isDeviceSuitable(VkPhysicalDevice device){
    QueueFamilyIndices indices = findQueueFamilies(device);
    
    return indices.isComplete();
}
```

更新循环使得可以提前跳出

```cpp
for(const auto& queueFamily : queueFamilies){
    if(queueFamily.queueFlags & VK_QUEUE_GRAPHICS_BIT){
        indices.graphicsFamily = i;
    }
    
    if(indices.isComplete()){
        break;
    }
    i++;
}
```

### Logical device and queues 逻辑设备与队列

在选择需要使用的物理设备后，我们需要设置一个逻辑设备与之交互。逻辑设备的创建类似于实例的创建，并且描述了我们需要使用的特性。由于我们已经查询了那些队列族是可用的，因此我们同时还要指定创建需要创建的队列。若存在不同的需求，甚至可以从同一个物理设备创建多个逻辑设备

首先创建一个新的类成员来存储逻辑设备句柄

```cpp
VkDevice device;
```

接下来在`initVulkan`中，添加一个`createLogicalDevice`函数

```cpp
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    pickPhysicalDevice();
    createLogicalDevice();
}

void createLogicalDevice() {

}
```

#### Specifying the queues to be created 指定要创建的队列

创建一个逻辑设备需要再次在struct中指定一系列细节，其中第一个将是`VkDeviceQueueCreateInfo`，这个结构描述了单个队列族所需的队列数量，现在我们只需要具有图形功能的队列

```cpp
    QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

    VkDeviceQueueCreateInfo queueCreateInfo{};
    queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
    queueCreateInfo.queueFamilyIndex = indices.graphicsFamily.value();
    ueueCreateInfo.queueCount = 1;
```

当前可用的驱动程序只允许为每个队列族创建少量的队列，实际上不需要多于一个的队列。这是因为可以在多个线程上创建所有的命令缓冲区，然后通过一个低开销的调用在主线程上一次性提交它们

Vulkan允许为队列分配优先级，以便使用0.0 - 1.0 之间的浮点数来影响命令缓冲区执行的调度，即使只有一个队列，这也是必须的。

```cpp
float queuePriority = 1.0f;
queueCreateInfo.pQueuePriorities = &queuePriority;
```

#### Specifying used device features 指定使用的设备特性

下一个要指定的信息是我们要使用的设备特性集。这些特性是我们在上一张利用`vkGetPhysicalDeviceFeatures`获取的，由于这里不需要任何特性，因此可以只是简单定义，当我们需要的时候再使用。

```cpp
VkPhysicalDeviceFeatures deviceFeatures{};
```

#### Creating the logical device 创建逻辑设备

在上面两个指定值指定结束后，就可以填充`VkDeviceCreateInfo`了

```cpp
VkDeviceCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO;
```

首先添加指向队列的创建信息和设备特性的指针

```cpp
createInfo.pQueueCreateInfos = &queueCreateInfo;
createInfo.queueCreateInfoCount = 1;
createInfo.pEnabledFeatures = &deviceFeatures;
```

其余信息与`VkInstanceCreateInfo`结构相似，并要求指定扩展与验证层，不同之处在于这些都是特定于设备的

一个设备特定扩展的例子是`VK_KHR_swapchain`，它允许渲染的图像从指定设备显示到窗口，但有的设备可能只支持计算操作而不支持渲染

`Vulkan`之前的实现对实例和设备的验证层进行了一定的区分，但在当前版本下，忽略了`VkDeviceCreateInfo`的`enabledLayerCount`与`ppEnabledLayerNames`字段，但也可以写入，使得代码可以向旧的版本实现兼容

```cpp
createInfo.enabledExtensionCount = 0;

if(enableValidationLayers){
    createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
    createInfo.ppEnabledLayerNames = validationLayers.data();
}
else {
    createInfo.enabledLayerCount = 0;
}
```

就这样，我们现在通过调用相应名为`vkCreateDevice`的函数实例化逻辑设备

```cpp
if(vkCreateDevice(physicalDevice, &createInfo, nullptr, &device) != VK_SUCCESS){
    throw std::runtime_error("failed to create logical device!");
}
// 参数是物理设备的接口，我们指定的创建信息，可选的分配回调指针和一个指向逻辑设备句柄的指针
```

在cleanup中对逻辑设备进行销毁

```cpp
void cleanup() {
    vkDestroyDevice(device, nullptr);
    
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

#### Retrieving queue handles 检索队列句柄

队列是根据逻辑设备的创建自动创建的，但是我们在创建逻辑设备后没有获得逻辑句柄，因此需要添加一个类成员来存储图形队列句柄

```cpp
VkQueue graphicsQueue;
```

当设备被销毁，设备的队列会被隐式清理，因此不需要在`cleanup`中声明

可以使用`vkGetDeviceQueue`来检索每个队列族的队列句柄，参数为逻辑设备、队列族、队列索引与一个指向用于存储队列句柄的指针，由于只需要从族中创建一个队列，因此只需要使用索引0

## Presentation 显示

### Window surface 窗口表面

由于Vulkan是一个与平台无关的API，所以它不能直接建立与系统窗口之间的接口。为了实现建立Vulkan与系统窗口之间的连接以显示在屏幕上，我们需要WSI(Window System Integration) 扩展。本章中讨论的第一个表面，即`VK_KHR_surface`。其公开了一个`VkSurfaceKHR`对象，以表示一种抽象类型的表面，用于呈现渲染的对象。这个表面将会返回至GLFW创建的窗口中。

`VK_KHR_surface`是一个实例级的扩展，且我们已经启用了它，因为它包含在`glfwGetRequiredInstanceExtensions`返回的列表之中。同时这个列表还包括一些其他的WSL扩展，我们将在接下来几章中使用。

创建实例后需要立刻创建窗口表面，因为它实际上会影响物理设备的选择，我们之所以推迟这样做，是因为窗口表面是渲染目标和表示这个更大主题的一部分，对此的解释会使基本设置变得混乱。还应该注意的是，窗口表面在 Vulkan 中是一个完全可选的组件，如果您只需要离屏渲染。Vulkan 允许你这样做，而不需要像创建一个不可见的窗口(OpenGL 必需的)。

#### Window surface creation 创建窗口表面

首先在调试回调的正下方添加一个表面类成员。

```cpp
VkSurfaceKHR surface;
```

尽管 `VkSurfaceKHR` 对象及其用法与平台无关，但它的创建并不意味着它不依赖于窗口系统的细节。例如，它需要在 Windows 上使用 HWND 和 HMODULE 句柄。因此，这个扩展有一个对特定平台的附加功能，在 Windows 上称为 `VK_KHR_win32_surface`，它也自动包含在 `glfwGetRequiredInstanceExtensions` 的列表中。

我将演示如何使用这个特定于平台的扩展在 Windows 上创建一个表面，但在本教程中我们不会实际使用它。使用像 GLFW 这样的库，却使用特定于平台的代码是没有意义的。实际上，GLFW 拥有 `glfwCreateWindowSurface`，可以为我们处理平台差异。尽管如此，在我们开始依赖它之前， 我们还是要知道它的底层是如何实现的

要访问本地平台函数，你需要更新顶部的 include

```cpp
#define VK_USE_PLATFORM_WIN32_KHR
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>
#define GLFW_EXPOSE_NATIVE_WIN32
#include <GLFW/glfw3native.h>
```

因为窗口表面是 Vulkan 对象，所以它附带了一个需要填充的 `VkWin32SurfaceCreateInfoKHR` 结构。它有两个重要的参数`hwnd` 和 `hinstance`。这些是窗口和进程的句柄。

```cpp
VkWin32SurfaceCreateInfoKHR createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_WIN32_SURFACE_CREATE_INFO_KHR;
createInfo.hwnd = glfwGetWin32Window(window);
createInfo.hinstance = GetModuleHandle(nullptr);
```

使用 `glfwGetWin32Window` 函数从 GLFW 窗口对象获取原始 `HWND`。`GetModuleHandle` 调用返回当前进程的 `HINSTANCE` 句柄。

之后可以使用 `vkCreateWin32SurfaceKHR` 创建表面，其中包括实例的参数、表面创建细节、自定义分配器和表面句柄的变量。从技术上讲，这是一个 WSI 扩展函数，但是它使用非常普遍，以至于标准 Vulkan loader 包含了它，因此与其他扩展不同，您不需要显式地加载它。

```cpp
if (vkCreateWin32SurfaceKHR(instance, &createInfo, nullptr, &surface) != VK_SUCCESS) {
    throw std::runtime_error("failed to create window surface!");
}
```

这个过程在其他的平台上也是类似的，比如Linux，其中的`vkCreateXcbSurfaceKHR`使用 XCB 连接和窗口作为 X11的创建细节

`glfwCreateWindowSurface`函数使用不同的实现为每个平台精确地执行这个操作。现在我们将把它整合到我们的程序中。在创建实例和 `setupDebugMessenger` 之后添加一个 `createSurface` 函数从 `initVulkan` 调用。

```cpp
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
}

void createSurface() {

}
```

调用使用简单的参数而不是结构体，这使得函数的实现非常简单:

```cpp
void createSurface(){
    if(glfwCreateWindowSurface(instance, window, nullptr, &surface) != VK_SUCCESS){
        throw std::runtime_error("failed to create window surface!");
    }
}
```

这些参数分别为`VkInstance`、 GLFW 窗口指针、自定义分配器和指向 `VkSurfaceKHR` 变量的指针，通过相关的平台调用传递`VkResult`

同时需要更改`cleanup`使得表面能在实例销毁前销毁

```cpp
void cleanup() {
    vkDestroyDevice(device, nullptr);
    
    if(enableValidationLayers){
        DestroyDebugUtilsMessengerEXT(instance, debugMessenger, nullptr);
    }
    vkDestroySurfaceKHR(instance, surface, nullptr);
    vkDestroyInstance(instance, nullptr);
    // 清理 vk 实例

    glfwDestroyWindow(window);
    // 清理窗口

    glfwTerminate();
    // 关闭glfw
}
```

#### Querying for presentation support 查询显示支持

虽然 Vulkan 实现可能支持窗口系统集成，但这并不意味着系统中的每个设备都支持它。因此，我们需要扩展 `isDeviceSuitable`，以确保设备能够将图像呈现到我们创建的表面。由于表示是一个特定于队列的特性，所以问题实际上是找到一个支持表示我们创建的表面的队列族。

实际上，支持绘图命令的队列族和支持表示的队列族可能不会重叠。因此，我们必须考虑到，通过修改 `QueueFamilyIndices` 结构，可能存在一个不同的表示队列:

```cpp
struct QueueFamilyIndices{
    std::optional<uint32_t> graphicsFamily;
    std::optional<uint32_t> presentFamily;
    
    bool isComplete(){
        return graphicsFamily.has_value() && presentFamily.has_value();
    }
};
```

接下来，我们将修改 `findQueueFamilies` 函数，以查找具有向窗口表面显示能力的队列族。要检查的函数是 `vkGetPhysicalDeviceSurfaceSupportKHR`，它将物理设备、队列族索引和曲面作为参数。在`VK_QUEUE_GRAPHICS_BIT`的同一个循环中添加对它的调用，并存储结果:

```cpp
QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device){
    QueueFamilyIndices indices;
    
    uint32_t queueFamilyCount = 0;
    vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, nullptr);
    
    std::vector<VkQueueFamilyProperties> queueFamilies(queueFamilyCount);
    vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, queueFamilies.data());
    
    int i = 0;
    for(const auto& queueFamily : queueFamilies){
        if(queueFamily.queueFlags & VK_QUEUE_GRAPHICS_BIT){
            indices.graphicsFamily = i;
        }
        
        VkBool32 presentSupport = false;
        vkGetPhysicalDeviceSurfaceSupportKHR(device, i, surface, &presentSupport);
        
        if(presentSupport){
            indices.presentFamily = i;
        }
        
        if(indices.isComplete()){
            break;
        }
        i++;
    }
    return indices;
}
```

在获得结果时，这些队列很可能是相同的队列族，但是在整个程序中，我们将其视为单独的队列，以采取统一的方法。当然，也可以通过逻辑判断选择同一队列同时支持绘图与表示的物理设备，以提高性能

#### 创建表现队列

剩下的一件事是修改逻辑设备创建过程，以创建表示队列并检索 `VkQueue` 句柄。为句柄添加一个成员变量:

```cpp
VkQueue presentQueue;
```

接下来，由于需要多个`VkDeviceQueueCreateInfo`结构来创建来自两个队列族的队列，由于这两个队列可能为一个队列，所以利用`set`来进行去重，并修改 `VkDeviceCreateInfo`来指向`vector`

```cpp
QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

std::vector<VkDeviceQueueCreateInfo> queueCreateInfos;
std::set<uint32_t> uniqueQueueFamiles = {indices.graphicsFamily.value(),indices.presentFamily.value()};

float queuePriority = 1.0f;
for(uint32_t queueFamily : uniqueQueueFamiles){
    VkDeviceQueueCreateInfo queueCreateInfo{};
    queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
    queueCreateInfo.queueFamilyIndex = queueFamily;
    queueCreateInfo.queueCount = 1;
    queueCreateInfo.pQueuePriorities = &queuePriority;
    queueCreateInfos.push_back(queueCreateInfo);
}

VkPhysicalDeviceFeatures deviceFeatures{};

VkDeviceCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO;

createInfo.queueCreateInfoCount = static_cast<uint32_t>(queueCreateInfos.size());
createInfo.pQueueCreateInfos = queueCreateInfos.data();

createInfo.pEnabledFeatures = &deviceFeatures;

createInfo.enabledExtensionCount = 0;
```

最后添加调用来检索队列句柄，若队列族相同，则两个句柄的值也很可能是相同的

```cpp
vkGetDeviceQueue(device, indices.presentFamily.value(), 0, &presentQueue);
```

### Swap chain 交换链

Vulkan 没有“默认帧缓冲区”的概念，因此它需要一个基础设施，这个基础设施拥有我们将提交给它们的缓冲区，然后我们才能在屏幕上可视化它们。这种基础设施称为交换链，必须在 Vulkan 中显式创建。交换链本质上是一个等待显示在屏幕上的图像队列。我们的应用程序从图像队列中获取图像并进行绘制，然后将它返回到队列中。队列的确切工作方式和从队列中显示图像的条件取决于交换链的设置方式，但交换链的一般用途是将图像的显示与屏幕的刷新率同步。

#### Checking for swap chain support 检查交换链支持

由于各种原因，并非所有的图形卡都能够将图像直接显示到屏幕上，例如，有些为服务器设计的图形卡，没有任何显示输出。其次，由于图像表示与窗口系统和与窗口相关的表面密切相关，它实际上并不是 Vulkan 内核的一部分。必须查询 `VK_KHR_swapchain` 设备扩展是否支持后，才可以启用该扩展。
 
 为此，我们将首先扩展 `isDeviceSuitable` 函数，以检查是否支持此扩展。我们之前已经了解了如何列出 `VkPhysicalDevice` 支持的扩展，因此这样做应该相当简单。注意 Vulkan 头文件提供了一个很好的宏 `VK_KHR_SWAPCHAIN_EXTENSION_NAME`，定义为 `VK_KHR_swapchain`。使用这个宏的优点是编译器可以捕获拼写错误。

首先声明所需的设备扩展名列表，类似于要启用的验证层列表。

```cpp
const std::vector<const char*> deviceExtensions = {
    VK_KHR_SWAPCHAIN_EXTENSION_NAME
};
// 设定启用的 Vulkan 设备扩展
```

接下来，创建一个新的函数 `checkDeviceExtensionSupport`，枚举扩展并检查所有必需的扩展是否都在其中，从 `isDeviceSuitable` 调用该函数作为附加检查:

```cpp
bool isDeviceSuitable(VkPhysicalDevice device){
    QueueFamilyIndices indices = findQueueFamilies(device);
    
    bool extensionsSupported = checkDeviceExtensionSupport(device);
    
    return indices.isComplete() && extensionsSupported;
}

bool checkDeviceExtensionSupport(VkPhysicalDevice device){
    uint32_t extensionCount;
    vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, nullptr);
    
    std::vector<VkExtensionProperties> availableExtensions(extensionCount);
    vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, availableExtensions.data());
    
    std::set<std::string> requiredExtensions(deviceExtensions.begin(),deviceExtensions.end());
    
    for(const auto& extension : availableExtensions){
        requiredExtensions.erase(extension.extensionName);
    }
    
    return requiredExtensions.empty();
}
// 检查Vulkan设备扩展可用性
```

这里使用一组字符串来表示未确认的必需扩展。这样我们就可以在列举可用扩展的序列时轻松地发现它们。当然，您也可以使用像 `checkValidationLayerSupport` 中那样的嵌套循环。性能差异在这里是无关紧要的。现在运行代码并验证您的显卡确实能够创建交换链。应该注意的是，正如我们在前一章中检查的那样，表示队列的可用性意味着交换链扩展必须得到支持。但是，对事件显式化总是好的，而且扩展必须显式启用。

#### Enabling device extensions 启用设备扩展

使用交换链首先需要启用 `VK_KHR_swapchain`扩展。启用扩展只需要对逻辑设备创建结构做一个小小的改变:

```cpp
// createLogicalDevice()
createInfo.enabledExtensionCount = static_cast<uint32_t>(deviceExtensions.size());
createInfo.ppEnabledExtensionNames = deviceExtensions.data();
```

当您这样做时，确保替换现有的行 `createInfo.enabledExtensionCount = 0;`

#### Querying details of swap chain support 查询交换链支持的详细信息

仅仅检查交换链是否可用是不够的，因为它实际上可能与我们的窗口表面不兼容。创建交换链还涉及到比实例和设备创建更多的设置，因此在继续之前，我们需要查询更多的细节。

基本上我们需要检查三类属性

- 基本表面功能(交换链中图像的最小/最大数量、图像的最小/最大宽度和高度)
- 表面格式(像素格式，颜色空间)
- 可用的表示模式

与 `findQueueFamilies` 类似，一旦查询了这些细节，我们将使用一个结构传递这些细节。上述三种类型的属性以下列结构和结构列表的形式出现:

```cpp
struct SwapChainSupportDetails{
    VkSurfaceCapabilitiesKHR capabilities;
    std::vector<VkSurfaceFormatKHR> formats;
    std::vector<VkPresentModeKHR> presentModes;
};
```

现在，我们将创建一个新函数 `querySwapChainSupport`，用于填充此结构。

```cpp
SwapChainSupportDetails querySwapChainSupport(VkPhysicalDevice device){
    SwapChainSupportDetails details;
    
    return details;
}
// 获取交换链属性细节
```

本节讨论如何查询包含此信息的结构。下一节将讨论这些结构的含义以及它们究竟包含哪些数据。

让我们从基本的表面能力开始。这些属性易于查询，并返回到单个 `VkSurfaceCapabilitiesKHR` 结构中。

```cpp
vkGetPhysicalDeviceSurfaceCapabilitiesKHR(device, surface, &details.capabilities);
```

这个函数在确定支持的功能时，会考虑指定的 `VkPhysicalDevice` 和 `VkSurfaceKHR` 窗口表面。所有支持查询函数都将这两个作为第一个参数，因为它们是交换链的核心组件。

下一步是查询支持的表面格式。因为这是一个 structs 列表，所以它遵循了熟悉的2个函数调用的惯例:

```cpp
uint32_t formatCount;
vkGetPhysicalDeviceSurfaceFormatsKHR(device, surface, &formatCount, nullptr);

if(formatCount != 0){
    details.formats.resize(formatCount);
    vkGetPhysicalDeviceSurfaceFormatsKHR(device, surface, &formatCount, details.formats.data());
}
```

确保调整vector的大小以保存所有可用的格式。最后，利用`vkGetPhysicalDeviceSurfacePresentModesKHR`使用相同的方式查询支持的表示模式

```cpp
uint32_t presentModeCount;
vkGetPhysicalDeviceSurfacePresentModesKHR(device, surface, &presentModeCount, details.presentModes.data());

if (presentModeCount != 0) {
    details.presentModes.resize(presentModeCount);
    vkGetPhysicalDeviceSurfacePresentModesKHR(device, surface, &presentModeCount, details.presentModes.data());
}
```

现在所有的细节都在结构体中，因此让我们再次扩展 `isDeviceSuitable` 来利用这个函数来验证交换链支持是否足够。如果至少有一种支持的图像格式和一种给定窗口表面的支持的表示模式，对于本次教程，交换链的支持就足够了。

```cpp
bool isDeviceSuitable(VkPhysicalDevice device){
    QueueFamilyIndices indices = findQueueFamilies(device);
    
    bool extensionsSupported = checkDeviceExtensionSupport(device);
    
    bool swapChainAdequate = false;
    if(extensionsSupported){
        SwapChainSupportDetails swapChainSupport = querySwapChainSupport(device);
        swapChainAdequate = !swapChainSupport.formats.empty() && !swapChainSupport.presentModes.empty();
    }
    return indices.isComplete() && extensionsSupported;
}
// 检查设备支持性
```

重要的是，我们只有在验证扩展可用之后才尝试查询交换链支持。函数的最后一行更改为:

```cpp
return indices.isComplete() && extensionsSupported && swapChainAdequate;
```

#### Choosing the right settings for the swap chain 为交换链选择正确的设置

如果`swapChainAdequate`的条件得到满足，那么支持肯定是充分的，但仍然可能有许多不同的最优化模式。现在，我们将编写一些函数来找到最佳交换链的正确设置。有三种类型的设置可以确定:

- Surface format (color depth) 表面格式(颜色深度)
- Presentation mode (conditions for "swapping" images to the screen) 表示模式(“交换”图像到屏幕的条件)
- Swap extent (resolution of images in swap chain) 交换面积(交换链中映像的分辨率)

对于这些设置中的每一个，都有一个理想值，如果它可用的话，我们将使用这个理想值，否则我们将创建一些逻辑来寻找下一个理想值

##### Surface format 表面格式

这个设置的函数是这样开始的。我们将 `SwapChainSupportDetails` 结构的格式成员作为参数传递。

每个 `VkSurfaceFormatKHR` 条目包含一个`format`和一个 `colorSpace` 成员。`format`成员指定颜色通道和类型。例如，`VK_FORMAT_B8G8R8A8_SRGB` 意味着我们分别用一个8位无符号整数存储 R,G,B 和 alpha 通道，总共每像素32位。`colorSpace` 成员指示是否支持 SRGB 颜色空间或不使用 `VK_COLOR_SPACE_SRGB_NONLINEAR_KHR` 标志。注意，在旧版本的规范中，这个标志曾被称为 `VK_COLORSPACE_SRGB_NONLINEAR_KHR`

对于色彩空间，若SRGB可用，我们将使用 SRGB，因为它能更准确的表示颜色。它也几乎是标准的图像色彩空间，就像我们稍后将要使用的纹理。同时我们也应该使用 SRGB 颜色格式，其中最常见的是 `VK_FORMAT_B8G8R8A8_SRGB`。

让我们浏览一下这个列表，看看首选的组合是否可用，若不可用，则可以根据可用格式的优先程度进行排序，但在大多数情况下，只需要使用指定的第一个格式即可

```cpp
VkSurfaceFormatKHR chooseSwapSurfaceFormat(const std::vector<VkSurfaceFormatKHR>& availableFormats){
    for(const auto& availableFormat : availableFormats){
        if(availableFormat.format == VK_FORMAT_B8G8R8_SRGB && availableFormat.colorSpace == VK_COLOR_SPACE_SRGB_NONLINEAR_KHR){
            return availableFormat;
        }
    }
    return availableFormats[0];
}
```

##### Presentation mode 表示模式

表示模式可以说是交换链最重要的设置，因为它代表了向屏幕显示图像的实际条件。Vulkan 有四种可能的模式:

- `VK_PRESENT_MODE_IMMEDIATE_KHR`: 引用程序所递交的影像会即时传送到屏幕上，可能会造成撕裂
- `VK_PRESENT_MODE_FIFO_KHR`:交换链是一个队列，其中显示在刷新显示时从队列的前面获取图像，并且程序在队列的后面插入已渲染的图像。如果队列已满，那么程序必须等待。这与现代游戏中的垂直同步最为相似。刷新显示的时刻称为“垂直空白”
- `VK_PRESENT_MODE_FIFO_RELAXED_KHR`:这个模式与之前一个模式唯一的不同在于，若应用程序延迟且最后一个垂直空白处队列为空，不是等待下一个垂直空白，而是图像到达的瞬间直接传送到屏幕，这可能导致明显的撕裂
- `VK_PRESENT_MODE_MAILBOX_KHR`:这是第二种模式的另一种变体。当队列已满时，不会阻塞应用程序，而是将已经排队的映像简单地替换为更新的映像。这种模式可以用来尽可能快地渲染帧，同时避免撕裂，从而比标准的垂直同步产生更少的延迟问题。这通常被称为“三重缓冲”，尽管三个缓冲区的存在本身并不一定意味着帧速率被解锁

只有 `VK_PRESENT_MODE_FIFO_KHR` 模式保证可用，所以我们需要再次编写一个函数来寻找可用的最佳模式:

```cpp
VkPresentModeKHR chooseSwapPresentMode(const std::vector<VkPresentModeKHR>& availablePresentModes){
    
    return VK_PRESENT_MODE_FIFO_KHR;
}
// 获取表示模式
```

我个人认为，如果能源使用不是一个问题，那么 `VK_PRESENT_MODE_MAILBOX_KHR` 是一个非常好的权衡。通过尽可能在下一个垂直空白前渲染新的图像，以在一个相当低的延迟下避免出现撕裂。在移动设备上，能源使用更为重要，你可能会想要使用 `VK_PRESENT_MODE_FIFO_KHR` 来代替。现在，让我们浏览一下列表，看看 `VK_PRESENT_MODE_MAILBOX_KHR` 是否可用:

```cpp
VkPresentModeKHR chooseSwapPresentMode(const std::vector<VkPresentModeKHR>& availablePresentModes){
    for(const auto& availablePresentMode : availablePresentModes){
        if(availablePresentMode == VK_PRESENT_MODE_MAILBOX_KHR){
            return availablePresentMode;
        }
    }
    return VK_PRESENT_MODE_FIFO_KHR;
}
// 获取表示模式
```

##### Swap extent 交换面积

这样就只剩下一个主要的属性，我们将为它添加最后一个函数:

交换面积是交换链映像的分辨率，它几乎总是精确地等于我们要绘制的窗口的像素分辨率(稍后再详细说明)。可能的分辨率范围在 `VkSurfaceCapabilitiesKHR` 结构中定义。Vulkan 告诉我们通过设置 `currentExtent` 成员中的宽度和高度来匹配窗口的分辨率。然而，一些窗口管理器允许我们在这里有所不同，通过以一些特殊值来设置 `currentExtent` 成员中的宽度和高度:`uint32_t`的最大值。在这种情况下，我们将选择在 minImageExtent 和 maxImageExtent 范围内最匹配窗口的分辨率。但是我们必须用正确的单位指定分辨率。

GLFW 在测量大小时使用两个单位: 像素和屏幕坐标。例如，我们在前面创建窗口时指定的分辨率`{WIDTH, HEIGHT}`时以屏幕坐标为标准。但 Vulkan 使用像素，因此交换链面积也必须以像素为单位指定。不幸的是，如果您使用高 DPI 显示器(如苹果的 Retina 显示器) ，屏幕坐标将不会一一对应像素。相反，由于更高的像素密度，窗口的像素分辨率将大于屏幕坐标分辨率。因此，如果 Vulkan 不能修复我们的交换面积，我们就不能使用原始的`{WIDTH, HEIGHT}`。因此，我们必须使用 `glfwGetFramebufferSize` 来查询窗口的像素分辨率，然后再将其与最小和最大图像范围进行匹配。

```cpp
VkExtent2D chooseSwapExtent(const VkSurfaceCapabilitiesKHR& capabilities){
    if(capabilities.currentExtent.width != UINT32_MAX){
        return capabilities.currentExtent;
    }
    else {
        int width, height;
        glfwGetFramebufferSize(window, &width, &height);
        
        VkExtent2D actualExtent = {
            static_cast<uint32_t>(width),
            static_cast<uint32_t>(height)
        };
        
        actualExtent.width = std::clamp(actualExtent.width, capabilities.maxImageExtent.width, capabilities.maxImageExtent.width);
        actualExtent.height = std::clamp(actualExtent.height, capabilities.minImageExtent.height, capabilities.maxImageExtent.height);
        
        return actualExtent;
    }
}
```

这里使用`clamp`函数将宽度和高度的值绑定到实现支持的允许的最小和最大范围之间。

#### Creating the swap chain 创建交换链

现在我们有了所有这些辅助函数来帮助我们在运行时做出选择，我们终于有了创建一个可工作的交换链所需的所有信息

创建一个 `createSwapChain` 函数，以这些调用的结果开始，并确保在创建逻辑设备后从 `initVulkan` 调用它。

```cpp
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
}
void createSwapChain(){
    SwapChainSupportDetails swapChainSupport = querySwapChainSupport(physicalDevice);
    
    VkSurfaceFormatKHR surfaceFormat = chooseSwapSurfaceFormat(swapChainSupport.formats);
    VkPresentModeKHR presentMode = chooseSwapPresentMode(swapChainSupport.presentModes);
    VkExtent2D extent = chooseSwapExtent(swapChainSupport.capabilities);
    
}
// 创建交换链
```

除了这些属性之外，我们还必须决定在交换链中需要多少图像。该实现指定了它运行所需的最小数量

```cpp
uint32_t imageCount = swapChainSupport.capabilities.minImageCount;
```

然而，简单地坚持这个最小值意味着我们有时不得不等待驱动程序完成内部操作，然后才能获得另一个图像进行渲染。因此，建议请求至少比最低要求多一个图像:

```cpp
uint32_t imageCount = swapChainSupport.capabilities.minImageCount + 1;
```

我们还应该确保在执行此操作时不超过图像的最大数量，其中0是一个特殊值，意味着没有最大值:

```cpp
if(swapChainSupport.capabilities.maxImageCount > 0 && imageCount > swapChainSupport.capabilities.maxImageCount){
    imageCount = swapChainSupport.capabilities.maxImageCount;
}
```

与传统的 Vulkan 对象一样，创建交换链对象需要填充大型结构。在指定交换链应该绑定到哪个表面之后，将指定交换链映像的详细信息:

```cpp
VkSwapchainCreateInfoKHR createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_SWAPCHAIN_CREATE_INFO_KHR;
createInfo.surface = surface;

createInfo.minImageCount = imageCount;
createInfo.imageFormat = surfaceFormat.format;
createInfo.imageColorSpace = surfaceFormat.colorSpace;
createInfo.imageExtent = extent;
createInfo.imageArrayLayers = 1;
createInfo.imageUsage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT;
```

`imageArrayLayers` 指定每个图像包含的图层数量。这总是1，除非你正在开发一个立体的3d 应用程序。`imageUsage` 位字段指定我们将在交换链中使用映像进行哪些操作。在本教程中，我们将直接渲染它们，这意味着它们被用作颜色附件。您还可以首先将图像渲染到单独的图像中，以执行诸如后处理之类的操作。在这种情况下，您可以使用 `VK_IMAGE_USAGE_TRANSFER_DST_BIT` 这样的值，并使用内存操作将渲染的图像转换为交换链图像。

```cpp
QueueFamilyIndices indices = findQueueFamilies(physicalDevice);
uint32_t queueFamilyIndices[] = {indices.graphicsFamily.value(), indices.presentFamily.value()};

if(indices.graphicsFamily != indices.presentFamily){
    createInfo.imageSharingMode = VK_SHARING_MODE_CONCURRENT;
    createInfo.queueFamilyIndexCount = 2;
    createInfo.pQueueFamilyIndices = queueFamilyIndices;
} // 并发模式
else {
    createInfo.imageSharingMode = VK_SHARING_MODE_EXCLUSIVE;
    createInfo.queueFamilyIndexCount = 0;
    createInfo.pQueueFamilyIndices = nullptr;
} // 独占模式
}
```

接下来，我们需要指定如何处理将跨多个队列族使用的交换链映像。如果图形队列家族与表示队列不同，那么在我们的应用程序中就会出现这种情况。我们将从图形队列中绘制交换链中的图像，然后在表示队列中提交它们。有两种方法可以处理从多个队列访问的图像:

- `VK_SHARING_MODE_EXCLUSIVE` 图像每次由一个队列族拥有，在将其用于另一个队列族之前，必须明确转让其所有权。这个选项提供了最好的性能
- `VK_SHARING_MODE_CONCURRENT` 图片可以跨多个队列族使用，不需要明确的所有权转移

如果队列家族不同，那么我们将在本教程中使用并发模式，以避免执行所有权章节，因为这些涉及到一些概念，稍后将更好地解释。并发模式要求您提前指定使用 `queueFamilyIndexCount` 和 `pQueueFamilyIndices` 参数在哪些队列家族所有权之间共享。如果图形队列族和表示队列族是相同的，这在大多数硬件上都是如此，那么我们应该坚持独占模式，因为并发模式要求您指定至少两个不同的队列族。

```cpp
createInfo.preTransform = swapChainSupport.capabilities.currentTransform;
```

我们可以指定一个特定的转换应该应用于交换链中的图像，如果它被支持的话(由`capabilities`中的 `supportedTransforms`属性表示) ，比如90度顺时针旋转或水平翻转。若不需要任何的转换，那么只需要指定为当前转换

```cpp
createInfo.compositeAlpha = VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR;
```

`compositeAlpha` 字段指定是否应该使用 alpha 通道与窗口系统中的其他窗口混合。你几乎总是希望简单地忽略 alpha 通道，因此设定为 `VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR`。

```cpp
createInfo.presentMode = presentMode;
createInfo.clipped = VK_TRUE;
```

`presentMode`成员的意思就是它本身的意思，它的值就是上文获取的值，若将`clipped`成员的属性设置为`VK_TRUE`,那么就意味着我们不关心被遮挡的像素的颜色，例如，因为另一个窗口在它们的前面。除非您真的需要读取这些像素并得到可预测的结果，可以通过启用剪切获得最佳性能。

```cpp
createInfo.oldSwapchain = VK_NULL_HANDLE;
```

最后一个属性`oldSwapChain`。使用 Vulkan 时，在应用程序运行时，交换链可能变得无效或未优化，例如，因为窗口的大小被调整了。在这种情况下，实际上需要从头开始重新创建交换链，并且必须在该字段中指定对旧交换链的引用。这是一个复杂的主题，我们将在以后的章节中学习更多。现在我们假设我们只创建一个交换链。

现在添加一个类成员来存储 `VkSwapchainKHR` 对象，创建交换链现在只需要调用`vkCreateSwapchainKHR`

```cpp
// class
VkSwapchainKHR swapChain;

...
// createSwapChain
if(vkCreateSwapchainKHR(device, &createInfo, nullptr, &swapChain) != VK_SUCCESS){
    throw std::runtime_error("failed to create swap chain!");
}
```

这些参数包括逻辑设备、交换链创建信息、可选的自定义分配器和一个指向用于存储句柄的变量的指针。在清理设备前，应该使用 `vkDestroySwapchainKHR` 清理交换链:

```cpp
// cleanup
vkDestroySwapchainKHR(device, swapChain, nullptr);
vkDestroyDevice(device, nullptr);
```

现在运行应用程序以确保成功创建交换链！如果此时你在 `vkCreateSwapchainKHR` 中出现了访问冲突错误，或者看到`Failed to find 'vkGetInstanceProcAddress' in layer SteamOverlayVulkanLayer.dll`这样的消息，那么请查看关于蒸汽覆盖层的 [FAQ 条目](https://vulkan-tutorial.com/FAQ)。

尝试删除 `createInfo.imageExtent = extent;` 在启用验证层的情况下运行。您将看到其中一个验证层立即捕捉到错误，并打印出一条有用的信息:

```shell
validation layer: Validation Error: [ VUID-VkSwapchainCreateInfoKHR-imageExtent-01274 ] Object 0: handle = 0x10605a218, type = VK_OBJECT_TYPE_DEVICE; | MessageID = 0x7cd0911d | vkCreateSwapchainKHR() called with imageExtent = (0,0), which is outside the bounds returned by vkGetPhysicalDeviceSurfaceCapabilitiesKHR(): currentExtent = (1600,1200), minImageExtent = (1,1), maxImageExtent = (16384,16384). The Vulkan spec states: imageExtent must be between minImageExtent and maxImageExtent, inclusive, where minImageExtent and maxImageExtent are members of the VkSurfaceCapabilitiesKHR structure returned by vkGetPhysicalDeviceSurfaceCapabilitiesKHR for the surface (https://vulkan.lunarg.com/doc/view/1.2.176.1/mac/1.2-extensions/vkspec.html#VUID-VkSwapchainCreateInfoKHR-imageExtent-01274)
```

#### Retrieving the swap chain images 检索交换链图像

现在已经创建了交换链，因此剩下的工作就是检索其中的 `VkImages` 的句柄。在后面的章节中，我们将在渲染操作中引用这些。添加一个类成员来存储句柄:

```cpp
std::vector<VkImage> swapChainImages;
// 存储VkImages句柄
```

这些映像是由交换链的实现创建的，一旦交换链被销毁，它们将自动清理，因此我们不需要添加任何清理代码。

现在添加代码来检索 `createSwapChain` 函数末尾的句柄，就在 `vkCreateSwapchainKHR` 调用之后。检索它们与我们从 Vulkan 检索一个对象数组的其他时间非常相似。请记住，我们只在交换链中指定了最小数量的映像，因此交换链可能使用比我们定义更多的映像。这就是为什么我们将首先使用 `vkGetSwapchainImagesKHR` 查询最终的图像数量，然后调整容器的大小，最后再次调用它来检索句柄。

```cpp
vkGetSwapchainImagesKHR(device, swapChain, &imageCount, nullptr);
swapChainImages.resize(imageCount);
vkGetSwapchainImagesKHR(device, swapChain, &imageCount, swapChainImages.data());
```

最后一件事，将我们为交换链映像选择的格式和范围存储在成员变量中。我们在以后的章节里会需要它们

```cpp
// class
VkFormat swapChainImageFormat;
VkExtent2D swapChainExtent;

// createSwapChain
swapChainImageFormat = surfaceFormat.format;
swapChainExtent = extent;
```

我们现在有了一组图像，可以绘制到窗口上，并可以显示。下一章将开始介绍如何将图像设置为渲染目标，然后我们开始查看实际的渲染绘图管线和绘制命令！


