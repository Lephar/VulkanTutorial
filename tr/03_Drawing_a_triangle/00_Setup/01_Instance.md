## Instance oluşturma

İlk yapacağınız şey bir *instance* oluşturarak Vulkan kütüphanesini hazırlamak
olmalı. Instance sizin uygulamanızla Vulkan arasındaki bağlantıdır, oluşturmak
için de uygulamanız hakkında sürücüye birkaç bilgi verilmesi gerekiyor.

Bir `createInstance` fonksiyonu oluşturup `initVulkan` içerisinde çağırarak
başlayalım.

```c++
void initVulkan() {
    createInstance();
}
```

Ayrıca instance tutmak için sınıfa bir değişken ekleyelim:

```c++
private:
VkInstance instance;
```

Şimdi instance oluşturmak için bir struct dolduracağız, uygulamamızla ilgili
bilgileri gireceğiz. Bu bilgiler opsiyonel ama sürücüye optimizasyon için
faydalı bilgiler sağlayabilir, mesela özel davranışlara sahip bilindik bir
grafik motoru kullanıyor olabiliriz. Bu struct `VkApplicationInfo`:

```c++
void createInstance() {
    VkApplicationInfo appInfo{};
    appInfo.sType = VK_STRUCTURE_TYPE_APPLICATION_INFO;
    appInfo.pApplicationName = "Hello Triangle";
    appInfo.applicationVersion = VK_MAKE_VERSION(1, 0, 0);
    appInfo.pEngineName = "No Engine";
    appInfo.engineVersion = VK_MAKE_VERSION(1, 0, 0);
    appInfo.apiVersion = VK_API_VERSION_1_0;
}
```

Daha önceden de bahsettiğimiz gibi Vulkan'daki çoğu structın `sType` değişkenine
elle değer atamamız gerekiyor. Bu ayrıca `pNext` alanına sahip birçok structtan
biri, ileride bir uzantı verisi tutabilir ama biz bunu `nullptr` olarak
bırakacağız.

Vulkan'da çoğu bilgi, fonksiyon parametreleriyle değil structlar ile gönderilir.
Şimdi bir struct daha dolduracağız, instance oluşturmak için gerekli bilgileri
içerecek. Bu seferki opsiyonel değil ve Vulkan sürücüsüne hangi global
uzantılarla hangi doğrulama katmanlarını kullanacağımızı söyleyecek. Burada
globalden kasıt, bu uzantıların tüm programa etki edecek olması, sadece belli
bir cihaza değil. Bir sonraki kısımda daha da netleşecek.

```c++
VkInstanceCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
createInfo.pApplicationInfo = &appInfo;
```

İlk iki parametre gayet net. Sonraki iki katman istenen global uzantıları
belirtecek. Genel bakış bölümünde değindiğimiz gibi, Vulkan platformdan bağımsız
bir API ve pencere yöneticisiyle haberleşmesi için bir uzantıya ihtiyacı var.
GLFW kullanışlı bir gömülü uzantı fonksiyonuyla gelmekte, structa
kaydedebilmemiz için gerekli uzantıları bize döndürebilir:

```c++
uint32_t glfwExtensionCount = 0;
const char** glfwExtensions;

glfwExtensions = glfwGetRequiredInstanceExtensions(&glfwExtensionCount);

createInfo.enabledExtensionCount = glfwExtensionCount;
createInfo.ppEnabledExtensionNames = glfwExtensions;
```

Structın son iki değişkeni, etkinleştirilecek global doğrulama katmanlarını
belirleyecek. Bunlara bir sonraki bölümde ayrıntılı değineceğimiz için şimdilik
boş bırakacağız.

```c++
createInfo.enabledLayerCount = 0;
```

Vulkan'ın instance oluşturması için gerekli her şeyi tanımladığımıza göre artık
`vkCreateInstance` fonksiyonunu çağırabiliriz:

```c++
VkResult result = vkCreateInstance(&createInfo, nullptr, &instance);
```

Gördüğünüz gibi Vulkan'daki nesne oluşturma fonksiyonlarının parametreleri
genelde şu kalıpta:

* Oluşturma bilgisini içeren structa bir pointer
* Kendi yer ayırıcı callback fonksiyonumuza bir pointer, bu derslerde hep `nullptr`
* Yeni oluşturulan objenin işleyicisini tutan değişkene bir pointer

