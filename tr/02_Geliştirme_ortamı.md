Bu bölümde Vulkan uygulamamızı yazacağımız geliştirme ortamımızı kurup birkaç
kullanışlı kütüphane yükleyeceğiz. Derleyiciler haricinde kullanacağımız tüm
araçlar Windows, Linux ve macOS ile uyumlu. Ama kurulum adımları birbirinden
biraz farklılık gösterdiği için burada ayrı ayrı anlatacağız.

## Windows

Eğer Windows'ta geliştiriyorsanız kodunuzu Visual Studio ile derlediğinizi
varsayacağım. Tam C++17 desteği için Visual Studio 2017 veya 2019
kullanmalısınız. Aşağıdaki adımlar VS 2017 için hazırlanmıştır ancak 2019'da da
rahatça uygulanabilir.

### Vulkan SDK

Vulkan uygulamamızı geliştirmek için ihtiyacımız olan en önemli parça Vulkan SDK
(yazılım geliştirme kiti). Header dosyaları, standart doğrulama katmanları,
hata ayıklama araçları ve Vulkan fonksiyonları için yükleyici (function loader)
bu paketle gelecektir. Yükleyici, çalışma zamanında sürücüden fonksiyonları
getirir. Eğer aşinalığınız varsa OpenGL için GLEW'e benzetilebilir.

