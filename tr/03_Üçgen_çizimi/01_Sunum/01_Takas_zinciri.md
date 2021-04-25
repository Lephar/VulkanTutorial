Vulkan'da "varsayılan kare arabelleği" (default framebuffer) kavramı yok, bu
yüzden ekranda göstermeden önce çizim yapılacak bu arabellekleri tutacak bir
altyapıya ihtiyaç var. Bu altyapının adı *takas zinciri* (swap chain) ve
Vulkan'da elle oluşturulmalı. Takas zinciri özünde, ekrana sunulmayı bekleyen
resimleri tutan bir kuyruk yapısı. Bizim uygulamamız çizmek için bu yapıdan bir
resim alacak ve sonrasında bu resmi yapıya geri verecek. Kuyruğun tam olarak
nasıl çalıştığı ve kuyruktan alıp ekranda bir resmi göstermenin koşulları bu
takas zincirinin nasıl kurulduğuyla ilgili, ama takas zincirinin asıl amacı
resmin sunulmasıyla ekranın yenilenme hızını senkronize etmek.

## Takas zinciri desteğinin kontrolü

Çeşitli sebeplerden dolayı ekrana direkt olarak çizim yapmayı desteklemeyen
ekran kartları var, örneğin bir grafik kartı sunucular için tasarlanmış olabilir
ve bir görüntü çıkışı yoktur. İkinci olarak resmin sunulması ciddi bir şekilde
pencere sistemine ve pencereyle eşleşmiş olan yüzeyle bağlantılı olduğundan,
Vulkan'ın çekirdek sisteminin bir parçası değil. Cihazın desteğini kontrol
ettikten sonra `VK_KHR_swapchain` cihaz uzantısını etkinleştirmelisiniz.

Bu amaç doğrultusunda `isDeviceSuitable` fonksiyonunu, bu uzantının desteğini de
kontrol edecek şekilde güncelleyelim. `VkPhysicalDevice` tarafından desteklenen
uzantıları nasıl listeleyeceğimizi görmüştük, bu yüzden basit bir şekilde
halledebileceğiz. Vulkan headerında `VK_KHR_SWAPCHAIN_EXTENSION_NAME` adlı
kullanışlı bir makro var, derleyicinin yazım yanlışlarını yakalaması açısından
yardımcı olabilir. `VK_KHR_swapchain` şeklinde tanımlanmış bir değer.

İlk olarak, etkinleştirilecek doğrulama katmanlarının listesine benzer bir liste
tanımlayalım, bu gerekli cihaz uzantılarını içerecek.

```c++
const std::vector<const char*> deviceExtensions = {
    VK_KHR_SWAPCHAIN_EXTENSION_NAME
};
```

Sonra `isDeviceSuitable` içerisinden çağrılacak `checkDeviceExtensionSupport`
adlı ek bir fonksiyon oluşturalım:

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    QueueFamilyIndices indices = findQueueFamilies(device);

    bool extensionsSupported = checkDeviceExtensionSupport(device);

    return indices.isComplete() && extensionsSupported;
}

