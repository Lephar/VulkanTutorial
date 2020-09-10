## Fiziksel aygıt seçimi

VkInstance aracılığıyla Vulkan kütüphanesini kurduktan sonra sistemde
istediğimiz özellikleri sağlayan bir fiziksel aygıt bulup seçmeliyiz. Aslında
istediğimiz kadar kart seçip aynı anda çalıştırabiliriz ama biz bu derslerde
ihtiyaçlarımızı karşılayan ilk kartla yetineceğiz.

`pickPhysicalDevice` adlı bir fonksiyon ekleyip `initVulkan` içerisinde çağırın.

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    pickPhysicalDevice();
}

void pickPhysicalDevice() {

}
```

Seçeceğimiz grafik kartı, sınıfa ekleyeceğimiz VkPhysicalDevice işleyicisinde
tutulacak. Bu obje VkInstance yok edildiğinde kendiliğinden temizlenecek, bu
nedenle `cleanup` fonksiyonuna yeni bir şey eklemeyeceğiz.

```c++
VkPhysicalDevice physicalDevice = VK_NULL_HANDLE;
```

Grafik kartlarını listelemek, uzantıları listelemeye çok benziyor, yine sadece
sayısını sorgulayarak başlayacağız.

```c++
uint32_t deviceCount = 0;
vkEnumeratePhysicalDevices(instance, &deviceCount, nullptr);
```

Eğer Vulkan destekli 0 cihaz görünüyorsa daha ileri gitmenin bir mantığı yok.

```c++
if (deviceCount == 0) {
    throw std::runtime_error("failed to find GPUs with Vulkan support!");
}
```

Eğer durum bu değilse tüm VkPhysicalDevice'ları tutacak bir diziye yer
ayırabiliriz.

```c++
std::vector<VkPhysicalDevice> devices(deviceCount);
vkEnumeratePhysicalDevices(instance, &deviceCount, devices.data());
```

Şimdi her birini değerlendirip ihtiyacımız olan işlemler için uygun olup
olmadıklarına bakacağız. Çünkü her grafik kartı eşit yaratılmamıştır. Bunun için
yeni bir fonksiyon ekleyeceğiz:

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    return true;
}
```

Ve herhangi bir fiziksel cihazın bu şartları sağlayıp sağlamadığını kontrol
ediyoruz. İhtiyaçlarımızı daha sonra buraya ekleyeceğiz:

```c++
for (const auto& device : devices) {
    if (isDeviceSuitable(device)) {
        physicalDevice = device;
        break;
    }
}

if (physicalDevice == VK_NULL_HANDLE) {
    throw std::runtime_error("failed to find a suitable GPU!");
}
```

Bir sonraki kısımda, `isDeviceSuitable` içerisinde kontrol edeceğimiz ilk
gereksinimimizi ekleyeceğiz. İleriki bölümlerde daha fazla Vulkan özelliği
kullanmaya başladıkça da bu fonksiyonu daha fazla kontrol yapacak şekilde
genişleteceğiz.

## Temel cihaz uygunluk kontrolleri

Bir cihazın uygunluğunu kontrol etmeye birkaç detayı sorgulayarak
başlayabiliriz. Cihaz ismi, tipi ve Vulkan versiyonu gibi basit özellikleri
vkGetPhysicalDeviceProperties yardımıyla sorgulayabiliriz:

```c++
VkPhysicalDeviceProperties deviceProperties;
vkGetPhysicalDeviceProperties(device, &deviceProperties);
```

Doku sıkıştırması, 64 bit kayan noktalı sayılar ve çoklu görüntü kapısında çizim
(VR için kullanışlı) gibi isteğe bağlı özellikler de vkGetPhysicalDeviceFeatures
kullanılarak sorgulanabilir:

```c++
VkPhysicalDeviceFeatures deviceFeatures;
vkGetPhysicalDeviceFeatures(device, &deviceFeatures);
```

Cihazlardan sorgulanabilecek daha farklı özellikler de var. Cihaz belleği ve
kuyruk aileleriyle ilgili bölümlerde tartışacağız (bir sonraki kısma
bakabilirsiniz).

