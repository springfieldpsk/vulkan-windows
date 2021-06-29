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

Vulkan 没有“默认帧缓冲”的概念，因此它需要一个基础设施，这个基础设施拥有我们将提交给它们的缓冲区，然后我们才能在屏幕上可视化它们。这种基础设施称为交换链，必须在 Vulkan 中显式创建。交换链本质上是一个等待显示在屏幕上的图像队列。我们的应用程序从图像队列中获取图像并进行绘制，然后将它返回到队列中。队列的确切工作方式和从队列中显示图像的条件取决于交换链的设置方式，但交换链的一般用途是将图像的显示与屏幕的刷新率同步。

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

### Image views 图像视图

要使用任何的`VkImage`,包括交换链中的 VkImage，我们必须在渲染管道中创建一个 `VkImageView` 对象,图像视图实际上就是对图像的视图。它描述了如何访问图像和访问图像的哪一部分，举个例子，它应该被视为没有任何 mipmapping 级别的二维纹理深度纹理。

在本章中，我们将编写一个 `createImageViews` 函数，它为交换链中的每个图像创建一个基本的图像视图，以便我们以后可以将它们用作颜色目标。

首先添加一个类成员来存储图像视图:

```cpp
std::vector<VkImageView> swapChainImageViews;
// 图像视图
```

创建 `createImageViews` 函数，并在交换链创建后立即调用它。

```cpp
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
}

void createImageViews() {

}
```

我们需要做的第一件事是调整列表的大小，以适应我们将要创建的所有图像视图:

```cpp
swapChainImageViews.resize(swapChainImages.size());
```

接下来，设置遍历所有交换链映像的循环。

```cpp
for(size_t i = 0;i < swapChainImages.size(); i ++ ){
    
}
```

创建图像视图的参数在 VkImageViewCreateInfo 结构中指定。

```cpp
VkImageViewCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
createInfo.image = swapChainImages[i];
```

`viewType` 和 `format` 字段指定应该如何解释图像数据。`viewType` 参数允许您将图像视为一维纹理、二维纹理、三维纹理和立方体映射。

```cpp
createInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
createInfo.format = swapChainImageFormat;
```

`components`字段允许你调整周围的颜色通道。例如，您可以将所有通道映射到红色通道以获得单色纹理。您还可以将常量值0和1映射到通道。在我们的示例中，我们将坚持使用默认映射。

```cpp
createInfo.components.r = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.g = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.b = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.a = VK_COMPONENT_SWIZZLE_IDENTITY;
```

`subresourceRange`字段描述了图像的用途以及应该访问图像的哪一部分。我们的图像将被视作没有任何mipmapping等级或多图层的彩色目标。

```cpp
createInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
createInfo.subresourceRange.baseMipLevel = 0;
createInfo.subresourceRange.layerCount = 1;
createInfo.subresourceRange.baseArrayLayer = 0;
createInfo.subresourceRange.levelCount = 1;
```

如果您正在开发立体图形3D应用程序，那么您将创建一个具有多个层的交换链。然后，您可以通过访问不同的图层，为每个表示左右眼视图的图像创建多个图像视图。

现在通过`vkCreateImageView`创建图像视图

```cpp
if(vkCreateImageView(device, &createInfo, nullptr, &swapChainImageViews[i]) != VK_SUCCESS){
    throw std::runtime_error("failed to create image views!");
}
```

与映像不同，图像视图是由我们显式创建的，因此我们需要在程序结束时添加一个类似的循环来再次销毁它们:

```cpp
void cleanup() {
    for(auto imageView : swapChainImageViews){
        vkDestroyImageView(device, imageView, nullptr);
    }
    ...
}
```

一个图像视图足以开始使用一个图像作为纹理，但是它还没有完全准备好用作渲染目标。这需要更多的间接步骤，称为 帧缓冲。但是首先，我们必须建立渲染管线。

## Graphics pipeline basics 渲染管线基础

### Introduction 引言

在接下来的几章中，我们将设置一个渲染管线，用于绘制第一个三角形。渲染管线是一系列的操作，通过这一系列操作使得网格上的顶点和纹理转化为渲染目标中的像素。以下是一个简化的概述:

![Vulkan_pipeline](vulkan_simplified_pipeline.svg)

The input assembler collects the raw vertex data from the buffers you specify and may also use an index buffer to repeat certain elements without having to duplicate the vertex data itself.

输入装配器从您指定的缓冲区中获取原始顶点数据，并且可以使用索引缓冲区来重复某些元素，而无需复制顶点数据本身。（个人认为这段的意思为可以通过索引缓冲区避免出现顶点数据的重复复制）

顶点数据载入后，通过顶点着色器渲染每个顶点，通常在这个阶段会将顶点位置从模型空间转换到屏幕空间，这个阶段处理结束的顶点数据将会沿管道传递到下一个阶段

下个阶段是镶嵌着色器，镶嵌着色器允许你基于某些规则进行几何细分以提升网格的质量，这通常用于细化砖墙或楼梯等的表面，以使得摄像机在这些表面附近时，看起来有更多的起伏

镶嵌着色器的数据经过处理将会传递到几何着色器，几何着色器在每个图元(三角形、直线、点)上运行，可以放弃这个阶段或者通过几何着色器输出比原始数据更多的图元。这个过程与镶嵌着色器类似，但是更加灵活。然而，它在今天的应用程序中并没有被广泛使用，因为除了英特尔的集成 gpu 之外，几何着色器在大多数显卡上的表现都不是很好。

光栅化阶段会将上个阶段处理好的图元离散成片段。这些片段填充帧缓冲中像素元素。任何位于屏幕范围之外的片段都会被丢弃，如图所示，顶点着色器输出的属性将会被插入片段中。通常情况下，那些被其他片段所覆盖的片段会因为深度测试而丢弃

所有没有被放弃的片段将会传入片段着色器中，在这个阶段，片段着色器会确定每个片段的颜色、深度值以及写入那个帧缓冲中。这些数据可以通过顶点着色器获得的结果插值得到，这些数据包括纹理坐标和用于光照计算的法线

颜色混合阶段将会混合帧缓冲中相同像素的不同片段。片段可以简单的相互覆盖、叠加或者根据透明度混合

在上图中，绿色的阶段被称为固定功能阶段，这些阶段可以通过调整参数来调整实际的操作，但这些阶段的工作方式是预定义的

橙色阶段是可编程的，这意味着可以将自己的代码上传到显卡上，从而精确的实现想要的操作。以片段着色器为例，通过对片段着色器的编程可以实现从纹理和光照到射线追踪器的任何东西。这些程序运行在多个GPU核心上，并行处理多个对象，比如并行的顶点和片段

如果您以前使用过像 `OpenGL` 和 `Direct3D` 这样的老 api，那么您将习惯于可以随意更改像 `glBlendFunc` 和 `OMSetBlendState` 这样的调用的任何管道设置。Vulkan 中的绘图管线几乎是不可改变的，所以如果你想改变着色器，绑定不同的帧缓冲或者改变混合函数，你必须从头开始重新创建管道。缺点是，您必须创建许多管道，它们代表您想要在呈现操作中使用的所有不同状态组合。但是，由于您将在管道中执行的所有操作都是事先知道的，因此驱动程序可以更好地优化它。

一些可编程的阶段是可选的，基于你打算做什么。例如，如果你只是绘制简单的几何,那么镶嵌和几何阶段可以禁用。如果您只对深度值感兴趣，那么您可以禁用片段着色阶段，这对阴影贴图生成非常有用。

在下一章，由于我们需要把一个三角形显示到屏幕上，所以将首先创建两个可编程阶段: 顶点着色器和片段着色器。固定功能的配置，如混合模式，视口，光栅化将在后面的章节中设置。在 Vulkan 中设置渲染管线的最后一部分将会涉及输入和输出帧缓冲器的规范。

创建一个 `createGraphicsPipeline` 函数，该函数在 `initVulkan` 中的 `createImageViews` 之后被调用。我们将在下面的章节中讨论这个函数。

```cpp
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
    createGraphicsPipeline();
}

...

void createGraphicsPipeline() {

}
```

### Shader modules 着色器模块