bool checkDeviceExtensionSupport(VkPhysicalDevice device) {
    return true;
}
```

Fonksiyonu, uzantıları gezip gerekli olanları içerdiğindi kontrol edecek şekilde
güncelleyelim.

```c++
bool checkDeviceExtensionSupport(VkPhysicalDevice device) {
    uint32_t extensionCount;
    vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, nullptr);

    std::vector<VkExtensionProperties> availableExtensions(extensionCount);
    vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, availableExtensions.data());

    std::set<std::string> requiredExtensions(deviceExtensions.begin(), deviceExtensions.end());

    for (const auto& extension : availableExtensions) {
        requiredExtensions.erase(extension.extensionName);
    }

    return requiredExtensions.empty();
}
```

Henüz onaylanmamış olan gerekli uzantıları göstermek için bir string seti
kullanmayı seçtim. Böylece bunların, var olan uzantıların listesini gezerken
kolayca üstünü çizebiliriz. Tabi `checkValidationLayerSupport` fonksiyonundaki
gibi iç içe döngüler de kullanabilirdik. Performans farkı yok sayılabilecek
kadar az. Şimdi kodu çalıştırın ve grafik kartınızın gerçekten bir takas zinciri
oluşturabileceğini görün. Geçen bölümde uygunluğunu kontrol ettiğimiz sunum
kuyruğunun, dolaylı yoldan takas zincirinin de bulunduğunu ima ettiğini de
söyleyebiliriz. Yine de her şeyi açıkça yapmakta fayda var, zaten uzantıların da
elle etkinleştirilmesi gerekiyor.

## Cihaz uzantılarını etkinleştirme

Bir takas zinciri kullanmak için önce `VK_KHR_swapchain` uzantısını
etkinleştirmeliyiz. Uzantıları etkinleştirmek için mantıksal aygıt oluşturma
structına ufak bir değişiklik yapsak yetiyor:

```c++
createInfo.enabledExtensionCount = static_cast<uint32_t>(deviceExtensions.size());
createInfo.ppEnabledExtensionNames = deviceExtensions.data();
```

Bunu yaparken `createInfo.enabledExtensionCount = 0;` satırını da güncellemeyi
unutmayın.

## Takas zinciri desteği ayrıntılarını sorgulama

Sadece takas zincirinin varlığını sorgulamak yeterli değil çünkü bizim pencere
yüzeyimizle uyumlu olmama ihtimali var. Ayrıca bir takas zinciri yaratmak,
instance ve mantıksal cihaz oluşturmaktan çok daha fazla ayar istiyor. Bu
sebeple devam etmeden önce biraz daha ayrıntıyı sorgulamak zorundayız.

Temel olarak kontrol etmemiz gereken üç farklı özellik var:

* Basit yüzey kabiliyetleri (takas zincirinde mümkün olan en az ve en çok resim sayısı, resimlerin en küçük ve en büyük genişlik ve yükseklik ölçüleri)
* Yüzey biçimleri (piksel formatı, renk uzayı)
* Var olan sunum modları

`findQueueFamilies` fonksiyonuna benzer olarak bu detayları sorguladıktan sonra
oradan oraya aktarmak için bir struct kullanacağız. Az önce bahsi geçen üç tip
özellik aşağıdaki struct ve struct listesi biçiminde geliyor:

```c++
struct SwapChainSupportDetails {
    VkSurfaceCapabilitiesKHR capabilities;
    std::vector<VkSurfaceFormatKHR> formats;
    std::vector<VkPresentModeKHR> presentModes;
};
```

Şimdi bu structı dolduracak olan `querySwapChainSupport` adlı yeni bir fonksiyon
ekleyeceğiz.

```c++
SwapChainSupportDetails querySwapChainSupport(VkPhysicalDevice device) {
    SwapChainSupportDetails details;

    return details;
}
```

Bu kısım, bu bilgileri içeren structları nasıl sorgulayacağımızı kapsayacak. Bu
structların anlamı ve tam olarak ne verisi içerdikleri bir sonraki kısımda
tartışılacak.

Basit yüzey kabiliyetleriyle başlayalım. Bu özellikleri sorgulamak kolay ve bize
tek bir `VkSurfaceCapabilitiesKHR` structı olarak döndürülüyorlar.

```c++
vkGetPhysicalDeviceSurfaceCapabilitiesKHR(device, surface, &details.capabilities);
```

Bu fonksiyon, desteklenen kabiliyetleri, belirlediğimiz `VkPhysicalDevice`'ı ve
`VkSurfaceKHR` pencere yüzeyini hesaba katarak belirliyor. Destek sorgulama
fonksiyonlarının tümünün ilk iki parametresi bunlar çünkü takas zincirinin
çekirdeğinde bunlar var.

Bir sonraki adım, desteklenen yüzey formatlarını sorgulamak. Bu bir struct
listesi olduğundan, bildiğiniz 2 fonksiyon çağrılı ritüeli izliyor:

```c++
uint32_t formatCount;
vkGetPhysicalDeviceSurfaceFormatsKHR(device, surface, &formatCount, nullptr);

