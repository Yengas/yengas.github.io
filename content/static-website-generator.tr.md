+++
type = "posts"
title = "Statik Website Oluşturucuları nedir? Neden tercih edilir?"
date = "2016-11-20T15:00:00+03:00"
Categories = ["Araç"]
Tags = ["jekyll", "hugo", "statik website", "aws", "github", "ghpages"]
+++

Web'in ilk zamanlarında, tamamen statik olarak çalışan websiteleri, daha sonradan kullanıcılar için özelleştirilmiş deneyimler sunmak ve kullanıcı girdisine göre işlem yaparak çıktı verebilmek için dinamik bir hal aldı. Fakat tarayıcılarımız güçlendikce; bir çok şeyi `Javascript` kullanarak yapabilir hale geldik. Bu durum hem websitelerinin istemci kısmının daha çok yük üstlenebilmesini, hemde bir websitesinin bir masaüstü veya mobil uygulama gibi, uzak sunucusundan tamamen bağımsız çalışabilmesine olanak sağladı.
 
 Tarayıcı ve Sunucularda çalışan sistemlerin birbirinden bağımsız çalışabilmesi ve geliştirilebilmesi; Javascript kullanarak sitemize entegre edebileceğimiz 3. parti servislerin sayısınıda arttırdı. Bu dönemde bir çok basit içerikli siteyi, tamamen statik olarak çalıştırabiliyoruz ve sunucu ihtiyaçlarımız için 3. parti sistemler kullanabiliyoruz(yorum, arama, analatik ve raporlama). 

Bu yazımda statik wesitesi oluşturmak için çokca kullanılan statik websitesi oluşturma araçlarından ve faydalarından, kısaca bahsetmek istiyorum.


## Nedir?
Statik websitesi oluşturucuları, verdiğimiz bir grup konfigürasyonu ve belirli formatlarda(örn. `markdown`) yazdığımız içerikleri, tamamen statik bir html/css/js yığınına dönüştürme görevini üstlenir. 

Belirli şablonlar kullanılarak hazırlanan websiteleri, değişken içerikten arındılır ve yeni bir içerik eklendiği zaman, bir araç çalıştırılarak, tüm site veya sadece değişen kısımları tekrardan oluşturulur. Böylelikle websitenizi her kullanıcı isteğinde dinamik olarak oluşturmak yerine, sitenizde değişiklik yaptığınız an, statik olarak oluşturmuş olursunuz. Sitenizin şablonuna ekleyeceğiniz Javascript dosyaları ilede, sitenizi olabildiğince çok iş yapabilecek hale getirebilirsiniz.

Bunu bir blog örneği vermek için kullanırsak, makale sayfası için bir şablon hazırladıktan sonra, bir araç çalıştırarak, belirli bir klasöre koyduğunuz tüm .txt formatında yazılmış makaleler için, .html çıktılı sayfalar almak olarak düşünebilirsiniz. Bu aracın tek yaptığı şey, şablonun içinde belirttiğiniz yere, .txt içerisinde bulunan makale içeriğini yapıştırarak, yeni bir dosya oluşturmak olur.

Çoğu statik website oluşturucusu, bu basit mantık üzerine, parçalı şablonlar, değişik içerik formatı desteği, değişkenler, kategori/etiket/seri yapıları, global ayarlar gibi sistemler bindirerek çalışır.

Verdiğimiz blog örneği için sitenize koyacağınız javascript kodlar ile; `disqus` benzeri bir sistem kullanarak yorum ihtiyaçlarınızı, `google analytics` kullanarak ise hit takip ve raporlama işlerinizi halledebilirsiniz.

## Avantajları
Kişisel blog veya ürün/kişi/kurum tanıtımı gibi çok fazla değişiklik görmeyen veya ağırlıklı olarak istemci tarafında çalışan websiteleri için statik websitesi oluşturucusu kullanmak mantıklıdır. 

Bu araçların diğer avantajlarını kısaca sıralamak gerekirse;