Örnek olarak diyelim ki uygulamamızın sadece geometri gölgeleyicilerini
(geometry shaders) destekleyen harici bir ekran kartında çalışmasını istiyoruz.
`isDeviceSuitable` fonksiyonumuz şöyle olurdu:

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    VkPhysicalDeviceProperties deviceProperties;
    VkPhysicalDeviceFeatures deviceFeatures;
    vkGetPhysicalDeviceProperties(device, &deviceProperties);
    vkGetPhysicalDeviceFeatures(device, &deviceFeatures);

    return deviceProperties.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU &&
           deviceFeatures.geometryShader;
}
```

Sadece cihaz uygun mu değil mi diye bakıp ilk uygun cihazı seçmek yerine,
cihazları puanlandırıp en yüksek puana sahip olanı da seçebilirsiniz. Bu şekilde
harici ekran kartına daha yüksek puan verip onun seçilme şansını
artırabilirsiniz. Buna benzer bir şeyi aşağıdaki gibi gerçekleyebilirsiniz:

```c++
#include <map>

...

void pickPhysicalDevice() {
    ...

    // Use an ordered map to automatically sort candidates by increasing score
    std::multimap<int, VkPhysicalDevice> candidates;

    for (const auto& device : devices) {
        int score = rateDeviceSuitability(device);
        candidates.insert(std::make_pair(score, device));
    }

    // Check if the best candidate is suitable at all
    if (candidates.rbegin()->first > 0) {
        physicalDevice = candidates.rbegin()->second;
    } else {
        throw std::runtime_error("failed to find a suitable GPU!");
    }
}

int rateDeviceSuitability(VkPhysicalDevice device) {
    ...

    int score = 0;

    // Discrete GPUs have a significant performance advantage
    if (deviceProperties.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU) {
        score += 1000;
    }

    // Maximum possible size of textures affects graphics quality
    score += deviceProperties.limits.maxImageDimension2D;

    // Application can't function without geometry shaders
    if (!deviceFeatures.geometryShader) {
        return 0;
    }

    return score;
}
```

Bu eğitim için bunların hepsini kodlamanıza gerek yok ama ilerde cihaz
seçiminizi nasıl tasarlayabileceğinizi göstermek için faydalı bir kısım. Tabi
tüm kartların isimlerini gösterip kullanıcının seçmesine de izin verebilirsiniz.

Daha yeni başladığımız için Vulkan desteği ihtiyacımız olan tek şey, bu yüzden
gördüğümüz ilk GPU'yu seçeceğiz:

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    return true;
}
```

Bir sonraki bölümde ihtiyacımız olacak ilk gerçek özelliği göreceğiz.

## Kuyruk aileleri

Üzerinde kısaca durmuştuk, Vulkan'daki çizimler, doku yükelemeleri gibi tüm
işlemler, komutun kuyruğa gönderilmesini gerektirir. Farklı *kuyruk ailelerinden*
türeyen farklı kuyruklar var ve bunların her biri komutların belirli bir alt
kümesini desteklemekte. Örneğin sadece hesap komutlarına izin veren veya sadece
bellek işlemleriyle ilgili komutlara izin veren kuyruk aileleri olabilir.

Cihaz tarafında hangi kuyruk kümelerinin desteklendiğini ve bunlardan
hangilerinin bizim istediğimiz komutları desteklediğini kontrol etmeliyiz. Bunun
için ihtiyacımız olan tüm kuyruk ailelerini getiren `findQueueFamilies` adlı bir
fonksiyon ekleyelim.

Şimdilik sadece grafik komutlarını destekleyen bir kuyruğa ihtiyacımız var, yani
fonksiyon şu şekilde olacak:

```c++
uint32_t findQueueFamilies(VkPhysicalDevice device) {
    // Logic to find graphics queue family
}
```

Ama sonraki bölümlerde başka kuyruklara da bakacağımız için şimdiden bunların
indislerini bir structta toplayacak şekilde hazırlanmamızda fayda var:

```c++
struct QueueFamilyIndices {
    uint32_t graphicsFamily;
};

QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device) {
    QueueFamilyIndices indices;
    // Logic to find queue family indices to populate struct with
    return indices;
}
```

