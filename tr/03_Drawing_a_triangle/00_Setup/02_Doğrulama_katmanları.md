## Doğrulama katmanları nedir?

Vulkan API, sürücüye en az yükü bindirme fikri üzerine tasarlanmıştır, bunun
gözle görülür sonuçlarından biri de hata kontrolünün çok kısıtlı bir şekilde
gelmesidir. Enumlara yanlış değerler vermek ve veya gerekli parametrelere null
atmak gibi hataların bile kontrolü yok ve bunlar genelde programın sessizce
kapanmasıyla veya belirsiz davranışlarla sonuçlanıyor. Vulkan her şeyi elle
yapmanızı beklediği için, mantıksal aygıt oluştururken istemeyi unuttuğumuz yeni
bir GPU özelliğini kullanmak gibi küçük hataları çok rahat yapabiliyorsunuz.

Ama bunlar, kontrollerin API'a eklenemeyeceği anlamına gelmiyor. Vulkan bu iş
için bize *doğrulama katmanları* (validation layers) adlı çok şık bir sistem
sunuyor. Doğrulama katmanları, Vulkan fonksiyon çağrılarına bağlanıp ek işlemler
uygulayan opsiyonel bir parça. Doğrulama katmanlarındaki yaygın işlemler şunlar:

* Resmi tanımlara göre parametre değerlerinin uygun kullanım kontrolü
* Bellek sızıntılarını bulmak için nesne oluşturma ve yok etme kontrolü
* Güvenli iş parçacığı kontrolü için çağrıların kaynağı olan iş parçacıklarının takibi
* Tüm çağrıları ve parametreleri standart çıktıya kayıtlama
* Profiling ve replaying için Vulkan çağrılarının takibi

Aşağıda, bir tanılama doğrulama katmanı fonksiyonunun nasıl yazılabileceğiyle
ile ilgili bir örnek var:

```c++
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

Bu doğrulama katmanları, ilgilendiğiniz tüm hata ayıklama işlevlerini içerecek
şekilde üst üste bindirilebilir. Doğrulama katmanlarını geliştirme sırasında
etkinleştirip programı yayınlarken devre dışı bırakarak hem hata ayıklama
özelliklerinden faydalanıp hem de son üründe performanstan taviz
vermeyebilirsiniz.

Vulkan'ın kendisi bir doğrulama katmanı içermiyor ancak LunarG'nin sağladığı
Vulkan SDK içerisinde, yaygın hataları kontrol eden çok hoş bir doğrulama
katmanı sağlanıyor. Bunlar ayrıca tamamen [açık kaynak](https://github.com/KhronosGroup/Vulkan-ValidationLayers)
olduğundan, ne tür hataların kontrol edildiğine bakıp kendiniz de katkıda
bulunabilirsiniz. Doğrulama katmanlarını kullanmak, programınızın belirsiz
davranışları kullanıp farklı sürücülerde hatalı çalışmasının önüne geçmek için
en iyi yol.

Doğrulama katmanları ancak sisteminize yüklendiyse kullanılabilir. Örneğin
LunarG doğrulama katmanları sadece Vulkan SDK yüklü bilgisayarlarda çalışabilir.

Önceden Vulkan'da instance ve cihaz özelinde olmak üzere iki tip doğrulama
katmanı vardı. Instance katmanları sadece instance'lar gibi global objeleri
kontrol ederken cihaz katmanları da o GPU'ya bağlı tüm çağrıları kontrol
ediyordu. Cihaz katmanları artık kullanılmıyor, yani instance katmanları tüm
Vulkan çağrılarına etki ediyor. Bazı versiyonlarda hala gerekebileceğinden,
resmi dökümantasyon cihaz katmanlarını da etkinleştirmenizi öneriyor. Biz de
ikisi için de aynı katmanları etkinleştireceğiz. Cihaz katmanlarına [sonraki](!en/Drawing_a_triangle/Setup/Logical_device_and_queues)
bölümlerde değineceğiz.

## Doğrulama katmanlarının kullanımı

Bu bölümde Vulkan SDK ile gelen standart tanı katmanlarını etkinleştireceğiz.
Uzantılar gibi, doğrulama katmanları da isimleri verilerek etkinleştirilmeli.
Faydalı standart katmanların tamamı SDK ile gelen `VK_LAYER_KHRONOS_validation`
adı altında paketlenmiş durumda.

Şimdi etkinleştirilecek katmanların adını ve katmanların etkinleştirilip
etkinleştirilmeyeceğini tutan iki değişken tanımlayalım. İkinci değişkenin
değerini hata atıklama modunda derleyip derlemeyeceğimize göre belirlemeyi
seçtim. `NDEBUG` makrosu C++ standardının bir parçası ve "hata ayıklama değil"
(not debug) demek.

```c++
const uint32_t WIDTH = 800;
const uint32_t HEIGHT = 600;