与早期的 api 不同，Vulkan 的着色器代码必须以字节码格式指定，而不是像 GLSL 和 HLSL 这样的人类可读的语法。这种字节码格式称为 SPIR-V，设计用于 Vulkan 和 OpenCL (两者都是 Khronos api)。这是一种可以用来编写图形和计算着色器的格式，但是我们将在本教程中集中讨论 Vulkan 图形管道中使用的着色器。

使用字节码格式的好处是，GPU 厂商编写的将着色代码转换为本机代码的编译器明显没有那么复杂。过去的经验表明，使用像 GLSL 这样人类可读的语法，一些 GPU 厂商在解释标准时相当灵活。如果你碰巧用其中一个厂商的 GPU 编写了非常重要的着色器，那么你将面临其他厂商的驱动程序因语法错误而拒绝你的代码的风险，或者更糟糕的是，你的着色器因为编译器错误而运行不同。使用简单的字节码格式，如 SPIR-V，希望能够避免这种情况。

但是，这并不意味着我们需要手工编写这个字节码。Khronos 发布了自己的独立于供应商的编译器，该编译器将 GLSL 编译为 SPIR-V。这个编译器被设计用来验证你的着色器代码是否完全符合标准，并且生成一个 SPIR-V 二进制文件，你可以在你的程序中使用它。您还可以将这个编译器包含为一个库，以便在运行时生成 SPIR-V，但在本教程中我们不会这样做。虽然我们可以通过 `glslangValidator.exe` 直接使用这个编译器，但是我们将使用 Google 的 `glslc.exe`。`Glslc` 的优点是，它使用与 `GCC` 和 `Clang` 等著名编译器相同的参数格式，并包含一些额外的功能，如 `includes`。它们都已经包含在 Vulkan SDK 中，因此您不需要下载任何额外的内容。

GLSL 是一个 c 式语法的着色语言。用它编写的程序有一个`main`函数，该函数可以调用所有的调用。GLSL 使用全局变量处理输入和输出，而不是使用参数作为输入和返回值作为输出。该语言包括许多有助于图形编程的特性，如内置矢量和矩阵原语。函数的运算，如交叉积，矩阵向量积和反射周围的一个向量包括在内。向量类型称为 `vec`，数字表示元素的数量。例如，一个3D 位置将被存储在一个 `vec3` 中。可以通过`.x`等成员访问单个组件。但是也可以同时从多个分量创建一个新的矢量。例如，表达式 `vec3(1.0, 2.0, 3.0).xy` 会生成 `vec2`。矢量的构造函数还可以组合矢量对象和标量值。例如，可以用`vec3(vec2(1.0, 2.0), 3.0)`构造 `vec3`。

正如前面提到的，我们需要编写一个顶点着色器和一个片段着色器来在屏幕上得到一个三角形。接下来的两个部分将介绍每个代码的 GLSL 代码，之后我将向您展示如何生成两个 SPIR-V 二进制文件并将它们加载到程序中。

#### Vertex shader 顶点着色器

顶点着色器处理每个传入的顶点。它以世界位置、颜色、法线和纹理坐标等属性作为输入。输出是在剪辑坐标系中的最终位置，以及需要传递给片段着色器的属性，如颜色和纹理坐标。然后这些值将被光栅化器插值到片段上以产生一个平滑的梯度。

剪辑坐标系是来自顶点着色器的四维向量，随后通过将整个向量除以其最后一个分量，将其转换为规范化的设备坐标系。这些标准化的设备坐标系将帧缓冲映射到[-1,1] by [-1,1]坐标系的标准化设备坐标，如下所示:

![device_coordinate](normalized_device_coordinates.svg)

如果你曾经涉足计算机图形学，你应该已经熟悉这些。如果您以前使用过 OpenGL，那么您将注意到 y 坐标的符号现在已经颠倒了。Z 坐标现在使用与 Direct3D 相同的范围，从0到1。

对于我们的第一个三角形，我们不会应用任何变换，我们只需直接指定三个顶点的位置作为规范化设备坐标创建形状

我们可以通过将它们输出为根据顶点着色器的剪辑坐标系，并将最后一个分量设置为1的方式，直接输出规范化的设备坐标。这种情况下，从剪辑坐标系到标准化设备坐标系的过程不会出现任何改变

通常这些坐标会被存储在一个顶点缓冲区中，但是在 Vulkan 中创建一个顶点缓冲区并用数据填充它不是一件简单的事。因此，我们将决定推迟这个操作，直到我们看到屏幕上弹出一个三角形的满足感。因此，我们将要做一些有点非传统的事情: 直接在顶点着色器中写入坐标。代码是这样的:

```cpp
#version 450

vec2 position[3] = vec2[](
     vec2(0.0, -0.5),
     vec2(0.5, 0.5),
     vec2(-0.5, 0.5)
);

void main(){
    gl_Position = vec4(position[gl_VertexIndex], 0.0, 1.0);
}
```

对每个顶点调用 `main` 函数。内置的 `gl_VertexIndex` 变量包含当前顶点的索引。这通常是顶点缓冲区的索引，但是在我们的例子中，它将是顶点数据硬编码数组的索引。每个顶点的位置是通过着色器中的常量数组访问的，并与虚拟 z 和 w 组件结合以产生剪辑坐标中的位置。内置的变量 `gl_Position` 作为输出。

#### Fragment shader 片段着色器

由顶点着色器的位置构成的三角形用片段填充屏幕上的一个区域。在这些片段上调用片段着色器以为 framebuffer (或 framebuffers)生成颜色和深度。这里用一个简单的片段着色器，输出一个由红色填充的三角形，如下所示:

```cpp
#version 450

layout(location = 0)out vec4 outColor;

void main(){
    outColor = vec4(1.0, 0.0, 0.0, 1.0);
}
```

对每个片段调用 `main` 函数，就像对每个顶点调用顶点着色器 `main` 函数一样。GLSL 中的颜色是四分量向量，r，g，b 和 alpha 通道在[0,1]范围内。与 `gl_Position` 在顶点着色器中不同，没有为当前片段输出颜色的内置变量。您必须为每个帧缓冲指定自己的输出变量，其中`layout(location = 0)`修饰符指定帧缓冲的索引。将红色写入到这个 `outColor` 变量，该变量链接到索引0处，也就是第一个(也是唯一的) 帧缓冲。

#### Per-vertex colors 每顶点颜色

仅仅只是用红色填充整个三角形也许不是很有趣，为什么不考虑为每个顶点设置颜色呢

我们必须对两个着色器做一些修改来实现这一点。首先，我们需要为三个顶点分别指定不同的颜色。顶点着色器现在应该包含一个颜色数组，就像它在位置上所做的那样:

```cpp
vec3 colors[3] = vec3[](
     vec3(1.0, 0.0, 0.0),
     vec3(0.0, 1.0, 0.0),
     vec3(0.0, 0.0, 1.0)
);
```

现在我们只需要将这些每个顶点的颜色传递给片段着色器，这样它就可以将它们的插值值输出到 帧缓冲。为顶点着色器添加一个颜色输出，并在 `main` 函数中写入:

```cpp
layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
    fragColor = colors[gl_VertexIndex];
}
```
接下来，我们需要在片段着色器中添加匹配的输入:

```cpp
layout(location = 0) in vec3 fragColor;

void main() {
    outColor = vec4(fragColor, 1.0);
}
```

输入变量不一定要使用相同的名称，它们将使用 `location` 指令指定的索引链接在一起。主函数已经被修改为输出颜色和 alpha 值。如上图所示，`fragColor` 的值将为三个顶点之间的片段自动插值，从而得到一个平滑的梯度。

#### Compiling the shaders 编译着色器

在项目的根目录中创建一个名为 `shaders` 的目录，并将顶点着色器存储在一个名为 `shader.vert` 的文件中，将片段着色器存储在该目录中一个名为 `shader.frag` 的文件中。GLSL 着色器没有官方的扩展，但是这两个通常被用来区分它们。

`shader.vert` 的内容应该是:

```cpp
#version 450

layout(location = 0) out vec3 fragColor;

vec2 position[3] = vec2[](
     vec2(0.0, -0.5),
     vec2(0.5, 0.5),
     vec2(-0.5, 0.5)
);

vec3 colors[3] = vec3[](
     vec3(1.0, 0.0, 0.0),
     vec3(0.0, 1.0, 0.0),
     vec3(0.0, 0.0, 1.0)
);

void main(){
    gl_Position = vec4(position[gl_VertexIndex], 0.0, 1.0);
    fragColor = colors[gl_VertexIndex];
}
```

