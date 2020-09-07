## Genel yapı

Geçtiğimiz bölümde doğru yapılandırılmış bir Vulkan projesi oluşturup örnek bir
kod ile denedik. Bu bölümde aşağıdaki kod ile sıfırdan başlayacağız:

```c++
#include <vulkan/vulkan.h>

#include <iostream>
#include <stdexcept>
#include <cstdlib>

class HelloTriangleApplication {
public:
    void run() {
        initVulkan();
        mainLoop();
        cleanup();
    }

private:
    void initVulkan() {

    }

    void mainLoop() {

    }

    void cleanup() {

    }
};

int main() {
    HelloTriangleApplication app;

    try {
        app.run();
    } catch (const std::exception& e) {
        std::cerr << e.what() << std::endl;
        return EXIT_FAILURE;
    }

    return EXIT_SUCCESS;
}
```

İlk olarak LunarG SDK içerisindeki Vulkan header dosyalarını ekledik.
Fonksiyonları, structları ve enumları bu header dosyası sağlıyor. `stdexcept` ve
`iostream`'i hataları yakalayıp bildirmek için ekledik. `EXIT_SUCCESS` ve
`EXIT_FAILURE` ise `cstdlib` dosyasından geliyor.

Programın kendisi bir sınıfa sarılmış durumda, Vulkan nesnelerini private eleman
olarak bu sınıfta saklayacağız. Fonksiyonları da bu sınıfa ekleyeceğiz, bunların
hemen hepsi `initVulkan` fonksiyonundan çağrılacak. Her şey hazır olduğunda asıl
döngüye girip karelerimizi çizmeye başlayacağız. `mainLoop` fonksiyonu, pencere
kapanana kadar çalışan bir döngü içerek. `mainLoop` kapanıp değer döndürdüğünde
`cleanup` fonksiyonu ile kullandığımız kaynakları temizleyeceğiz.

Ciddi bir hata oluşursa hata mesajı ile `std::runtime_error` exception
fırlatılacak. Geri `main` fonksiyonuna dönüp bu mesaj komut satırına
bastırılacak. Birkaç farklı exception işleyebilmek için daha genel olan
`std::exception` yakalayacağız. Yakında karşılaşabileceğimiz örnek hatalardan
biri gerekli bir uzantının desteklenmemesi olabilir.

Müteakip çoğu bölümde `initVulkan`'dan çağrılacak yeni bir fonksiyon
ekleyeceğiz. Bunun yanında sınıfa private bir Vulkan nesnesi ekleyip `cleanup`
fonksiyonunda bunları temizleyeceğiz.

## Kaynak yönetimi

Nasıl ki `malloc` ile ayrılan tüm bellek parçaları bir yerde `free` ile geri
salınmak zorundaysa oluşturduğumuz tüm Vulkan nesneleri de işimiz bittiğinde
elle temizlenmeli. C++'ta [RAII](https://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization)
konsepti veya `<memory>` başlığındaki akıllı pointerlar aracılığıyla bu işi
otomatize etmek mümkün. Ama bu derslerde Vulkan nesneleri için yer ayırma ve
temizleme işlemlerini açık bir şekilde elle yapmayı seçtik. Sonuçta Vulkan'ın
olayı yanlışlıkların önüne geçmek için tüm işlemleri açık bir şekilde
yaptırması. API'ın işleyişine alışmak için nesnelerin ömrüyle ilgili işlemlerin
de açık olmasında fayda var.

Bu eğitimin sonunda C++ özelliklerinden yararlanarak otomatik kaynak
yönetiminizi yazabilirsiniz. Sınıflarınıza Vulkan nesnelerini constructor
parametresi olarak verip, destructorda da bunları temizleyebilirsiniz. Veya
izin gereksinimlerinize göre `std::unique_ptr` ve `std::shared_ptr` tiplerine
kendi yer ayırıcınızı ve temizleyicinizi yazabilirsiniz. Büyük Vulkan projeleri
için RAII önerilen yöntemdir ama öğrenmek için arka planda neler döndüğünü
incelemek her zaman iyidir.

