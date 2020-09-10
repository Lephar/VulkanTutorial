Vulkan platform bağımsız bir API olduğundan kendi başına pencere yöneticisiyle
etkileşemez. Sonuçları ekranda göstermek için Vulkan ile pencere yöneticisi
arasındaki bağlantıyı WSI (pencere sistem entegrasyonu) uzantıları ile
kurmalıyız. Bu bölümde ilkini konuşacağız, o da `VK_KHR_surface`. Çizilmiş
resimleri ekranda gösterebilmemiz için bize soyut bir yüzey olan `VkSurfaceKHR`
nesnesi sunar. Programımızdaki yüzey, GLFW ile oluşturduğumuz pencere tarafından
desteklenecek.

`VK_KHR_surface` uzantısı instance düzeyinde bir uzantıdır ve biz aslında bunu
etkinleştirdik çünkü `glfwGetRequiredInstanceExtensions` tarafından döndürülen
listede zaten var. Bu liste önümüzdeki birkaç bölümde kullanacağımız diğer WSI
uzantılarını da içeriyor.

Pencere yüzeyi, instance'dan hemen sonra oluşturulmalı çünkü fiziksek aygıt
seçimimizi etkileyebilir. Bu işlemi şimdiye kadar ertelememizin sebebi, pencere
yüzeylerinin daha büyük bir konu olan çizim hedefleri ve sunumun bir parçası
olması. Bunları önceki bölümlerde anlatmak, temel kurulumumuzda dağınıklığa yol
açardı. Ayrıca pencere yüzeylerinin, sadece ekran dışı çizimlere ihtiyacınız
varsa, Vulkan için tamamen isteğe bağlı bir konsept olduğunu unutmayın. Vulkan
bunu, görünmez bir pencere oluşturmak gibi (OpenGL için gerekli) hilelere
başvurmadan yapmanıza izin veriyor.

## Penceri yüzeyi oluşturma

Hata ayıklama habercisinin altına `surface` sınıf değişkenini ekleyerek
başlayalım.

```c++
VkSurfaceKHR surface;
```

Her ne kadar `VkSurfaceKHR` nesnesi ve kullanımı platform bağımsız olsa da,
pencere yöneticisi detaylarına bağlı olduğundan oluşturma kısmı bağımsız değil.
Örneğin Windows'ta `HWND` ve `HMODULE` işleyicilerine ihtiyacı var. Bu sebeple
uzantıya platform bağımlı bir eklenti mevcut, Windows'taki ismi
`VK_KHR_win32_surface` ve bu `glfwGetRequiredInstanceExtensions` listesi ile
zaten eklendi.

Windows'ta bu platform bağımlı uzantılarla nasıl yüzey oluşturulduğunu
göstereceğim ama derslerde bu şekilde kullanmayacağız. GLFW gibi bir kütüphane
kullanıp da sonra bu işleri platform bağımlı kodlarla halletmek zaten mantıklı
olmaz. GLFW bizim için `glfwCreateWindowSurface` fonksiyonuyla platform
farklılıklarını hallediyor. Yine de buna bel bağlamadan önce arka planda ne
yaptığını bilmekte fayda var.

Pencere yüzeyi bir Vulkan objesi olduğundan doldurulması gereken
`VkWin32SurfaceCreateInfoKHR` adlı bir structla beraber geliyor. İki önemli
parametresi var: `hwnd` ve `hinstance`. Bunlar pencere ve işlem işleyicileri.

```c++
VkWin32SurfaceCreateInfoKHR createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_WIN32_SURFACE_CREATE_INFO_KHR;
createInfo.hwnd = glfwGetWin32Window(window);
createInfo.hinstance = GetModuleHandle(nullptr);
```

`glfwGetWin32Window` fonksiyonu, GLFW pencere nesnesinden ham bir `HWND` istemek
için kullanılır. `GetModuleHandle` çağrısı da şu anki işlemin `HINSTANCE`
işleyicisini döndürür.

Bundan sonra `vkCreateWin32SurfaceKHR` çağrısıyla yüzey oluşturulabilir.
Parametreler instance, yüzey oluşturma ayrıntıları, yer ayırıcılar ve yüzey
işleyicisini tutacak olan değişkendir. Teknik olarak bir WSI uzantısı fonksiyonu
ama o kadar çok kullanılıyor ki standart Vulkan yükeyicisi tarafından
getirilenlere dahil. Bu yüzden diğer uzantılar gibi elle yüklemeniz gerekmiyor.

```c++
if (vkCreateWin32SurfaceKHR(instance, &createInfo, nullptr, &surface) != VK_SUCCESS) {
    throw std::runtime_error("failed to create window surface!");
}
```

İşlem Linux gibi diğer platformlar da benzer, `vkCreateXcbSurfaceKHR` fonksiyonu
X11 ile oluşturma ayrıntıları olarak XCB bağlantısı ve pencere değişkenin
alıyor.

`glfwCreateWindowSurface` fonksiyonu tam olarak bu işlemleri yapıyor, her
platform için ayrı kod ile. Şimdi bu halini programımıza ekleyeceğiz.
`initVulkan` içerisinde çağrılacak bir `createSurface` fonksiyonu ekleyip
instance oluşturma ve `setupDebugMessenger`'dan hemen sonra çağıralım.