if (formatCount != 0) {
    details.formats.resize(formatCount);
    vkGetPhysicalDeviceSurfaceFormatsKHR(device, surface, &formatCount, details.formats.data());
}
```

Vektörün, mevcut tüm formatları tutabilecek şekilde yeniden boyutlandırıldığına
emin olun. Ve son olarak, `vkGetPhysicalDeviceSurfacePresentModesKHR` ile aynı
şekilde desteklenen sunum modlarını sorgulayın:

```c++
uint32_t presentModeCount;
vkGetPhysicalDeviceSurfacePresentModesKHR(device, surface, &presentModeCount, nullptr);

if (presentModeCount != 0) {
    details.presentModes.resize(presentModeCount);
    vkGetPhysicalDeviceSurfacePresentModesKHR(device, surface, &presentModeCount, details.presentModes.data());
}
```

Tüm bu detaylar artık structta, `isDeviceSuitable` fonksiyonunu, takas zinciri
desteğinin yeterli olduğunu onaylayacak şekilde genişletelim. Eğer verdiğimiz
pencere yüzeyi için en az bir tane resim formatı, bir tane de sunum modu varsa,
bu ders için takas zinciri desteği yeterli diyebiliriz.

```c++
bool swapChainAdequate = false;
if (extensionsSupported) {
    SwapChainSupportDetails swapChainSupport = querySwapChainSupport(device);
    swapChainAdequate = !swapChainSupport.formats.empty() && !swapChainSupport.presentModes.empty();
}
```

Takas zinciri desteğinin sorgusunu, sadece uzantının varlığından emin olduktan
sonra yapmamız önemli. Fonksiyonun son satırı şuna dönüşecek:

```c++
return indices.isComplete() && extensionsSupported && swapChainAdequate;
```

## Takas zinciri için doğru ayarların seçilmesi

Eğer `swapChainAdequate` koşulları sağlanmışsa destek kesinlikle yeterlidir ama
hala verimliliği değişen bir sürü farklı mod olabilir. Şimdi mümkün olan en iyi
takas zinciri ayarlarını bulmak için birkaç fonksiyon yazalım. Belirleyeceğimiz
üç çeşit ayar var:

* Yüzey formatı (renk derinliği)
* Sunum modu (resimleri ekrana getirmek için gereken koşullar)
* Takas ölçüsü (takas zincirindeki resimlerin çözünürlüğü)

Bu ayarların her biri için aklımızdan en iyi olan değeri belirleyip eğer
destekleniyorsa bunu seçeceğiz, yoksa bir sonraki en iyisini belirlemek için bir
mantık işleyeceğiz.

### Yüzey formatı

Bu ayar için olan fonksiyon aşağıdaki gibi başlayacak. Daha sonra
`SwapChainSupportDetails` structının `formats` değişkenini argüman olarak
atacağız.

```c++
VkSurfaceFormatKHR chooseSwapSurfaceFormat(const std::vector<VkSurfaceFormatKHR>& availableFormats) {

}
```

Her `VkSurfaceFormatKHR` bir `format` ve `colorSpace` değişkeni içeriyor.
`format` renk kanallarını ve tiplerini belirtiyor. Örneğin
`VK_FORMAT_B8G8R8A8_SRGB` B (mavi), G (yeşil), R (kırmızı) ve alfa kanallarını
bu sırayla 8 bit işaretsiz tamsayı olarak, her piksel 32 bit olacak şekilde
tuttuğumuz anlamına gelir. `colorSpace` değişkeni ise
`VK_COLOR_SPACE_SRGB_NONLINEAR_KHR`bitine göre SRGB renk uzayının
desteklenip desteklenmediğini belli eder. Bu değerin, tanımların eski
versiyonlarında `VK_COLORSPACE_SRGB_NONLINEAR_KHR` şeklinde olduğunu bilmenizde
fayda var.

Eğer SRBG mevcutsa renk uzayı olarak bunu kullanacağız çünkü
[daha isabetli görünen renkler sağlıyor](http://stackoverflow.com/questions/12524623/).
Ayrıca resimler için neredeyse standart olmuş durumda, örneğin ileride
kullanacağımız dokular da genelde bu formatta. Bu sebeple renk formatı olarak da
SRBG formatlarından birini kullanmalıyız, bunların en yaygınlarından biri
`VK_FORMAT_B8G8R8A8_SRGB`.

Liste üzerinde dolaşıp tercihlerimizin uygun olup olmadığına bakalım:

```c++
for (const auto& availableFormat : availableFormats) {
    if (availableFormat.format == VK_FORMAT_B8G8R8A8_SRGB && availableFormat.colorSpace == VK_COLOR_SPACE_SRGB_NONLINEAR_KHR) {
        return availableFormat;
    }
}
```

Eğer bu da başarısız olursa formatları ne kadar iyi olduklarına göre puanlayıp
sıralayabiliriz ama genellikle tanımlı olan ilk formatı kullanmak yeterlidir.

```c++
VkSurfaceFormatKHR chooseSwapSurfaceFormat(const std::vector<VkSurfaceFormatKHR>& availableFormats) {
    for (const auto& availableFormat : availableFormats) {
        if (availableFormat.format == VK_FORMAT_B8G8R8A8_SRGB && availableFormat.colorSpace == VK_COLOR_SPACE_SRGB_NONLINEAR_KHR) {
            return availableFormat;
        }
    }

    return availableFormats[0];
}
```

### Sunum modu

Sunum modu belki de bir takas zincirinin en önemli ayarıdır. Resimleri ekranda
göstermek için gereken koşulları belirtir. Vulkan'da kullanabileceğimiz dört
tane sunum modu var:

* `VK_PRESENT_MODE_IMMEDIATE_KHR` (anlık): Uygulamanız tarafından gönderilen
resimler anında ekrana yansıtılır, kırılmalara sebep olabilir.
* `VK_PRESENT_MODE_FIFO_KHR` (ilk giren ilk çıkar): Takas zinciri burada bir
kuyruk yapısındadır. Ekran yenilendiği zaman, kuyruğun ilk resmini alır ve
gösterir, program da çizilen resimleri kuyruğun sonuna ekler. Eğer kuyruk
doluysa program beklemek zorundadır. Bu modern oyunlarda bulunan dikey eş
zamanlamaya (vertical sync) en çok benzeyen modeldir. Ekranın yenilendiği an
"dikey boşluk" (vertical blank) olarak bilinir.
* `VK_PRESENT_MODE_FIFO_RELAXED_KHR` (ilk giren ilk çıkar, gevşetilmiş): Bu
modun bir öncekinden tek farkı, eğer program geç kalmışsa ve kuyruk bir önceki
dikey boşlukta tamamen boşsa, bir sonraki dikey boşluğu beklemek yerine resim
çizildiği anda ekrana gönderilir. Bu da görülür kırılmalara yol açabilir.
* `VK_PRESENT_MODE_MAILBOX_KHR` (posta kutusu): Bu da ikinci modun bir
varyasyonu. Kuyruk doluyken programı bekletmek yerine, kuyrukta bekleyen
resimler yenileriyle değiştirilir. Bu mod üçlü arabellekleme (triple buffering)
için kullanılabilir. Bu mod ekran kırılmasını çoğunlukla engellerken, ikili
arabellekleme (double buffering) kullanan standart dikey eş zamanlamaya göre de
çok daha az gecikmeye sahiptir.

Sadece `VK_PRESENT_MODE_FIFO_KHR` modunun mevcut olması garantidir, bu yüzden
mevcut olan en iyi moda bakan bir fonksiyon yazacağız.

```c++
VkPresentModeKHR chooseSwapPresentMode(const std::vector<VkPresentModeKHR>& availablePresentModes) {
    return VK_PRESENT_MODE_FIFO_KHR;
}
```

Bence üçlü arabellekleme gayet verilebilecek bir taviz. Kırılmayı önlerken düşük
gecikme de sağlayabiliyor. Tam bir sonraki dikey boşluktan hemen önceki en
güncel resimleri sunmaya çalışıyor. Bu yüzden listeyi gezip bu modun varlığını
kontrol edelim:

```c++
VkPresentModeKHR chooseSwapPresentMode(const std::vector<VkPresentModeKHR>& availablePresentModes) {
    for (const auto& availablePresentMode : availablePresentModes) {
        if (availablePresentMode == VK_PRESENT_MODE_MAILBOX_KHR) {
            return availablePresentMode;
        }
    }

    return VK_PRESENT_MODE_FIFO_KHR;
}
```

### Takas ölçüsü

Sadece bir tane önemli özellik kaldı, bunun için son olarak aşağıdaki fonksiyonu
ekleyeceğiz:

```c++
VkExtent2D chooseSwapExtent(const VkSurfaceCapabilitiesKHR& capabilities) {

}
```

Takas ölçüsü, takas zincirindeki resimlerin çözünürlüğüdür ve neredeyse her
zaman (birazdan daha ayrıntılı açıklanacak olan) _iç piksellerine_ çizim
yaptığımız pencerenin çözünürlüğüne eşittir. Mümkün olan çözünürlük aralığı
`VkSurfaceCapabilitiesKHR` structında tanımlıdır. Vulkan bize, `currentExtent`
değişkenindeki genişlik ve yükseklik boyutlarına uyarak pencerenin çözünürlüğüne
uymamızı söylüyor. Ama bazı pencere yöneticileri, `currentExtent`'teki özel bir
değer ile, başka değerler kullanabileceğimizi belirtebilir, bu özel değer de
`uint32_t`'nin alabileceği en büyük değerdir. Bu durumda `minImageExtent` ve
`maxImageExtent` aralığında penceremizin çözünürlüğüne en çok uyan değeri
seçeceğiz. Ancak çözünürlüğü doğru birimle ifade etmeliyiz.

GLFW boyutları ölçerken iki farklı birim kullanıyor: pikseller ve [ekran koordinatları](https://www.glfw.org/docs/latest/intro_guide.html#coordinate_systems).
Örneğin pencereyi oluştururken verdiğimiz `{WIDTH, HEIGHT}` biçimindeki
çözünürlük ekran koordinatları biriminde. Ama Vulkan birim olarak pikselleri
kullanıyor, bu neden takas zinciri ölçüsü de piksellerle ifade edilmeli. Ne
yazık ki yüksek DPI'a (inç başına nokta) sahip bir ekran (Apple'ın Retina
ekranları gibi) kullanıyorsanız, ekran koordinatları ile pikseller biribine
denk gelmez. Yüksek piksel yoğunluğundan dolayı, pencerenin piksel birimli
çözünürlüğü, ekran koordinatı birimli çözünürlüğünden yüksek olacaktır. Yani
Vulkan bizim için takas ölçüsünü düzeltmiyorsa, orijinal `{WIDTH, HEIGHT}`
ikilisi kullanamayız. Bunun yerine `glfwGetFramebufferSize` ile pencerenin
çözünürlüğünü piksel bazında sorgulayıp bunu minimum ve maksimum resim ölçüsüyle
eşleştirmeliyiz.

```c++
#include <cstdint> // Necessary for UINT32_MAX
#include <algorithm> // Necessary for std::min/std::max

