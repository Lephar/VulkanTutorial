Önceki API'ların aksine, Vulkan'da gölgeleyici kodu [GLSL](https://en.wikipedia.org/wiki/OpenGL_Shading_Language)
ve [HLSL](https://en.wikipedia.org/wiki/High-Level_Shading_Language) gibi insan
tarafından okunabilir kod formatlarında değil bayt kodu biçiminde olmalı. Bu
bayt kodu formatının adı [SPIR-V](https://www.khronos.org/spir) ve hem Vulkan
hem de OpenCL (ikisi de Khronos API'ı) tarafından tarafından kullanılmak üzere
tasarlandı. Hem hesap hem de grafik gölgelendirici yazmak için kullanılabilen
bir format ancak biz bu derslerde sadece Vulkan'ın grafik boru hattında
kullanılacak olan gölgelendiricilerle çalışacağız.

Bayt kodunun avantajı, GPU üreticilerinin yazdığı gölgeleyici derleyicilerinin
çok daha az karmaşık bir yapıda olmasını sağlaması. Geçmişte gördük ki, GLSL
gibi insan tarafından okunabilir sözdizimlerinde, bazı GPU üreticileri
standartları yorumlama konusunda çok esnek davranıyordu. Eğer GPU
üreticilerinden biri için, basit olmayan bir gölgeleyici kodu yazıyorduysanız,
diğer üreticilerin sürücülerinin sözdiziminizi reddetme riski vardı. Hatta daha
da kötüsü, derleyicideki yazılım hatalarından dolayı gölgeleyicinizin tamamen
farklı çalışma ihtimalı vardı. SPIR-V gibi düz bir bayt kodu formatıyla bu tür
sıkıntıların önüne geçilmesi umuluyor.

Tabi ki bu, bayt kodunu elle yazacağımız anlamına gelmiyor. Khronos, GLSL kodunu
SPIR-V formatına derleyen üreticiden bağımsız kendi derleyicisini yayınladı. Bu
derleyici, kodunuzun standartlara tamamen uyduğunu doğrulamak ve programınızda
kullanabilmeniz için SPIR-V binarysini üretmek için tasarlandı. Hatta SPIR-V
binarysini çalışma zamanında üretmek için bu derleyiciyi kütüphane olarak da
programınıza eklemeniz mümkün ama biz bu derslerde bu yöntemi izlemeyeğiz.
Derleyiciyi direkt olarak `glslangValidator.exe` ile kullanabiliriz ama bunun
yerine Google'ın geliştirdiği `glslc`'yi kullanacağız. `glslc` kullanmanın
avantajı, GCC ve Clang gibi bilindik derleyicilerle aynı parametre formatını
kullanması ve *include*'lar gibi ek işlevlere sahip olması. İkisi de Vulkan SDK
içerisinde mevcut, yani ek bir şey indirmeniz gerekmiyor.

GLSL, C'ye benzer sözdizimine sahip bir gölgelendirici dili. Bunda yazılmış
programlar, her obje için çalışan bir `main` fonksiyonuna sahip. Girdi için
parametreler kullanıp değerleri çıktı olarak döndürmek yerine GLSL girdi ve
çıktıları global değişkenlerle sağlamakta. Bu dil, bütünleşik vektörler ve
matrisler için temel değişkenler gibi, grafik işlemlerinde bize yardımcı olacak
birçok özelliğe sahip. Çapraz çarpım, matris-vektör çarpımı ve bir vektöre göre
yansıma gibi işlemler içermekte. Vektör tipi `vec` ve eleman sayısını belirten
bir sayı ile belirtilmekte. Örneğin 3 boyutta bir pozisyon `vec3`'de saklanır.
Bunun tek bir bileşenine `.x` şeklinde erişebileceğimiz gibi aynı anda birden
çok bileşeninden yeni bir vektör üretmemiz de mümkün. Örneğin
`vec3(1.0, 2.0, 3.0).xy` ifadesi, bir `vec2` ile döndürür. Ayrıca vektör
constructorları vektör objeleri ve skaler değerler karışımı da alabilir. Örneğin
bir `vec3` değişkeni `vec3(vec2(1.0, 2.0), 3.0)` şeklinde oluşturulabilir.

Önceki bölümlerde bahsettiğimiz gibi, ekrana bir üçgen çizebilmek için bir köşe
gölgelendirici ve bir parçacık gölgelendirici yazmalıyız. Önümüzdeki iki kısım,
bu ikisi için de gereken GLS kodu üstüne olacak. Sonrasında da bunlardan iki
SPIR-V binarysi üretip kodumuza yüklemeyi göreceğiz.

## Köşe gölgelendirici (vertex shader)

Köşe gölgelendirici, gelen tüm köşeleri (vertex) işler. Girdi olarak köşenin
dünyadaki konumu, rengi, normali ve doku koordinatı gibi özelliklerini alır.
Çıktısı ise kesit koordinatlarındaki son konumu ve parçacık gölgelendiriciye
aktarılacak olan rengi ve doku koordinatları gibi özellikleri oluyor. Bu
değerler, yumuşak bir geçiş oluşturmak için pikselleştirici tarafından
parçacıklar üzerinde interpole edilir.

*Kesit koordinatı* (clip coordinate) köşe gölgeleyiciden gelen ve sonradan tüm
vektör, son elemanına bölünerek *normalleştirilmiş cihaz koordinatlarına*
(normalized device coordinate) dönüştürülen dört boyutlu bir vektördür. Bu
normalleştirilmiş cihaz koordinatları, kare arabelleklerini aşağıda göründüğü
gibi [-1, 1]'den [-1, 1]'e bir koordinat sistemine eşleyen birer
[homojen koordinattır](https://en.wikipedia.org/wiki/Homogeneous_coordinates):

![](/images/normalized_device_coordinates.svg)

Daha önceden bilgisayar grafikleriyle uğraştıysanız bunlara aşinalığınız vardır.
Eğer OpenGL kullandıysanız Y koordinatının işaretinin artık tersine döndüğünü
farketmişsinizdir. Z koordinatları da artık Direct3D ile aynı 0 ile 1 aralığını
kullanmakta.

İlk üçgenimiz için herhangi bir dönüşüm uygulamayacağız. Bu üç köşenin
konumlarını aşağıdaki şekli oluşturmak için direkt olarak normalleştirilmiş
cihaz koordinatları biçiminde vereceğiz:

![](/images/triangle_coordinates.svg)

Köşe gölgelendiricide son değerlerine `1` vererek normalleştirilmiş cihaz
koordinatlarını direkt kesit koordinatları olarak çıktılayabiliriz. Böylece
kesit koordinatlarını normalleştirilmiş cihaz koordinatlarına dönüştüren bölme
işlemi hiçbir şey değiştirmez.

Normalde bu koordinatlar bir verteks arabelleğinde tutulur ama Vulkan'da verteks
arabelleği oluşturma ve içini verilerle doldurma işlemi çok kolay değil. Bu
yüzden üçgenimizi ekranda görüp tatmin olduktan sonraki bir vakte erteledim. Bu
sırada hiç alışılmamış bir şey yapıp tüm koordinatları direkt olarak köşe
gölgelendiricinin içinde saklayacağız. Kod şöyle görünecek:

```glsl
#version 450

vec2 positions[3] = vec2[](
    vec2(0.0, -0.5),
    vec2(0.5, 0.5),
    vec2(-0.5, 0.5)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
}
```

`main` fonksiyonu her köşe için çağrılır. `gl_VertexIndex` değişkeni şu anki
köşenin indisini tutar. Bu genelde verteks arabelleğine bir indistir ancak bizim
durumumuzda doğrudan kodlanmış dizimize bir indis olacak. Tüm köşelerin
konumlarına gölgelendiricideki sabit diziden erişilecek ve bunlar, bir kesit
koordinatı üretebilmek için işlevsiz birer `z` ve `w` bileşeniyle
birleştirilecek. Gömülü `gl_Position` değişkeni çıktı olarak işlemekte.

## Parçacık gölgelendirici (fragment shader)

Köşe gölgelendiriciden gelen konumlardan oluşturulmuş bir üçgen, parçacıklardan
oluşmuş bir ekranda bir alan kaplar. Parçacık gölgelendirici, kare arabelleğine
(veya arabelleklerine) renk ve derinlik yazmak için bu parçacıkların her biri
için çağrılır. Tüm üçgen için kırmızı renk çıktısı veren basit bir parçacık
gölgelendirici aşağıdaki gibi görünür:

```glsl
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(1.0, 0.0, 0.0, 1.0);
}
```

Nasıl köşe gölgelendiricideki `main` fonksiyonu her köşe için çağrılıyorsa
buradaki `main` fonksiyonu da her parçacık için çağrılır. GLSL'deki renkler
[0, 1] aralığında R (kırmızı), G (yeşil), B (mavi), ve alfa kanalına sahip 4
bileşenli vektörlerdir. Köşe gölgelendiricideki `gl_Position`'ın aksine şu anki
parçacık için renk çıktısı verecek gömülü bir değişken yoktur. Her kare
arabelleği için `layout(location = 0)` niteleyicisinin kare arabelleği indisini
belirteceği şekilde kendi çıktı değişkenimizi belirlemeliyiz. Kırmızı renk `0`
indisindeki ilk (ve tek) kare arabelleğine bağlı olan `outColor` değişkenine
yazılacak.

## Köşeye özel renk

Tamamı kırmızı bir üçgen çizmek hiç ilgi çekici değil, şöyle bir şey daha güzel
görünmez miydi?

![](/images/triangle_coordinates_colors.png)

Buna ulaşmak için iki gölgelendiricide de birkaç değişiklik yapmalıyız. İlk
olarak üç köşe için de farklı birer renk belirlemeliyiz. Köşe gölgelendirici
artık konum dizisi gibi bir de renk dizisi içermeli:

```glsl
vec3 colors[3] = vec3[](
    vec3(1.0, 0.0, 0.0),
    vec3(0.0, 1.0, 0.0),
    vec3(0.0, 0.0, 1.0)
);
```

Şimdi köşeye özel renkleri parçacık gölgelendiriciye de atmalıyız ki interpole
edilmiş değerleri kare arabelleğine basabilsin. Köşe gölgelendiriciye renk için
bir çıktı ekleyip `main` içerisinde bu değişkene değer atayalım:

```glsl
layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
    fragColor = colors[gl_VertexIndex];
}
```

Şimdi parçacık gölgelendiriciye bunla eşleşecek bir girdi ekleyelim:

```glsl
layout(location = 0) in vec3 fragColor;

void main() {
    outColor = vec4(fragColor, 1.0);
}
```

Girdi değişkeni aynı isimde olmak zorunda değil, birbirlerine `location`
direktifinde belirtilen indis ile bağlanacaklar. `main` fonksiyonu rengin
yanında alfa değerini de çıktı verecek şekilde değiştirildi. Yukarıdaki şekilde
gösterildiği gibi `fragColor` değerleri, üç köşenin arasındaki parçacıklar için
otomatik olarak interpole edilecek ve yumuşak bir geçiş elde edilecektir.

## Gölgelendiricileri derleme

Proje klasörünüzün altına `shaders` adlı bir klasör oluşturun. Köşe
gölgelendiricinizi bu klasör içerisinde `shader.vert`, parçacık
gölgelendiricinizi de yine aynı klasör içinde `shader.frag` adlı bir dosyaya
koyun. GLSL gölgeleyicilerin resmi bir uzantısı yok ama ayırt etmek için genelde
bu iki uzantı kullanılır.

`shader.vert`'in içeriği şöyle olmalı:

```glsl
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(location = 0) out vec3 fragColor;

vec2 positions[3] = vec2[](
    vec2(0.0, -0.5),
    vec2(0.5, 0.5),
    vec2(-0.5, 0.5)
);

vec3 colors[3] = vec3[](
    vec3(1.0, 0.0, 0.0),
    vec3(0.0, 1.0, 0.0),
    vec3(0.0, 0.0, 1.0)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
    fragColor = colors[gl_VertexIndex];
}
```

Ve `shader.frag`'inki şöyle:

```glsl
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(location = 0) in vec3 fragColor;

layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(fragColor, 1.0);
}
```

`glslc` kullanarak bu ikisini SPIR-V bayt kodu formatına derleyeceğiz.

**Windows**

Aşğıdaki içeriğe sahip `compile.bat` isimli bir dosya oluşturun:

```bash
C:/VulkanSDK/x.x.x.x/Bin32/glslc.exe shader.vert -o vert.spv
C:/VulkanSDK/x.x.x.x/Bin32/glslc.exe shader.frag -o frag.spv
pause
```

`glslc.exe`'ye giden yolu, Vulkan SDK kurulu olan konumla değiştirin.
Çalıştırmak için dosyaya çift tıklayın.

**Linux**

Aşağıdaki içeriğe sahip bir `compile.sh` dosyası oluşturun:

```bash
/home/user/VulkanSDK/x.x.x.x/x86_64/bin/glslc shader.vert -o vert.spv
/home/user/VulkanSDK/x.x.x.x/x86_64/bin/glslc shader.frag -o frag.spv
```

`glslc.exe`'ye giden yolu, Vulkan SDK kurulu olan konumla değiştirin.
`chmod +x compile.sh` komutuyla bu dosyanın çalıştırılabilir olduğuna emin olun
ve sonrasında çalıştırın.

**Platforma özel yönergelerin sonu**

Bu iki komut, derleyiciye GLSL kaynak dosyalarını okuyup `-o` (output)
parametresiyle SPIR-V bayt kodu olarak çıktı vermesini söyler.

Eğer gölgelendiricinizde sözdizimi hatası varsa tahmin edeceğiniz gibi derleyici
size satır numarasını ve hatayı söyleyecektir. Örneğin bir noktalı virgülü
silip komutları tekrar çalıştırmayı deneyin. Ayrıca derleyiciyi parametresiz
çalıştırarak ne tür parametreleri desteklediğini görün. Örneğin bayt kodunuzu da
insan tarafından okunabilir bir formatta çıktılayabilir. Böylece
gölgelendiricinizin tam olarak ne yaptığını ve bu aşamada uygulanan
optimizasyonları kontrol edebilirsiniz.

Gölgelendiricileri komut satırından derlemek en kolay yöntem ve biz de bu
derslerde bu şekilde kullanacağız. Tabi gölgelendiricilerinizi direkt olarak
kodunuzun içinde derlemeniz de mümkün. Vulkan SDK, GLSL kodunuzu SPIR-V
formatına kodunuzun içerisinde derlemenizi sağlayan [libshaderc](https://github.com/google/shaderc)
kütüphanesini de içeriyor.

## Gölgelendiriciyi yükleme

Artık SPIR-V gölgelendiricilerini oluşturabildiğimize göre, bunları programımıza
yükleyip bir noktada grafik boru hattına bağlama zamanımız geldi. Öncelikle
dosyalardaki binary veriyi okumak için basit bir yardımcı fonksiyon yazacağız.

```c++
#include <fstream>

...

static std::vector<char> readFile(const std::string& filename) {
    std::ifstream file(filename, std::ios::ate | std::ios::binary);

    if (!file.is_open()) {
        throw std::runtime_error("failed to open file!");
    }
}
```

`readFile` fonksiyonu dosyanın tüm baytlarını okuyup bunu `std::vector`'e
sarılmış bir bayt dizisi şeklinde döndürecek. Dosyayı iki parametreyle açarak
başlayacağız.

* `ate`: Dosyayı okumaya sondan başla
* `binary`: Dosyayı binary olarak oku (metin dönüşümlerinden kaçın)

Okumaya sondan başlamamızın avantajı, okuma pozisyonunu kullanarak dosyanın
boyutunu bulup arabelleğe buna göre yer açabilmemiz:

```c++
size_t fileSize = (size_t) file.tellg();
std::vector<char> buffer(fileSize);
```

Bundan sonra dosyanın başına gidip tüm baytları tek seferde okuyabiliriz:

```c++
file.seekg(0);
file.read(buffer.data(), fileSize);
```

Ve son olarak dosyayı kapatıp baytları döndürürüz:

```c++
file.close();

return buffer;
```

Bu fonksiyonu şimdi `createGraphicsPipeline` içerisinden iki gölgelendiricinin
de bayt kodlarını yüklemek için çağıracağız:

```c++
void createGraphicsPipeline() {
    auto vertShaderCode = readFile("shaders/vert.spv");
    auto fragShaderCode = readFile("shaders/frag.spv");
}
```

Arabelleklerin boyutlarını ekrana yazdırıp dosyaların gerçek boyutlarıyla
karşılaştırarak gölgelendiricilerin doğru yüklendiğinden emin olun. Kod binary
olduğundan null ile biten bir string olması gerekmiyor. İleride tam boyutunu
kullanacağımız işlemler olacak.

## Gölgelendirici modüllerini oluşturma

Kodu boru hattına göndermeden önce bir `VkShaderModule` objesine sarmalıyız.
Bunun için `createShaderModule` adlı yardımcı bir fonksiyon yazalım.

```c++
VkShaderModule createShaderModule(const std::vector<char>& code) {

}
```

Fonksiyon, bayt kodunu içeren arabelleği parametre olarak alıp bundan bir
`VkShaderModule` oluşturacak.

Gölgelendirici modülünü oluşturmak gayet basit, bayt kodunu içeren arabelleğe ve
kodun uzunluğunu tutan değişkene bir işaretçi vermemiz yeterli. Bu bilgiler
`VkShaderModuleCreateInfo` structında olacak. Dikkat etmemiz gereken bir husus,
bayt kodun uzunluğu bayt türünde belirtiliyor ancak bayt kodunun işaretçisi
`char` değil bir `uint32_t` işaretçisi türünde. Bu sebeple aşağıdaki gibi
`reinterpret_cast` kullanarak işaretçiyi cast etmeliyiz. Ancak bunun gibi bir
cast uyguladığımızda verinin `uint32_t` hizalama (alignment) gereksinimlerini
sağladığından da emin olmalıyız. Neyse ki veri bir `std::vector` içerisinde
saklanıyor ve bunun varsayılan yer ayırıcısı (default allocator) en kötü ihtimal
hizalama gereksinimlerini bile sağlayan bir yapıda.

```c++
VkShaderModuleCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO;
createInfo.codeSize = code.size();
createInfo.pCode = reinterpret_cast<const uint32_t*>(code.data());
```

Şimdi bir `vkCreateShaderModule` çağrısıyla `VkShaderModule` oluşturulabilir:

```c++
VkShaderModule shaderModule;
if (vkCreateShaderModule(device, &createInfo, nullptr, &shaderModule) != VK_SUCCESS) {
    throw std::runtime_error("failed to create shader module!");
}
```

Parametreler önceki nesne oluşturma fonksiyonları ile aynı: mantıksal aygıt,
oluşturma bilgisi içeren structa bir işaretçi, kendi yer ayırıcımıza opsiyonel
bir işaretçi ve işleyici için bir çıktı değişkeni. Kodu içeren arabellek, modül
oluşturulduktan hemen sonra temizlenebilir. Oluşturduğumuz gölgelendirici
modülünü döndürmeyi de unutmayın:

```c++
return shaderModule;
```

Gölgelendirici modülleri, daha önce dosyadan yüklediğimiz gölgelendirici bayt
kodunun ve bunun içindeki fonksiyonların etrafını saran ince bir katmandır.
SPIR-V bayt kodunun makine koduna derlenip bağlanması ve GPU tarafından
çalıştırılması, grafik boru hattı oluşturulduğunda gerçekleşir. Bu da
gölgelendirici modüllerini, boru hattını oluşturma işlemi bittiği anda yok
edebileceğimiz anlamına geliyor. Bunları sınıf değişkenleri olarak tanımlamak
yerine `createGraphicsPipeline` içinde yerel değişken olarak tanımlama sebebimiz
de bu.

```c++
void createGraphicsPipeline() {
    auto vertShaderCode = readFile("shaders/vert.spv");
    auto fragShaderCode = readFile("shaders/frag.spv");

    VkShaderModule vertShaderModule = createShaderModule(vertShaderCode);
    VkShaderModule fragShaderModule = createShaderModule(fragShaderCode);
}
```

Temizleme işlemi, fonksiyonun sonunda iki tane `vkDestroyShaderModule`
çağrısıyla yapılmalı. Bu bölümde kalan tüm kod bu satırların öncesine
eklenecektir.

```c++
    ...
    vkDestroyShaderModule(device, fragShaderModule, nullptr);
    vkDestroyShaderModule(device, vertShaderModule, nullptr);
}
```

## Gölgelendirici aşamasının oluşturulması

Gölgelendiricileri kullanabilmek için bunları boru hattının spesifik bir
aşamasına atamamız gerekiyor. Bunu da boru hattını oluşturmanın gerçek bir
parçası gibi `VkPipelineShaderStageCreateInfo` structı kullanarak yapacağız.

Yine `createGraphicsPipeline` içerisinde köşe gölgelendirici ile ilgili
structları doldurarak başlayacağız.

```c++
VkPipelineShaderStageCreateInfo vertShaderStageInfo{};
vertShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
vertShaderStageInfo.stage = VK_SHADER_STAGE_VERTEX_BIT;
```

Zorunlu `sType` alanı dışındaki ilk adım, Vulkan'a bu gölgelendiricinin hangi
boru hattı aşamasında kullanılacağını söylemek. Geçen kısımda bahsettiğimiz
bütün programnlanabilir aşamalar için bir enum değeri var.

```c++
vertShaderStageInfo.module = vertShaderModule;
vertShaderStageInfo.pName = "main";
```

Sonraki iki alan, kodu içeren gölgelendirici modülü ve *giriş noktası*
(entrypoint) olarak bilinen giriş fonksiyonu. Bu da, birden fazla parçacık
gölgelendiriciyi tek bir gölgelendirici modülde birleştirip farklı giriş
noktaları vererek farklı davranışlar elde edebileceğimiz anlamına geliyor. Ama
bizim durumumuzda sadece `main` kullanılacak.

Bir alan daha var, o da isteğe bağlı `pSpecializationInfo` değeri. Bu derste
kullanmayacağız ama tartışmaya değer. Gölgelendiricilerdeki sabitlere değer
atamamızı sağlıyor. İçindeki sabite, boru hattını oluşturma sırasında, farklı
değerler vererek tek bir gölgelendirici modülünden, farklı davranışlar elde
edebilmemizi sağlar. Gölgelendiriciyi, değişkenler yardımıyla çalışma zamanında
yapılandırmaktan çok daha verimli çünkü derleyici bu değerleri kullanarak `if`
ifadelerini yok etmek gibi optimizasyonlar yapabilir. Bu tür bir sabitiniz yoksa
bu alana `nullptr` atayabilirsiniz, bizim struct oluşturucumuz bunu otomatik
yapıyor.

Structı parçacık gölgelendiriciye uyacak şekilde değiştirmek de gayet basit:

```c++
VkPipelineShaderStageCreateInfo fragShaderStageInfo{};
fragShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
fragShaderStageInfo.stage = VK_SHADER_STAGE_FRAGMENT_BIT;
fragShaderStageInfo.module = fragShaderModule;
fragShaderStageInfo.pName = "main";
```

Bu iki structı içeren bir dizi oluşturarak da tamamlayalım. Bu diziye asıl boru
hattı oluşturma aşamasında bir referans vereceğiz.

```c++
VkPipelineShaderStageCreateInfo shaderStages[] = {vertShaderStageInfo, fragShaderStageInfo};
```

Boru hattının programlanabilir aşamalarını yapılandırmak için yapmamız gereken
her şey bu kadar. Bir sonraki bölümde sabit fonksiyon aşamalarına bakacağız.

[C++ kodu](/code/09_shader_modules.cpp) /
[Köşe gölgelendirici](/code/09_shader_base.vert) /
[Parçacık gölgelendirici](/code/09_shader_base.frag)
