Bu bölüme Vulkan'a kısa bir giriş ve Vulkan'ın çözmeyi amaçladığı sorunlarla
başlayacağız. Sonrasında ilk üçgenimizi çizmek için gerekenlerden bahsedeceğiz.
Bu size, önümüzdeki her bölümün büyük resimdeki yerini göstermekte yardımcı
olacaktır. Vulkan'ın yapısı ve genel kullanım biçimleriyle de kapanışı
yapacağız.

## Vulkan'ın çıkış noktası

Daha önceki grafik API'ları gibi Vulkan da [GPU'lar](https://en.wikipedia.org/wiki/Graphics_processing_unit)
üzerinde çoklu platform bir soyutlama sağlamak üzere tasarlanmıştır. Bu
API'ların birçoğunun sorunu, tasarlandıkları çağın gereği olarak sabit bir
işlevselliğe sahip olmalarıdır. Yapılandırılabilir de olsalar bu kısıtlayıcı bir
etken. Programcı verteks verisini standart bir formatta vermeliydi ve
ışıklandırma ile gölgelendirme konularında GPU üreticisinin insafına kalıyordu.

Ekran kartı teknolojileri geliştikçe daha programlanabilir işlevsellikler
sağlamaya başladılar. Tüm bunlar var olan API'lara bir şekilde entegre edilmek
zorundaydı. Bu da ideal olmayan soyutlamalara sebep olmaya başladı. Sürücü
tarafında programcının niyetini anlamaya çalışmak ve bunları modern grafik
mimarilerine eşlemek gerekiyordu. Oyun performanslarını artırmak için sıklıkla
sürücü güncellemeleri olması ve bunlardan bazılarının performansı çok ciddi
şekilde artırmasının sebebi de tam olarak bu. Sürücülerin bu denli karmaşık
olmasından dolayı geliştiriciler de çoğu zaman farklı üreticiler arasındaki
tutarsızlıklarla baş etmek zorunda kalıyordu. [Gölgeleyiciler](https://en.wikipedia.org/wiki/Shader)
tarafından kabul edilen sözdizimlerinin farklılıkları buna güzel bir örnek.
Eklenen yeni özelliklerin yanı sıra, geçtiğimiz yıllarda güçlü grafik
donanımlarına sahip mobil cihazların sayısında da ciddi bir artış yaşandı.
Bu mobil GPU'lar, enerji tüketimi ve boyut kısıtlarından dolayı farklı
mimarilere sahipler. Örneğin [tiled rendering](https://en.wikipedia.org/wiki/Tiled_rendering),
programcıya bu özellik üzerinde yeterince kontrol verildiği takdirde çok ciddi
performans artışlarına sebep olabilmekte. Çağın yol açtığı bir diğer sorun ise
çok çekirdekli mimari desteğinin kısıtlı olması. Bu da [CPU](https://en.wikipedia.org/wiki/Central_processing_unit)
tarafında darboğazlara sebep olmaktaydı.

Vulkan tüm bu problemleri, modern grafik mimarilerine göre sıfırdan tasarlandığı
için çözüyor. Programcıya daha detaylı bir API sunup niyetini daha rahat
anlatmasına izin vererek, sürücüyü fazlalık yüklerden kurtarıyor. Birden fazla
iş parçacığının eş zamanlı olarak komut göndermesine olanak sağlıyor. Standart
bir bayt kodu formatı ve derleyicisi getirerek, gölgeleyici derlemelerindeki
tutarsızlıklardan kurtarıyor. Ve son olarak, modern grafik kartlarının grafik
işlemleri dışındaki kullanımlarını da göz önünde bulundurarak, grafik ve
hesaplama işlevlerinin hepsini tek bir çatı altında birleştiriyor.

## Üçgen çizmek için ne lazım

Şimdi iyi tasarlanmış bir Vulkan programında üçgen çizmek için gerekli aşamalara
genel bir bakış ile başlayacağız. Tüm bu kavramların üzerinden ilgili bölümlerde
ayrıntılı olarak geçilecek. Bu kısım sadece, parçaların bütündeki bağlantılarını
kurabilmeniz için var.

### 1. Adım - Instance oluşturma ve fiziksel aygıt seçimi

Her Vulkan uygulaması `VkInstance` aracılığıyla Vulkan API'ını kurarak başlar.
Instance, uygulamanızı betimleyerek ve gereken uzantıları (extensions)
belirterek oluşturulur. Oluşturulduktan sonra Vulkan destekli donanımları
sorgulatarak ileriki işlemlerde kullanmak üzere bir veya birden fazla
`VkPhysicalDevice` (fiziksel aygıt) seçebilirsiniz. VRAM boyutu ve aygıt
özelliklerini gibi detayları sorgulayarak kullanacağınız cihaza karar
verebilirsiniz. Örneğin sadece harici ekran kartını kullanmayı tercih
edebilirsiniz.

### 2. Adım - Mantıksal aygıt ve kuyruk aileleri

Gerekli donanımı seçtikten sonra bir `VkDevice` (mantıksal aygıt) oluşturmamız
gerekiyor. Çoklu görüntü kapısı (multi viewport) ve 64 bit noktalı sayılar gibi
kullanacağımız `VkPhysicalDeviceFeatures` (fiziksel aygıt özellikleri) bu
kısımda belirlenecek. Ayrıca hangi kuyruk ailelerini (queue families) kullanmak
istediğimizi de belirtmemiz gerekli. Vulkan'da kullanacağımız çizim komutları,
bellek işlemleri gibi çoğu operasyon, `VkQueue`'ya (kuyruğa) gönderilerek
asenkron bir şekilde çalıştırılır. Kuyruklar kuyruk ailelerinden seçilir. Her
kuyruk ailesi, elemanı olan kuyruklarda belirli işlemleri destekler. Örneğin
grafik, hesaplama ve bellek aktarımı işlemleri için ayrı kuyruk aileleri
olabilir. Gerekli kuyruk ailelerinin varlığı da fiziksel aygıt seçiminde bir
etken olabilir. Vulkan destekli bir donanımın hiç grafik işlevine sahip olmaması
gayet muhtemel. Ancak günümüzdeki neredeyse tüm Vulkan destekli grafik kartları
bizim ilgilendiğimiz kuyruk işlemlerini desteklemekte.

### 3. Adım - Pencere yüzeyi ve takas zinciri

Eğer sadece ekran dışı çizimlerle ilgilenmiyorsanız, çizilen kareleri sunmak
için bir pencere (window) oluşturmanız gerekiyor. Pencereler kullandığınız
platformun kendi API'ları veya [GLFW](http://www.glfw.org/) ve [SDL](https://www.libsdl.org/)
gibi kütüphaneler aracılığıyla oluşturulabilir. Biz bu derslerde GLFW
kullanacağız. Ayrıntılar sonraki bölümde.

Pencereye çizebilmek için iki bileşene daha ihtiyacımız var: `VkSurfaceKHR`
(pencere yüzeyi) ve `VkSwapchainKHR` (takas zinciri). `KHR` son eki, bu
nesnelerin bir Vulkan uzantısına ait olduğunu belirtiyor. Vulkan tamamen
platformdan bağımsız bir API, bu nedenle pencere yöneticisyle (windows manager)
iletişim kurabilmek için standart bir WSI (pencere sistem arayüzü) uzantısı
kullanmamız gerekiyor. Pencere yüzeyi (windows surface) ise pencereler üzerinde
platformdan bağımsız bir soyutlama sağlayan bir yapı. Genellikle platformun
kendi pencere işleyicisine bir referans verilerek oluşturulmakta. Örneğin
Windows için işleyici `HWND`. Neyse ki GLFW kütüphanesi bizi bu tür platform
ayrıntılarından kurtaracak işlevsellikle birlikte geliyor.

Takas zinciri ise üzerine çizim yapacağımız hedefler topluluğudur. Temel amacı
şu an üzerine çizim yaptığımız karenin ekranda gösterilenden farklı olmasını
sağlamaktır. Bu, sadece tamamlanmış karelerin ekranda gösterilmesi açısından
önemlidir. Ne zaman bir kareye çizim yapmak istesek, takas zincirinden üzerine
çizim yapabileceğimiz bir kare istememiz gerekiyor. Çizimimiz bittiğindeyse
bu kare takas zincirine geri döndürülür ve zamanı geldiğinde ekranda gösterilir.
Çizim hedeflerimizin sayısı ve ekranda gösterilmesi için gereken şartlar, sunum
moduna bağlıdır. En yaygın sunum modları çift tamponlama (double buffering veya
v-sync) ve üçlü tamponlamadır (triple buffering). Takas zinciri bölümünde
bunlara daha ayrıntılı bakacağız.

Bazı platformlar araya hiç pencere yöneticisi sokmadan `VK_KHR_display` ve
`VK_KHR_display_swapchain` uzantıları aracılığıyla ekrana çizim yapmanıza izin
vermekte. Bu size tüm ekranı içeren bir çizim yüzeyi oluşturma şansı tanır.
Örnek olarak kendi pencere yöneticinizi yazmak için bunları kullanabilirsiniz.

### 4. Adım - Resim görünümleri and kare arabellekleri

Takas zincirinden alınan bir kareyi çizebilmek için bunu bir `VkImageView`
(resim görünümü) ve `VkFramebuffer` (kare arabelleği) içine sarmamız gerekmekte.
Resim görünümleri, kullanılacak bir resmin belirli bölgelerine referans veren
yapılardır. Kare arabellekleriyse renk, derinlik ve kalıp hedefleri olarak
kullanılacak resim görüntülerine referans verir. Takas zincirinde birden fazla
resim olabileceğinden, hepsi için birer tane resim görünümü ve kare arabelleğini
önceden oluşturup, zamanı geldiğinde doğru olanı seçip ekranda göstereceğiz.

### 5. Adım - Çizim geçişleri

Vulkan'da çizim geçişleri (render passes), çizim işlemleri sırasında
kullanacağımız resimlerin türlerinin, nasıl kullanılacaklarının ve içeriklerine
nasıl davranılacağının belirlendiği kısımdır. İlk üçgen uygulamamızda Vulkan'a
sadece tek resim kullanacağımızı, bunun renk verisi tutacağını ve her çizimden
hemen önce bu resmin düz bir renk ile temizleneceğini söyleyeceğiz. Çizim geçişi
sadece resimlerin türlerini belirleyecek, bu resimleri gerekli yerlere asıl
bağlama işlemini `VkFramebuffer` gerçekleştirecek.

### 6. Adım - Grafik boru hattı

Vulkan'da grafik boru hattı, bir `VkPipeline` nesnesi oluşturularak kurulur.
Ekran kartının yapılandırılabilir durumları burada belirlenir. Görüntü kapısı
boyutu, derinlik arabelleği işlemleri ve `VkShaderModule` (gölgeleyici modülü)
kullanan programlanabilir durumlar bunların en net örnekleri. `VkShaderModule`
nesneleri, gölgeleyici bayt kodundan oluşturulur. Ayrıca sürücü, boru hattında
hangi çizim hedeflerinin kullanılacağını da bilmeli, bunu da çizim geçişinin
referansını vererek belirliyoruz.

Vulkan'ın diğer API'lara karşı en öne çıkan özelliklerinden biri, boru hattının
neredeyse tüm yapılandırılmasının önceden halledilmesi gerekmesi. Bu sebeple,
eğer başka bir gölgelendiriciye geçmek isterseniz veya verteks diziliminizde
ufak değişiklikler yaparsanız tüm grafik boru hattını baştan oluşturmanız
gerekmekte. Yani çizim işlemleriniz için gerekecek her durum için önceden birer
`VkPipeline` oluşturmanız gerekiyor. Yalnızca görüş kapısı boyutu, arka plan
temizleme rengi gibi bazı basit ayarlar dinamik olarak değiştirilebilir. Tüm
durum değişkenleri de açık bir şekilde tanımlanmalı. Örneğin varsayılan bir
renk karıştırma yöntemi mevut değil, elle belirtilmesi gerekiyor.

İyi haber ise, zamanı geldiğinde yapılacak anlık değişiklikler yerine tüm bu
düzenlemeleri önceden yaptığımız için ve tamamen farklı bir boru hattına geçiş
yapmak gibi önemli durum değişiklikleri önceden net bir şekilde belirtildiği
için sürücüye daha fazla optimizasyon fırsatı sağlıyoruz ve çalışma zamanı
performansımız daha tahmin edilebilir oluyor.

### 7. Adım - Komut havuzları ve komut arabellekleri

Daha önceden de bahsettiğimiz gibi, çizim işlemleri gibi Vulkan'da yapmak
istediğimiz birçok operasyonu önceden bir kuyruğa yollamamız gerekiyor. Bu
operasyonlar gönderilmeden önce bir `VkCommandBuffer` (komut arabelleği)
yapısına kaydedilmeli. Bu komut kuyruklarının alanları, gerekli kuyruk ailesine
bağlı olan `VkCommandPool` (komut havuzu) yapısından ayrılır. Basit bir üçgen
çizmek için aşağıdaki işlemleri içeren bir komut arabelleği kaydetmemiz
gerekiyor:

* Çizim geçişini başlat
* Grafik boru hattını bağla
* 3 tane verteks çiz
* Çizim geçişini sonlandır

Kare arabelleğindeki resim, takas zincirinin bize vereceği resme bağlı olduğu
için, mümkün olan tüm resimler için ayrı birer komut arabelleği kaydedip, çizim
zamanında bunlardan doğru olanını seçmemiz gerekmekte. Bir diğer yöntem de
ekrana çizilecek her kare için komut arabelleğini baştan kaydetmek olurdu ki,
bu hiç verimli bir yöntem değil.

### 8. Adım - Ana döngü

Çizim komutlarımızı komut arabelleklerine kaydettikten sonra ana döngümüz gayet
basit görünüyor. İlk olarak `vkAcquireNextImageKHR` (sıradaki resmi getir)
fonksiyonu ile takas zincirinden bir resim alıyoruz. Sonrasında bu resim için
uygun olan komut arabelleğini seçip `vkQueueSubmit` (kuyruğu gönder) fonksiyonu
ile çalıştırabiliriz. Ve son olarak ekranda göstermek için resmi takas zincirine
`vkQueuePresentKHR` (kuyruğu sun) fonksiyonu ile geri döndürüyoruz.

Kuyruğa gönderilen işlemler asenkron bir şekilde çalıştırılır. Bu sebeple doğru
bir çalışma sırası sağlayabilmek için semaforlar gibi senkronizasyon nesnelerine
ihtiyacımız olacak. Çizim komut arabelleğinin çalıştırılması, çizilecek resmin
getirilmesini bekleyecek şekilde ayarlanmalı. Yoksa ekranda gösterilmek için
halen bellekten okunmakta olan bir resme çizim yapmaya başlayabiliriz.
`vkQueuePresentKHR` çağrısı da çizimin bitmesini beklemek zorunda. Bunun için de
çizim bittiğinde sinyal alan ikinci bir semafor kullanacağız.

### Özet

Bu yıldırım hızındaki turumuz, ilk üçgenimizi çizerken karşılaşacaklarımız
hakkında kısa bir bilgi vermiştir. Ancak gerçek hayatta karşımıza çıkacak bir
program verteks arabelleklerine yer ayırma, ortak arabellekleri (uniform
buffers) oluşturma ve doku resimlerini (texture images) yükleme gibi birçok adım
daha içeriyor. Bunların hepsine akabindeki bölümlerde değinilecek olsa da biz en
başta kolaydan başlayacağız. Zaten Vulkan yeterince dik bir öğrenme eğrisine
sahipken işleri daha baştan zorlaştırmanın gereği yok. Hatta başlangıçta bir
verteks arabelleği kullanmak yerine koordinatları verteks gölgeleyicisine gömmek
gibi hilelere bile başvuracağız. Çünkü verteks arabelleklerini yönetmek için
komut arabelleklerine de biraz aşina olmak gerekiyor.

Yani kısaca ilk üçgenimizi çizmek için şunları yapacağız:

* `VkInstance` oluştur
* Desteklenen bir fiziksel aygıt seç (`VkPhysicalDevice`)
* Çizim ve sunum için birer `VkDevice` ve `VkQueue` oluştur
* Bir pencere, pencere yüzeyi ve takas zinciri oluştur
* Takas zinciri resimlerini birer `VkImageView`'a sar
* Çizim hedeflerini ve kullanımlarını belirten çizim geçişini oluştur
* Çizim geçişleri için çizim arabelleklerini oluştur
* Grafik boru hattını kur
* Çizim komutları için her bir takas zinciri resmine ait komut arabelleklerini
ayır ve kaydet
* Sırası gelen resmi alarak, doğru çizim komut arabelleğini kuyruğa yollayarak
ve resmi geri takas zincirine göndererek kareleri ekrana çiz

Bir sürü adım var ama önümüzdeki her bölümde bu adımların her biri çok basit ve
açık bir şekilde anlatılacak. Eğer bir adımın, programın tümüyle olan bağlantısı
hakkında aklınıza takılan bir şeyler olursa bu bölüme tekrar dönmelisiniz.

## API konseptleri

Bu bölümde Vulkan API'ının daha alt seviyede nasıl yapılandığından bahsederek
kapanışı yapacağız.

### Kodlama usülleri

Vulkan'daki tüm fonksiyonlar, enumlar ve structlar, [LunarG](https://www.lunarg.com/)
tarafından geliştirilmiş olan [Vulkan SDK](https://lunarg.com/vulkan-sdk/) ile
birlikte gelen `vulkan.h` header dosyasında tanımlanmıştır. Bu SDK (yazılım
geliştirme kiti) için yükleme adımlarına bir sonraki bölümde değineceğiz.

Fonksiyonlar küçük harfle başlayan `vk` ön ekine sahip, enumlar ve structlar
gibi tipler `Vk` öne ekine sahip, ve enum değerleri `VK_` ön ekine sahip. API
fonksiyonlara parametre sağlamak için bu structları sıklıkla kullanmakta.
Örneğin nesne oluşturma genelde şu biçimde gerçekleştiriliyor:

```c++
VkXXXCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_XXX_CREATE_INFO;
createInfo.pNext = nullptr;
createInfo.foo = ...;
createInfo.bar = ...;

VkXXX object;
if (vkCreateXXX(&createInfo, nullptr, &object) != VK_SUCCESS) {
    std::cerr << "failed to create object" << std::endl;
    return false;
}
```

Vulkan'daki çoğu struct, structın tipini `sType` (tip) elemanında elle
belirtmenizi gerektiriyor. `pNext` (sıradaki) elemanları ise uzantı structlarına
referans verebilir ancak bu derslerde her zaman `nullptr` olacak. Bir nesne
oluşturan veya yok eden fonksiyonların `VkAllocationCallbacks` (yer ayırma geri
çağrıları) parametresi mevcut, sürücü belleği için kendi yer ayırma
fonksiyonunuzu kullanmanızı sağlıyor ancak bu da derslerde `nullptr` olarak
kalacak.

Neredeyse tüm fonksiyonlar `VkResult` (sonuç) tipinde bir değer döndürür,
başarılıysa `VK_SUCCESS` (başarı), bir hatayla karşılaşıldıysa da belli bir hata
kodu. Resmi tanımlarda her fonksiyon için karşınıza çıkabilecek hata kodlarını
ve bunların ne anlama geldiklerini bulabilirsiniz.

### Doğrulama katmanları

Daha önce de bahsedildiği gibi Vulkan yüksek performans ve düşük sürücü yükü
için tasarlanmıştır. Bu nedenle çok kısıtlı hata kontrolü ve debug kapasitesiyle
gelecektir. Eğer bir hata yaparsanız sürücü bir hata kodu vermek yerine
genellikle sessizce kapanacaktır. Daha da kötüsü, sizin kartınızda çalışıyor
gibi görünüp başka kartlarda tamamen hüsrana uğrayacaktır.

Vulkan, *doğrulama katmanları* (validation layers) aracılığıyla kapsamlı bir
hata kontrolüne izin vermekte. Doğrulama katmanları, API ile sürücü arasında kod
sokarak ek fonksiyon parametre kontrolleri ve bellek yönetimi hata takibi gibi
kontroller yapmanıza olanak sağlayan bir yapı. İşin güzel tarafı, bu katmanları
geliştirme sırasında etkinleştirip uygulamayı yayınlarken de sıfır yük
oluşturacak şekilde devre dışı bırakabilmeniz. İsteyen kendi doğrulama
katmanlarını yazabilir ancak biz bu derslerde LunarG'nin sağladığı Vulkan SDK
ile beraber gelen standart doğrulama katmanlarını kullanacağız. Ayrıca bu
katmanlardan debug mesajlarını alabilmek için bir geri çağrı fonksiyonu
(callback function) yerleştirmemiz gerekli.

Vulkan her işlem için açık seçik ve detaylı olduğundan doğrulama katmanları da
çok kapsamlı. Hatta neden siyah bir ekranla karşı karşıya kaldığınızı anlamak
OpenGL ve Direct3D'ye göre çok daha kolay bile olabilir!

Kodumuzu yazmaya başlamadan önce tek bir adımımız kaldı, o da [geliştirme ortamımızı kurmak](!tr/Geliştirme_Ortamı).
