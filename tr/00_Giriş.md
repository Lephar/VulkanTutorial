## Hakkında

Bu kurs size [Vulkan](https://www.khronos.org/vulkan/) grafik ve hesaplama
API'ının temellerini öğretecektir. Vulkan, [Khronos grup](https://www.khronos.org/)
(OpenGL'in geliştiricileri) tarafından üretilmiş, modern grafik kartlarında çok
daha iyi soyutlama sağlayan yeni bir API (uygulama programlama arayüzü). Bu yeni
arayüz, programınızın ne yapacağını [OpenGL](https://en.wikipedia.org/wiki/OpenGL)
ve [Direct3D](https://en.wikipedia.org/wiki/Direct3D) gibi diğer API'lara göre
daha iyi tarif etmenize izin vererek daha yüksek performans elde etmenizi sağlar
ve daha az beklenmedik sürücü davranışına sebep olur. Vulkan'ın arkasında yatan
fikirler [Direct3D 12](https://en.wikipedia.org/wiki/Direct3D#Direct3D_12) ve
[Metal](https://en.wikipedia.org/wiki/Metal_(API))'inkilere çok benzer ama
Vulkan tamamen çoklu platform olma avantajına sahip; uygulamanızı Windows, Linux
ve Android platformlarında aynı anda geliştirmenize olanak sağlar.

Ancak bütün bunlardan faydalanmanız için çok daha ayrıntılı bir API ile
çalışmayı göze almış olmanız gerekiyor. Grafikle ilgili her türlü detay
programınız tarafından sıfırdan kurulmalı. Buna kare arabelleklerinin
oluşturulması ve doku resimleri gibi nesneler için bellek yönetimi de dahil.
Grafik sürücünüz size daha az yardımcı olacak, yani uygulamanızda doğru
davranışı elde etmeniz için daha çok iş yapmanız gerekecek.

Buradan çıkarılması gereken mesaj, Vulkan'ın herkes için olmadığıdır. Hedef
kitlesi yüksek performanslı bilgisayar grafiklerine hevesli ve bu alanda emek
harcamayı göze almış kişilerdir. Eğer bilgisayar grafiklerinden ziyade oyun
geliştirme ile ilgileniyorsanız OpenGL veya Direct3D kullanmak isteyebilirsiniz.
Yakın zamanda Vulkan tarafından yerleri doldurulacak gibi görünmüyor. Bir başka
alternatif ise [Unreal Engine](https://en.wikipedia.org/wiki/Unreal_Engine#Unreal_Engine_4)
veya [Unity](https://en.wikipedia.org/wiki/Unity_(game_engine)) gibi oyun
motorları kullanmak. Temelinde Vulkan kullanırken size çok daha üst seviye bir
arayüz sağlayabilirler.

Bütün bunları aradan çıkardığımıza göre bu dersleri takip edebilmeniz için
gerekenlerden bahsedelim:

* Vulkan uyumlu bir ekran kartı ve sürücüsü ([NVIDIA](https://developer.nvidia.com/vulkan-driver), [AMD](http://www.amd.com/en-us/innovations/software-technologies/technologies-gaming/vulkan), [Intel](https://software.intel.com/en-us/blogs/2016/03/14/new-intel-vulkan-beta-1540204404-graphics-driver-for-windows-78110-1540))
* C++ tecrübesi (RAII konseptine ve öndeğer atama listelerine aşinalık)
* C++17 özelliklerini destekleyen bir derleyici (Visual Studio 2017+, GCC 7+, veya Clang 5+)
* Belli bir seviye 3 boyutlu bilgisayar grafikleri tecrübesi

OpenGL veya Direct3D konseptlerine hakim olduğunuz varsayılmayacak ancak 3
boyutlu bilgisayar grafikleri temellerini bilmeniz gerekiyor. Mesela perspektif
projeksiyonun arkasında yatan matematiğe değinilmeyecek. Bilgisayar grafik
konseptlerine etkili bir giriş yapabilmek için [bu çevirimiçi kitaba](https://paroj.github.io/gltut/)
ve aşağıdaki kaynaklara göz atabilirsiniz:

* [Ray tracing in one weekend](https://github.com/petershirley/raytracinginoneweekend) (Bir haftasonunda ışın izleme)
* [Physically Based Rendering book](http://www.pbr-book.org/) (Fizik temelli çizim kitabı)
* Vulkan kullanılan gerçek açık kaynak kodlu motorlar [Quake](https://github.com/Novum/vkQuake) ve [DOOM 3](https://github.com/DustinHLand/vkDOOM3)

İsterseniz C++ yerine C kullanabilirsiniz ancak başka bir lineer cebir
kütüphanesi kullanmanız ve kodunuzu kendiniz yapılandırmanız gerekecek.
Mantığı kurmak ve kaynakları yönetmek için sınıflar ve RAII gibi C++ özellikleri
kullanacağız. Bu derslerin Rust geliştiricileri için [alternatif bir versiyonu](https://github.com/bwasty/vulkan-tutorial-rs)
da mevcut.

Başka diller kullanan geliştiricilerin daha rahat takip edebilmesi ve temel API
hakkında tecrübe kazanabilmek için Vulkan'ın orijinal C API'ını kullanacağız.
Eğer C++ kullanıyorsanız daha yeni olan [Vulkan-Hpp](https://github.com/KhronosGroup/Vulkan-Hpp)
bağlantılarını tercih edebilirsiniz. Sizi bazı zahmetli ayrıntılardan kurtarıp
bir takım hataların önüne geçecektir.

## E-kitap

Eğer bu dersleri e-kitap olarak okumayı tercih ederseniz EPUB veya PDF
versiyonlarını buradan indirebilirsiniz:

* [EPUB](https://raw.githubusercontent.com/Overv/VulkanTutorial/master/ebook/Vulkan%20Tutorial%20tr.epub)
* [PDF](https://raw.githubusercontent.com/Overv/VulkanTutorial/master/ebook/Vulkan%20Tutorial%20tr.pdf)

## Eğitim yapısı

Vulkan'ın nasıl çalıştığının ve ekrana bir üçgen çizmek için gerekenlerin kısa
bir özetiyle başlayacağız. Bu küçük adımların her biri, büyük resimdeki
rollerini gördüğünüzde anlam kazanacaktır. Sonrasında [Vulkan SDK](https://lunarg.com/vulkan-sdk/),
lineer cebir için [GLM](http://glm.g-truc.net/) ve pencere oluşturma için [GLFW](http://www.glfw.org/)
kütüphanelerini kullanarak geliştirme ortamımızı kuracağız. Windows'ta Visual
Studio, macOS'te Xcode ve Ubuntu'da GCC kullanarak bütün bunları nasıl
kuracağınız anlatılacak.

Bundan sonra ilk üçgenimizi çizmek için gereken tüm temel Vulkan parçaları
gerçeklemeye başlayacağız. Tüm bölümler kabaca şu yapıda olacak:

* Yeni bir kavram ve bunun altında yatan mantığa giriş
* İlgili API çağrılarını kullanarak bunu programa entegre etme
* Bazı parçaları yardımcı fonksiyonlar kullanarak soyutlama

Bütün bölümler birbirinin devamı şeklinde yazılmış da olsa, tüm bu bölümleri
Vulkan'ın ilgili özelliklerini öğrenmek için bağımsız olarak okuyabilirsiniz.
Yani bu site size bir referans olarak da faydalı olacaktır. Tüm Vulkan
değişkenleri ve fonksiyonları, resmi tanım sayfalarına bir link içermektedir.
İstediğiniz takdirde tıklayarak ayrıntılarına ulaşabilirsiniz. Vulkan çok yeni
bir API, yani bu tanımlarda da eksikler olabilir. Böyle bir durumla
karşılaştığınızda [bu Khronos deposuna](https://github.com/KhronosGroup/Vulkan-Docs)
geribildirimde bulunabilirsiniz.

Daha önce de bahsettiğimiz gibi Vulkan API, size grafik donanımı üzerinde mümkün
olan en yüksek kontrolü sağlayabilmek için çok parametreli, görece ayrıntılı bir
yapıda. Bu da doku üretmek gibi basit işlemler için bile tekrarlanması gereken
birçok adımın bulunmasına neden oluyor. Bu adımları kolaylaştırmak için dersler
boyunca kendi yardımcı fonksiyonlarımızı yazıyor olacağız.

Ayrıca her bölümün sonunda, o noktaya kadar yazılmış olan kodun tam haline bir
bağlantı bulacaksınız. Kodun yapısıyla ilgili aklınıza takılanlar veya
uygulamanızda çıkabilecek sorunlar için bu kodu karşılaştırma amaçlı
kullanabilirsiniz. Tüm kodlar, doğruluğundan emin olmak için birden fazla
üreticinin ekran kartlarında denenmiştir. Konu ile alakalı her türlü sorunuzu
sorabilmeniz için ayrıca her bölümün sonunda birer yorum kısmı da mevcut. Daha
rahat yardım alabilmeniz için lütfen işletim sisteminizi, sürücü versiyonunuzu,
kaynak kodunuzu, uygulamadan beklediğiniz davranışı ve elde ettiğiniz davranışı
da belirtin.

Bu eğitim tamamen topluluğun emekleriyle geliştirilmeyi amaçlamaktadır. Vulkan
hala çok yeni bir API ve en iyi yöntemler henüz tam anlamıyla belirlenmiş değil.
Eğer dersler veya site hakkında herhangi bir geribildiriminiz varsa lütfen [GitHub depomuza](https://github.com/Overv/VulkanTutorial)
sorun bildirmekten veya katkıda bulunmaktan çekinmeyin. Ayrıca eğitimde yapılan
güncellemelerden haberdar olmak için depoyu *izle* seçeneği de mevcut.

Feleğin çemberinden geçip ilk üçgenimizi ekranda gördükten sonra programımızı
lineer dönüşümler, yüzey dokuları ve 3 boyutlu modeller içerecek şekilde
genişleteceğiz.

Eğer daha önceden grafik API'ları ile uğraştıysanız ekrana ilk şekli çizmek için
birçok adım olabileceğini bilirsiniz. Bu ilk adımlardan Vulkan'da bolca var. Bu
adımların her birinin aslında gayet anlaşılabilir olduğunu ve alakasız
hissettirmediğini göreceksiniz. Ayrıca bilmelisiniz ki sıkıcı görünen üçgenimizi
ekrana çizdikten sonra yüzey dokusuna sahip üç boyutlu bir model çizmek o kadar
da fazla iş gerektirmeyecek, üçgenden sonraki her adım daha da tatmin edici
olacak.

Eğer dersleri takip ederken herhangi bir sorunla karşılaşırsanız öncelikle Sıkça
Sorulan Sorular kısmını kontrol ediniz. Sorununuzun çözümü zaten burada
açıklanmış olabilir. Problem hala devam ediyorsa en ilgili gördüğünüz konunun
yorumlar kısmında sormaktan çekinmeyin.

Geleceğin yüksek hızlı grafik arayüzünde kaybolmaya hazırsanız [haydi başlayalım!](!tr/Genel_Bakış)
