Grafik boru hattında herhangi bir `VkImage`'ı kullanabilmek için, buna takas
zincirindekiler de dahil, `VkImageView` objesine sarmalıyız. Resim görünümü tam
anlamıyla resim üzerine geçirilmiş bir görünüm. Resme nasıl erişileceğini ve
hangi parçalarına erişileceğini belirtir. Örneğin 2 boyutlu bir doku resmine
hiç mipmap seviyesi olmayan bir derinlik dokusu olarak davranmamızı
söyleyebilir.

Bu bölümde, ileride renk hedefi olarak kullanabilmemiz için tüm takas zinciri
resimlerini birer resim görüntüsüne saracak bir `createImageViews` fonksiyonu
oluşturacağız.

İlk olarak resim görüntülerini saklayacağımız bir sınıf değişkeni tanımlayalım:

```c++
std::vector<VkImageView> swapChainImageViews;
```

`createImageViews` fonksiyonunu oluşturun ve takas zinciri oluşturma
fonksiyonundan hemen sonra çağırın.

```c++
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

İlk yapmamız gereken şey, listeyi tüm resim görüntülerimizin sığacağı şekilde
yeniden boyutlandırmak olacak:

```c++
void createImageViews() {
    swapChainImageViews.resize(swapChainImages.size());

}
```

Sonrasında tüm takas zinciri resimlerini gezen bir döngü kuracağız.

```c++
for (size_t i = 0; i < swapChainImages.size(); i++) {

}
```

Resim görünümü oluşturma parametreleri `VkImageViewCreateInfo` structında
belirlenir. İlk birkaç parametre gayet net.

```c++
VkImageViewCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
createInfo.image = swapChainImages[i];
```

`viewType` ve `format` alanları resim verisinin nasıl yorumlanması gerektiğini
belirtir. `viewType` parametresi resimlere 1 boyutlu, 2 boyutlu, 3 boyutlu
dokular veya küp haritaları (cube maps) olarak davranmanıza olanak sağlar.

```c++
createInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
createInfo.format = swapChainImageFormat;
```

`components` alanı, renk kanallarını dönüştürmemizi (swizzle) sağlar. Örneğin
tek renkli (monochrome) bir doku için tüm renk kanallarını kırmızı kanalına
bağlayabilirsiniz. Ayrıca bir kanalı `0` ve `1` sabitlerine eşleyebilirsiniz.
Bizim durumumuzda varsayılan eşleme kullanılacak.

```c++
createInfo.components.r = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.g = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.b = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.a = VK_COMPONENT_SWIZZLE_IDENTITY;
```

`subresourceRange` alanı resmin amacını ve hangi parçalarına erişileceğini
belirtecek. Bizim resimlerimiz herhangi bir mipmap seviyesine veya çoklu bir
katmana sahip olmayan bir renk hedefi olarak kullanılacak.

```c++
createInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
createInfo.subresourceRange.baseMipLevel = 0;
createInfo.subresourceRange.levelCount = 1;
createInfo.subresourceRange.baseArrayLayer = 0;
createInfo.subresourceRange.layerCount = 1;
```

Eğer bir stereographic 3D uygulama üzerine çalışıyor olsaydık birden fazla
katmanlı bir takas zinciri oluştururduk. Farklı katmanlar aracılığıyla sağ ve
sol gözün gördüğü alanları temsil eden kısımlara erişmek için her resim başına
birden çok resim görünümü oluşturabilirdik.

Şu anda bu görünümleri oluşturmak için sadece `vkCreateImageView` çağrısı kaldı:

```c++
if (vkCreateImageView(device, &createInfo, nullptr, &swapChainImageViews[i]) != VK_SUCCESS) {
    throw std::runtime_error("failed to create image views!");
}
```

Resimlerin aksine, resim görünümleri bizim tarafımızdan elle oluşturuldu, bu
yüzden benzer bir döngü ekleyerek programın sonunda bunları temizlemeliyiz:

```c++
void cleanup() {
    for (auto imageView : swapChainImageViews) {
        vkDestroyImageView(device, imageView, nullptr);
    }

    ...
}
```

Bir resim görünümü, bir resmi doku olarak kullanmaya başlamak için yeterli ama
henüz çizim hedefi olarak kullanılmaya hazır değiller. Bunun için bir üçkağıt
daha gerekiyor, kare arabellekleri... Ama öncesinde grafik boru hattını
kurmalıyız.

[C++ kodu](/code/07_image_views.cpp)
