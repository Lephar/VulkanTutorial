## Giriş

Bir fiziksel aygıt seçtikten sonra, bunla arayüzlemek için bir *mantıksal aygıt*
kurmalıyız. Mantıksal aygıt oluşturma süreci instance ile benzer ve kullanmak
istediğimiz özellikleri belirtmeye yarıyor. Hangi kuyruk ailelerinin bulunduğunu
önceki bölümde sorguladık, kuyrukları da burada oluşturacağız. Hatta farklı
gereksinimleriniz varsa tek bir fiziksel aygıttan, birden çok mantıksal aygıt da
yaratmanız mümkün.

Mantıksal aygıt işleyicisini saklayacak bir sınıf değişkeni tanımlayarak
başlayalım:

```c++
VkDevice device;
```

Sonra, `initVulkan` içerisinden çağrılacak bir `createLogicalDevice` fonksiyonu
ekleyelim:

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    pickPhysicalDevice();
    createLogicalDevice();
}

void createLogicalDevice() {

}
```

## Oluşturulacak kuyrukların belirtilmesi

Mantıksal aygıt oluşturulması yine structlar içerisinde birkaç ayrıntı vermeyi
içeriyor, bunlardan ilki de `VkDeviceQueueCreateInfo` olacak. Bu struct, bir
kuyruk ailesinden kaç tane kuyruk oluşturmak istediğimizi içeriyor. Şimdilik
sadece bir tane grafik destekli kuyruğa ihtiyacımız var.

```c++
QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

VkDeviceQueueCreateInfo queueCreateInfo{};
queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
queueCreateInfo.queueFamilyIndex = indices.graphicsFamily.value();
queueCreateInfo.queueCount = 1;
```

Şu anki sürücüler her kuyruk ailesinden az sayıda kuyruk oluşturmanıza izin
vermekte, zaten birden fazla kuyruğa da genelde ihtiyacınız olmaz. Bunun sebebi
tüm komut arabelleklerini birçok iş parçacığından birden oluşturup asıl iş
parçacığından az yüke sahip tek bir çağrı ile hepsini gönderebilmemiz.

Vulkan `0.0` ile `1.0` arasındaki kayan noktalı sayılar aracılığıyla, kuyruklara
öncelik atayıp komut arabelleklerinin çalışırkenki zamanlama algoritmalarına
müdahale etmenize olanak sağlamakta. Tek bir kuyruğunuz varken bile bu gerekli:

```c++
float queuePriority = 1.0f;
queueCreateInfo.pQueuePriorities = &queuePriority;
```

## Kullanılan cihaz özelliklerinin belirtilmesi

Vereceğimiz bir sonraki bilgi kullanacağımız cihaz özellikleri olacak. Bunlar
geçen bölümde  `vkGetPhysicalDeviceFeatures` aracılığıyla desteğini
sorguladığımız geometri gölgelendirici gibi özellikler. Şimdilik özel bir şeye
ihtiyacımız yok, sadece tanımlayıp her şeyi `VK_FALSE` yapabiliriz. Vulkan'la
daha ilginç şeyler yapmaya başladığımızda bu structa tekrar döneceğiz.

```c++
VkPhysicalDeviceFeatures deviceFeatures{};
```

## Mantıksal aygıtı oluşturma

Önceki iki struct hazır olduğuna göre asıl `VkDeviceCreateInfo` structını
doldurmaya başlayabiliriz.

```c++
VkDeviceCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO;
```

İlk olarak kuyruk oluşturma bilgisi ve cihaz özellikleri structlarına birer
işaretçi verelim:

```c++
createInfo.pQueueCreateInfos = &queueCreateInfo;
createInfo.queueCreateInfoCount = 1;