const std::vector<const char*> validationLayers = {
    "VK_LAYER_KHRONOS_validation"
};

#ifdef NDEBUG
    const bool enableValidationLayers = false;
#else
    const bool enableValidationLayers = true;
#endif
```

Gerekli tüm katmanların olup olmadığını kontrol edecek olan
`checkValidationLayerSupport` fonksiyonunu ekleyelim. İlk olarak kullanılabilir
tüm katmanları `vkEnumerateInstanceLayerProperties` ile listeleyelim. Kullanımı,
instance bölümünde tartıştığımız `vkEnumerateInstanceExtensionProperties`
fonksiyonu ile aynı.

```c++
bool checkValidationLayerSupport() {
    uint32_t layerCount;
    vkEnumerateInstanceLayerProperties(&layerCount, nullptr);

    std::vector<VkLayerProperties> availableLayers(layerCount);
    vkEnumerateInstanceLayerProperties(&layerCount, availableLayers.data());

    return false;
}
```

Sonra `validationLayers` listesindeki tüm katmanların `availableLayers`
listesinde de olup olmadığını kontrol edelim. `strcmp` için `<cstring>`
kütüphanesini eklemeniz gerekebilir.

```c++
for (const char* layerName : validationLayers) {
    bool layerFound = false;

    for (const auto& layerProperties : availableLayers) {
        if (strcmp(layerName, layerProperties.layerName) == 0) {
            layerFound = true;
            break;
        }
    }

    if (!layerFound) {
        return false;
    }
}