Bu SDK [LunarG sitesinden](https://vulkan.lunarg.com/), sayfanın altındaki tuşla
indirilebilir. Hesap oluşturmanız şart değil ancak faydalı olabilecek ek
dökümantasyonlara ulaşmak için oluşturabilirsiniz.

![](/images/vulkan_sdk_download_buttons.png)

Yüklemeye devam ederken SDK yükleme klasörüne dikkat edin. İlk yapacağımız şey
grafik kartımız ve sürücümüzün Vulkan'ı düzgün bir şekilde desteklediğinden emin
olmak. Yüklemeyi yaptığınız klasöre gidin, `Bin` klasörünü açın ve `vkcube.exe`
örneğini çalıştırın. Resimdekine benzer bir görsel görmeniz gerekiyor:

![](/images/cube_demo.png)

Eğer bir hata mesajıyla karşılaşırsanız sürücünüzün güncel olduğundan, Vulkan
bileşenlerinin düzgün yüklendiğinden ve grafik kartınızın desteklendiğinden emin
olun. Başlıca üreticilerin sürücü sayfalarına giden bağlantılar için [giriş](!tr/Giriş)
bölümüne bakabilirsiniz.

Bu klasörde geliştirme süresince işimize yarayacak bir program daha mevcut.
`glslangValidator.exe` ve `glslc.exe` programları, insan tarafından okunabilir
[GLSL](https://en.wikipedia.org/wiki/OpenGL_Shading_Language) gölgeleyici
kodlarını, bayt koduna derlemek için kullanılacak. Bunlara ayrıntılı bir şekilde
[gölgeleyici modülleri](!en/Drawing_a_triangle/Graphics_pipeline_basics/Shader_modules)
bölümünde değineceğiz. `Bin` klasörü ayrıca Vulkan yükleyicisi ve doğrulama
katmanlarının binary dosyalarını içerirken, `Lib` klasörü de kütüphane
dosyalarını içermekte.

Son olarak Vulkan header dosyalarını içeren bir `Include` klasörü mevcut. Diğer
klasörlere de göz atabilirsiniz ama bu derslerde onlara ihtiyacımız olmayacak.

### GLFW

Daha önce bahsettiğimiz gibi, Vulkan'ın kendisi platformdan bağımsız bir API ve
çizdiklerimizi yansıtabileceğimiz bir pencere oluşturmak için araçlar içermiyor.
Vulkan'ın çoklu platform yapısından faydalanabilmek ve Win32'nin çileleriyle
boğuşmamak için pencereleri hem Windows'u, hem Linux'u hem de macOS'i
destekleyen [GLFW kütüphanesi](http://www.glfw.org/) ile oluşturacağız. Aynı
amaç için geliştirilmiş [SDL](https://www.libsdl.org/) gibi kütüphaneler de
mevcut ama GLFW kullanarak pencere oluşturmanın yanı sıra Vulkan'a özel bir
takım platform bağımlı şeyleri de soyutlayabileceğiz.

[GLFW resmi sitesinden](http://www.glfw.org/download.html) son sürüme
erişebilirsiniz. Bu derslerde biz 64 bit binary dosyalarını kullanacağız ama
tabi isterseniz 32 bitte derlemeyi de seçebilirsiniz. Bu durumda Vulkan SDK
içerisinde `Lib` değil de `Lib32` klasöründeki binary dosyalarıyla bağladığınza
emin olun. İndirdikten sonra arşivden çıkarın ve kolay erişebileceğiniz bir
konuma taşıyın. Ben Belgelerim içerisindeki Visual Studio klasörünün içine
Libraries (kütüphaneler) adlı bir klasör açmayı seçtim.

![](/images/glfw_directory.png)

### GLM

DirectX 12'nin aksine Vulkan lineer cebir işlemleri için bir kütüphane
içermiyor, bu sebeple kendimiz bir tane indireceğiz. [GLM](http://glm.g-truc.net/),
grafik API'ları ile kullanılmak üzere tasarlanmış ve OpenGL ile sıklıkla
kullanılan güzel bir kütüphane.

GLM sadece header dosyalarından oluşan bir kütüphane, [son versiyonunu](https://github.com/g-truc/glm/releases)
indirmeniz ve düzgün bir konuma taşımanız yeterli. Şimdi buna benzer bir klasör
yapınız olmalı:

![](/images/library_directory.png)

### Visual Studio Kurulumu

Tüm gereksinimleri yüklediğimize göre şimdi bir Visual Studio projesi
kurabiliriz. Sonra da her şeyin düzgün çalışıp çalışmadığından emin olmak için
kısa bir kod yazalım.

Visual Studio'yu başlatın ve yeni bir `Windows Desktop Wizard` (Windows masaüstü
sihirbazı) projesi oluşturmak için bir isim girip `OK` tuşuna basın.

![](/images/vs_new_cpp_project.png)

Hata ayıklama mesajlarımızı yazdırabileceğimiz bir konsolumuz olması için
uygulama tipi olarak `Console Application (.exe)` (konsol uygulaması) seçili
olduğundan emin olun ve Visual Studio'nun fazlalık kodlar eklememesi için `Empty
Project` (boş proje) seçeneğini işaretleyin.

![](/images/vs_application_settings.png)

`OK` tuşuna basın ve bir C++ kaynak kodu dosyası ekleyin. Bunun nasıl
yapılacağını zaten biliyorsunuzdur ama yine de adımların tam olması açısından
aşağıda ekran görüntüleri mevcut.

![](/images/vs_new_item.png)

![](/images/vs_new_source_file.png)

Şimdi aşağıdaki kodu dosyaya ekleyin. Şimdilik ne olduğunu anlamaya çalışmanıza
gerek yok. Sadece bir Vulkan programı derleyip çalıştırabildiğimizden emin olmak
istiyoruz. Bir sonraki bölümde sıfırdan başlayacağız.

```c++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>

#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/vec4.hpp>
#include <glm/mat4x4.hpp>

#include <iostream>

int main() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    GLFWwindow* window = glfwCreateWindow(800, 600, "Vulkan window", nullptr, nullptr);

    uint32_t extensionCount = 0;
    vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);

    std::cout << extensionCount << " extensions supported\n";

    glm::mat4 matrix;
    glm::vec4 vec;
    auto test = matrix * vec;

    while(!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }

    glfwDestroyWindow(window);

    glfwTerminate();

    return 0;
}
```

Şimdi projeyi yapılandırıp hatalardan kurtulalım. `Project Properties` (proje
ayarları) sayfasını açın ve `All Configurations` (tüm yapılandırmalar)
seçeneğinin seçili olduğuna emin olun. Çünkü çoğu değişiklik hem `Debug` (hata
ayıklama) hem de `Release` (yayınlama) modlarına etki edecek.

![](/images/vs_open_project_properties.png)

![](/images/vs_all_configs.png)

Şimdi `C++`, `General` (genel), `Additional Include Directories` (ek include
klasörleri) sayfasına gidin ve açılır listeden `<Edit...>` (düzenle) seçeneğini
seçin.

![](/images/vs_cpp_general.png)

Vulkan, GLF ve GLM'in header klasörlerini ekleyin:

![](/images/vs_include_dirs.png)

Şimdi kütüphane klasörleri için `Linker` (bağlayıcı), `General` (genel)
altındaki editörü açın:

![](/images/vs_link_settings.png)

Ve Vulkan ile GLFW obje dosyalarının konumlarını ekleyin:

![](/images/vs_link_dirs.png)

`Linker`, `Input` (girdi) kısmına gidin ve `Additional Dependencies` (ek
gereksinimler) açılır listesinden `<Edit...>` seçeneğine tıklayın.

![](/images/vs_link_input.png)

Vulkan ve GLFW obje dosyalarının isimlerini girin:

![](/images/vs_dependencies.png)

Ve son olarak derleyici desteğini C++17'ye çekin:

![](/images/vs_cpp17.png)

Şimdi `Project Properties` sayfasını kapatabilirsiniz. Eğer her şeyi doğru
yaptıysanız artık kodda vurgulanan herhangi bir hata görmemeniz lazım.

Son olarak 64 bit modda derlediğinizden emin olun:

![](/images/vs_build_mode.png)

Projeyi derleyip çalıştırmak için `F5`'e basın. Şimdi buna benzer bir komut
istemi görüyor olmanız lazım:

![](/images/vs_test_window.png)

Uzantı sayısı sıfırdan farklı olmalı. Tebrikler, artık [Vulkan'la oynamaya](!en/Drawing_a_triangle/Setup/Base_code)
hazırsınız!

## Linux

Bu kısım Ubuntu kullanıcılarına göre hazırlanmıştır ama `apt` komutlarını
dağıtımınızın paket yöneticisine göre değiştirerek kendinize uygun hale
getirebilirsiniz. Ayrıca C++17 destekli bir de derleyiciye (GCC 7+ veya Clang
5+) ve make'e ihtiyacınız olacak.

### Vulkan Paketleri

Vulkan uygulamamızı geliştirirken Linux'ta ihtiyacımız olacak olan en önemli
parçalar Vulkan yükleyicisi (loader), doğrulama katmanları ve bilgisayarımızın
Vulkan destekli olup olmadığınızı anlamamıza yarayacak birkaç komut satırı
uygulaması:

* `sudo apt install vulkan-tools`: Komut satırı uygulamaları, özellikle `vulkaninfo` ve `vkcube` önemli. Bilgisayarınızın Vulkan desteğinin olduğuna emin olmak için bunları çalıştırın.
* `sudo apt install libvulkan-dev`: Vulkan yükleyicisini kurar. Yükleyici, çalışma zamanında sürücüden fonksiyonları getirir. Eğer aşinalığınız varsa OpenGL için GLEW'e benzetilebilir.
* `sudo apt install vulkan-validationlayers-dev`: Standart doğrulama katmanlarını kurar. Bunlar Vulkan uygulamamızda hata ayıklamanın bel kemiği. Önümüzdeki bölümde ayrıntısına ineceğiz.

Eğer kurulumlar başarılıysa Vulkan kısmını halletmişsinizdir. `vkcube` komutunu
çalıştırıp aşağıdaki pencereyi gördüğünüzden emin olun:

![](/images/cube_demo_nowindow.png)

Eğer bir hata mesajıyla karşılaşırsanız sürücünüzün güncel olduğundan, Vulkan
bileşenlerinin düzgün yüklendiğinden ve grafik kartınızın desteklendiğinden emin
olun. Başlıca üreticilerin sürücü sayfalarına giden bağlantılar için [giriş](!tr/Giriş)
bölümüne bakabilirsiniz.

### GLFW

Daha önce bahsettiğimiz gibi, Vulkan'ın kendisi platformdan bağımsız bir API ve
çizdiklerimizi yansıtabileceğimiz bir pencere oluşturmak için araçlar içermiyor.
Vulkan'ın çoklu platform yapısından faydalanabilmek ve X11'in çileleriyle
boğuşmamak için pencereleri hem Windows'u, hem Linux'u hem de macOS'i
destekleyen [GLFW kütüphanesi](http://www.glfw.org/) ile oluşturacağız. Aynı
amaç için geliştirilmiş [SDL](https://www.libsdl.org/) gibi kütüphaneler de
mevcut ama GLFW kullanarak pencere oluşturmanın yanı sıra Vulkan'a özel bir
takım platform bağımlı şeyleri de soyutlayabileceğiz.

GLFW aşağıdaki komutla yüklenebilir:

```bash
sudo apt install libglfw3-dev
```

### GLM

DirectX 12'nin aksine Vulkan lineer cebir işlemleri için bir kütüphane
içermiyor, bu sebeple kendimiz bir tane indireceğiz. [GLM](http://glm.g-truc.net/),
grafik API'ları ile kullanılmak üzere tasarlanmış ve OpenGL ile sıklıkla
kullanılan güzel bir kütüphane.

GLM sadece header dosyalarından oluşan bir kütüphane, aşağıdaki komutla
`libglm-dev` paketini kurabilirsiniz:

```bash
sudo apt install libglm-dev
```

### Shader Compiler

İnsan tarafından okunabilir [GLSL](https://en.wikipedia.org/wiki/OpenGL_Shading_Language)
gölgeleyici kodlarını, bayt koduna derlemek için kullanılacak olan programlar
dışında hemen hemen her şeyimiz hazır.

En popüler iki gölgeleyici derleyicisi Khronos Grup'un `glslangValidator`'ı ve
Google'ın `glslc` programı. İkincisi GCC ve Clang benzeri bir kullanıma sahip
olduğundan bunu kullanacağız. Google'ın [resmi olmayan binary dosyalarını](https://github.com/google/shaderc/blob/main/downloads.md)
indirin ve `glslc` dosyasını `/usr/local/bin` klasörünüze taşıyın. Yetkilerinize
göre `sudo` komutuna ihtiyacınız olabileceğini unutmayın. `glslc` komutunu
çalıştırarak programı test edebilirsiniz, parametre göndermediğimiz için haklı
olarak hata verecektir:

`glslc: error: no input files`

`glslc` ile ilgili ayrıntılara [gölgeleyici modülleri](!en/Drawing_a_triangle/Graphics_pipeline_basics/Shader_modules)
kısmında değineceğiz.

### Makefile projesi oluşturma

Tüm gereksinimleri yüklediğimize göre şimdi bir makefile projesi kurabiliriz.
Sonra da her şeyin düzgün çalışıp çalışmadığından emin olmak için kısa bir kod
yazalım.

Uygun bir konuma `VulkanTest` gibi bir isme sahip bir dizin açın ve içinde
`main.cpp` adlı bir kaynak kodu dosyası oluşturup aşağıdaki kodu kopyalayın.
Şimdilik ne olduğunu anlamaya çalışmanıza gerek yok. Sadece bir Vulkan programı
derleyip çalıştırabildiğimizden emin olmak istiyoruz. Bir sonraki bölümde
sıfırdan başlayacağız.

```c++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>

#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/vec4.hpp>
#include <glm/mat4x4.hpp>

#include <iostream>

int main() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    GLFWwindow* window = glfwCreateWindow(800, 600, "Vulkan window", nullptr, nullptr);

    uint32_t extensionCount = 0;
    vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);

    std::cout << extensionCount << " extensions supported\n";

    glm::mat4 matrix;
    glm::vec4 vec;
    auto test = matrix * vec;

    while(!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }

    glfwDestroyWindow(window);

    glfwTerminate();

    return 0;
}
```

Şimdi bu basit Vulkan kodunu derleyip çalıştırmak için bir makefile yazacağız.
`Makefile` isminde boş bir dosya oluşturun. Makefile hakkında değişkenlerin ve
kuralların nasıl işlediğine dair bir miktar tecrübeniz olduğunu varsayacağım.
Eğer yoksa [bu eğitim](https://makefiletutorial.com/) ile hızlı bir giriş
yapabilirsiniz.

İlk olarak dosyanın geri kalan kısmını basit tutabilmek için birkaç değişken
tanımlayacağız. Derleyiciye vereceğimiz parametreleri tutacak `CFLAGS`
değişkenini tanımlayın:

```make
CFLAGS = -std=c++17 -O2
```

Modern C++ (`-std=c++17`) kullanacağız ve optimizasyon seviyesini O2'ye
çekeceğiz. Programı daha hızlı derlemek için O2 parametresini kaldırabilirsiniz
ama yayınlanacak (release) hali için tekrar açmayı unutmayın.

Benzer şekilde linker parametreleri için `LDFLAGS` değişkenini tanımlayın:

```make
LDFLAGS = -lglfw -lvulkan -ldl -lpthread -lX11 -lXxf86vm -lXrandr -lXi
```

`-lglfw` GLFW için, `-lvulkan` Vulkan yükleyicisini bağlıyor.
Geri kalanı da GLFW'nin gereksinimleri olan alt seviye sistem kütüphaneleri:
çoklu iş parçacığı ve pencere yöneticisi.

`VulkanTest`'i derleyecek kural şimdi gayet basitçe yazılabilir. Yalnız satır
başlarında boşluklar için boşluk karakteri yerine tab karakteri kullandığınızdan
emin olun.

```make
VulkanTest: main.cpp
    g++ $(CFLAGS) -o VulkanTest main.cpp $(LDFLAGS)
```

Makefile dosyasını kaydedin, `main.cpp` ve `Makefile` dosyalarının olduğu
dizinde `make` komutunu çalıştırarak her şeyin doğru olduğundan emin olun. Bu
size `VulkanTest` adlı çalıştırılabilir bir dosya vermelidir.

Şimdi iki kural daha tanımlayacağız, bunlar `test` ve `clean`. İlki programı
çalıştıracak, ikincisi ise çalıştırılabilir dosyayı silecek:

```make
.PHONY: test clean

test: VulkanTest
    ./VulkanTest

clean:
    rm -f VulkanTest
```

`make test` komutunu çalıştırdığınızda programınızı düzgün bir şekilde
çalıştığını ve ekrana Vulkan uzantılarının sayısını yazdırdığını görmelisiniz.
Uygulamanız, pencereyi kapattığınızda (`0`) kodu dönerek başarılı bir şekilde
kapanmalı. Makefile dosyanızın bütünü aşağıdakine benzer olmalı:

```make
CFLAGS = -std=c++17 -O2
LDFLAGS = -lglfw -lvulkan -ldl -lpthread

VulkanTest: main.cpp
    g++ $(CFLAGS) -o VulkanTest main.cpp $(LDFLAGS)

.PHONY: test clean

test: VulkanTest
	./VulkanTest

clean:
    rm -f VulkanTest
```

Bu dizini Vulkan projeleriniz için bir taslak olarak kullanabilirsiniz. Klasörü
kopyalayın, `HelloTriangle` gibi bir isim verin ve `main.cpp` içerisindeki tüm
kodu silin.

İşte şimdi [asıl macera](!en/Drawing_a_triangle/Setup/Base_code) için
hazırsınız!

## macOS

Bu kısımda Xcode ve [Homebrew paket yöneticisi](https://brew.sh/) kullandığınız
varsayılacak. Ayrıca macOS 10.11 veya daha yüksek bir versiyona sahip
olmalısınız ve cihazınız [Metal API](https://en.wikipedia.org/wiki/Metal_(API)#Supported_GPUs)
desteklemeli.

### Vulkan SDK

Vulkan uygulamamızı geliştirmek için ihtiyacımız olan en önemli parça Vulkan SDK
(yazılım geliştirme kiti). Header dosyaları, standart doğrulama katmanları,
hata ayıklama araçları ve Vulkan fonksiyonları için yükleyici (function loader)
bu paketle gelecektir. Yükleyici, çalışma zamanında sürücüden fonksiyonları
getirir. Eğer aşinalığınız varsa OpenGL için GLEW'e benzetilebilir.

Bu SDK [LunarG sitesinden](https://vulkan.lunarg.com/), sayfanın altındaki tuşla
indirilebilir. Hesap oluşturmanız şart değil ancak faydalı olabilecek ek
dökümantasyonlara ulaşmak için oluşturabilirsiniz.

![](/images/vulkan_sdk_download_buttons.png)

macOS uyumlu SDK versiyonu kendi içerisinde [MoltenVK](https://moltengl.com/)
kullanıyor. macOS Vulkan'ı desteklemediğinden MoltenVK tüm Vulkan çağrılarını
Apple'ın Metal grafik çerçevesi çağrılarına dönüştüren bir katman olarak
çalışmakta. Böylece Metal'in de hata ayıklama ve performans getirilerinden
faydalanabiliyoruz.

İndirdikten sonra istediğiniz bir konuma arşivi çıkarın. Xcode projenizde bu
klasöre referans vereceğinizi unutmayın. Çıkardığınız klasörün içindeki
`Applications` dizininde test amaçlı kullanabileceğiniz birkaç çalıştırılabilir
dosya bulunmalı. `vkcube` programını çalıştırdığınızda aşağıdakini görmelisiniz:

![](/images/cube_demo_mac.png)

### GLFW

Daha önce bahsettiğimiz gibi, Vulkan'ın kendisi platformdan bağımsız bir API ve
çizdiklerimizi yansıtabileceğimiz bir pencere oluşturmak için araçlar içermiyor.
Vulkan'ın çoklu platform yapısından faydalanabilmek için pencereleri hem
Windows'u, hem Linux'u hem de macOS'i destekleyen [GLFW kütüphanesi](http://www.glfw.org/)
ile oluşturacağız. Aynı amaç için geliştirilmiş [SDL](https://www.libsdl.org/)
gibi kütüphaneler de mevcut ama GLFW kullanarak pencere oluşturmanın yanı sıra
Vulkan'a özel bir takım platform bağımlı şeyleri de soyutlayabileceğiz.

macOS'te GLFW yüklemek için Homebrew paket yöneticisini kullanacağız. macOS'te
Vulkan desteği bu metnin yazıldığı zamanki stabil versiyon olan 3.2.1 için halen
tamamlanmış değil. Bu yüzden `glfw3` paketi aracılığıyla son versiyonu
kuracağız:

```bash
brew install glfw3 --HEAD
```

### GLM

Vulkan lineer cebir işlemleri için bir kütüphane içermiyor, bu sebeple kendimiz
bir tane indireceğiz. [GLM](http://glm.g-truc.net/), grafik API'ları ile
kullanılmak üzere tasarlanmış ve OpenGL ile sıklıkla kullanılan güzel bir
kütüphane.

GLM sadece header dosyalarından oluşan bir kütüphane, `glm` paketiyle
kurabiliriz:

```bash
brew install glm
```

### Xcode kurulumu

Tüm gereksinimleri yüklediğimize göre şimdi Vulkan için bir Xcode projesi
kurabiliriz. Buradaki yönergelerin çoğu, tabiri caizse "amelelik",
gereksinimleri projeye bağlamak için yapmamız gerekiyor. Ayrıca bu kısımda
`vulkansdk` dizininden her bahsedişimizde aklınıza arşivden çıkardığımız Vulkan
SDK dizini gelmeli.

Xcode'u başlatın ve yeni bir Xcode projesi oluşturun. Açılan pencerede sırasıyla
Application (uygulama) ve Command Line Tool (komut satırı aracı) seçeneklerini
seçin.

![](/images/xcode_new_project.png)

`Next`'e (sıradaki) tıklayın, projenize bir isim verin ve `Language` (dil)
olarak `C++`'ı seçin.

![](/images/xcode_new_project_2.png)

Tekrar `Next`'e tıklayınca projeniz oluşturulacaktır. Şimdi kendi üretilmiş olan
`main.cpp` dosyasındaki kodu aşağıdakiyle değiştirin:

```c++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>

#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/vec4.hpp>
#include <glm/mat4x4.hpp>

#include <iostream>

int main() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    GLFWwindow* window = glfwCreateWindow(800, 600, "Vulkan window", nullptr, nullptr);

    uint32_t extensionCount = 0;
    vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);

    std::cout << extensionCount << " extensions supported\n";

    glm::mat4 matrix;
    glm::vec4 vec;
    auto test = matrix * vec;

    while(!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }

    glfwDestroyWindow(window);

    glfwTerminate();

    return 0;
}
```

Şimdilik bu kodun ne yaptığını anlamanıza gerek yok. Sadece her şeyin
çalıştığından emin olmak için birkaç API çağrısı yapıyoruz.

Xcode hali hazırda bulamadığı kütüphanelerle ilgili hatalar veriyor olmalı.
Şimdi projeyi, bu hataları giderecek şekilde düzenleyeceğiz. *Project Navigator*
(proje gezgini) panelinden kendi projenizi seçin. *Build Settings* (derleme
ayarları) tabını açın ve şunları yapın:

* **Header Search Paths** (header arama konumları) alanını bulun ve `/usr/local/include` (Homebrew header dosyalarını buraya yüklüyor, glm ve glfw3 header dosyaları burada olmalı) ve `vulkansdk/macOS/include` konumlarına bir bağlantı ekleyin.
* **Library Search Paths** (kütüphane arama konumları) alanını bulun ve `/usr/local/lib` (burası da Homebrew'in kütüphaneleri yüklediği yer, glm ve glfw3 dosyaları burda olmalı) ve `vulkansdk/macOS/lib` konumlarına bir bağlantı ekleyin.

Buna benzer görünüyor olmalı (tabi dosyaları koyduğunuz yere göre konumlar
değişebilir):

![](/images/xcode_paths.png)

Şimdi *Build Phases* (yapım aşamaları) tabındaki **Link Binary With Libraries**
(binary dosyayı kütüphane ile bağla) ile `glfw3` ve `vulkan` çerçevelerini
ekleyeceğiz. İşleri kolaylaştırmak için projeye dinamik kütüphaneleri
ekleyeceğiz (eğer statik çerçeveleri kullanmak isterseniz dökümantasyona göz
atabilirsiniz).

* GLFW için `/usr/local/lib` klasöründe `libglfw.3.x.dylib` dosyasını bulacaksınız (x burada versiyon numarası, Homebrew ile indirdiğiniz zamana göre farklılık gösterecektir). Bu dosyayı Xcode'daki Linked Frameworks (bağlanan çerçeveler) ve Libraries (kütüphaneler) tabına sürükleyip bırakın.
* Vulkan için `vulkansdk/macOS/lib` klasörüne gidin. Aynı işlemleri `libvulkan.1.dylib` ve `libvulkan.1.x.xx.dylib` dosyaları için yapacaksınız (yine x indirdiğiniz SDK versiyonunu belirtmekte).

Bu kütüphaneleri ekledikten sonra **Copy Files** (dosyaları kopyala) ile aynı
tabda `Destination`'ı (hedef) "Frameworks"'e (çerçeve) çevirin, alt konumu
(subpath) temizleyin ve "Copy only when installing" (sadece yüklerken kopyala)
seçeneğini kaldırın. "+" işaretine basın ve üç çerçevenin üçünü de ekleyin.

Xcode şöyle görünmeli:

![](/images/xcode_frameworks.png)

Kurmamız gereken son birkaç şey de ortam değişkenleri. Xcode'da araç çubuğundan
sırasıyla `Product` (ürün), `Scheme` (şema), `Edit Scheme...`'e (şemayı düzenle)
gidin ve `Arguments` (argümanlar) tabından aşağıdaki ortam değişkenlerini
ekleyin:

* VK_ICD_FILENAMES = `vulkansdk/macOS/share/vulkan/icd.d/MoltenVK_icd.json`
* VK_LAYER_PATH = `vulkansdk/macOS/share/vulkan/explicit_layer.d`

Şuna benzer görünmeli:

![](/images/xcode_variables.png)

Sonunda tamamladık! Eğer şimdi projemizi çalıştırırsak (yapılandırma seçiminize
göre Debug (hata ayıklama) ve Release (yayınlama) modlarından birinde)
aşağıdakini görüyor olmalısınız:

![](/images/xcode_output.png)

Uzantı sayısı sıfırdan farklı olmalı. Diğer çıktılar kütüphanelerin,
yapılandırmanıza göre farklı mesajlar alıyor olabilirsiniz.

Artık [asıl şeylere](!en/Drawing_a_triangle/Setup/Base_code) geçmeye hazırsınız!