### Performans
Dinamik websitesi geliştirmek ne kadar gelişmiş bir sektör olsada, tamamen statik olan içeriği kullanıcıza göndermekten daha performanslı bir şey olamaz. Statik Websitelerinde performans ile ilgili sorun yaratabilecek işlemler, Javascript ile kullanıcı tarafında yapacağınız işlemler olacaktır.

Statik olarak sunduğunuz websitenizi barındıran sunucunuz, dinamik bir websiteye göre anlık olarak daha fazla kişiye içerik servis edebilir.

### Güvenlik
`En güvenli bilgisayar, fişi çekik bilgisayardır.` sözünü hepimiz duymuşuzdur. Statik Websiteleri hata yapma şansınızı azaltır. Eğer statik websiteniz, çok fazla sunucu iletişimi içeren bir webapp'e dönüşmemişse, bir hackerın sitenizi ele geçirmek için yapabileceği tek şey; hosting bilgilerinizi çalmak veya bulmak olabilir.

### Sunucu Yönetimi
Statik bir websitenin sunucu maliyeti neredeyse yoktur. Hatta sadece 3. parti servisler kullanarak dinamik içerik gösteriyorsanız; sunucunuzun tek yapması gereken şey .html dosya servis etmekdir.

Bu yüzden veritabanı, programlama dili versiyonu, gerekli kütüphaneler gibi şeylerle uğraşmak ve bunları güncel tutmak zorunda kalmazsınız.

## Teknolojiler
Statik olarak website oluşturmak için bir çok araç ve gereç bulabilirsiniz. Benim kendi websitemde kullandığım araç: [Hugo](https://gohugo.io) adı verilen, şablonlama sistemi için `Golang Template` kullanan bir oluşturucu. Golang ile yazıldığı için, tek bir çalıştırılabilir dosya ile website oluşturmaya başlamanız mümkün.

Sektörde en çok kullanılan statik website oluşturucusu ise [Jekyll](https://jekyllrb.com/) olarak görünüyor.

Bu konuda pek bir deneyimim olmadığı için daha fazla bir şey yazmak istemiyorum. [Yararlı Kaynaklar](#yararlı-kaynaklar) bölmesinde, en çok kullanılan statik website oluşturucuları görebilirsiniz. Bir sonraki websitenizi yazmaya başlamadan önce, bu teknolojileri araştırıp, statik website oluşturmaya bir şans vermenizi tavsiye ederim. 

## Barındırma (Hosting)
Statik Website oluşturucularının en hoş kısımlarından biride, bu projeleri barındırmak için sahip olduğunuz seçeneklerdir. Temelde ihtiyacınız olan tek şey .html servis eden bir sunucu olduğu için; bir çok hosting firması ücretli/ücretsiz statik websiteler için hosting sağlamaktadır. Bunun en güzel örneği Github üzerinde tamamen ücretsiz olarak elde edebileceğiniz `.github.io` domaini ve ücretsiz hostingi. `Github Pages` adı verilen bu sistemde yapmanız gereken tek şey; adı `kullanıcıadınız.github.io` olacak şekilde bir git projesi oluşturmak, ve statik websitenizin html/css/js içeriğini buraya koymak. Bu işlemi el ile sitenizi her değiştirdiğinizdede yapabilir, 3. parti bir servis ile otomatiğede bağlayabilirsiniz. Böylece tamamen ücretsiz bir şekilde, websitenizi yayınlayabilirsiniz.

Alternatif hosting yöntemleri için kişisel sunucunuza `Nginx` web sunucusu kurabilir, veya `Amazon Web Services` gibi cloud çözümleri inceleyebilirsiniz.

Eğer `Hugo` ile `Github Pages` üzerine kurduğum, tamamen otomatik olarak çalışan bu blog'u nasıl yaptığımı incelemek isterseniz, [Bu bloğu nasıl yaptım?](/tr/personal-website) başlıklı makalemi okuyabilirsiniz.

## Yararlı Kaynaklar
- [Statik websitelerinizde kullanabileceğiniz, servisler ve teknolojiler](https://github.com/aharris88/awesome-static-website-services)
- [En çok kullanılan statik website oluşturucuları](https://www.staticgen.com/)