Peki ya bir kuyruk ailesi desteklenmiyorsa? `findQueueFamilies` fonksiyonunda
bir exception fırlatabilirdik ama bu fonksiyon cihazın uygunluğuyla ilgili
kararlar almak için doğru yer değil. Örneğin özelleşmiş bir transfer kuyruk
ailesini *tercih edebiliriz* ama varlığı şart olmayabilir. Bu yüzden belli bir
kuyruk ailesinin bulunup bulunmadığını belirtmek için bir yol belirlemeliyiz.

Bir kuyruk ailesinin olmadığını, belli bir sayı ile belirtmemiz mümkün değil
çünkü teorik olarak `uint32_t`'in alabileceği tüm değerler geçerli bir kuyruk
ailesini indisliyor olabilir, buna 0 da dahil. Neyse ki C++17 ile, bir değerin
var olup olmadığını belirten bir veri yapısı eklendi:

```c++
#include <optional>

...

std::optional<uint32_t> graphicsFamily;

std::cout << std::boolalpha << graphicsFamily.has_value() << std::endl; // false

graphicsFamily = 0;

std::cout << std::boolalpha << graphicsFamily.has_value() << std::endl; // true
```

`std::optional`, bir değer atanana kadar içinde hiçbir şey tutmayan bir yapı.
Herhangi bir zamanda `has_value()` fonksiyonunu çağırarak içinde bir değer tutup
tutmadığını sorgulayabilirsiniz. Mantığı şu şekilde değiştirelim:

```c++
#include <optional>

...

struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;
};

QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device) {
    QueueFamilyIndices indices;
    // Assign index to queue families that could be found
    return indices;
}
```

Şimdi `findQueueFamilies` fonksiyonunu yazmaya başlayabiliriz:

```c++
QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device) {
    QueueFamilyIndices indices;

    ...

    return indices;
}
```

Kuyruk ailelerinin listesini getirtme işlemini tam da tahmin ettiğiniz gibi
`vkGetPhysicalDeviceQueueFamilyProperties` kullanarak yapacağız:

```c++
uint32_t queueFamilyCount = 0;
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, nullptr);

std::vector<VkQueueFamilyProperties> queueFamilies(queueFamilyCount);
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, queueFamilies.data());
```

VkQueueFamilyProperties structı kuyruk ailesiyle ilgili, desteklenen işlem
tipleri ve bu aileden oluşturulabilecek kuyruk sayısı gibi bir takım ayrıntıyı
tutmakta. `VK_QUEUE_GRAPHICS_BIT` destekleyen en az bir aile bulmalıyız.

```c++
int i = 0;
for (const auto& queueFamily : queueFamilies) {
    if (queueFamily.queueFlags & VK_QUEUE_GRAPHICS_BIT) {
        indices.graphicsFamily = i;
    }

    i++;
}
```

Şimdi süslü bir kuyruk ailesi bulma fonksiyonumuz olduğuna göre, bunu
`isDeviceSuitable` fonksiyonu içerisinde, cihazın bizim kullanmak istediğimiz
komutları işleyebileceğinden emin olmak için kullanabiliriz:

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    QueueFamilyIndices indices = findQueueFamilies(device);

    return indices.graphicsFamily.has_value();
}
```

Bunu biraz daha kullanışlı bir hale getirmek için structın içine jenerik bir
kontrol ekleyelim:

```c++
struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;

    bool isComplete() {
        return graphicsFamily.has_value();
    }
};

...

bool isDeviceSuitable(VkPhysicalDevice device) {
    QueueFamilyIndices indices = findQueueFamilies(device);

    return indices.isComplete();
}
```

Bu özelliği, `findQueueFamilies` fonksiyonunu zamanında bitirmek için de
kullanabiliriz:

```c++
for (const auto& queueFamily : queueFamilies) {
    ...

    if (indices.isComplete()) {
        break;
    }

    i++;
}
```

Güzel, şu anda doğru fiziksel aygıtı bulmak için gerekenlerin hepsi bu kadar!
Bir sonraki adım fiziksel aygıtla arayüzlemek üzere [mantıksal aygıt oluşturma](!en/Drawing_a_triangle/Setup/Logical_device_and_queues).

[C++ kodu](/code/03_physical_device_selection.cpp)