...

VkExtent2D chooseSwapExtent(const VkSurfaceCapabilitiesKHR& capabilities) {
    if (capabilities.currentExtent.width != UINT32_MAX) {
        return capabilities.currentExtent;
    } else {
        int width, height;
        glfwGetFramebufferSize(window, &width, &height);

        VkExtent2D actualExtent = {
            static_cast<uint32_t>(width),
            static_cast<uint32_t>(height)
        };

        actualExtent.width = std::max(capabilities.minImageExtent.width, std::min(capabilities.maxImageExtent.width, actualExtent.width));
        actualExtent.height = std::max(capabilities.minImageExtent.height, std::min(capabilities.maxImageExtent.height, actualExtent.height));

        return actualExtent;
    }
}
```

`max` ve `min` fonksiyonları `WIDTH` ve `HEIGHT` değerlerini, izin verilen en
büyük ve en küçük ölçü aralığına sıkıştırmamıza yarıyor.

## Takas zincirinin oluşturulması

Artık alacağımız kararlarda bize çalışma zamanında yardımcı olması için
oluşturduğumuz fonksiyonlar da hazır olduğuna göre çalışan bir takas zinciri
oluşturmak için gereken tüm bilgilere sahibiz.

Bu fonksiyonlarla başlayan ve `initVulkan` içerisinde mantıksal aygıt
yarattıktan hemen sonra çağrılacak olan bir `createSwapChain` fonksiyonu
yazalım:

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
}

void createSwapChain() {
    SwapChainSupportDetails swapChainSupport = querySwapChainSupport(physicalDevice);

    VkSurfaceFormatKHR surfaceFormat = chooseSwapSurfaceFormat(swapChainSupport.formats);
    VkPresentModeKHR presentMode = chooseSwapPresentMode(swapChainSupport.presentModes);
    VkExtent2D extent = chooseSwapExtent(swapChainSupport.capabilities);
}
```