`shader.frag`  的内容应该是:

```cpp
#version 450

layout(location = 0) in vec3 fragColor;

layout(location = 0) out vec3 outColor;

void main(){
    outColor = vec4(fragColor, 1.0);
}
```

现在，我们将使用 `glslc` 程序将这些代码编译成 SPIR-V 字节码。

建立一个当前系统下的脚本文件，windows下使用bat，linux与macos使用sh，填入以下代码

```shell
# sh
/home/user/VulkanSDK/x.x.x.x/x86_64/bin/glslc shader.vert -o vert.spv
/home/user/VulkanSDK/x.x.x.x/x86_64/bin/glslc shader.frag -o frag.spv

# bat
C:/VulkanSDK/x.x.x.x/Bin32/glslc.exe shader.vert -o vert.spv
C:/VulkanSDK/x.x.x.x/Bin32/glslc.exe shader.frag -o frag.spv
pause
```

将 glslc.exe 的路径替换为您安装 Vulkan SDK 的路径。windows下双击该文件以运行它，Linux与macOS下使用 chmod + x compile.sh 使脚本可执行并运行它。

这两个命令告诉编译器读取 GLSL 源文件，并使用-o (输出)标志输出一个 SPIR-V 字节码文件。

如果你的着色器包含一个语法错误，那么编译器会告诉你行号和问题，正如你所期望的那样。例如，尝试省略分号，然后再次运行编译脚本。还可以尝试运行没有任何参数的编译器，以查看它支持哪些类型的标志。例如，它还可以将字节码输出为人类可读的格式，这样您就可以确切地看到着色程序在做什么，以及在这个阶段应用的任何优化。

在命令行上编译着色器是最简单的选项之一，也是我们将在本教程中使用的选项，但是也可以直接从自己的代码中编译着色器。Vulkan SDK 包括 libshadec，它是一个库，用于从程序内部将 GLSL 代码编译为 SPIR-V。

#### Loading a shader 加载着色器

现在我们已经有了生成 SPIR-V 着色器的方法，是时候将它们加载到我们的程序中，以便在某个时候将它们插入渲染管线。我们将首先编写一个简单的 helper 函数来从文件中加载二进制数据。

```cpp
#include <fstream>

...
// class
static std::vector<char> readFile(const std::string& filename) {
    std::ifstream file(filename, std::ios::ate | std::ios::binary);

    if (!file.is_open()) {
        throw std::runtime_error("failed to open file!");
    }
}
```

readFile 函数将读取指定文件中的所有字节，并将它们返回到由 std: : vector 管理的字节数组中。我们首先用两个标志打开文件:

- `ate` 从文件末尾开始读取
- `binary` 以二进制方式读取(避免文本转换)

从文件末尾开始读取的好处是，我们可以使用读取位置来确定文件的大小并分配一个缓冲区:

```cpp
size_t fileSize = (size_t) file.tellg();
std::vector<char> buffer(fileSize);
```

之后，我们可以返回到文件的开头，并一次读取所有字节，读取结束后关闭文件并返回字节:

```cpp
file.seekg(0);
file.read(buffer.data(), fileSize);
file.close();

return buffer;
```

现在，我们从 `createGraphicsPipeline` 中调用这个函数来加载两个着色器的字节码:

```cpp
void createGraphicsPipeline(){
    auto vertShaderCode = readFile("shaders/vert.spv");
    auto fragShaderCode = readFile("shaders/frag.spv");
    
}
```

通过打印缓冲区的大小并检查它们是否与实际的文件大小(以字节为单位)匹配，确保着色器正确加载。请注意，代码不需要为空终止，因为它是二进制代码，我们稍后将明确其大小。

#### Creating shader modules 创建着色器模块

在将代码传递给管道之前，我们必须将其封装在 `VkShaderModule` 对象中。让我们创建一个辅助函数 `createShaderModule` 来实现这一点。

```cpp
VkShaderModule createShaderModule(const std::vector<char>& code){
    
}
// 创建着色器模块
```

该函数将以字节码作为参数获取一个缓冲区，并从中创建一个 `VkShaderModule`。

创建一个着色器模块很简单，我们只需要用字节码和它的长度指定一个指向缓冲区的指针。这些信息是在`VkShaderModuleCreateInfo`结构中指定的。其中一个问题是，字节码的大小是以字节为单位指定的，但是字节码指针是一个`uint32_t`指针，而不是一个`char`指针。因此，我们需要使用`reinterpret_cast`来强制转换指针。当您执行这样的强制转换时，还需要确保数据满足`uint32_t` 的对齐要求。幸运的是，数据存储在 `std::vector`中，其中默认分配器已经确保数据满足最坏情况下的对齐要求。

```cpp
VkShaderModuleCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO;
createInfo.codeSize = code.size();
createInfo.pCode = reinterpret_cast<const uint32_t*>(code.data());
```

可以通过调用 `vkCreateShaderModule` 来创建 `VkShaderModule`:

```cpp
VkShaderModule shaderModule;
if(vkCreateShaderModule(device, &createInfo, nullptr, &shaderModule) != VK_SUCCESS){
    throw std::runtime_error("failed to create shader module!");
}
```

这些参数与之前的对象创建函数相同: 逻辑设备、创建信息结构的指针、自定义分配器的可选指针和处理输出变量。带有字节码的缓冲区可以在创建着色器模块后立即释放。不要忘记返回创建的着色器模块:

```cpp
return shaderModule;
```

着色器模块只是我们之前从文件中加载的着色器字节码和其中定义的函数的一个简单包装器。渲染管线的创建完成后， SPIR-V 字节码将会编译和链接成以供GPU执行的机器代码。这意味着一旦管道创建完成，我们就可以再次销毁着色器模块，这就是为什么我们在 createGraphicsPipeline 函数中将它们设置为局部变量，而不是类成员:

然后，通过向 `vkDestroyShaderModule` 添加两个调用，清理应该发生在函数的末尾。本章中剩下的所有代码都将插入到这些行之前。

```cpp
...
vkDestroyShaderModule(device, fragShaderModule, nullptr);
vkDestroyShaderModule(device, vertShaderModule, nullptr);
```

#### Shader stage creation 创建着色器阶段

要实际使用着色器，我们需要通过 `VkPipelineShaderStageCreateInfo` 结构将着色器分配到特定的管道级，作为实际管道创建过程的一部分。

我们将首先填充顶点着色器的结构，同样是在 `createGraphicsPipeline` 函数中。

```cpp
VkPipelineShaderStageCreateInfo vertShaderStageInfo{};
vertShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
vertShaderStageInfo.stage = VK_SHADER_STAGE_VERTEX_BIT;
```

第一步，除了必须的 sType 成员，是告诉 Vulkan 在哪个管道阶段着色器将被使用。与之前章节中描述的每个可编程阶段一样，是一个 enum 值。

```cpp
vertShaderStageInfo.module = vertShaderModule;
vertShaderStageInfo.pName = "main";
```

接下来的两个成员指定包含代码的着色器模块和要调用的函数，称为入口点。这意味着可以将多个着色器片段组合成单个着色器模块，并使用不同的入口点来区分它们的行为。然而，在这种情况下，我们将坚持使用标准的 main。

还有一个(可选)成员，`pSpecializationInfo`，我们不会在这里使用，但是值得讨论。它允许您为着色器常量指定值。您可以使用单个着色器模块，通过为其中使用的常量指定不同的值，可以在管道创建时配置其行为。这比在渲染时使用变量配置着色器更有效率，因为编译器可以进行优化，比如消除依赖于这些值的 if 语句。如果没有这样的常量，那么可以将成员设置为 nullptr，这是我们的结构初始化时自动完成的。

修改结构体以适应片段着色器是很容易的:

```cpp
VkPipelineShaderStageCreateInfo fragShaderStageInfo{};
fragShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
fragShaderStageInfo.stage = VK_SHADER_STAGE_FRAGMENT_BIT;
fragShaderStageInfo.module = fragShaderModule;
fragShaderStageInfo.pName = "main";
```