createInfo.pEnabledFeatures = &deviceFeatures;
```

Geri kalan bilgiler `VkInstanceCreateInfo` structına benziyor, uzantıları ve
doğrulama katmanlarını belirtmemiz gerekiyor. Aradaki fark, bu seferkilerin
cihaz özelinde olması.

Cihaz özelindeki uzantılara örnek olarak `VK_KHR_swapchain` verilebilir, bu
cihazda çizdiğimiz resimlerin bir pencerede gösterilmesini sağlıyor. Sistemde
bu özellikten yoksun cihazların olması muhtemel, örneğin cihaz sadece hesaplama
işlemlerini destekliyor olabilir. Takas zinciri bölümünde bu uzantıya tekrar
geleceğiz.

Vulkan'ın önceki versiyonları instance ve cihaz özelindeki doğrulama katmanları
arasında bir ayrıma sahipti ama [durum artık böyle değil](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#extendingvulkan-layers-devicelayerdeprecation).
Bu da `VkDeviceCreateInfo` structının `enabledLayerCount` ve
`ppEnabledLayerNames` değişkenlerinin güncel sürücüler tarafından yok sayıldığı
anlamına geliyor. Ama bunlara yine de atama yapmak, eski sürümlerle de uyumlu
olmak için iyi bir fikir:

```c++
createInfo.enabledExtensionCount = 0;

if (enableValidationLayers) {
    createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
    createInfo.ppEnabledLayerNames = validationLayers.data();
} else {
    createInfo.enabledLayerCount = 0;
}
```

Cihaz özelinde bir uzantıya şimdilik ihtiyacımız olamayacak.

Bu kadar. Gayet açık bir şekilde isimlendirilmiş olan `vkCreateDevice` çağrısını
kullanarak mantıksal aygıtımızı oluşturmaya hazırız.

```c++
if (vkCreateDevice(physicalDevice, &createInfo, nullptr, &device) != VK_SUCCESS) {
    throw std::runtime_error("failed to create logical device!");
}
```

Parametreler sırasıyla arayüzlenecek olan fiziksel cihaz, az önce belirttiğimiz
kuyruk ve kullanım bilgisi, isteğe bağlı yer ayırıcı callback işaretçisi ve
mantıksal aygıtı saklayacak olan işleyiciye bir işaretçi. Instance oluşturma
fonksiyonu gibi bu da, olmayan bir uzantının etkinleştirilmesi veya
desteklenmeyen bir özelliğin istenmesi gibi durumlara bağlı olarak bir hata kodu
döndürmekte.

Bu aygıt `cleanup` içerisinde `vkDestroyDevice` fonksiyonuyla yok edilmeli:

```c++
void cleanup() {
    vkDestroyDevice(device, nullptr);
    ...
}
```

Mantıksal aygıtlar, instance'lar ile direkt olarak bir etkileşim içinde
değildir, bu yüzden parametreler arasında instance yok.

## Kuyruk işleyicilerini getirtme

Kuyruklar, mantıksal aygıtlarla birlikte otomatik olarak oluşturulurlar ama
henüz bunlarla etkileşmek için bir işleyiciye sahip değiliz. İlk olarak grafik
kuyruğunun işleyicisini tutmak için bir sınıf değişken oluşturun:

```c++
VkQueue graphicsQueue;
```

Aygıt kuyrukları, aygıtlar yok edildiğinde kendiliğinden temizlenir, bu yüzden
`cleanup`'a bir şey eklememize gerek yok.

Her kuyruk ailesi için `vkGetDeviceQueue` fonksiyonu ile kuyruk işleyicisini
getirtebiliriz. Parametreler mantıksal aygıt, kuyruk ailesi, kuyruk indisi ve
kuyruk işleyicisini tutacak değişkene bir işaretçidir. Bu aileden tek bir kuyruk
oluşturacağımızdan indis olarak basitçe `0` kullanabiliriz.

```c++
vkGetDeviceQueue(device, indices.graphicsFamily.value(), 0, &graphicsQueue);
```

Mantıksal aygıt ve kuyruk işleyicilerimizle artık grafik kartını bir şeyler
yapmak için kullanmaya başlayabiliriz! Sonraki birkaç bölümde, sonuçları pencere
yöneticisine sunabilmek için gereken kaynakları ayarlayacağız.

[C++ kodu](/code/04_logical_device.cpp)