return true;
```

Şimdi bu fonksiyonu `createInstance` içerisinde kullanabiliriz:

```c++
void createInstance() {
    if (enableValidationLayers && !checkValidationLayerSupport()) {
        throw std::runtime_error("validation layers requested, but not available!");
    }

    ...
}
```

Şimdi programı hata ayıklama modunda çalıştırın ve kodda herhangi bir hata
bulunmadığına emin olun. Eğer hata alırsanız SSS bölümüne bakın.

Ve son olarak, katmanların etkinleştirilme durumuna göre `VkInstanceCreateInfo`
structındaki katman sayısı ve isimleri değişkenlerini güncelleyelim:

```c++
if (enableValidationLayers) {
    createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
    createInfo.ppEnabledLayerNames = validationLayers.data();
} else {
    createInfo.enabledLayerCount = 0;
}
```

Eğer kontrol başarılıysa `vkCreateInstance` fonksiyonu hiçbir zaman
`VK_ERROR_LAYER_NOT_PRESENT` hatası döndürmemeli, ama yine de emin olmak için
programınızı çalıştırıp deneyin.

## Mesaj callback fonksiyonu

Doğrulama katmanları varsayılan olarak, hata ayıklama mesajlarını standart
çıktıya basar ancak kendimiz bir callback fonksiyonu vererek bu mesajları
kendimiz de işleyebiliriz. Bu ayrıca mesajları filtreleme imkanı da sunar,
sonuçta ciddi hatalar dışındaki mesajların hepsi önemli değil. Eğer bunu şimdi
yapmak istemiyorsanız bu konunun son bölümüne atlayabilirsiniz.

Mesajları ve ilgili detayları işleyecek olan callback fonksiyonunu oluşturmak
için `VK_EXT_debug_utils` uzantısını kullanarak bir callbacke sahip hata
ayıklama habercisi oluşturacağız.

Doğrulama katmanlarının etkinleştirilme durumuna göre gerekli uzantıların
listesini döndürecek bir `getRequiredExtensions` fonksiyonu yazalım:

```c++
std::vector<const char*> getRequiredExtensions() {
    uint32_t glfwExtensionCount = 0;
    const char** glfwExtensions;
    glfwExtensions = glfwGetRequiredInstanceExtensions(&glfwExtensionCount);

    std::vector<const char*> extensions(glfwExtensions, glfwExtensions + glfwExtensionCount);

    if (enableValidationLayers) {
        extensions.push_back(VK_EXT_DEBUG_UTILS_EXTENSION_NAME);
    }

    return extensions;
}
```

GLFW tarafından istenen uzantılar her zaman şart, ama hata ayıklama habercisi
için olanlar duruma göre ekleniyor. Farkettiyseniz burada
`VK_EXT_DEBUG_UTILS_EXTENSION_NAME` adlı bir makro kullandım, bunun değeri
"VK_EXT_debug_utils" stringine eşit. Bu makroyu kullanmak, stringde yazım hatası
yapmanızın önüne geçiyor.

Şimdi `createInstance` içerisinde bu fonksiyonu kullanabiliriz:

```c++
auto extensions = getRequiredExtensions();
createInfo.enabledExtensionCount = static_cast<uint32_t>(extensions.size());
createInfo.ppEnabledExtensionNames = extensions.data();
```

Programı şimdi çalıştırın ve `VK_ERROR_EXTENSION_NOT_PRESENT` hatası
almadığınıza emin olun. Bu uzantının varlığını elle kontrol etmemize gerek yok,
doğrulama katmanının varlığından dolayı garanti olması lazım.

Şimdi hata ayıklama callbackinin nasıl göründüğüne bakalım. Sınıfa ait statik
ve `PFN_vkDebugUtilsMessengerCallbackEXT` prototipinde bir `debugCallback`
fonksiyonu ekleyelim. `VKAPI_ATTR` ve `VKAPI_CALL` ifadeleri, fonksiyonun Vulkan
tarafından çağrılabilmesi için doğru imzaya sahip olmasını sağlıyor.

```c++
static VKAPI_ATTR VkBool32 VKAPI_CALL debugCallback(
    VkDebugUtilsMessageSeverityFlagBitsEXT messageSeverity,
    VkDebugUtilsMessageTypeFlagsEXT messageType,
    const VkDebugUtilsMessengerCallbackDataEXT* pCallbackData,
    void* pUserData) {

    std::cerr << "validation layer: " << pCallbackData->pMessage << std::endl;

    return VK_FALSE;
}
```

İlk parametre mesajın önem seviyesini belirtiyor, değerleri şu flaglerden biri:

* `VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT`: Tanılama mesajı
* `VK_DEBUG_UTILS_MESSAGE_SEVERITY_INFO_BIT_EXT`: Kaynak oluşturma gibi bilgilendirme mesajları
* `VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT`: Hata olması şart olmayan ama genelde programdaki sorunlu bir davranışı belirten uyarı mesajlar
* `VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT`: Hatalı kullanımlar ve programın bozulmasına sebep olacak davranışlarla ilgili mesajlar

Bu enumların değerleri, mesajların önemlerinin birbiriyle olan büyüklük küçüklük
ilişkisi karşılaştırılabilecek şekilde ayarlanmıştır, örneğin:

```c++
if (messageSeverity >= VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT) {
    // Message is important enough to show
}
```

`messageType` parametresi aşağıdaki değerleri alabilir:

* `VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT`: Resmi tanımlarla veya performansla ilgisi olmayan genel olaylar
* `VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT`: Resmi tanımlara uymayan veya muhtemel hatalı kullanımla ilgili olaylar
* `VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT`: Vulkan'ın verimsiz kullanılmış olabileceği olaylar

`pCallbackData` parametresi, mesajlarla ilgili detayları içeren
`VkDebugUtilsMessengerCallbackDataEXT` structına işaret eder, en önemli
elemanları şunlar:

* `pMessage`: Null ile biten bir string formatında hata ayıklama mesajı
* `pObjects`: Mesajla ilgili Vulkan nesnelerinin işleyicilerini tutan bir dizi
* `objectCount`: Dizideki eleman sayısı

Son olarak `pUserData` parametresi de callback kurulumu sırasında belirtilen
bir işaretçi aracılığıyla fonksiyona kendi verinizi göndermenizi sağlıyor.

Callback bir boolean değeri döndürüyor, bu mesaja sebep olan Vulkan çağrısının
yarıda kesilip kesilmeyeceğini belirliyor. Eğer callback true dönerse, çağrı
`VK_ERROR_VALIDATION_FAILED_EXT` hatasıyla kapanıyor. Bu genelde sadece
doğrulama katmanlarının kendi doğruluğunu test etmek için kullanılır, bu yüzden
her zaman `VK_FALSE` döndürmelisiniz.

Şimdi tek kalan, Vulkan'a bu callback hakkında gerekli bilgiyi vermek. İlginç
bir şekilde Vulkan'da bu hata ayıklama callbacki bile elle oluşturulup yok
edilmesi gereken bir işleyici ile yönetiliyor. Bu callback, *hata ayıklama habercisinin*
(debug messenger) bir parçası ve bunlardan istediğiniz kadar üretebilirsiniz.
Instance'ın hemen altına bir sınıf değişkeni olarak işleyiciyi ekleyelim:

```c++
VkDebugUtilsMessengerEXT debugMessenger;
```

Şimdi `initVulkan` tarafından `createInstance`'tan hemen sonra çağrılacak olan
`setupDebugMessenger` fonksiyonunu ekleyelim:

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
}

void setupDebugMessenger() {
    if (!enableValidationLayers) return;

}
```