最后定义一个包含这两个结构的数组，稍后我们将在实际的管道创建步骤中使用该数组来引用它们

```cpp
VkPipelineShaderStageCreateInfo shaderStages [] = {
    vertShaderStageInfo,fragShaderStageInfo
};
```

描述管道的可编程阶段的全部内容。在下一章中，我们将研究固定函数阶段。

### Fixed functions 固定函数

旧的图形 api 为绘图管线的大部分阶段提供了默认状态。在 Vulkan 你必须明确的一切，从视口大小到颜色混合功能。在本章中，我们将填充所有的结构来配置这些固定函数操作。

#### Vertex input 顶点输入

`VkPipelineVertexInputStateCreateInfo` 结构描述将传递给顶点着色器的顶点数据的格式。它大致从两个方面描述了格式

- 绑定： 数据之间的间距以及数据为逐顶点或逐实例(详见[instancing](https://en.wikipedia.org/wiki/Geometry_instancing))
- 属性描述：传递给顶点着色器的属性类型，绑定从哪个偏移量加载它们

因为我们很难直接在顶点着色器中对顶点数据进行编码，所以我们将填充这个结构来指定现在没有顶点数据可以加载。我们将在顶点缓冲区一章再考虑它。

```cpp
VkPipelineVertexInputStateCreateInfo vertexInputInfo{};
vertexInputInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO;
vertexInputInfo.vertexBindingDescriptionCount = 0;
vertexInputInfo.pVertexBindingDescriptions = nullptr;
vertexInputInfo.vertexAttributeDescriptionCount = 0;
vertexInputInfo.pVertexAttributeDescriptions = nullptr;
```

`pVertexBindingDescriptions` 和 `pVertexAttributeDescriptions` 成员指向一个结构数组，该结构数组描述了家在顶点数据的上述细节，在`shaderStages`数组之后将这个结构添加到 `createGraphicsPipeline` 函数中。

#### Input assembly 输入汇编

`VkPipelineInputAssemblyStateCreateInfo` 结构描述了两件事: 从顶点绘制何种几何图形，以及是否应该启用`primitiveRestart`。前者是在`topology`成员中指定的，可以具有如下值

- `VK_PRIMITIVE_TOPOLOGY_POINT_LIST`：根据顶点绘制点
- `VK_PRIMITIVE_TOPOLOGY_LINE_LIST`：在不重复使用的情况下每两个顶点绘制一条线
- `VK_PRIMITIVE_TOPOLOGY_LINE_STRIP`：每条线的最后一个顶点作为下个线的开始顶点
- `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST`： 在不重复使用的情况下每三个顶点绘制一个三角形
- `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_STRIP`：每个三角形的第二个顶点和第三个顶点用作下一个三角形的前两个顶点

通常，顶点是按照索引顺序从顶点缓冲区加载的，但是如果使用元素缓冲区，您可以指定自己使用的索引。这允许您执行诸如重用顶点之类的优化。如果您将 `primitiveRestartEnable` 成员设置为 `VK_TRUE`，那么可以使用特殊的索引`0xFFFF` 或 `0xFFFFFFFF` 来分解 `_STRIP`拓扑模式中的线和三角形。

我们打算在整个教程中绘制三角形，所以我们将坚持以下数据的结构:

#### Viewports and scissors 视口和裁剪

视口基本上描述了要渲染值帧缓冲的区域。这总会是`(0, 0)` 到 `(width, height)`，在本教程中也是如此。

请记住，交换链及其映像的大小可能与窗口的 `WIDTH` 和 `HEIGHT` 不同。交换链映像稍后将用作帧缓冲，因此我们应该坚持使用它们的大小。

`minDepth` 和 `maxDepth` 值指定用于帧缓冲的深度值范围。这些值必须在`[0.0f, 1.0f]`范围内，但 `minDepth` 可能高于 `maxDepth`。如果你没有特别的需求的话，那么你应该坚持使用`0.0f` 和 `1.0f`的标准值

视口定义了从图像到帧缓冲的转换，裁剪矩阵定义了实际存储像素的区域。裁剪矩阵外面的任何像素都将被光栅化器丢弃。它们的功能就像一个过滤器，而不是一个转换器。区别如下图所示。注意，左边的裁剪矩阵只是许多可能的结果之一，只要它大于视口。

![viewports_scissors](viewports_scissors.png)

在本教程中，我们只是想绘制整个帧缓冲，所以我们将指定一个裁剪矩阵完全覆盖它:

```cpp
VkRect2D scissor{};
scissor.offset = {0, 0};
scissor.extent = swapChainExtent;
```

现在，需要使用`VkPipelineViewportStateCreateInfo`结构将这个视口和裁剪矩形结合成一个视口状态。在一些显卡上可以使用多个视口和裁剪矩形，这样它的成员就可以引用它们的一个数组。使用多重需要启用 GPU 特性(请参阅创建逻辑设备)

```cpp
VkPipelineViewportStateCreateInfo viewportState{};
viewportState.sType = VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO;
viewportState.viewportCount = 1;
viewportState.pViewports = &viewport;
viewportState.scissorCount = 1;
viewportState.pScissors = &scissor;
```

#### Rasterizer 光栅化器

光栅化器采用的几何形状是由顶点着色器的顶点形成，并把它变成片段，由片段着色。它也执行[深度测试](https://en.wikipedia.org/wiki/Z-buffering)，[面剔除](https://en.wikipedia.org/wiki/Back-face_culling)和裁剪测试，它可以配置为输出片段填补整个多边形或只是边缘(线框渲染)。所有这些都是使用`VkPipelineRasterizationStateCreateInfo`结构配置的。

```cpp
VkPipelineRasterizationStateCreateInfo rasterizer{};
rasterizer.sType = VK_STRUCTURE_TYPE_PIPELINE_RASTERIZATION_STATE_CREATE_INFO;
rasterizer.depthClampEnable = VK_FALSE;
```

如果 `depthClampEnable` 设置为 `VK_TRUE`，那么超出近平面和远平面的片段将被保留而不是丢弃它们。这在一些特殊情况下很有用，比如阴影图。使用这个功能需要启用 GPU 特性。

```cpp
rasterizer.rasterizerDiscardEnable = VK_FALSE;
```

如果 `rasterizerDiscardEnable` 被设置为 `VK_TRUE`，那么几何图形就永远不会通过光栅化阶段。这基本上禁用了对帧缓冲的任何输出。

```cpp
rasterizer.polygonMode = VK_POLYGON_MODE_FILL;
```

polygonMode 决定如何为几何图形生成片段。以下模式可用:

- `VK_POLYGON_MODE_FILL`:用片段填充多边形区域
- `VK_POLYGON_MODE_LINE`:将多边形的边画成线
- `VK_POLYGON_MODE_POINT`:多边形顶点作为点绘制

使用填充以外的任何模式都需要启用 GPU 特性。

`lineWidth`成员是简单的，它根据线条的粗细绘制片段的数量。支持的最大线宽取决于硬件，任何宽于`1.0f`的线都需要启用 `wideLines` GPU 特性。

```cpp
rasterizer.cullMode = VK_CULL_MODE_BACK_BIT;
rasterizer.frontFace = VK_FRONT_FACE_CLOCKWISE;
```

`cullMode`变量确定要使用的面剔除类型。你可以禁用剔除，剔除正面，剔除背面，或两者兼而有之。`frontFace` 变量指定正面的顶点顺序，可以是顺时针或逆时针方向。

```cpp
rasterizer.depthBiasEnable = VK_FALSE;
rasterizer.depthBiasConstantFactor = 0.0f;
rasterizer.depthBiasClamp = 0.0f;
rasterizer.depthBiasSlopeFactor = 0.0f;
```

光栅化器可以通过增加一个常量值或基于片段的斜率偏置来改变深度值。这有时用于阴影图，但我们不会使用它。只需将 `depthBiasEnable` 设置为 `VK_FALSE`。

#### Multisampling 多重采样

`VkPipelineMultisampleStateCreateInfo` 结构配置多重采样，这是执行抗锯齿的方法之一。它的工作原理是组合多个光栅化为同一像素的多边形的片段着色器结果。这主要发生在边缘，这也是最明显的混淆伪影发生的地方。因为如果只有一个多边形映射到一个像素，那么它不需要多次运行片段着色器，这比简单地渲染到更高的分辨率然后缩放要节约资源得多。启用它需要启用 GPU 特性。

```cpp
VkPipelineMultisampleStateCreateInfo multisampling{};
multisampling.sType = VK_STRUCTURE_TYPE_PIPELINE_MULTISAMPLE_STATE_CREATE_INFO;
multisampling.sampleShadingEnable = VK_FALSE;
multisampling.rasterizationSamples = VK_SAMPLE_COUNT_1_BIT;
multisampling.minSampleShading = 1.0f;
multisampling.pSampleMask = nullptr;
multisampling.alphaToCoverageEnable = VK_FALSE;
multisampling.alphaToOneEnable = VK_FALSE;
```

我们将在后面的章节中重新讨论多重采样，现在让我们保持禁用它。

#### Depth and stencil testing 深度和模板测试

如果您使用深度和/或模板缓冲区，那么还需要使用 `VkPipelineDepthStencilStateCreateInfo` 配置深度和模板测试。我们现在还不需要，所以我们可以简单地传递一个 nullptr，而不是一个指向这样一个结构的指针。我们将在深度缓冲章节重新使用。

#### Color blending 颜色混合

片段着色器返回颜色后，需要将其与已经在帧缓冲中的颜色组合在一起。这种转换被称为颜色混合，有两种方法可以实现:

- 将新旧颜色混合，得到最终的颜色
- 使用位操作将新旧值结合在一起

可以通过两种结构体来配置颜色混合。第一个结构 `VkPipelineColorBlendAttachmentState` 包含每个附加帧缓冲的配置，第二个结构 `VkPipelineColorBlendStateCreateInfo` 包含全局颜色混合设置。在我们的例子中，我们只有一个 帧缓冲:

```cpp
VkPipelineColorBlendAttachmentState colorBlendAttachment{};
colorBlendAttachment.colorWriteMask = VK_COLOR_COMPONENT_R_BIT | VK_COLOR_COMPONENT_G_BIT | VK_COLOR_COMPONENT_B_BIT | VK_COLOR_COMPONENT_A_BIT;
colorBlendAttachment.blendEnable = VK_FALSE;
colorBlendAttachment.srcColorBlendFactor = VK_BLEND_FACTOR_ONE;
colorBlendAttachment.dstColorBlendFactor = VK_BLEND_FACTOR_ZERO;
colorBlendAttachment.colorBlendOp = VK_BLEND_OP_ADD;
colorBlendAttachment.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE;
colorBlendAttachment.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO;
colorBlendAttachment.alphaBlendOp = VK_BLEND_OP_ADD;
```

这个每帧缓冲结构允许您配置第一种颜色混合方式。将要执行的操作可以用以下伪代码进行演示:

```cpp
if (blendEnable) {
    finalColor.rgb = (srcColorBlendFactor * newColor.rgb) <colorBlendOp> (dstColorBlendFactor * oldColor.rgb);
    finalColor.a = (srcAlphaBlendFactor * newColor.a) <alphaBlendOp> (dstAlphaBlendFactor * oldColor.a);
} else {
    finalColor = newColor;
}

finalColor = finalColor & colorWriteMask;
```

如果`blendEnable`设置为`VK_FALSE`，那么片段着色器中的新颜色将不经修改通过。否则，执行两个混合操作来计算一个新的颜色。结果颜色与 `colorWriteMask` 进行 AND 运算以确定实际通过哪些通道。

最常见的使用颜色混合的方法是实现 alpha 混合，我们希望新的颜色与旧的颜色混合，基于它的不透明度。然后，`finalColor` 的计算方法如下:

```cpp
finalColor.rgb = newAlpha * newColor + (1 - newAlpha) * oldColor;
finalColor.a = newAlpha.a;
```

这可以通过以下参数来实现:

```cpp
colorBlendAttachment.blendEnable = VK_TRUE;
colorBlendAttachment.srcColorBlendFactor = VK_BLEND_FACTOR_SRC_ALPHA;
colorBlendAttachment.dstColorBlendFactor = VK_BLEND_FACTOR_ONE_MINUS_SRC_ALPHA;
colorBlendAttachment.colorBlendOp = VK_BLEND_OP_ADD;
colorBlendAttachment.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE;
colorBlendAttachment.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO;
colorBlendAttachment.alphaBlendOp = VK_BLEND_OP_ADD;
```

您可以在规范中的 `VkBlendFactor` 和 `VkBlendOp` 枚举中找到所有可能的操作。

第二个结构引用所有帧缓冲的结构数组，并允许您设置混合常量，您可以在上述计算中将其用作混合因子。

```cpp
VkPipelineColorBlendStateCreateInfo colorBlending{};
colorBlending.sType = VK_STRUCTURE_TYPE_PIPELINE_COLOR_BLEND_STATE_CREATE_INFO;
colorBlending.logicOpEnable = VK_FALSE;
colorBlending.logicOp = VK_LOGIC_OP_COPY;
colorBlending.attachmentCount = 1;
colorBlending.pAttachments = &colorBlendAttachment;
colorBlending.blendConstants[0] = 0.0f;
colorBlending.blendConstants[1] = 0.0f;
colorBlending.blendConstants[2] = 0.0f;
colorBlending.blendConstants[3] = 0.0f;
```

如果您想使用第二种混合方法(按位组合) ，那么您应该将 `logicOpEnable` 设置为 `VK_TRUE`。然后可以在`logicOp`字段中指定位操作。请注意，这将自动禁用第一个方法，就像您为每个附加的帧缓冲设置 `blendEnable` 为 `VK_FALSE` 一样！`colorWriteMask` 也将在这种模式下使用，以确定帧缓冲中的哪些通道将实际受到影响。还可以禁用这两种模式，正如我们在这里所做的，在这种情况下，片段的颜色将不加修改地写入帧缓冲。

#### Dynamic state 动态状态

我们在前面的结构中指定的有限数量的状态实际上可以在不重新创建管道的情况下进行更改。例如视口的大小、行宽和混合常量。如果你想这样做，那么你必须填写一个 `VkPipelineDynamicStateCreateInfo`  结构，如下所示:

```cpp
VkDynamicState dynamicStates[] = {
    VK_DYNAMIC_STATE_VIEWPORT,
    VK_DYNAMIC_STATE_LINE_WIDTH
};

VkPipelineDynamicStateCreateInfo dynamicState{};
dynamicState.sType = VK_STRUCTURE_TYPE_PIPELINE_DYNAMIC_STATE_CREATE_INFO;
dynamicState.dynamicStateCount = 2;
dynamicState.pDynamicStates = dynamicStates;
```

这将导致忽略这些值的配置，您将被要求在绘图时指定数据。我们将在以后的章节中回到这里。如果您没有任何动态状态，那么稍后可以用 nullptr 替换这个结构。

#### Pipeline layout 管道布局

你可以在着色器中使用`uniform`值，这些值是全局变量，类似于动态状态变量，可以在绘制时更改它们以改变着色器的行为，而无需重新创建它们。它们通常用于将变换矩阵传递给顶点着色器，或者在片段着色器中创建纹理采样器。

在创建管道时，需要通过创建 `VkPipelineLayout` 对象来指定这些`uniform`值。尽管在未来的章节中我们不会使用它们，但是我们仍然需要创建一个空的管道布局。

创建一个类成员来保存这个对象，因为稍后我们会从其他函数引用它:

```cpp
VkPipelineLayout pipelineLayout;
```

然后在 `createGraphicsPipeline` 函数中创建对象:

```cpp
VkPipelineLayoutCreateInfo pipelineLayoutInfo{};
pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
pipelineLayoutInfo.setLayoutCount = 0;
pipelineLayoutInfo.pSetLayouts = nullptr;
pipelineLayoutInfo.pushConstantRangeCount = 0;
pipelineLayoutInfo.pPushConstantRanges = 0;

if(vkCreatePipelineLayout(device, &pipelineLayoutInfo, nullptr, &pipelineLayout) != VK_SUCCESS){
    throw std::runtime_error("failed to create pipeline layout!");
}
```

该结构还指定了推送常量，这是另一种将动态值传递给着色器的方式，我们将在以后的章节中介绍。管道布局将在程序的整个生命周期中被引用，所以它应该在最后被销毁:

```cpp
void cleanup() {
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    ...
}
```

#### Conclusion 总结

这就是所有的固定函数状态！从头开始建立这一切需要做大量的工作，但优势在于我们现在几乎完全意识到了中东绘图管线正在发生的一切！这可以减少出现意外行为的机会，因为某些组件的默认状态可能不是您所期望的。

然而，在我们最终创建绘图管线之前，还有一个对象需要创建，那就是渲染通道。

### Render passes 渲染通道

#### Setup 设置

在我们完成创建管道之前，我们需要告诉 Vulkan 渲染时将要使用的帧缓冲 附件。我们需要指定有多少颜色和深度缓冲区，每个缓冲区需要使用多少样本，以及在整个渲染操作中应该如何处理它们的内容。所有这些信息都包装在一个渲染通道对象中，为此我们将创建一个新的 `createRenderPass` 函数。在`createGraphicsPipeline`之前从 `initVulkan` 调用这个函数。

#### Attachment description 附件描述

在我们的例子中，我们只有一个颜色缓冲区附件，由交换链中的一个图像表示。

```cpp
VkAttachmentDescription colorAttachment{};
colorAttachment.format = swapChainImageFormat;
colorAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
```

颜色附件的`format`应该与交换链映像的格式匹配，而且我们还没有对多采样进行任何操作，因此我们将坚持使用一个样本。

```cpp
colorAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_LOAD;
colorAttachment.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
```

`loadOp` 与 `storeOp` 决定在渲染之前和渲染之后如何处理附件中的数据。对于 `loadOp`，我们有以下选择:

- `VK_ATTACHMENT_LOAD_OP_LOAD` 保留附件的现有内容
- `VK_ATTACHMENT_LOAD_OP_CLEAR` 在开始时将值清除为常数
- `VK_ATTACHMENT_LOAD_OP_DONT_CARE` 现有内容未定义，我们不在乎

在我们的示例中，在绘制新帧之前，我们将使用 clear 操作将帧缓冲清除为黑色。对`storeOp`而言只有两种可能性

- `VK_ATTACHMENT_STORE_OP_STORE` 渲染的内容将存储在内存中，以后可以读取
- `VK_ATTACHMENT_STORE_OP_DONT_CARE` 在渲染操作之后，将不定义帧缓冲的内容

我们将在屏幕上渲染一个三角形，所以我们在这里使用存储操作

```cpp
colorAttachment.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
colorAttachment.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
```

`loadOp` 和 `storeOp` 适用于颜色和深度数据，`stencilLoadOp`/`stencilStoreOp` 适用于模版数据。我们的应用程序不会对模版缓冲区做任何事情，因此加载和存储的结果是不相关的。

```cpp
colorAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
colorAttachment.finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;
```

在 Vulkan 中，纹理和帧缓冲由具有特定像素格式的 `VkImage` 对象表示，但是内存中的像素布局可以根据您试图对图像进行的操作进行更改。

一些最常见的布局如下:

- `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL` 作为颜色附件的图片
- `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR` 将在交换链中显示的图像
- `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL` 用作内存复制操作目标的图像

我们将在贴图这一章更深入地讨论这个话题，但是现在重要的是要知道，图像需要转换到特定的布局，以适配他们下一步要参与的操作。

`initialLayout` 指定在渲染开始之前图像的布局。`finalLayout` 指定在渲染结束时自动转换到的布局。对 `initialLayout` 使用 `VK_IMAGE_LAYOUT_UNDEFINED` 意味着我们不关心图像之前的布局是什么。这个特殊值的警告是，不能保证图像的内容被保留，但是这并不重要，因为我们无论如何都要清除它。我们希望在渲染后使用交换链准备好显示图像，这就是为什么我们使用 `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR` 作为 `finalLayout`。

#### Subpasses and attachment references 子通道和附件引用

一个渲染通道可以由多个子通道组成。子通道是依赖于先前通道中帧缓冲 内容的后续渲染操作，例如一个接一个应用的后处理效果序列。如果您将这些渲染操作分组到一个渲染通道中，那么Vulkan可以重新排序这些操作并节省内存带宽以获得更好的性能。然而，对于我们的第一个三角形，我们将坚持单个子通道。

每个子传递都引用一个或多个我们在前面部分中使用结构描述的附件。这些引用本身就是 `VkAttachmentReference` 结构，如下所示:

```cpp
VkAttachmentReference colorAttachmentRef{};
colorAttachmentRef.attachment = 0;
colorAttachmentRef.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
```

`attachment`参数通过附件描述数组中的索引指定要引用的附件。我们的数组由一个 `VkAttachmentDescription`组成，因此它的索引为0。`layout`指定了我们希望附件在使用此引用的子通道期间具有的布局。Vulkan会在子通道启动时自动将附件转换到这个布局。我们打算使用附件作为一个颜色缓冲区和`VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`布局将给我们提供最佳的性能，就像它的名字所暗示的那样。

子通道使用 `VkSubpassDescription` 结构描述:

```cpp
VkSubpassDescription subpass{};
subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
```

Vulkan将来也可能支持计算子通道，所以我们必须明确说明这是一个图形子通道。接下来，我们指定对颜色附件的引用:

```cpp
subpass.colorAttachmentCount = 1;
subpass.pColorAttachments = &colorAttachmentRef;
```

这个数组中附件的索引直接从片段着色器引用，即我们之前使用的`layout(location = 0) out vec4 outColor`指令

子通道可以引用以下其他类型的附件:

- `pInputAttachments` 从着色器读取的附件
- `pResolveAttachments` 用于多采样彩色附件的附件
- `pDepthStencilAttachment` 深度和模板数据附件
- `pPreserveAttachments` 此子通道不使用但必须为其保留数据的附件

#### Render pass 渲染通道

现在已经描述了附件和引用它的基本子通道，我们可以创建渲染通道本身。创建一个新的类成员变量，将 `VkRenderPass` 对象保存在 `pipelineLayout` 变量之上:

```cpp
VkRenderPass renderPass;
VkPipelineLayout pipelineLayout;
```

然后可以通过填充 `VkRenderPassCreateInfo` 结构以及一系列的附件和子通道来创建 `render pass `对象。`VkAttachmentReference` 对象使用此数组的索引引用附件。

```cpp
VkRenderPassCreateInfo renderPassInfo{};
renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
renderPassInfo.attachmentCount = 1;
renderPassInfo.pAttachments = &colorAttachment;
renderPassInfo.subpassCount = 1;
renderPassInfo.pSubpasses = &subpass;

if(vkCreateRenderPass(device, &renderPassInfo, nullptr, &renderPass) != VK_SUCCESS){
    throw std::runtime_error("failed to create render pass!");
}
````

就像管道布局一样，渲染通道在整个程序中都会被引用，所以它应该只在最后被清理:

```cpp
void cleanup() {
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    vkDestroyRenderPass(device, renderPass, nullptr);
    ....
}
```

这是一个很大的工作量，但是在下一章中，所有的工作都集中在一起，最终创建了绘图管线对象！

#### Conclusion 总结

我们现在可以结合前面章节中的所有结构和对象来创建绘图管线！以下是我们现在拥有的对象类型，简单回顾一下:

- 着色器阶段: 着色器模块定义了渲染管线的可编程阶段的功能
- 固定功能状态: 定义渲染管线固定功能阶段的所有结构，比如输入组件、光栅、视口和颜色混合
- 管线布局: 着色器引用的`uniform`值和`push`值，可以在绘制时更新
- 渲染通道: 管道阶段引用的附件及其用法

所有这些都完全定义了绘图管线的功能，因此我们现在可以开始填充在 `createGraphicsPipeline` 函数结尾处的 `VkGraphicsPipelineCreateInfo` 结构。但是在调用 `vkDestroyShaderModule` 之前，因为在创建过程中仍然需要使用这些函数。

```cpp
VkGraphicsPipelineCreateInfo pipelineInfo{};
pipelineInfo.sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO;
pipelineInfo.stageCount = 2;
pipelineInfo.pStages = shaderStages;
```

我们首先引用 `VkPipelineShaderStageCreateInfo` 结构的数组。

```cpp
VkGraphicsPipelineCreateInfo pipelineInfo{};
pipelineInfo.sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO;
pipelineInfo.stageCount = 2;
pipelineInfo.pStages = shaderStages;
pipelineInfo.pVertexInputState = &vertexInputInfo;
pipelineInfo.pInputAssemblyState = &inputAssembly;
pipelineInfo.pViewportState = &viewportState;
pipelineInfo.pRasterizationState = &rasterizer;
pipelineInfo.pMultisampleState = &multisampling;
pipelineInfo.pDepthStencilState = nullptr;
pipelineInfo.pColorBlendState = &colorBlending;
pipelineInfo.pDynamicState = nullptr;
```

然后我们引用了所有描述固定函数阶段的结构。

```cpp
pipelineInfo.layout = pipelineLayout;
```

然后是管道布局，它是 Vulkan 句柄而不是结构指针。

```cpp
pipelineInfo.renderPass = renderPass;
pipelineInfo.subpass = 0;
```

最后，我们有渲染通道的参考和子通道的索引，这个渲染管线将被使用。也可以使用这个管道而不是这个特定实例的其他渲染通道，但是它们必须与 `renderPass` 兼容。[这里](https://www.khronos.org/registry/vulkan/specs/1.0/html/vkspec.html#renderpass-compatibility)描述了兼容性的要求，但是我们不会在本教程中使用这个特性。

```cpp
pipelineInfo.basePipelineHandle = VK_NULL_HANDLE;
pipelineInfo.basePipelineIndex = -1;
```

实际上还有两个参数: `basePipelineHandle` 和 `basePipelineIndex`。Vulkan 允许您通过从现有管道派生出一个新的绘图管线。管道派生的想法是，当管道与现有管道具有许多共同功能时，设置管道的成本会更低，并且可以更快地在来自同一父管道的管道之间进行切换。您可以使用 `basePipelineHandle` 指定现有管道的句柄，也可以使用 `basePipelineIndex` 引用即将通过索引创建的另一个管道。现在只有一个管道，所以我们只需指定一个空句柄和一个无效的索引。只有在 `VkGraphicsPipelineCreateInfo` 的 `flags` 字段中也指定了 `VK_PIPELINE_CREATE_DERIVATIVE_BIT` 标志时，才会使用这些值。

现在通过创建一个类成员来保存 `VkPipeline` 对象来准备最后一步:

```cpp
VkPipeline graphicsPipeline;
```

最后，创建一个渲染管线:

```cpp
if(vkCreateGraphicsPipelines(device, VK_NULL_HANDLE, 1, &pipelineInfo, nullptr, &graphicsPipeline) != VK_SUCCESS){
    throw std::runtime_error("failed to create graphics pipeline!");
}
```

`vkCreateGraphicsPipelines` 函数实际上比 Vulkan 中通常的对象创建函数有更多的参数。它被设计为在一个调用中接收多个 `VkGraphicsPipelineCreateInfo` 对象并创建多个 `VkPipeline` 对象。

第二个参数，我们已经为其传递了 `VK_NULL_HANDLE`参数，它引用了一个可选的 `VkPipelineCache` 对象。管道缓存可以用来存储和重用与管道创建相关的数据，这些数据通过对 `vkCreateGraphicsPipelines`的多次调用进行存储和重用，如果缓存存储在一个文件中，甚至可以跨程序执行进行存储和重用。这使得以后可以显著加快管道的创建。我们将在管道缓存章节中讨论这个问题。

所有常见的绘图操作都需要渲染管线，所以它也应该只在程序结束时被销毁:

```cpp
void cleanup() {
    vkDestroyPipeline(device, graphicsPipeline, nullptr);
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    ...
}
```

现在运行您的程序，确认所有这些艰苦的工作已经导致了成功的管道创建！我们已经非常接近看到屏幕上弹出的东西。在接下来的几章中，我们将根据交换链映像设置实际的帧缓冲区，并准备绘图命令。

### drawing 绘图

#### Framebuffers 帧缓冲

在过去的几章中，我们已经讨论了很多关于帧缓冲的内容，并且我们已经设置了渲染通道，期望使用与交换链映像相同格式的单个帧缓冲，但是我们实际上还没有创建任何渲染通道。

在渲染通道创建期间指定的附件通过将它们包装到`VkFramebuffer`对象中来绑定。 帧缓冲区对象引用所有表示附件的 `VkImageView` 对象。 在我们的例子中，只有一个：颜色附件。 然而，我们用于附件的图像取决于当我们检索一个用于展示的图像时交换链返回的那个图像。这意味着我们必须为交换链中的所有图像创建一个帧缓冲区，并使用与绘制时检索到的图像对应的帧缓冲区。

为此，创建另一个 `std::vector` 类成员来保存帧缓冲:

```cpp
std::vector<VkFramebuffer> swapChainFramebuffers;
```


我们将在一个新的函数`createFramebuffers`中为这个数组创建对象，这个函数在创建渲染管线数组之后被 `initVulkan` 调用:

```cpp
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createFramebuffers();
}

...

void createFramebuffers() {

}
```

首先调整容器的大小以保存所有帧缓冲:

```cpp
void createFramebuffers(){
    swapChainFramebuffers.resize(swapChainImageViews.size());
}
```

然后我们将遍历图像视图并从中创建帧缓冲:

```cpp
for(size_t i = 0; i < swapChainImageViews.size();i++){
    VkImageView attachments[] = {
        swapChainImageViews[i]
    };
    
    VkFramebufferCreateInfo framebufferInfo{};
    framebufferInfo.sType = VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO;
    framebufferInfo.renderPass = renderPass;
    framebufferInfo.attachmentCount = 1;
    framebufferInfo.pAttachments = attachments;
    framebufferInfo.width = swapChainExtent.width;
    framebufferInfo.height = swapChainExtent.height;
    framebufferInfo.layers = 1;
    
    if(vkCreateFramebuffer(device, &framebufferInfo, nullptr, &swapChainFramebuffers[i]) != VK_SUCCESS){
        throw std::runtime_error("failed to create framebuffer!");
    }
}
```

如您所见，帧缓冲的创建非常简单。我们首先需要指定帧缓冲区需要与哪个`renderPass`兼容。您只能使用与其兼容的渲染通道的帧缓冲，这大致意味着它们使用相同数量和类型的附件。

`attachmentCount` 和 `pAttachments` 参数指定应该绑定到渲染通道 `pAttachment` 数组中的相应附件描述的 `VkImageView` 对象。

`width`和`height`参数是不言自明的，`layers`是指图像数组中的层数。我们的交换链映像是单个映像，因此层数为1。

我们应该在图像视图和渲染通道之前删除它们所基于的帧缓冲区，但只能在我们完成渲染之后：

```cpp
for(auto framebuffer : swapChainFramebuffers){
    vkDestroyFramebuffer(device, framebuffer, nullptr);
}
```

我们现在已经到达了一个里程碑，我们拥有渲染所需的所有对象。在下一章中，我们将编写第一个实际的绘图命令。

#### Command buffers 命令缓冲区

Vulkan中的命令，比如绘图操作和内存传输，不直接使用函数调用来执行。您必须在命令缓冲区对象中记录要执行的所有操作。这样做的好处是所有设置绘图命令的繁重工作都可以提前在多线程中完成。之后，您只需要告诉 Vulkan 在主循环中执行命令。

##### Command pools 命令池

在创建命令缓冲区之前，我们必须创建一个命令池。命令池管理用于存储缓冲区的内存，并从中分配命令缓冲区。添加一个新的类成员来存储 `VkCommandPool`:

```cpp
VkCommandPool commandPool;
```

然后创建一个新函数 `createCommandPool`，并在创建帧缓冲后从 `initVulkan` 调用它。

```cpp
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createFramebuffers();
    createCommandPool();
}

...

void createCommandPool() {

}
```

创建命令池只需要两个参数:

```cpp
QueueFamilyIndices queueFamilyIndices = findQueueFamilies(physicalDevice);

VkCommandPoolCreateInfo poolInfo{};
poolInfo.sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO;
poolInfo.queueFamilyIndex = queueFamilyIndices.graphicsFamily.value();
poolInfo.flags = 0;
```

命令缓冲区是通过将它们提交到设备队列之一来执行的，例如我们检索的图形和表示队列。每个命令池只能分配在单一类型队列上提交的命令缓冲区。我们将录入绘图命令，所以我们选择图形队列族。

命令池有两种可选的标志:

- `VK_COMMAND_POOL_CREATE_TRANSIENT_BIT`: 提示命令缓冲区将经常使用的新命令记录下来(可能会导致内存分配行为出现改变)
- `VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT` 允许命令缓冲区单独记录，如果没有这个标志，它们都必须一起重置

我们将只在程序开始时记录命令缓冲区，然后在主循环中多次执行它们，因此我们不会使用这两个标志中的任何一个。

```cpp
if(vkCreateCommandPool(device, &poolInfo, nullptr, &commandPool) != VK_SUCCESS){
    throw std::runtime_error("failed to create command pool!");
}
```

使用`vkCreateCommandPool`函数完成命令池的创建。它没有任何特殊的参数。命令将在整个程序中用于在屏幕上绘制内容，因此命令池只能在最后被销毁:

```cpp
void cleanup() {
    vkDestroyCommandPool(device, commandPool, nullptr);

    ...
}
```

##### Command buffer allocation 命令缓冲区分配

我们现在可以开始分配命令缓冲区，并在其中记录绘图命令。由于其中一个绘图命令涉及绑定正确的 `VkFramebuffer`，因此实际上我们必须再次为交换链中的每个映像记录一个命令缓冲区。为此，创建一个`VkCommandBuffer`对象列表作为类成员。命令缓冲区将在其命令池被销毁时自动释放，因此我们不需要显式的清理。

```cpp
std::vector<VkCommandBuffer> commandBuffers;
```

现在，我们将开始处理 createCommandBuffers 函数，该函数为每个交换链映像分配和记录命令。

```cpp
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createFramebuffers();
    createCommandPool();
    createCommandBuffers();
}