Bu özelliklerin yanı sıra takas zincirinde kaç tane resim olacağına da karar
vermeliyiz. Gerçekleme bize çalışmak için en az kaç resme ihtiyacı olduğunu
söylüyor:

```c++
uint32_t imageCount = swapChainSupport.capabilities.minImageCount;
```

Ama öylece en az sayıda resimle idare etmek demek, bazen sürücünün içinde dönen
işlemlerin bitmesi için üzerine çizmek için yeni bir resmin gelmesini
bekleyebileceğimiz anlamına gelebiliyor. Bu yüzden mümkün olan en düşük sayıdan
en az bir fazla resim istemek önerilir:

```c++
uint32_t imageCount = swapChainSupport.capabilities.minImageCount + 1;
```

Bunu yaparken üst resim limitini de geçmediğimize dikkat etmeliyiz. Burada 0
özel bir değer, bir üst limit olmadığını belirtiyor:

```c++
if (swapChainSupport.capabilities.maxImageCount > 0 && imageCount > swapChainSupport.capabilities.maxImageCount) {
    imageCount = swapChainSupport.capabilities.maxImageCount;
}
```

Vulkan objelerinde gelenek olduğu gibi, takas zincirini oluşturmak için de
büyükçe bir struct doldurmamız gerekiyor. Gayet tanıdık bir şekilde başlıyor:

```c++
VkSwapchainCreateInfoKHR createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_SWAPCHAIN_CREATE_INFO_KHR;
createInfo.surface = surface;
```

Takas zincirinin hangi yüzeye bağlanacağını belirttikten sonra, takas zinciri
resimlerinin ayrıntılarını belirlemeliyiz:

```c++
createInfo.minImageCount = imageCount;
createInfo.imageFormat = surfaceFormat.format;
createInfo.imageColorSpace = surfaceFormat.colorSpace;
createInfo.imageExtent = extent;
createInfo.imageArrayLayers = 1;
createInfo.imageUsage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT;
```

`imageArrayLayers` değişkeni her resmin kaç katmandan oluştuğunu belirtir. Bu
stereoskopik 3D bir uygulama geliştirmiyorsanız her zaman 1 olur. `imageUsage`
bit alanı, bu resmi ne tür işlemlerde kullanacağımızı belirtmek için. Bu derste
direkt olarak resimlerin üzerine çizim yapacağız, bu yüzden renk eklentisi
(color attachment) olarak kullanılacak. Çizim sonrası işlemler (post-processing)
yapmak için çizimleri ayrı birer resme yapmak da mümkün. Böyle bir durumda
`VK_IMAGE_USAGE_TRANSFER_DST_BIT` bitini kullanıp bellek operasyonlarıyla
çizilmiş resmi takas zincirindeki resme taşıyabilirsiniz.

```c++
QueueFamilyIndices indices = findQueueFamilies(physicalDevice);
uint32_t queueFamilyIndices[] = {indices.graphicsFamily.value(), indices.presentFamily.value()};

if (indices.graphicsFamily != indices.presentFamily) {
    createInfo.imageSharingMode = VK_SHARING_MODE_CONCURRENT;
    createInfo.queueFamilyIndexCount = 2;
    createInfo.pQueueFamilyIndices = queueFamilyIndices;
} else {
    createInfo.imageSharingMode = VK_SHARING_MODE_EXCLUSIVE;
    createInfo.queueFamilyIndexCount = 0; // Optional
    createInfo.pQueueFamilyIndices = nullptr; // Optional
}
```