```c++
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

GLFW çağrısı struct yerine basit parametreler alıyor, bu da fonksiyonumuzu
çok açık bir hale getiriyor:

```c++
void createSurface() {
    if (glfwCreateWindowSurface(instance, window, nullptr, &surface) != VK_SUCCESS) {
        throw std::runtime_error("failed to create window surface!");
    }
}
```

Parametreler `VkInstance`, GLFW pencere işaretçisi, yer ayırıcılar ve
`VkSurfaceKHR` değişkenine bir işaretçi. İlgili platform çağrılarından de
`VkResult` sonucunu bize taşıyor. GLFW yüzeyi yok etmek için bir fonksiyon
sağlamıyor ama bunu orijinal API aracılığıyla rahatça yapabiliriz:

```c++
void cleanup() {
        ...
        vkDestroySurfaceKHR(instance, surface, nullptr);
        vkDestroyInstance(instance, nullptr);
        ...
    }
```

Yüzeyin, instance'tan önce yok edildiğine emin olun.

## Sunum desteğini sorgulama

Vulkan gerçeklemesinin pencere sistem entegrasyonunu desteklemesi, sistemdeki
cihazın da desteklediği anlamına gelmez. Bu sebeple `isDeviceSuitable`
fonksiyonunu, cihazın çizdiğimiz resimleri ekranda gösterebileceğinden emin
olacak şekilde genişletmemiz gerekiyor. Sunum, kuyruk özelinde bir özellik
olduğundan, asıl sorun oluşturduğumuz yüzeye sunum yapma desteği olan bir kuyruk
ailesi bulmak oluyor.

Çizim destekli kuyruklarla sunum destekli kuyrukların aynı olmaması mümkün. Bu
yüzden `QueueFamilyIndices` structını, farklı bir sunum kuyruğu olabilecekmiş
gibi güncellemeliyiz.

```c++
struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;
    std::optional<uint32_t> presentFamily;

    bool isComplete() {
        return graphicsFamily.has_value() && presentFamily.has_value();
    }
};
```

Sonra, `findQueueFamilies` fonksiyonunu, pencere yüzeyimize sunum yapabilecek
bir kuruk ailesi bulacak şekilde güncelleyelim. Bunu kontrol edecek fonksiyon
`vkGetPhysicalDeviceSurfaceSupportKHR`, parametre olarak fiziksel aygıt, kuyruk
ailesi indisi ve yüzey alıyor. `VK_QUEUE_GRAPHICS_BIT` ile aynı döngü
içerisinde buna da bir çağrı ekleyin.

```c++
VkBool32 presentSupport = false;
vkGetPhysicalDeviceSurfaceSupportKHR(device, i, surface, &presentSupport);
```

Hemen ardından booleanın değerini kontrol edin ve sunum kuyruk ailesinin
indisini saklayın:

```c++
if (presentSupport) {
    indices.presentFamily = i;
}
```

Bu ikisi çok büyük ihtimalle aynı kuyruk ailesi olacaktır ama program boyunca
ortak bir yaklaşım sağlamak açısından farklıymış gibi davranacağız. İsteseydik
daha yüksek performans için özellikle hem çizimi hem de sunumu destekleyen bir
kuyruğa sahip cihaz seçecek bir mantık ekleyebilirdik.

## Sunum kuyruğunu oluşturma

Mantıksal aygıt oluşturma akışını güncelleyerek sunum kuyruğunu oluşturup bir
`VkQueue` işleyicisine atmak kaldı. Bunun için bir sınıf değişkeni oluşturalım:

```c++
VkQueue presentQueue;
```

Şimdi ailelerin ikisinden de kuyruk oluşturmak için birden fazla
`VkDeviceQueueCreateInfo` structına ihtiyacımız var. Bunu yapmanın şık bir yolu,
ihtiyacımız olan kuyruklar için gerekli tüm farklı kuyruk aileleri için bir set
oluşturmak.

```c++
#include <set>

...

QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

std::vector<VkDeviceQueueCreateInfo> queueCreateInfos;
std::set<uint32_t> uniqueQueueFamilies = {indices.graphicsFamily.value(), indices.presentFamily.value()};

float queuePriority = 1.0f;
for (uint32_t queueFamily : uniqueQueueFamilies) {
    VkDeviceQueueCreateInfo queueCreateInfo{};
    queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
    queueCreateInfo.queueFamilyIndex = queueFamily;
    queueCreateInfo.queueCount = 1;
    queueCreateInfo.pQueuePriorities = &queuePriority;
    queueCreateInfos.push_back(queueCreateInfo);
}
```

Ve `VkDeviceCreateInfo`'yu bu vektöre işaret verecek şekilde güncelleyelim.

```c++
createInfo.queueCreateInfoCount = static_cast<uint32_t>(queueCreateInfos.size());
createInfo.pQueueCreateInfos = queueCreateInfos.data();
```

Eğer kuyruk aileleri aynıysa indisini sadece bir kez atmamız yeterli. Son olarak
da kuyruk işleyicisini getirtmek için bir çağrı ekleyelim:

```c++
vkGetDeviceQueue(device, indices.presentFamily.value(), 0, &presentQueue);
```

Eğer kuyruk aileleri aynıysa muhtemelen iki işleyici de aynı değere sahip
olacaktır. Bir sonraki bölümde takas zincirlerine ve resimleri yüzeyde
göstermemize nasıl olanak sağladıklarına bakacağız.

[C++ kodu](/code/05_window_surface.cpp)