...

void createCommandBuffers() {
    commandBuffers.resize(swapChainFramebuffers.size());
}
```

命令缓冲区使用 `vkAllocateCommandBuffers` 函数分配，该函数接受 `VkCommandBufferAllocateInfo` 结构作为参数，指定命令池和分配的缓冲区数量:

```cpp
VkCommandBufferAllocateInfo allocInfo{};
allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
allocInfo.commandPool = commandPool;
allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
allocInfo.commandBufferCount = (uint32_t) commandBuffers.size();

if(vkAllocateCommandBuffers(device, &allocInfo, commandBuffers.data()) != VK_SUCCESS){
    throw std::runtime_error("failed to allocate command buffers!");
}
```

`Level` 参数指定分配的命令缓冲区是主命令缓冲区还是辅命令缓冲区。

- `VK_COMMAND_BUFFER_LEVEL_PRIMARY` 可以提交到队列执行，但不能从其他命令缓冲区调用
- `VK_COMMAND_BUFFER_LEVEL_SECONDARY` 不能直接提交，但可以从主命令缓冲区调用

这里我们不会使用辅助命令缓冲区的功能，但是您可以想象，辅助命令缓冲区对主命令缓冲区重用常见操作是很有帮助的。

##### Starting command buffer recording 开始记录命令缓冲区

我们通过调用`vkBeginCommandBuffer`开始记录命令缓冲区，并使用一个简单的`VkCommandBufferBeginInfo`结构作为参数，该结构指定了有关这个特定命令缓冲区使用情况的一些细节。

```cpp
for(size_t i = 0; i < commandBuffers.size(); i++){
    VkCommandBufferBeginInfo beginInfo{};
    beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
    beginInfo.flags = 0;
    beginInfo.pInheritanceInfo = nullptr;
    
    if(vkBeginCommandBuffer(commandBuffers[i], &beginInfo) != VK_SUCCESS){
        throw std::runtime_error("failed to begin recording command buffer!");
    }
}
```

`flags` 参数指定如何使用命令缓冲区:

- `VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT` 命令缓冲区将在执行一次后立即重新记录
- `VK_COMMAND_BUFFER_USAGE_RENDER_PASS_CONTINUE_BIT` 这是一个辅助命令缓冲区，将完全在单个渲染通道中。
- `VK_COMMAND_BUFFER_USAGE_SIMULTANEOUS_USE_BIT` 当命令缓冲区在等待执行时，命令缓冲区可以重新提交

这些标志现在都不适用于我们。

`pInheritanceInfo`参数仅与辅助命令缓冲区相关。它指定从调用主命令缓冲区继承哪个状态。

如果命令缓冲区已被记录一次，则调用 vkBeginCommandBuffer 将隐式重置它。在完成命令缓冲区的建立后，不可以再对命令缓冲区附加指令。

##### Starting a render pass 启动一个渲染通道

绘图从使用`vkCmdBeginRenderPass`启动渲染通道开始。使用 `VkRenderPassBeginInfo`  结构中的某些参数配置渲染通道。

```cpp
VkRenderPassBeginInfo renderPassInfo{};
renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO;
renderPassInfo.renderPass = renderPass;
renderPassInfo.framebuffer = swapChainFramebuffers[i];
```

第一个参数是渲染通道本身和要绑定的附件。我们为每个交换链映像创建一个 帧缓冲，将其指定为颜色附件。

```cpp
renderPassInfo.renderArea.offset = {0,0};
renderPassInfo.renderArea.extent = swapChainExtent;
```

接下来的两个参数定义渲染区域的大小。渲染区域定义着色器加载和存储的位置。此区域外的像素将具有未定义的值。它应该与附件的大小匹配，以获得最佳性能。

```cpp
VkClearValue clearColor = {0.0f, 0.0f, 0.0f, 1.0f};
renderPassInfo.clearValueCount = 1;
renderPassInfo.pClearValues = &clearColor;
```

最后两个参数定义了用于`VK_ATTACHMENT_LOAD_OP_CLEAR`的清晰值，我们将其用作颜色附件的加载操作。我已经将透明颜色定义为纯黑色，不透明度为100% 。

```cpp
vkCmdBeginRenderPass(commandBuffers[i], &renderPassInfo, VK_SUBPASS_CONTENTS_INLINE);
```

渲染通道现在可以开始了。记录命令的所有函数都可以通过它们的`vkCmd`前缀来识别。它们都返回 `void`，因此在我们完成录制之前不会有错误处理。

每个命令的第一个参数总是记录命令的命令缓冲区。第二个参数指定我们刚才提供的渲染通道的详细信息。最后一个参数控制如何在渲染通道中提供绘图命令。它可以有两个值:

- `VK_SUBPASS_CONTENTS_INLINE` 渲染通道命令将嵌入主命令缓冲区本身，不会执行辅助命令缓冲区
- `VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS` 将从辅助命令缓冲区执行渲染通道命令

##### Basic drawing commands 基本绘图命令

我们现在可以绑定渲染管线:

```cpp
vkCmdBindPipeline(commandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, graphicsPipeline);
```

第二个参数指定管道对象是图形管道还是计算管道。我们现在已经告诉 Vulkan 哪些操作要在渲染管线中执行，哪些附件要在片段着色器中使用，所以剩下的就是告诉它绘制三角形:

```cpp
vkCmdDraw(commandBuffers[i], 3, 1, 0, 0);
```

实际的 `vkCmdDraw` 函数有点虎头蛇尾，但是由于我们事先指定了所有信息，所以它非常简单。除了命令缓冲区，它还有以下参数:

- `vertexCount` 即使我们没有顶点缓冲区，我们在技术上仍然有3个顶点绘制
- `instanceCount` 用于实例渲染，如果没有使用则使用1
- `firstVertex` 用作顶点缓冲区的偏移量，定义最低值为`gl_VertexIndex`
- `firstInstance` 用作实例渲染的偏移量，定义最小值`gl_InstanceIndex`.

##### Finishing up 完成
 
 渲染通道现在可以结束了:

```cpp
vkCmdEndRenderPass(commandBuffers[i]);
```

我们已经完成了命令缓冲区的记录:

```cpp
if(vkEndCommandBuffer(commandBuffers[i]) != VK_SUCCESS){
    throw std::runtime_error("failed to record command buffer!");
}
```

在下一章中，我们将为主循环编写代码，它将从交换链中获取映像，执行正确的命令缓冲区，并将完成的映像返回到交换链。
