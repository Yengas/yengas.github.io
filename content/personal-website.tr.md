+++
type = "posts"
title = "Bu siteyi nasıl yaptım?"
date = "2016-11-20T16:30:00+03:00"
Categories = ["Kişisel", "Proje", "Continous Integration"]
Tags = ["wercker", "github", "markdown", "hugo", "golang", "ghpages"]
+++

> [2023-08-16] Daha minimal bir design ve aydınlık/karanlık teması desteği sebebi ile, [anubis teması](https://github.com/Mitrichius/hugo-theme-anubis)'na geçiş yaptım.

> [2020-05-14] [Github Actions](https://github.com/features/actions) yayınlandıktan sonra, bu projeyi wercker yerine Github Actions kullanacak şekilde değiştirdim. Blog yazıları ve Hugo konfigürasyonu [Yengas/yengas.github.io](https://github.com/Yengas/yengas.github.io/tree/source) projesinin *source* branch'inde yer alıyor. Bu branch'de yapılan her değişiklikte Github Actions html/css/js dosyalarını oluşturup, aynı projenin *master* branch'ine bu dosyaları gönderiyor.

Web platformunu bırakıp, `Android` ve daha sonrada diğer platform ve teknolojiler ile ilgilenmeye başladığımdan beri, arayüzden çok, backend sistemler ile uğraşmaya başladım. Bu dönemde tamamladığım çoğu proje; ya tek sayfalık bir arayüze sahipti, yada direk olarak arkaplanda çalışıyordu.

Kişisel bir websitesi açmak istediğim zaman, son zamanlarda ne kadar Web ile ilgilenmemiş olsamda, hackernews benzeri sitelerden hakkında çok okuduğum: `static website generator`'ları kullanmak istedim. Kısaca bir araştırma yaptıktan sonra, benim için en mantıklı tercihin `Hugo` olduğuna karar verdim. Hosting olarak ise tüm projeyi `git` ile yönetebilmek için, `Github`'ın herkese bedava olarak sağladığı `Github Pages` altyapısını kullanmaya karar verdim.

Aklımdaki senaryo, Github üzerinde 2 farklı proje oluşturmak, bunlardan birine `personal-website` adını vermek; bu proje içerisinde Hugo konfigürasyonlarımı, makaleleri tutmak, diğer projeye `yengas.github.io` adını vermek ve burada ise Hugo'nun oluşturduğu html/css/js website çıktısını barındırmaktı. Bu yapının üzerine kuracağım otomatik bir sistem, `personal-website`'ye herhangi bir değişiklik gönderdiğimde, otomatik olarak `yengas.github.io`'nun güncellenmesine sebep olacaktı. Github otomatik olarak `kullaniciadiniz.github.io` adı ile oluşturduğunuz projeleri websitesi olarak yayınladığı için; git veya github kullanarak, istediğim yerden websitemi otomatik olarak güncelleyebilecektim.

İlk başlarda bu otomatik sistemi oluşturmak için bir `docker-compose` konfigürasyonu hazırlayıp, kişisel sunucuma `Jenkins` kurmayı ve kendi sunucumda bir `Continous Integration` ortamı hazırlamayı düşünüyordum. Fakat biraz daha araştırdıktan ve Hugo dökümantasyonlarını okuduktan sonra, `wercker` ismi verilen 3. parti bir servis ile, bu otomatik build işlemini yapabileceğimi öğrendim.

Bu kurulum ile tamamen ücretsiz bir şekilde, herhangi bir hosting veya domain ücreti vermeden, kişisel bir websitesi oluşturmayı başardım. Böylece sitemin performansı Github Pages'a ve sitemin güvenliği ise Github şifremi saklamama dayalı olan bir sistem kurmuş oldum.

Sistemi kurarken Github'ın `.github.io` siteler için `master` branch'i dışında herhangi bir branch'e website upload edememe, wercker üzerined bulunan hazır build/deploy scriptlerinin eski olması gibi sorunlar yaşadım. Bu sebepten dolayı wercker konfigrasyonumun küçük bir kısmını el ile düzenlemem gerekti.

Aynı zamanda sitemin birden fazla dili desteklemesini, ve seri şeklinde makaleler yazabilmeyi istiyordum. Bu aşamadada biraz `Hugo` ve `go templates` öğrenip, hazır olarak kullandığım [redlounge](https://github.com/tmaiaroto/hugo-redlounge) temasını el ile düzenlemem gerekti.

Benzer bir sistem kurmak isterseniz, sitemdeki bir hatayı düzeltmek isterseniz; [personal-website](https://github.com/Yengas/personal-website), sitemin html/css/js halinin nasıl host edildiğini görmek isterseniz; [yengas.github.io](https://github.com/Yengas/yengas.github.io) projesini inceleyebilirsiniz. Hugo kullanmayı düşünüyorsanız, kullanmakta olduğum [özelleştirilmiş hugo-redlounge](https://github.com/Yengas/hugo-redlounge) temamada bakmanızı öneririm.