Şimdi haberci ve bunun callbacki hakkında detayları içeren structı dolduralım:

```c++
VkDebugUtilsMessengerCreateInfoEXT createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT;
createInfo.messageSeverity = VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT;
createInfo.messageType = VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT;
createInfo.pfnUserCallback = debugCallback;
createInfo.pUserData = nullptr; // Optional
```

`messageSeverity` alanı, callbackin çağrılacağı önem seviyelerini belirtmenize
yarar. Ben burada `VK_DEBUG_UTILS_MESSAGE_SEVERITY_INFO_BIT_EXT` haricindeki tüm
tipleri belirttim, böylece muhtemelen sorunlarla ilgili mesajları alırken detay
bilgilerle boğuşmayacağız.

Benzer bir şekilde `messageType` alanı da callbackin çağrılacağı mesaj tiplerini
belirtir. Burada hepsini etkinleştirdik, gerekli görmediğiniz anda
istediklerinizi kapatabilirsiniz.

Son olarak `pfnUserCallback` alanı da callback fonksiyonumuza bir işaretçi
tutar. İsterseniz `pUserData` alanına da bir işaretçi atayarak fonksiyondaki
`pUserData` alanına gönderilecek veriyi belirleyebilirsiniz. Bu kısımda mesela
program sınıfımız olan `HelloTriangleApplication`'a bir işaretçi
gönderebilirsiniz.

Doğrulama katmanları mesajlarını ve callbacklerini yapılandırabileceğiniz daha
fazla yol var ama şu anki hali bu dersler için yeterince iyi. Daha fazla bilgi
için [uzantı tanımlarına](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VK_EXT_debug_utils)
göz atabilirsiniz.

Bu struct, `VkDebugUtilsMessengerEXT` objesini oluşturmak için
`vkCreateDebugUtilsMessengerEXT` fonksiyonuna gönderilmeli. Ne yazık ki bu
bir uzantı fonksiyonu olduğundan otomatik olarak yüklenmiyor, adresini
`vkGetInstanceProcAddr` fonksiyonu yardımıyla kendimiz getirtmeliyiz. Bunu arka
planda halledecek olan aracı bir fonksiyonu `HelloTriangleApplication` tanımının
hemen altına ekledim.