Eğer her şey yolunda gittiyse şu anda `VkInstance` elemanında instance
işleyicisi tutuluyor olmalı. Neredeyse tüm Vulkan fonksiyonları, değeri
`VK_SUCCESS` veya bir hata kodu olan, `VkResult` tipinde bir sonuç döndürür.
Instance oluşturmanın sonucunu kontrol etmek için sonuçu tutmamıza gerek yok,
sadece başarı değişkeniyle karşılaştırabiliriz:

```c++
if (vkCreateInstance(&createInfo, nullptr, &instance) != VK_SUCCESS) {
    throw std::runtime_error("failed to create instance!");
}
```

Şimdi düzgünce instance oluşturulduğundan emin olmak için programınızı
çalıştırın.

## Uzantı desteği kontrolü

Eğer `vkCreateInstance` dökümantasyonuna bakarsanız muhtemelen hatalar arasında
`VK_ERROR_EXTENSION_NOT_PRESENT` (uzantı mevcut değil) diye bir hata kodu
göreceksiniz. Uzantıyı her türlü isteyip eğer bu hatayla karşılaşırsak programı
kapatabiliriz. Pencere sistemi gibi gerekli uzantılar için bu mantıklı bir
yöntem. Peki ya opsiyonel bir özellik sağlayacak uzantıların varlığını kontrol
etmek istersek?

Instance oluşturmadan, desteklenen uzantıların listesini çekebilmek için
`vkEnumerateInstanceExtensionProperties` fonksiyonu mevcut. Bu fonksiyon
parametre olarak, desteklenen uzantı sayısını tutacak değişkene ve uzantıların
detaylarını tutacak bir `VkExtensionProperties` dizisine pointer alıyor. İlk
parametre de  desteklenen uzantıları, doğrulama katmanlarına göre filtreleme
yapmamızı sağlıyor ancak bunu şimdilik yok sayacağız.

Uzantı detaylarını tutacak diziyi oluşturmadan önce kaç uzantı olduğunu bilmemiz
gerekli. Son parametreyi boş bırakarak sadece uzantı sayısını sorgulayabiliriz:

```c++
uint32_t extensionCount = 0;
vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);
```

Şimdi uzantı detaylarını tutacak diziye yer ayırmak için `#include <vector>`
satırını ve aşaıdaki kod parçasını ekleyin:

```c++
std::vector<VkExtensionProperties> extensions(extensionCount);
```

Sonunda uzantı detaylarını sorgulayabiliriz:

```c++
vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, extensions.data());
```

Her `VkExtensionProperties` structı, bir uzantının ismini ve versiyonunu içerir.
Hepsini basit bir for döngüsü ile listeleyebiliriz (`\t` tab karakteriyle
hizalamak için):

```c++
std::cout << "available extensions:\n";

for (const auto& extension : extensions) {
    std::cout << '\t' << extension.extensionName << '\n';
}
```

Vulkan desteğiyle ilgili daha fazla bilgi vermek isterseniz bu kodu
`createInstance` fonksiyonuna ekleyebilirsiniz. Kendinizi denemek için
`glfwGetRequiredInstanceExtensions` fonksiyonundan dönen tüm uzantıların,
desteklenen uzantılar arasında olup olmadığını test eden bir fonksiyon yazmaya
çalışın.

## Temizlik

`VkInstance`, tam programdan çıkmadan önce yok edilmeli. `cleanup` fonksiyonunun
içinde `vkDestroyInstance` fonksiyonu kullanılarak yok edilebilir:

```c++
void cleanup() {
    vkDestroyInstance(instance, nullptr);

    glfwDestroyWindow(window);

    glfwTerminate();
}
```

`vkDestroyInstance` fonksiyonunun parametreleri gayet açıklayıcı. Önceki bölümde
bahsettiğimiz gibi Vulkan'daki yer ayırma ve yeri boşaltma fonksiyonları
opsiyonel bir yer ayırıcı callback parametresine sahip. Her zamanki gibi bunu
yok sayıp `nullptr` göndereceğiz. Sonraki bölümlerde oluşturacağımız diğer tüm
Vulkan kaynakları, instance yok edilmeden önce temizlenmeli.

Instance oluşturduktan sonraki karmaşık adımlara geçmeden önce, [doğrulama katmanlarına](!en/Drawing_a_triangle/Setup/Validation_layers)
göz atıp hata ayıklama seçeneklerimizi incelemenin tam zamanı!

[C++ kodu](/code/01_instance_creation.cpp)
