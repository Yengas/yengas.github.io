+++
title = "Node.JS Event Loop, Sync/Async, Promise, Async/Await ve RxJS üzerine..."
date = "2018-03-10T03:28:00+03:00"
Categories = ["Node.JS", "Programming Paradigms"]
Tags = ["node.js", "promise", "async/await", "event loop", "performance", "rxjs"]
+++

Bu makalede Event Loop, Sync/Async, Promise, Async/Await, RxJS kavramlarını anlatmaya ve tüm bu kavramları tek bir örnek üzerinde kodlayarak göstermeye çalışacağım. Kendi sisteminizde denemeler yapmak istiyorsanız Node versiyonunuzun 8'in üzerinde olduğundan emin olun.

Bu makalede yazılan tüm kodların, çalışır hallerine, [yengas/async-blog-post](https://github.com/Yengas/async-blog-post)@Github adresinden ulaşabilirsiniz.

## Problem
İlk önce problemimiz ile başlayalım... Elimde daha önce izlemek için kaydettiğim Ghibli filmlerinin id'leri var. Bu filmlerin hepsinin adını ve açıklamasını almak istiyorum.

İlk önce elimdeki dosyanın formatına bakalım... 
```
2baf70d1-42bb-4437-b551-e5fed5a87abe
12cfb892-aac0-4c5b-94af-521852e46d6a
58611129-2dbc-4a81-a72f-77ddfc1b1b49
ea660b10-85c4-4ae3-8a5f-41cea3648e3e
4e236f34-b981-41c3-8c65-f8c9000b94e7
ebbb6b7c-945c-41ee-a792-de0e43191bd8
1b67aa9a-2e4a-45af-ac98-64d6ad15b16c
ff24da26-a969-4f0e-ba1e-a122ead6c6e3
0440483e-ca0e-4120-8c50-4c8cd9b965d6
45204234-adfd-45cb-a505-a8e7a676b114
dc2e6bd1-8156-4886-adff-b39e6043af0c
90b72513-afd4-4570-84de-a56c312fdf81
cd3d059c-09f4-4ff3-8d63-bc765a5184fa
112c1e67-726f-40b1-ac17-6974127bb9b9
758bf02e-3122-46e0-884e-67cf83df1786
2de9426b-914a-4a06-a3a0-5e6d9d3886f6
45db04e4-304a-4933-9823-33f389e8d74d
67405111-37a5-438f-81cc-4666af60c800
578ae244-7750-4d9f-867b-f3cd3d6fecf4
5fdfb320-2a02-49a7-94ff-5ca418cae602
```
Her satır, takip ettiğim bir bitcoin'i ifade ediyor. Güzel... Yeni satır karakteri '\n' ile, satırları birbirinden ayırmam yetecek. Ama ilk önce tüm listeyi işlemeye başlamadan önce, tek bir id için başlık ve açıklama alabileceğimden emin olmam lazım.

## Tek bir film, sync olarak işlem yapmak.
Algoritma basit. Bir değişkende, bilgilerini bulmak istediğim filmin id'sini tutacağım, API'ye bir istek göndereceğim, gelen cevabı JSON olarak işleyip, obje'ye dönüştüreceğim, ve obje'min içindeki başlık ve açıklama bilgisini ekrana yazdıracağım! Tamamen düz mantık.

```js
// Sync olarak request yapmamızı sağlayacak, kütüphane.
const request = require('sync-request');
// bilgilerini almak istediğimiz filmin id'si.
const movieID = '2baf70d1-42bb-4437-b551-e5fed5a87abe';

// API'ye yaptığımız istek
const result = request('GET', `https://ghibliapi.herokuapp.com/films/${movieID}`);
// API'den gelen string cevabı, javascript objesini çevirmek için JSON parse işlemi
const body = JSON.parse(result.body.toString());
// obje içerisindeki title ve description değerlerinin, aynı isimle değişkene atanması.
const { title, description } = body;

// Artık sonucu ekrana yazdırabiliriz, umarız şimdiye kadar hata olmamıştır...
console.log(`Film Adı: ${title}`);
console.log(`Filmin açıklaması: ${description}`);
```

bu script'i çalıştırdığım zaman, çıktı olarak
```
Film Adı: Castle in the Sky
Filmin açıklaması: The orphan Sheeta inherited a mysterious crystal that links her to the mythical sky-kingdom of Laputa. With the help of resourceful Pazu and a rollicking band of sky pirates, she makes her way to the ruins of the once-great civilization. Sheeta and Pazu must outwit the evil Muska, who plans to use Laputa's science to make himself ruler of the world.
```
görüyorum. Tamda istediğimiz şey! Tek bir film id için, filmin başlığını ve açıklamasını aldık. Artık bir sonraki adıma geçip, tüm id'ler için bu işlemi yapabiliriz!

## Birden fazla film, sync olarak işlem yapmak.
Düz mantık devam ettiğim zaman, kodumda pek bir şey değişmemesi lazım. Tek yapmam gereken; tek bir film yerine, dosyamda bulunan her bir film için aynı kodu çalıştırmak. Kodumu düzenleyip, baktığımda, durum gerçektende böyle. Tek yapmamız gereken koda 2 şey eklemek oldu.

Sabit bir film id'si yerine, tüm dosyayı okuyup, her satırı diziye atamak:
```js
// Tüm film id'lerini, dosyamızı okuyarak ve '\n' karakterinden split yaparak, bir diziye atıyalım.
const movieIDs = fs.readFileSync('../data/ghibli_movies.txt').toString().split('\n');
```

Ve istek gönderdiğimiz satırdan itibaren, tüm kodumu, for/let of döngüsü içine almak:
```js
for(let movieID of movieIDs){
    const result = ...
}
```

Elimizdeki kodun son hali:
```js
// Dosya okumak için node.js kütüphanemizi alalım.
const fs = require('fs');
// Sync olarak request yapmamızı sağlayacak, kütüphane.
const request = require('sync-request');
// Tüm film id'lerini, dosyamızı okuyarak ve '\n' karakterinden split yaparak, bir diziye atıyalım.
const movieIDs = fs.readFileSync('../data/ghibli_movies.txt').toString().split('\n');

for(let movieID of movieIDs){
  // API'ye yaptığımız istek
  const result = request('GET', `https://ghibliapi.herokuapp.com/films/${movieID}`);
  // API'den gelen string cevabı, javascript objesini çevirmek için JSON parse işlemi
  const body = JSON.parse(result.body.toString());
  // obje içerisindeki title ve description değerlerinin, aynı isimle değişkene atanması.
  const { title, description } = body;

  // Artık sonucu ekrana yazdırabiliriz, umarız şimdiye kadar hata olmamıştır...
  console.log(`Film Adı: ${title}`);
  console.log(`Filmin açıklaması: ${description}`);
}
```

Sonuç olarak, artık kodumuz tüm film id'leri için başlık ve açıklamaları aldı ve bize gösterdi. Sorunumuz artık çözüldü. Bu problem için daha fazla kod yazmak gereksiz bile. Ama bir şeyi çok yanlış yapıyoruz. Bunu anlamak için, bir Node.js işleminin çalışma mantığına ve Event Loop'a değinelim.

## Node.js işlemi ve Async
Yazdığımız Javascript kodunun, bir şekilde bilgisayarın anlayabileceği makine koduna dönüşmesi ve çalışması lazım. Tarayıcı'da bunu bizim için tarayıcının kendisi yaparken, Node.js kullanırken, `node <script_dosyamiz>.js` komutunu kullanarak bu işlemi başlatıyoruz. Bu komut çalıştırıldığında, işletim sistemimiz'de, bizim Javascript kodumuzu çalıştıran bir işlem oluşuyor. Ve Node.js'in yardımları sayesinde, işletim sistemimizi kullanarak; dosya okuyabiliyoruz ve ağ üzerinden istekler gönderebiliyoruz. 

Buraya kadar olan kısım, şuana kadar yazdığımız kodu açıklıyor. Fakat burada daha optimize bir kod yazmamız için düşünmemiz gereken şey şu; bizim dosya okuma işlemimiz Harddisk veya SSD tarafından, ağ üzerinden yaptığımız istekler ise, ağ donanımımız üzerinden yapılıyor. Aslında sadece; dosyadan okuduğumuz değerleri diziye dönüştürmek, her bir dizi elemanı için yapılan istekleri `JSON.parse` ile obje'ye dönüştürmek ve içerisindeki değerleri bu objeden çıkarıp, değişkene atamak için CPU'muzu kullanıyoruz.

Ama kodumuzdaki düz mantığa geri döndüğümüzde, her bir isteğin gönderilmeden önce, bir önceki isteğin bitmesini ve ekrana yazdırılmasını beklediğimizi görüyoruz. Bu durumdan kurtulmak, şu anki problemimize baktığımızda; bütün programlama yöntemlerimizi değiştirmemize değmeyecek gibi görünsede, ideal bir dünyada yapmak istediğimiz şey; tüm bilgisayar donanımlarımızı optimal şekilde kullanmaktır.

Yani, bir kullanıcınıza ağ üzerinden veri gönderirken, diğerinin sadece CPU kullanacak işlemini bekletmek istemezsiniz. Burada bizi kurtaracak olan terim Async. Async işlem yapmak demek, birden fazla iş parçacağının birbiri ile kesişen zamanlarda çalışabilmesi demektir. Node.JS için bu, "aynı anda" çalışmak anlamına gelmez.

Şöyle örnek vermek gerekirse; biz ağ donanımımız işlem yaparken, CPU ile işlem yapabiliriz ama, CPU'muz birden fazla çekirdeğe sahip olsa bile, aynı anda 2 tane CPU gerektiren işlemi çalıştıramayız. Çünkü Node.JS bunu aynı işlem içinde desteklememektedir. Bu tek bir Node.js işleminin maksimum performans sınırını düşürmektedir. Fakat beraberinde birden fazla çekirdek kullanmanın getirebileceği sorunları çözmekte, ve bir çok async işleme kavramını kolaylaştırmaktadır. Sektörde Node.JS'in bu sorunu çözmek için ortaya attığı tek çekirdekli Async yapı dışında, birden fazla çekirdek ile ve async çalışmayı kolaylaştıran yazılım paradigmaları bulunmaktadır(örn. green threading, channels, functional programming).

Çoğu basit seviyede uygulama, CPU'dan çok, diğer donanımları beklerken zaman harcamaktadır. Kaldı ki, bir Node.JS uygulamasında, sıkıntınız CPU olduğu zaman, birden fazla Node.JS işlemi çalıştırarak, birden fazla CPU kullanabilirsiniz. Burada dikkate alınması gereken durumlar tabiki olacaktır ama, bu bölümü kısa tutmak için, load balancing, state yönetimi gibi konulara girmek istemiyorum.

Özet olarak, elimizdeki uygulamanın performansını iyileştirmek için, bir ağ isteği bitmeden, bir başkasını başlatabilir, ve ilk hangisi sonuçlanırsa onu işleyebiliriz. Ve bu işlem, Node.JS'in tek çekirdek kullanan yapısında, oldukca basit olacaktır.

## Event Loop
Bir farklı detayına girmek istemediğim, ve basitleştirme yaparak anlatmak istediğim konu ise, Event Loop. Node.JS çalışmak isteyen tüm fonksiyonları, kendi içinde bir sıraya koyar. Uygulamanızı ilk başlattığınız kod sayfanız, ilk çalışan fonksiyonunuz olarak düşünülür. Daha sonra, bu ana fonksiyonunuz içinde yaptığınız her "bloklayıcı(sync)" fonksiyon çağrısı, bitmesi beklenecek şekilde çalıştırılır. Eğer async olarak çalışacak olan bir fonksiyon çağrılırsa, bu async işlem bittiğinde çalışması için başka bir fonksiyon daha verebilirsiniz. Node.JS arkaplanda bu fonksiyonu sıraya koyarak, çalıştırmakta olduğu fonksiyona devam edecek, daha sonra çalışmakta olan fonksiyon ve async işlem bittikten sonra çalışması için verdiğiniz fonksiyonu çağırıcaktır. 

Bunu daha iyi anlayabilmek için basit bir örnek yapalım, daha sonra async programlama kısmına geçiş yapalım.

```js
setTimeout(function yuzMilisaniyeSonra(){ console.log('1'); }, 100);
setImmediate(function birSonrakiFonksiyon(){ console.log('2'); });
setImmediate(function birSonrakiFonksiyon(){ console.log('4'); });
console.log('start!');
```

Bu kodu çalıştırdığımız zaman ekranda göreceğimiz çıktı:

```
start!
2
4
1
```
olacaktır. Tüm scriptimizi tek bir fonksiyon olarak kabul edelim. İlk önce 1. satır, `setTimeout` fonksiyonu ile 100 milisaniye sonra çalışması için bir fonksiyonu sıraya koyuyoruz, `setImmediate` ile event loop'a direk olarak 2 fonksiyonu arka arkaya ekliyoruz. Bu fonksiyonlar sadece Event Loop'a eklendi, ve daha çalışmadı. `setTimeout` ve `setImmediate` fonksiyonları sonlandıktan sonra, ana fonksiyonumuza hep geri dönüyoruz. Ve en son satırımızdaki `console.log` fonksiyonu çağrılıp, ekrana `start!` yazdırıldıktan sonra, sırasıyla Event Loop'daki fonksiyonlar çalışmaya başlıyor.

`setTimeout` ile sıraya koyduğumuz fonksiyonumuz, 100 ms saniye geçmeden çalışamıyacağı için, sırasıyla event loop'a giren 2 yazdırma, 4 yazdırma fonksiyonları yazdırılıyor. Ve 100 ms geçtikten sonra ekrana 1 yazdırılıyor ve programımız sonlanıyor.

İlk kodumuza döndüğümüz zaman, `fs.readFileSync('../data/ghibli_movieis.txt')` satırımızı görebiliriz. Burada bu fonksiyon, dosya okuma işlemi bitene kadar, event loop'daki diğer fonksiyonların çalışmasını engellemiş oluyor. Hoş, bizim örneğimizde, event loopda başka bir fonksiyon olmadığı, ve programın çalışmaya devam etmesi için bu dosyayı okumamız gerektiği için bir önemi yok ama; `fs.readFile('../data/ghibli_movies.txt', function(err, data){ console.log(err, data); })` yaparak; dosya okuma işlemini başlatıp, işlem bittikten sonra çağrılmak üzere bir fonksiyonu event loop'a ekleyebiliriz. Böylece dosya okuma işlemi, işletim sistemi tarafından, donanıma iletilir ve yapılırken, CPU ve Event Loop üzerinde başka fonksiyonlar çağırabiliriz. Alternatif olarak kısaca `fs.readFile('../data/ghibli_movies.txt', console.log);`'da yapabiliriz. `console.log`'da bir fonksiyon olduğu için, yazdığımız isimsiz fonksiyon gibi, dosya okuma işlemi bittikten sonra, sonuç parametreleri ile birlikte çağrılacaktır.

## Promiseler ve Async/Await
Şimdi geldik, esas ilginç bölüme. Async değerler ile çalışmak için, Event Loop'a fonksiyon koymamız gerektiğini söyledik. Ama bunu eski, error first Node.js yöntemi ile yapmak zorunda değiliz. Çünkü bu yöntem, bizi; kod seviyesinde, Node.JS'in uydurduğu bir yapıya uymaya ve tüm async fonksiyonlara, standart tipte (hata, sonuç) tipinde parametreler kabul eden, fonksiyonlar vermeye zorluyor. Halbüki, sync çalışıyormuş gibi fonksiyonlar yazıp, sadece gerektiği yerde, async kavramları işin içine karıştırmamızı sağlayan bir yapı bulunmaktadır. Buna Promise denir.

Örnek vermek gerekirse:
```js
// Bu fonksiyonun tek bildiği, Buffer tipiinde bir parametre aldığı ve her satırdan bir dizi oluşturması
function idListesineCevir(dosyaIcerigiBuffer){
	return dosyaIcerigiBuffer.toString().split('\n');
}

// bu fonksiyonu sync olarak kullanabiliriz.
idListesineCevir(fs.readFileSync('../data/ghibli_movies.txt'))

// bu fonksiyonu async olarak callback ile kullanabiliriz
fs.readFile('../data/ghibli_movies.txt', function(err, buffer){
	if(err) ...
	const sonuc = idListesineCevir(buffer);
});

// bu fonksiyonu async olarak, Promise ile kullanabiliriz
// Buffer döndüren bir Promise'i, Id listesi döndüren bir Promise'e dönüştürür.
readFilePromise('../data/ghibli_movies.txt').then(idListesineCevir)
```
Burada dikkat çekmek istediğim şey, sync olarak bu fonksiyonumuzu kullandığımız zaman direk olarak, fonksiyon parametresi olarak Buffer veriyoruz. Promise kullanımı buna yakın ve temiz görünürken, eski tarz Node.js error first callback kullandığımız zaman, tamamen bu yapıya özel ekstra kod yazmamız gerekiyor. Error first callback kullanırken, hata olma durumunu kontrol etmek bizim sorumluluğumuzdayken, Promise'lerde başarılı sonuçlanma, ve hata durumları için ayrı fonksiyonlar tanımlayabiliyoruz. Aynı zamanda Promise'leri birbiri ardına bağlayarak kullanabilirken, error first callbackler ile bunu yapmak biraz daha uğraş verici oluyor. 

Örnek vermek gerekirse:
```js
// Promise veya callback olmadan, sync kullanım. Eğer bir hata olursa, exception throwlanır.
const ids = idListesineCevir(fs.readFileSync('../data/ghibli_movies.txt');
console.log(islemYap(ids[0]));

// zincirleme şekilde dosya okuma sonucunu id listesine çevir, ilk film id'si için işlem başlat, sonucu ekrana yazdır
readFilePromise('../data/ghibli_movies.txt')
	.then(idListesineCevir)
	.then((ids) => islemYap(ids[0]))
	.then(console.log)
	// Herhangi bir adımda hata oluşurse, bu fonksiyon çağırılacak.
	.catch(console.log);

// aynı işlemi callback ile yapmak istediğimizde sonuç:
fs.readFile('../data/ghibli_movies.txt', (err, buffer) => {
	if(err) console.log(err);
	const id = idListesineCevir(buffer)[0];
	islemYap(id, function(err, result){
		if(err) return console.log(err);
		console.log(result);
	});
})

```

Yukarıdaki callback, daha düzgün bir şekilde, yeni fonksiyonlar tanımlanarak yazılabilir. Veya yardımcı bir kütüphane kullanılarak, daha düz hale getirilebilir, fakat... Promiseler için en önemli ve kritik özellik, Node'un son versiyonlarında, kullanabilmeye başladığımız, async/await anahtar kelimeleri. Promise'ler, temelde düşünüldüğü zaman, sync fonksiyonlar ile async fonksiyonlar arasında bağlantı sağlayan, sync olarak parametre verilerek başlayan, gelecekte ya başarılı bir değer, ya da hata döndüren birer fonksiyondur.

Promiselerin dönüş yapısı sync fonksiyonların exception throwlamasına benzetilebilir. Ve gelecekte değer döndürme belirtmek ve beklemek için özel anahtar kelimeler kullanılabilir. Async/Await tamamen bunu yapmak için yaratılmıştır. Basit anlamda düşünürsek, sizin sync'e benzer şekilde yazdığınız kodu, Promise'lerdeki `.then` ve `.catch` yapısına çevirir. Promise'ler ile tamamen uyumlu şekilde çalışırlar.

Daha önceki örnekte bahsettiğim gibi, eğer bir Promise'e fonksiyon vererek, içinde sync işlem yapsanız bile, sonuçta elinizde gene bir Promise olacak. Async/Await'de bu mantıkla çalışmaktadır. Await kelimesi sadece async fonksiyonlar(yani Promise'ler) içinde kullanılabilir ve async fonksiyonlar Promise'den başka bir şey döndüremez.

bir async fonksiyonun, promise olarak nasıl göründüğünü daha iyi anlıyalım.
```js
async function z(){
  return 4;
}
async function x(){
  const r = await z();
  const r2 = await z();
  return r + r2 + 5;
}

// async/await olmasaydı... iç içe .then yaptığımız bir promise gibi düşünebilirdik.
function x2(){
  return Promise.resolve().then(() => z().then(r => z().then(r2 => r + r2 + 5)))
}

x().then(console.log)
x2().then(console.log)
```
hem x, hemde x2 fonksiyonu, sonuç olarak 13 döndürecektir.

Async/Await hakkında bilgi sahibi olduktan sonra, yukarıdaki Promise ve Error First Callbacklere alternatif olarak, kodumuzu tekrar yazarsak;
```js
async function main(){
	const buffer = await readFilePromise('../data/ghibli_movies.txt');
	const id = idListesineCevir(buffer)[0];
	console.log(await islemYap(id)); 
}

// Main bir async fonksiyon, yani Promise olduğu için, .then ve .catch kullanabiliriz.
main().then(console.log).catch(console.log)
```
Bu kodun, ilk yazdığımız Sync örneğe ne kadar benzediğini kolayca görebilirsiniz. Async faydalarının hepsini Promise yapısını kullanarak, yine elde ediyoruz. Fakat, async/await kullanarak sync kod'a oldukca benzeyen bir yapıya sahip oluyoruz! Oldukca güzel bir özellik!

## Async olarak, tek bir film ile işlem yapmak.
Şimdi problemimize geri dönelim. Yaptığımız işlemlerin sync olduğunu, ve async kullanmamız gerektiğini söylemiştik. Şimdi async/await kullanarak kodumuz tekrar yazalım.

```js
// promise döndürecek şekilde async istek yapmamızı sağlayacak olan kütüphane.
const request = require('request-promise-native');

// sync benzeri kod yazabilmek için, async (promise döndüren) bir fonksiyon yazalım.
async function main(){
  const movieID = "2baf70d1-42bb-4437-b551-e5fed5a87abe";

  // API'ye yaptığımız istek
  const resultStr = await request(`https://ghibliapi.herokuapp.com/films/${movieID}`);
  // API'den gelen string cevabı, javascript objesini çevirmek için JSON parse işlemi
  const body = JSON.parse(resultStr);
  // obje içerisindeki title ve description değerlerinin, aynı isimle değişkene atanması.
  const { title, description } = body;

  // Artık sonucu ekrana yazdırabiliriz.
  console.log(`Film Adı: ${title}`);
  console.log(`Filmin açıklaması: ${description}`);
}

// main fonksiyonunu çalıştır, ve dönen promise değerinde sonuç ve hata olması durumunda, konsola çıktı ver.
main().then(console.log, console.log);
// main fonksiyonunun bloklamadığını kanıtlamak için, ekrana yazı yazalım.
console.log('Main fonksiyonu çağrıldı!');
```

Bu scripti çalıştırdığımız zaman, çıktı olarak gözlemleyeceğimiz şey;
```
Main fonksiyonu çağrıldı!
Film Adı: Castle in the Sky
Filmin açıklaması: The orphan Sheeta inherited a mysterious crystal that links her to the mythical sky-kingdom of Laputa. With the help of resourceful Pazu and a rollicking band of sky pirates, she makes her way to the ruins of the once-great civilization. Sheeta and Pazu must outwit the evil Muska, who plans to use Laputa's science to make himself ruler of the world.
undefined
```
Burada görüldüğü üzere, `Main fonksiyonu çağrıldı!` yazısı, main fonksiyonu çağrıldıktan sonra yazılmış olsa bile, main fonksiyonu bloklamadığı için, ilk önce bu yazıyı görüyoruz. Daha sonra ağ isteğimiz sonuçlanıyor, film bilgileri yazdırılıyor ve main fonksiyonumuzun döndürdüğü Promise sonlanıyor. Main fonksiyonundan herhangi bir değer döndürmediğimiz için, sonuç undefined oluyor ve `.then` fonksiyonunda bu undefined değeri ekrana yazdırılıyor.

İlk sync versiyonumuza çok benzer bir şekilde, ağ işlemi; async olan bir kod yazmış olduk.

## Async olarak, birden fazla film ile işlem yapmak
İşlerin biraz kafa karıştırıcı hal aldığı durum burası. Aync/await kodun sync'e benzerliği sayesinde, kolayca birden fazla filmi async olarak işleyebiliriz.

```js
// Dosya okumak için node.js kütüphanemizi alalım.
const fs = require('fs');
// node.js error-first callbackleri Promise'e çevirmemize yardımcı olan fonksiyon
const { promisify } = require('util');
// readFile error first async fonksiyonunu, promise döndürecek şekilde çeviriyoruz.
const readFilePromise = promisify(fs.readFile);
// promise döndürecek şekilde async istek yapmamızı sağlayacak olan kütüphane.
const request = require('request-promise-native');

// sync benzeri kod yazabilmek için, async (promise döndüren) bir fonksiyon yazalım.
async function main(){
  // Dosya okuyan async kodu başlatalım, sonuctaki her bir satırı, dizi elemanı olarak alalım.
  const movieIDs = (await readFilePromise('../data/ghibli_movies.txt')).toString().split('\n');

  // Her bir dizi elemanı için async olarak istek gönderelim.
  for(let movieID of movieIDs){
    // API'ye yaptığımız istek
    const resultStr = await request(`https://ghibliapi.herokuapp.com/films/${movieID}`);
    // API'den gelen string cevabı, javascript objesini çevirmek için JSON parse işlemi
    const body = JSON.parse(resultStr);
    // obje içerisindeki title ve description değerlerinin, aynı isimle değişkene atanması.
    const { title, description } = body;

    // Artık sonucu ekrana yazdırabiliriz.
    console.log(`Film Adı: ${title}`);
    console.log(`Filmin açıklaması: ${description}`);
  }
}

// main fonksiyonunu çalıştır, ve dönen promise değerinde sonuç ve hata olması durumunda, konsola çıktı ver.
main().then(console.log, console.log);
// main fonksiyonunun bloklamadığını kanıtlamak için, ekrana yazı yazalım.
console.log('Main fonksiyonu çağrıldı!');
```

Bu kodu çalıştırdığımızda, ağ işleminin async olmasına rağmen, beklediğimizden yavaş çalıştığını göreceğiz. Hani async sayesinde birden fazla ağ isteğini aynı anda yapabilecektik? İlk geleni işleyecektik? Buradaki sıkıntı async/await'in, promise'e dönüştürülürken, takip etmesi gereken adımlar. Şuanda bu yapı ile özet olarak şu şekilde bir yapıya sahip oluyoruz:

```js
let promise = Promise.resolve();
for(let movieID of movieIDs){
    promise = promise.then(() => request(...movieID)).then((result) => ...));
}
```

yani tek bir Promise zinciri oluşturuluyor. Bu yüzden network isteklerimizden biri bitmeden, bir sonraki başlamıyor. Burada uygulamanın yavaş çalışmasına rağmen, not düşmek istediğim bir konu var. Tüm ağ isteklerini, sırayla başlatmak yerine, hepsini aynı anda başlatabiliriz. Ama bu bizim Node.js uygulamamızı blokladığımız zamanı değiştirmeyecek. Sadece bu işlemin daha hızlı sonlanmasını/toplamda daha az zaman almasını sağlayacak. Burada ne demek istiyorum?

Diyelim ki, her gönderdiğimiz istek 300 milisaniye alıyor, ve gelen cevabın işlenip ekrana yazdırılması ise 10 ms sürüyor. 300 milisaniyelik ağ isteği async olduğu için, nodejs işlemimiz bu ağ isteği gönderildikten sonra, çalışmasına devam edebilir, başka bir yerde, CPU harcayan bir kodunuz varsa(şuan bizim scriptimizde yok) onu çalıştırabilir. Ama ağ isteğinden cevap geldikten ve sıra işleme/yazdırma kodumuza geldiğinde 10 ms boyunca CPU'yu bu fonksiyonumuz bloklayacak. Daha sonra 300 milisaniye süren, yeni bir ağ isteği başlatılacak, ve yine CPU kullanmak isteyen farklı bir kod parçacağı varsa o çağrılabilecek. Yani biz aslında bu kod ile, her film id'si için sadece 10 milisaniye bloklamış oluyoruz. Ve toplam CPU bloklama süremiz, `10 * film id sayısı` milisaniye oluyor. Fakat algoritmamız: `ağ isteği gönder, cevabın gelmesini bekle, işle ve yazdır, ağ isteği gönder...` şeklinde olduğu için, toplam işlem süremiz:  `(300 + 10) * film id sayısı` milisaniye oluyor.

Bu durumu, algoritmamımızı: `bir filmid için; ağ isteği başlatıp, sonucu işleyen ve yazdıran bir promise zinciri oluştur, tüm film id için bu promise zincirlerini aynı anda başlat` şeklinde değiştirirsek, bloklama süremiz yine değişmeyecek. Her işlemi işlemesi ve yazdırması 10 milisaniye süreceği için, gene `10 * film id sayısı` milisaniye CPU bloklamış olacağız. Fakat ağ istekleri aynı anda başlatıldığı için, hepsi farklı sürede bitecek, ve en uzun süren isteğin 500 ms olduğunu varsayarsak, `500 + (10 * film id sayısı)` milisaniye sonra, programımız sonlanmış olacak. Bu da script'in çalışma süresinde çok büyük bir düşüşe sebep olmuş olacak. Hatta kendi bilgisayarımdan ve internet bağlantımdan örnek vermem gerekirse; şuanki kod 25 saniyede çalışırken, optimize edilmiş şekilde yazdığım kod, 1.5 saniye sürüyor. 20 tane film id'si olduğunu varsayarsak, demek ki tek bir isteğin sonuçlanması ortalama 1.25 saniye sürüyor, ve tüm ağ isteklerini aynı anda başlattığımda, en uzun süren işlem 1.5 saniye tutuyor. Burada unutmamamız gereken, bloklama süresinin iki örnek içinde aynı olduğu. Yani eğer ağ istekleri 10 milisaniye tutsaydı, ve CPU işlemleri 1.5 saniye tutsaydı, iki örnekte benzer sürede bitecekti.
 
Bu işlemin, daha hızlı biten versiyonunu yazabilmek için async/await fonksiyonları ile Promise bilgilerimizi birbiri ile birleştirip, ortaya bir karışım çıkarmamız gerkiyor. Bundan önce, uygulamamızı fonksiyonlara ayrıştırıp, kodumuzu daha düzenli hale getirelim. Ve son olarak, promise ile daha performanslı olarak yazmaya çalışalım.

### Fonksiyonlara ayrıştırmak
Sync veya Async nasıl kodlarsak kodlayalım, uygulamamızın bazı kısımları tamamen saf, sync olarak çalışacak mantıklar içermektedir. Örnek vermek gerekirse, dosya okuma işlememiz sync/async olabilir ama, sonuç olarak elimizde olan değer string olduğu için, dosyayı dizi listesine çevirme işlemimiz hep aynı sync kod ile yapılıyor.

Aynı şekilde network isteğimiz sync/async olabilir ama gelen string cevabı hep JSON'a parselayıp, içindeki title ve description bilgisini aynı şekilde alıyoruz. Ve ekrana yazma fonksiyonumuzda aynı şekilde hep aynı kalıyor.

Bu fonksiyonları ayırıp, bir önceki async/await kodumuzu tekrar yazdığımızda böyle bir şey elde ediyoruz:
```js
// Dosya okumak için node.js kütüphanemizi alalım.
const fs = require('fs');
// node.js error-first callbackleri Promise'e çevirmemize yardımcı olan fonksiyon
const { promisify } = require('util');
// readFile error first async fonksiyonunu, promise döndürecek şekilde çeviriyoruz.
const readFilePromise = promisify(fs.readFile);
// promise döndürecek şekilde async istek yapmamızı sağlayacak olan kütüphane.
const request = require('request-promise-native');

function convertFileBufferToIDArray(buffer){
  return buffer.toString().split('\n');
}

// verilen string'i json parse yaparak, film modeline çevirme işlemi.
function parseResponseToTitleAndDescription(resultStr){
  const body = JSON.parse(resultStr);
  return { title, description } = body;
}

// verilen başlık ve açıklamayı, konsola yazdırma
function outputMovieToConsole({ title, description }){
  console.log(`Film Adı: ${title}`);
  console.log(`Filmin açıklaması: ${description}`);
}

// string olarak gelen cevabı film modeline işle ve ekrana bastır.
function parseAndOutput(resultStr){
  const movie = parseResponseToTitleAndDescription(resultStr);
  return outputMovieToConsole(movie);
}

// sync benzeri kod yazabilmek için, async (promise döndüren) bir fonksiyon yazalım.
async function main(){
  // Dosya okuyan async kodu başlatalım, sonuctaki her bir satırı, dizi elemanı olarak alalım.
  const buffer = await readFilePromise('../data/ghibli_movies.txt');
  const movieIDs = convertFileBufferToIDArray(buffer);

  // Her bir dizi elemanı için async olarak istek gönderelim.
  for(let movieID of movieIDs){
    // API'ye yaptığımız istek
    const resultStr = await request(`https://ghibliapi.herokuapp.com/films/${movieID}`);
    parseAndOutput(resultStr);
  }
}

// main fonksiyonunu çalıştır, ve dönen promise değerinde sonuç ve hata olması durumunda, konsola çıktı ver.
main().then(console.log, console.log);
// main fonksiyonunun bloklamadığını kanıtlamak için, ekrana yazı yazalım.
console.log('Main fonksiyonu çağrıldı!');
```
Uygulamamızın sabit mantığını fonksiyonlara taşıyarak, async içerisindeki kodu oldukca azalttık. Artık sync veya farklı bir şekilde kod yazsak bile, async kısma bulaşmayan kodumuz aynı çalışacak.

### Promise kullanarak çoklu işlemde performansı arttırmak
Bir Promise zinciri oluşturmak yerine, Promise oluşturma işlemini async await'e bırakmak yerine, kendimiz belirleyerek, daha performanslı bir kod elde edebiliriz.

```js
// sync benzeri kod yazabilmek için, async (promise döndüren) bir fonksiyon yazalım.
async function main(){
  // Dosya okuyan async kodu başlatalım, sonuctaki her bir satırı, dizi elemanı olarak alalım.
  const buffer = await readFilePromise('../data/ghibli_movies.txt');
  const movieIDs = convertFileBufferToIDArray(buffer);

  // Promise all kullanarak, aynı anda birden fazla async işlem başlatıp, hepsinin bitmesini bekleyebiliriz.
  // await ile bu işlemler bitmeden main fonksiyonunu sonlandırmak istemediğimiz söylüyoruz.
  await Promise.all(
      // her bir film id için,
      movieIDs.map((movieID) => 
        // async bir işlem başlat
        request(`https://ghibliapi.herokuapp.com/films/${movieID}`)
          // bu işlem bittikten sonra, sonucu işle ve ekrana yazdır.
          .then(parseAndOutput)
      )
  )

  console.log('Main fonksiyonunun çalışması bitti.');
}
```

Main fonksiyonumuzu bu şekilde değiştirdikten sonra, film çekilme işleminin hızlandığını ve çıktıların sıralarının her çalıştırmada değiştiğini görüyoruz. Artık tüm network istekleri eş zamanlı olarak çalıştırılıyor ve ilk biten istek için işleme ve ekrana yazdırma işlemi yapılıyor.

Özet olarak; async/await'in okunulabilirlik açısından sağladığı faydaların, Promise kullanımı bilmeden, performansı kötü etkileyici bir faktör olabildiğini, ve Promise primitivelerini kullanarak uygulamamızı hızlandırabileceğimizi gördük.

Fakat şöyle bir sıkıntı var. Bu tarz bir yapıda, eş zamanlı olarak kaç işlem başlatabileceğimizin, işlemlerin iptal edilebilmesinin kontrolü bizde değil. Ne kadar Promise'ler ile çalışırken yardımcı kütüphaneler ile bu konularda biraz çözüm bulabilsekte, Promise'ler tek bir async veri ile çalışmak üzere geliştirildiği için bu konularda eksiklik çekiyoruz. Burada devreye; kurtarıcımız, RxJS giriyor.

### RxJs ile async, birden fazla film işlemenin yeniden yazılması
Promise'ler Async ve Tek veriler ile kullanıma uygun olduğu gibi, RxJS ise, async veri akışı ile çalışmak istediğimiz zamanda kullanılıyor. Promise'ler, error first callbackler için nasıl güzel bir alternatif ise, RxJs'de Node.JS streamleri için o kadar güzel bir alternatif. Promise'ler gelecekte başarılı veya başarısız olabilecek tekil verileri ifade ederkeen, RxJs oberservable pattern kullanarak, zaman bağlı olarak bir veri akışını ifade ediyor. RxJs'de Stream üzerindeki verileri tek tek işlemek için bir fonksiyon, hata olması durumunda başka bir fonksiyon ve Stream'in sonuçlanmasında farklı bir fonksiyon tanımlayabiliyorsunuz.

RxJs Streamleri, Promise'ler gibi zincirlenme özelliklerine sahip, ve aynı zamanda kendi methodları içerisinde Promise'leride destekliyor. RxJs'in bir güzel özelliği ise, zamana bağlı işlemler için [operator](http://reactivex.io/documentation/operators.html) adını verdiği methodlar içermesi. Bu methodlar ile el ile implemente etmesi zor olan bir çok veri akışı işleme özelliğini bedavaya alıyorsunuz.

Az önceki kodumuza, eş zamanlı olarak maksimum 5 istek çağırmayı ve eğer toplam işlem süresi 2 saniyeyi geçerse tüm istekleri durdurmayı ekliyelim:

İlk önce rxjs'i projemize ekliyoruz.
```js
// rxjs kütüphanesini ekliyelim
const Rx = require('rxjs/Rx');
// kullanacağımız operatörleri alalım
const { mergeMap, takeUntil } = require('rxjs/operators');
```

Daha sonrada projemizin sadece Main fonksiyon kısmını değiştiriyoruz:
```js
const timeoutPromise = new Promise((resolve, reject) => setTimeout(resolve, 2000));
// sync benzeri kod yazabilmek için, async (promise döndüren) bir fonksiyon yazalım.
async function main(){
  // Dosya okuyan async kodu başlatalım, sonuctaki her bir satırı, dizi elemanı olarak alalım.
  const buffer = await readFilePromise('../data/ghibli_movies.txt');
  const movieIDs$ = Rx.Observable.from(convertFileBufferToIDArray(buffer));

  // movieIDs streamini işlemek için bir hat oluştur, promise'e döndür ve bitmesini bekle.
  await movieIDs$.pipe(
      // her akıştan gelen veri için, verilen fonksiyondaki işlemi yap.
      mergeMap((movieID) =>
        // movieID için istek gönder, sonucu işle.
        request(`https://ghibliapi.herokuapp.com/films/${movieID}`).then(parseAndOutput),
        undefined,
        // aynı anda en fazla 5 promise bekliyor durumda olsun.
        5
      ),
      // timeoutPromise bitene kadar çalış!
      takeUntil(timeoutPromise)
  ).toPromise();

  console.log('Main fonksiyonunun çalışması bitti.');
}
```

bunun sonucunda elde ettiğimiz scriptin, 2 saniyeden fazla çalışması durumunda kendini sonlandırdığını, ve anlık olarak en fazla 5 istek gönderdiğini görebiliriz. Umarım RxJs'in operatörleri sayesinde kafanızdaki fikirleri, sonlu/sonsuz bir veri akışı üzerinde gerçekleştirmenin ne kadar kolay olduğunu anlayabilmişsinizdir.

Bu kullanım, JS dünyasında Thread olmadığı için, diğer dillere göre daha bir kolay ve eğlencelidir.

Son oluşturduğumuz script'de küçük bir hata'da bulunmaktadır! Yaptığımız istekleri Promise olarak yaptığımız için, stream timeout sebebi ile sonlanması, Promise'ler cancel edilebilir olmadığı için, istekleri durduramamaktadır. Bunuda [RxJs.DOM.Ajax](https://github.com/Reactive-Extensions/RxJS-DOM/blob/master/doc/operators/ajax.md) kullanarak çözebiliriz.

## Çıkarımlar
- Bilgisayar donanımımızı ve kaynaklarımızı en optimize şekilde kullanmak için async işlem yapmaya ihtiyacımız vardır.
- Node.JS'in tek çekirdekli yapısı, dili parallelikten yoksun bırakırken, async olarak çalışmayı kolaylaştırmaktadır.
- Node.JS scriptleri birden fazla işlem olarak çalıştırılarak paralellik elde edilebilir.
- Sync/Async durumlara bağlı olmayan işlemlerinizi ayrıştırarak, ortak kullanılabilen fonksiyonlar oluşturabilirsiniz.
- Promise gibi yapılara dil seviyesinde uyumluluk sağlanarak, daha okunulabilir kod yazılabilir. Örn. async/await.
- Async tekli veriler için Promise kullanabilirsiniz. Async/Await syntaxı ile Promiseler ile, Sync kod'a benzer bir yapıya sahip olabilirsiniz, ama performans için her zaman iyi olmayabilir.
- Async/Await kullanırken, performans veya okunulabilirlik için Promise fonksiyonları kullanabilirsiniz.
- Sonlu veya sonsuz, async olarak işlenebilecek veri akışları için, Node.JS Stream'lerine alternatif olarak RxJS kullanabilirsiniz.
- RxJS operatörleri, zamana bağlı veri akışları ile çalışmayı kolaylaştırmaktadır. Ve Promiseler ile uyumlu bir şekilde çalışabilir.