Sonrasında, birden çok kuyruk ailesi tarafından kullanılacak olan takas zinciri
resimlerinin nasıl idare edileceğini belirlemeliyiz. Eğer grafik kuyruk ailesi
ile sunum kuyruk ailesi farklıysa bizim programımızda da durum bu olacak. Takas
zincirindeki resimlere grafik kuyruğundan çizim yapıp sonrasında bunları sunum
zincirinden ekrana göndereceğiz. Birden çok kuyruktan erişilen resimleri idare
etmek için iki yol var:

* `VK_SHARING_MODE_EXCLUSIVE` (tek başına kullanım): Resim bir anda yalnızca tek
kuyruk tarafından kullanılıyor olabilir ve başka bir kuyruk ailesine
aktarılırken bunun sahipliği elle değiştirilmelidir. Bu seçenek en iy
performansı sağlar.
* `VK_SHARING_MODE_CONCURRENT` (eş zamanlı kullanım): Resimler birden çok kuyruk
ailesi tarafından, elle sahiplik aktarımı yapılmaksızın kullanılabilir.

Eğer kuyruk aileleri farklıysa sahiplikle ilgili bölümlerle uğraşmamak için eş
zamanlı modu kullanacağız çünkü bu işlem daha sonra anlatılması daha iyi olacak
kavramlar içeriyor. Eş zamanlı mod `queueFamilyIndexCount` ve
`pQueueFamilyIndices` parametreleri aracılığıyla hangi kuyruk aileleri arasında
paylaşılacağını önceden belirlememizi gerektiriyor. Eğer grafik ve sunum kuyruk
aileleri aynıysa ki neredeyse tüm donanımlarda durum böyledir, tek başına
kullanım modunda kalmalıyız, çünkü eş zamanlı kullanım modu en az iki farklı
kuyruk ailesi belirtmenizi istiyor.

```c++
createInfo.preTransform = swapChainSupport.capabilities.currentTransform;
```

Eğer `capabilities` içindeki `supportedTransforms` arasında saat yönünde 90
derece döndürmek veya yatay düzlemde çevirmek gibi işlemler varsa, bu tür
işlemlerin takas zinciri resimlerine uygulanmasını istediğimizi belirtebiliriz.
Hiçbir dönüşüm uygulamak istemediğimizi belirtmek için şu anki dönüşümü
parametre olarak verebiliriz.

```c++
createInfo.compositeAlpha = VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR;
```

`compositeAlpha` alanı, alfa kanalı kullanılarak pencere yöneticisindeki diğer
pencerelerle bir renk karışımı yapılıp yapılmayacağını belirtiyor. Neredeyse
her zaman alfa kanalını yok sayarız, bu yüzden
`VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR` veriyoruz.

```c++
createInfo.presentMode = presentMode;
createInfo.clipped = VK_TRUE;
```

`presentMode` zaten yeterince açık. Eğer `clipped` değişkenine `VK_TRUE`
atanmışsa bu önü kapalı olan piksellerin renkleriyle ilgilenmediğimiz anlamına
geliyor. Örneğin sistemdeki başka bir pencere bir kısmın önüne geçmiş olabilir.
Eğer gerçekten bu pikselleri de okuyup tahmin edilebilir değerler elde etmenize
gerek yoksa bu kırpmayı (clipping) etkinleştirerek en iyi performansı elde
edebilirsiniz.

```c++
createInfo.oldSwapchain = VK_NULL_HANDLE;
```

Tek bir alan kaldı, o da `oldSwapChain`. Vulkan'da programınız hala çalışırken
takas zincirinin geçersiz veya verimsiz olması ihtimali var, örneğin pencereyi
yeniden boyutlandırma durumunda. Böyle bir durumda takas zinciri sıfırdan tekrar
oluşturulmalı ve eski haline burada bir referans veirlmeli. Bu
[ileriki bir bölümde](!en/Drawing_a_triangle/Swap_chain_recreation)
öğreneceğimiz karışık bir konu. Şimdilik sadece tek bir takas zinciri
yaratacağımızı varsayalım.