```c++
VkResult CreateDebugUtilsMessengerEXT(VkInstance instance, const VkDebugUtilsMessengerCreateInfoEXT* pCreateInfo, const VkAllocationCallbacks* pAllocator, VkDebugUtilsMessengerEXT* pDebugMessenger) {
    auto func = (PFN_vkCreateDebugUtilsMessengerEXT) vkGetInstanceProcAddr(instance, "vkCreateDebugUtilsMessengerEXT");
    if (func != nullptr) {
        return func(instance, pCreateInfo, pAllocator, pDebugMessenger);
    } else {
        return VK_ERROR_EXTENSION_NOT_PRESENT;
    }
}
```

Eğer fonksiyon yüklenemezse `vkGetInstanceProcAddr` fonksiyonu bir `nullptr`
döndürecek. Şimdi uzantı nesnesini oluşturmak için bu fonksiyonu çağıracağız.

```c++
if (CreateDebugUtilsMessengerEXT(instance, &createInfo, nullptr, &debugMessenger) != VK_SUCCESS) {
    throw std::runtime_error("failed to set up debug messenger!");
}
```

Sondan ikinci parametre yine kendi yer ayırıcımız için opsiyonel bir işaretçi,
her zamanki gibi `nullptr` verip geçeceğiz. Diğer parametreler zaten
anlaşılıyor. Hata ayıklama habercimiz, instance'a ve onun katmanlarına özel
olduğundan, bu ilk parametre olarak verilmeli. Bu kullanımı diğer *çocuk*
(child) nesnelerde de göreceğiz.

`VkDebugUtilsMessengerEXT` objesi de bir `vkDestroyDebugUtilsMessengerEXT`
çağrısıyla temizlenmeli. `vkCreateDebugUtilsMessengerEXT` gibi bu fonksiyon da
elle yüklenmeli.

`CreateDebugUtilsMessengerEXT` altına başka bir aracı fonksiyon yazalım:

```c++
void DestroyDebugUtilsMessengerEXT(VkInstance instance, VkDebugUtilsMessengerEXT debugMessenger, const VkAllocationCallbacks* pAllocator) {
    auto func = (PFN_vkDestroyDebugUtilsMessengerEXT) vkGetInstanceProcAddr(instance, "vkDestroyDebugUtilsMessengerEXT");
    if (func != nullptr) {
        func(instance, debugMessenger, pAllocator);
    }
}
```

Bu fonksiyonun sınıfa ait statik bir fonksiyon olduğundan veya sınıfın dışına
yazıldığından emin olun. `cleanup` içerisinde çağırabiliriz:

```c++
void cleanup() {
    if (enableValidationLayers) {
        DestroyDebugUtilsMessengerEXT(instance, debugMessenger, nullptr);
    }

    vkDestroyInstance(instance, nullptr);

    glfwDestroyWindow(window);

    glfwTerminate();
}
```

## Instance oluşturma ve yok etme sırasında hata ayıklama

Her ne kadar doğrulama katmanıyla hata ayıklama kurulumunu yapmış olsak da hala
her durumu kapsayamadık. `vkCreateDebugUtilsMessengerEXT` çağrısı düzgünce
tanımlanmış bir instance gerektiriyor ve `vkDestroyDebugUtilsMessengerEXT`
fonksiyonunun instance yok edilmeden önce çağrılması gerekiyor. Bu da
`vkCreateInstance` ve `vkDestroyInstance` fonksiyonlarında hata ayıklama
yapamamamıza sebep oluyor.

