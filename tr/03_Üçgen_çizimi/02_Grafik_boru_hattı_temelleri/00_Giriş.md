Önümüzdeki birkaç bölüm boyunca, ilk üçgenimizi çizmek için yapılandırılmış bir
grafik boru hattı dizeceğiz. Grafik boru hattı, meshinizin vertekslerini ve
dokularını alıp çizim hedeflerindeki pikseller haline getiren işlemler
dizisidir. Basitleştirilmiş bir özet aşağıda gösterilmiştir:

![](/images/vulkan_simplified_pipeline.svg)

*Girdi çeviricisi* (input assembler), belirlediğiniz arabelleklerden ham verteks
verisini toplar. Gerekirse indis arabelleği kullanarak, verteks verisini
çoğaltmaya ihtiyaç duymadan belli elemanları tekrarlamak için kullanılabilir.

*Köşe gölgelendirici* (vertex shader) her köşe (vertex) için çalışır ve
genelde pozisyonları model uzayından ekran uzayına taşımak için gereken
dönüşümleri uygular. Ayrıca vertekse özel verileri de boru hattına taşır.

*Tesselasyon gölgelendiriciler* (tessellation shaders) mesh kalitesini artırmak
için belli kurallar dahilinde geometrileri parçalara bölmeye yarar. Bu genelde
tuğla duvarlar, merdivenler gibi yüzeylerde, yakındayken daha az düz görünsünler
diye kullanılır.

*Geometri gölgelendirici* (geometry shader) her temel yapı (üçgen, çizgi, nokta)
için çalışır ve bunları yok etmek veya daha küçük şekillere bölmek için
kullanılabilir. Bu tesselasyon gölgelendiriciye benzer ama çok daha esnektir.
Ancak günümüzde çok az uygulamada kullanılır çünkü çoğu grafik kartında pek iyi
bir performansa sahip değildir, Intel'in bütünleşik ekran kartları hariç.

*Pikselleştirme* (rasterisation) aşaması temel yapıları (primitives)
*parçacıklara* (fragments) indirger. Bunlar, kare arabelleklerini dolduracak
olan piksellerdir. Ekranın dışında kalan tüm parçacıklar atılır ve köşe
gölgelendirici tarafından gönderilen tüm öznitelikler şekilde gösterildiği gibi
parçacıklar arasında interpole edilir. Genellikle derinlik testi sebebiyle,
başka temel yapıların arkasında kalan parçacıklar da atılır.

*Parçacık gölgelendirici* (fragment shader) hayatta kalan tüm parçacıklar için
çalışır ve parçacıkların hani kare arabelleklerine yazılıp hangi renk ve
derinlik değerlerine sahip olacağını belirler. Bunu köşe gölgelendiriciden gelen
veriyi interpole ederek de yapabilir, bu veri doku koordinatları ve ışıklandırma
için normaller gibi şeyleri içerebilir.

*Renk karıştırma* (color blending) aşamasında aynı piksele denk gelen
parçacıkları karıştırmak için işlemler uygulanır. Parçacıklar basitçe birbirinin
üzerine yazabilir, toplanabilir veya saydamlığa bağlı olarak karıştırılabilir.

Yeşil renkli aşamalar *sabit fonksiyon* (fixed-function) aşamaları olarak
bilinir. Bu aşamalar, parametreler aracılığıyla işlemlerinde değişiklik
yapılmasına izin verir ama nasıl çalıştıkları önceden belirlenmiştir.

Turuncu renkliler ise `programlanabilir` aşamalardır. Grafik kartına kendi
kodunuzu yükleyerek tam olarak istediğiniz işlemleri yapmanıza izin verir. Bu,
örnek olarak parçacık gölgelendirici kullanarak doku geçirme ve ışıklandırmadan
ışın takibine kadar her şeyi yapmanıza olanak sağlar. Bu programlar birçok GPU
çekirdeğinde eş zamanları çalışarak köşeler ve parçacıklar gibi birçok objeyi
aynı anda işler.

Önceden OpenGL ve Direct3D gibi eski API'ları kullandıysanız `glBlendFunc` ve
`OMSetBlendState` gibi çağrılarla her türlü boru hattı ayarını değiştirmeye
alışmışsınızdır. Vulkan'daki grafik boru hattı hemen her zaman sabittir, bu
yüzden eğer gölgelendirici değiştirmek, başka bir kare arabelleği bağlamak veya
renk karıştırma fonksiyonunu değiştirmek isterseniz boru hattını sıfırdan terar
oluşturmalısınız. Bunun dezavantajı, çizim işlemlerinizde gerekecek her türlü
kombinasyon için ayrı bir boru hattı oluşturmanız gerekmesidir. Ama boru
hattında yapacağınız tüm işlemler önceden bilineceği için sürücü sizin kodunuzu
çok daha iyi optimize edebilir.

Yapacağınız işlemlere göre bazı programlanabilir aşamalar isteğe bağlıdır.
Örneğin sadece basit şekiller çiziyorsanız tesselasyon ve geometri
gölgelendiricileri devre dışı bırakılabilir. Eğer sadece derinlik değerleriyle
ilgileniyorsanız parçacık gölgelendirici aşamasını devre dışı bırakabilirsiniz.
Bu mesela [gölge haritası](https://en.wikipedia.org/wiki/Shadow_mapping)
oluşturmak için gayet faydalıdır.

Bir sonraki bölümde ekrana üçgen çizmek için gerekli ilk iki programlanabilir
aşamamızı oluşturacağız: köşe gölgelendirici ve parçacık gölgelendirici.
Karıştırma modu, görüntü kapısı, pikselleştirme gibi sabit fonksiyon ayarları
ondan sonraki bölümde yapılacak. Vulkan'da grafik boru hattını kurmanın son
parçası ise girdi ve çıktı kare arabelleklerinin tanımlarını içerecek.

`initVulkan` içerisinde `createImageViews`'dan hemen sonra çağrılacak olan
`createGraphicsPipeline` fonksiyonunu oluşturalım. Önümüzdeki bölümlerde bu
fonksiyonla uğraşacağız.

```c++
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

[C++ kodu](/code/08_graphics_pipeline.cpp)