Şimdi `VkSwapchainKHR` tutacak bir sınıf değişkeni ekleyelim:

```c++
VkSwapchainKHR swapChain;
```

Şu an takas zincirini yaratmak için `vkCreateSwapchainKHR` fonksiyonunu
çağırmamız yeterli:

```c++
if (vkCreateSwapchainKHR(device, &createInfo, nullptr, &swapChain) != VK_SUCCESS) {
    throw std::runtime_error("failed to create swap chain!");
}
```

Parametreler mantıksal aygıt, takas zinciri oluşturma bilgisi, yer ayırıcı,
ve işleyiciyi tutacak değişkene bir işaretçi. Ve tahmin edeceğiniz gibi cihazı
yok etmeden önce `vkDestroySwapchainKHR` kullanarak temizlememiz gerekiyor:

```c++
void cleanup() {
    vkDestroySwapchainKHR(device, swapChain, nullptr);
    ...
}
```

Şimdi takas zincirinin düzgünce oluşturulduğundan emin olmak için programınızı
çalıştırın. Eğer şu an `vkCreateSwapchainKHR`'da erişim ihlali hatası
alıyorsanız veya
`Failed to find 'vkGetInstanceProcAddress' in layer SteamOverlayVulkanLayer.dll`
gibi bir mesajla karşılaşıyorsanız Steam overlay hakkındaki [SSS kısmına](!en/FAQ)
bakınız.

Eğer doğrulama katmanları etkinse `createInfo.imageExtent = extent;` satırını
silmeyi deneyin. Doğrulama katmanlarının anında bu hatayı yakalayıp faydalı bir
mesaj bastırdığını göreceksiniz.

![](/images/swap_chain_validation_layer.png)

## Takas zinciri resimlerini getirtme

Takas zinciri oluşturuldu, tek kalan içerisindeki `VkImage`'ların işleyicilerini
getirtmek. Çizim işlemleriyle ilgili olan bölümlerde bunlara referans vereceğiz.
İşleyicileri tutmak için sınıf değişkeni ekleyelim:

```c++
std::vector<VkImage> swapChainImages;
```

Resimler kütüphane tarafından oluşturuldu ve takas zinciri yok edildiğinde
bunlar da otomatik olarak yok edilecektir. Yani herhangi bir temizleme koduna
ihtiyacımız yok.

İşleyicileri getirtme kodunu `createSwapChain`'in en sonuna ekliyorum,
`vkCreateSwapchainKHR` çağrısının hemen altına. Bunları getirtmek, Vulkan'daki
diğer obje dizileri getirtmeye çok benzer. Bir önceki bölümde sadece minimum
resim sayısını belirttiğimizi unutmayın, yani kütüphane bize daha fazla resme
sahip bir takas zinciri döndürebilir. Bu yüzden ilk önce
`vkGetSwapchainImagesKHR` çağrısıyla resimlerin nihai sayısını sorgulayacağız.
Sonrasında vektörü yeniden boyutlandırıp bunu tekrar çağırarak işleyicileri
alabiliriz.

```c++
vkGetSwapchainImagesKHR(device, swapChain, &imageCount, nullptr);
swapChainImages.resize(imageCount);
vkGetSwapchainImagesKHR(device, swapChain, &imageCount, swapChainImages.data());
```

Son bir şey kaldı, sonraki bölümlerde ihtiyacımız olacağı için takas zinciri
resimleri için seçtiğimiz formatı ve ölçüyü sınıf değişkenlerinde saklayacağız.

```c++
VkSwapchainKHR swapChain;
std::vector<VkImage> swapChainImages;
VkFormat swapChainImageFormat;
VkExtent2D swapChainExtent;

...

swapChainImageFormat = surfaceFormat.format;
swapChainExtent = extent;
```

Şimdi üzerine çizim yapıp ekrana sunabileceğimiz bir resim topluluğumuz var.
Bir sonraki bölümde bu resimleri nasıl çizim hedefi olarak atayacağımızı görüp
daha sonrasında da asıl grafik boru hattına ve çizim komutlarına bakacağız!

[C++ kodu](/code/06_swap_chain_creation.cpp)