Ama [uzantı dökümanını](https://github.com/KhronosGroup/Vulkan-Docs/blob/master/appendices/VK_EXT_debug_utils.txt#L120)
dikkatlice okursanız, özellikle bu iki çağrı için ayrı bir haberci oluşturmanın
bir yolu olduğunu göreceksiniz. `VkInstanceCreateInfo` structının `pNext` uzantı
alanında `VkDebugUtilsMessengerCreateInfoEXT` structına bir işaretçi göndererek
bunu halledebiliyoruz. İlk olarak haberci oluşturma bilgisiyle ilgili kısmı
ayrı bir fonksiyona taşıyalım:

```c++
void populateDebugMessengerCreateInfo(VkDebugUtilsMessengerCreateInfoEXT& createInfo) {
    createInfo = {};
    createInfo.sType = VK_STRUCTURE_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT;
    createInfo.messageSeverity = VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT;
    createInfo.messageType = VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT;
    createInfo.pfnUserCallback = debugCallback;
}

...

void setupDebugMessenger() {
    if (!enableValidationLayers) return;

    VkDebugUtilsMessengerCreateInfoEXT createInfo;
    populateDebugMessengerCreateInfo(createInfo);

    if (CreateDebugUtilsMessengerEXT(instance, &createInfo, nullptr, &debugMessenger) != VK_SUCCESS) {
        throw std::runtime_error("failed to set up debug messenger!");
    }
}
```

Şimdi bu fonksiyonu `createInstance` içerisinde kullanabiliriz:

```c++
void createInstance() {
    ...

    VkInstanceCreateInfo createInfo{};
    createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
    createInfo.pApplicationInfo = &appInfo;

    ...

    VkDebugUtilsMessengerCreateInfoEXT debugCreateInfo;
    if (enableValidationLayers) {
        createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
        createInfo.ppEnabledLayerNames = validationLayers.data();

        populateDebugMessengerCreateInfo(debugCreateInfo);
        createInfo.pNext = (VkDebugUtilsMessengerCreateInfoEXT*) &debugCreateInfo;
    } else {
        createInfo.enabledLayerCount = 0;

        createInfo.pNext = nullptr;
    }

    if (vkCreateInstance(&createInfo, nullptr, &instance) != VK_SUCCESS) {
        throw std::runtime_error("failed to create instance!");
    }
}
```

`vkCreateInstance` çağrısından önce yok edilmediğinden emin olmak için
`debugCreateInfo` değişkeni if ifadesinin dışında olmalı. Bu yolla
`vkCreateInstance` ve `vkDestroyInstance` çağrıları sırasında otomatik olarak
kullanılıp, sonrasında da yok edilecek ek bir hata ayıklama habercisi oluşturmuş
olduk.

## Deneme

Şimdi doğrulama katmanlarını sahada görmek için bilinçli olarak bir hata
yapalım. `cleanup` içerisindeki `DestroyDebugUtilsMessengerEXT` çağrısını
silelim ve programı çalıştıralım. Kapandığı zaman şuna benzer bir şeyler
görmelisiniz:

![](/images/validation_layer_test.png)

>Eğer hiçbir mesaj görmüyorsanız [kurulumunuzu kontrol edin](https://vulkan.lunarg.com/doc/view/1.2.131.1/windows/getting_started.html#user-content-verify-the-installation).

Eğer mesajı hangi çağrının tetiklediğini görmek isterseniz mesaj callbackine
bir breakpoint ekleyip stack trace'e bakabilirsiniz.

## Yapılandırma

Doğrulama katmanlarının davranışı için, `VkDebugUtilsMessengerCreateInfoEXT`
structında ayarladığımız flaglerden çok daha fazla ayar var. Vulkan SDK klasörü
içerisindeki `Config` klasörüne gidin. Burada katmanları nasıl ayarlayacağınızı
anlatan `vk_layer_settings.txt` dosyasını bulacaksınız.

Kendi programınızının katman ayarlarını değiştirmek için dosyayı projenizin
`Debug` ve `Release` klasörlerini kopyalayıp istediğiniz davranışı elde etmek
için yönergeleri izleyin. Yalnız biz bu derslerin devamında katmanları
varsayılan ayarlarıyla kullandığınızı varsayacağız.

Bu eğitim boyunca bilinçli olarak birkaç hata yapıp, bu hataları yakalamada
doğrulama katmanlarının ne kadar faydalı olduğunu ve Vulkan'la çalışırken tam
olarak ne yaptığınızı bilmenin ne kadar önemli olduğunu göstereceğim. Şimdi
[sistemdeki Vulkan aygıtlarına](!en/Drawing_a_triangle/Setup/Physical_devices_and_queue_families)
bakma zamanı.

[C++ kodu](/code/02_validation_layers.cpp)