Vulkan nesneleri ya `vkCreateXXX` gibi fonksiyonlarla direkt olarak yaratılır ya
da `vkAllocateXXX` fonksiyonlarıyla başka bir nesne aracılığıyla yerleri
ayrılır. Bu nesnelerin başka bir yerde kullanılmayacağına emin olduğumuz zaman
ise bunların karşılığı olan `vkDestroyXXX` ve `vkFreeXXX` fonksiyonlarıyla yok
edilir. Bu fonksiyonların parametreleri nesnelere göre farklılık gösterirken
hepsinde ortak olan bir `pAllocator` parametresi var. Bu opsiyonel parametre
bellek ayırıcısına kendi callback fonksiyonunuzu vermenize yarar ama biz bu
derslerde bu parametreyi yok sayıp her zaman `nullptr` göndereceğiz.

## GLFW Entegrasyonu

Eğer sadece ekran dışı hesaplamalarla ilgileniyorsanız Vulkan bir pencere
olmadan da kullanılabilir ama ekrana bir şeyler çizdirmek çok daha eğlenceli!
İlk olarak `#include <vulkan/vulkan.h>` satırını aşağıdakiyle değiştireceğiz:

```c++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>
```

Böylece GLFW kendi tanımlarını ekleyip Vulkan headerlarını da kendi
yükleyecektir. `initWindow` fonksiyonu oluşturun ve `run` fonksiyonu içine
ilk çağrılacak şekilde ekleyin. GLFW başlatmasını ve pencere oluşturmayı burada
halledeceğiz.

```c++
void run() {
    initWindow();
    initVulkan();
    mainLoop();
    cleanup();
}

private:
    void initWindow() {

    }
```

`initWindow`'daki ilk çağrı GLFW'i başlatan `glfwInit()` olmalıdır. GLFW özünde
OpenGL için tasarlandığından, OpenGL içeriği oluşturmaması gerektiğini aşağıdaki
gibi özellikle belirtmeliyiz:

```c++
glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
```

Boyutu değişen pencereleri halletmek ekstra yük getirdiğinden buna daha sonra
bakacağız, şimdilik pencere boyutlandırmasını devre dışı bırakalım:

```c++
glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);
```

Sadece pencereyi yaratmak kaldı. Pencere referansını tutmak için sınıfa private
bir `GLFWwindow* window;` değişkeni ekleyin ve aşağıdaki gibi pencere oluşturun:

```c++
window = glfwCreateWindow(800, 600, "Vulkan", nullptr, nullptr);
```

İlk üç parametre pencerenin genişliğini, yüksekliğini ve başlığını belirtmekte.
Dördüncü parametre pencereyi açmak istediğiniz monitörü belirmek için
kullanılabilirken son parametre ise sadece OpenGL ile kullanırken işe yaramakta.

Doğrudan kodlanmış genişlik ve yükseklik değerleri yerine sabit değişkenler
kullanmak daha mantıklı çünkü bu değerlere ileride de ihtiyaç duyacağız.
Aşağıdaki tanımları `HelloTriangleApplication` sınıf tanımının üzerine ekledim:

```c++
const uint32_t WIDTH = 800;
const uint32_t HEIGHT = 600;
```

ve pencere oluşturma kodunu aşağıdakiyle değiştirdim:

```c++
window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
```

Şimdi şöyle bir `initWindow` fonksiyonuna sahip olmalısınız:

```c++
void initWindow() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);

    window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
}
```

Pencere kapatılana kadar veya bir hatayla karşılaşana kadar programın çalışması
için `mainLoop` fonksiyonuna aşağıdaki gibi bir olay döngüsü eklemeliyiz:

```c++
void mainLoop() {
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }
}
```

Bu kod yeterince açık olmalı, sürekli dönüyor ve kullanıcının X işaretiyle
pencereyi kapatması gibi olayları kontrol ediyor. Ayrıca her bir kareyi
çizdirmek için çağıracağımız fonksiyonlar da buraya eklenecek.

Pencere kapandığı anda kaynakları temizleyip GLFW'i kapatmalıyız. Bu bizim ilk
`cleanup` kodumuz olacak:

```c++
void cleanup() {
    glfwDestroyWindow(window);

    glfwTerminate();
}
```

Bu kodu çalıştırdığınızda ekranda `Vulkan` başlıklı bir pencere görmelisiniz,
siz kapatana kadar da orda kalmalı. Vulkan uygulamamızın isketeletini
tamamladığımıza göre [ilk Vulkan objemizi oluşturalım](!en/Drawing_a_triangle/Setup/Instance)!

[C++ kodu](/code/00_base_code.cpp)
