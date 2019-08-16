+++
title = "Bir Satranç Oyunu, Node.JS ile İşlemler Arası İletişim ve İş Dağıtımı"
date = "2019-08-16T12:00:00+03:00"
Categories = ["Node.JS", "Architecture"]
Tags = ["node.js", "performance", "ipc", "webassembly", "ipc"]
+++

Bu makalenin konusu, Facebook gruplarından birinde karşılaştığım güzel bir sorudan geliyor. Soru hem temel konularda bilgi sahibi olmayı, hem de kullanılacak aracı iyi tanımayı gerektiriyor. Bu konuda fikirlerimi kısa bir cevap olarak yazarsam, haksızlık ederim diye düşündüm. Ve aynı zamanda bir şeyler öğrenmek ve kod maymunluğu yapmak için eğlenceli bir konu bulmuş oldum. Ortaya bu makale ve github üzerinde yayınladığım [örnek kodlar](https://github.com/Yengas/stockfish-cluster-example) ortaya çıktı. Soruya bakarak, esas konumuzu yavaş yavaş işleyelim.

## Bir Satranç Oyunu
<center><img src="/img/articles/selim-question.png" alt="drawing" style="width:70%; max-width: 600px; @media (min-width: 768px) { width: 50%; }"/></center>

Anlaşıldığı üzere, arkadaş bir satranç oyunu yapıyor ve kullanıcıyı bilgisayara karşı yarıştırmak istiyor. Stockfish motorunu(hamle önerisi yapan bir program), uygulamalarını yayınladığı ortama kurmuş, ve şimdi bunu Node.js socket sunucusu ile entegre ederken, kafasında soru işaretleri var. Bu soruya bakınca aklımda ilişkili sorular uçmaya başlıyor:

1. Bu motoru istemci tarafında çalıştırabiliyor muyuz? Sunucu'da uğraşmamak, bizi fazlaca mühendislikten kurtarabilir.
2. Stockfish motorunu kendi kontrolümüzde, Node.js işlemi içerisinde çalıştırabiliyor muyuz?
3. Oldu da Node.js içerisinde çalıştırdık. Bunun bize CPU/Memory maliyeti ne olacak?
4. Bu motor herhangi bir state tutuyor mu? Örn. bir kullanıcı oyuna başladı, başlangıç itibari ile tüm hamleleri sırası ile mi motora veriyoruz? Yoksa her istekte tüm tahta durumunu mu veriyoruz? 
5. Ne kadar kullanıcımız olacak?

Araştırmaya koyulmam lazım. Her sorunun cevabı dizayn ettiğimiz sistemi değiştirebilir. Ama ilk önce bir duraksayıp, bu sorunun neden makale yazmaya değer olduğunu konuşalım.

### Başka Birinin Kodu
Projelerimizde ihtiyacımız olan özellikleri karşılamak için başkalarının kodunu hep kullanıyoruz. Resim küçültüp, büyütmek için olabilir. PDF bir dosya çıktı almak için olabilir. QRCode oluşturmak için olabilir. Bu tarz ihtiyaçlarımızı NPM çoğu zaman karşılıyor. Ama bazı ihtiyaçlarımızı karşılayacak kodlar, kütüphane olarak bulunmayabilir. Böyle durumlarda artık kendi projemizin dışına adım atmamız gerekiyor.

Burada işin içine işlemler arası iletişim ([IPC](https://stackoverflow.com/questions/40005935/which-kind-of-inter-process-communication-ipc-mechanism-should-i-use-at-which)) dediğimiz kavram devreye giriyor. Eğer uygulamanın bir konsol arayüzü varsa, ilk akla gelende "komut çalıştırmak" oluyor. Evet bunu yapmak oldukca kolay, [child_process](https://nodejs.org/api/child_process.html) kullanılarak, istediğiniz programı başlatabilirsiniz. Başlayan programın stdin ve stdout "dosyalarını" kendi Node.js işleminize bağlayarak da, istediğiniz gibi iletişim kurabilirsiniz.

Bu yöntemle birlikte düşünülmesi gereken şeyler artıyor. Bizim kodumuzun çalıştığı her yerde, bu uygulamanın kurulu olması gerekiyor. Hem de aynı versiyon ve konfigürasyon ile. Kullanmak istediğimiz tüm özelliklerin komut satırı üzerinden sağlanıyor olması gerekiyor. Eğer kendi kodumun içine bu fonksiyonaliteyi gömebiliyorsam, bunlarla uğraşmak istemem.

Neyse ki Stockfish durumunda, NPM üzerinde bir kütüphane [bulunuyor](https://www.npmjs.com/package/stockfish). Ama bu seferde başka sorunlar ortaya çıkıyor. Bu kütüphanenin nasıl çalıştığını anlamam lazım. Satranç hamlesi önermek CPU ucuz bir işlem olmamalı. Ben bu kütüphaneden bir fonksiyon çağırdığımda, CPU kullanımından dolayı sunucum kitlenecek mi? Sonuçta Node.js tek thread ve tek CPU demek. Bu kütüphanedeki bir fonksiyon 1 saniye bizi kitlerse, o 1 saniye hiç bir kullanıcı hizmet alamıyacak demek.

Araştırma yapmam lazım.

## Stockfish ve Satranç Hamle Önerisi
Üç taş oyunu gibi oyunları bilgisayara oynatmaya çalıştığımızda, bilgisayarın tüm ihtimalleri değerlendirmesi çok kısa sürüyor. Kazanan strateji de önceden belirli oluyor. Ama satranç gibi çok fazla ihtimal içeren kompleks oyunlarda, tüm ihtimalleri değerlendirmek imkansız. Bu yüzden Stockfish gibi satranç motorları, tahtanın şu anki durumu üzerinden, potansiyel hamlelerin ne kadar mantıklı olduğunu(hangi hamlelerin kaybetme ihtimalini azaltacağını) hesaplamak için geleceğe dönük bir arama yapıyorlar. Problemin doğası gereği, bu ucuz bir işlem değil. CPU'nun ısınmasına sebep olabilir. Stockfish'i kendi Node process'imiz içerisinde çalıştırmak istiyoruz. O yüzden bunu aklımıza tutalım. 

Ama ilk önce bu motor ile nasıl konuşacağımıza, hamle önermek için bizden ne isteyeceğine bakalım. İnternette kısa bir araştırma sonucunda, [aradığımı](https://chess.stackexchange.com/a/12581) buldum. [Universal Chess Interface](http://wbec-ridderkerk.nl/html/UCIProtocol.html) (UCI) adı verilen bir protokol sayesinde, Stockfish'e tahtamızın şu anki durumunu veriyoruz, ve istediğimiz sayıda hamle önerisi alabiliyoruz. En basit örnek şuna indirgenebilir:

```
position fen r3kb1r/p2npppp/2p2n2/3N4/8/5N2/PPPP1PPP/R1B2RK1 b kq - 0 10
go
```

Kriptik görünen `r3kb1r/p2npppp/2p2n2/3N4/8/5N2/PPPP1PPP/R1B2RK1 b kq - 0 10` kısmının [açıklaması](http://kirill-kryukov.com/chess/doc/fen.html) aslında oldukca [basit](http://www.chessgames.com/fenhelp.html). *FEN* adı verilen bu notasyon, tahtanın konumunu, sıranın hangi oyuncuda olduğu, kaçıncı hamle olduğunu gibi bilgileri barındırıyor. Önemli sorularımızdan biri cevaplandı. Her hamle önerisinde, tahtanın tüm pozisyonunu Stockfish'e verebiliyoruz ve [önerilen](https://chess.stackexchange.com/a/12670) bu. Bu demek oluyor ki, 1 tane motor çalıştırıp, eş zamanlı birden fazla kullanıcıya hizmet verebiliriz. 

Aynı zamanda araştırmalarımda Stockfish'in multi-thread çalışabildiğini, düşünme zamanını kısıtlayabildiğimizi, arama derinliğini limitleyebildiğimizi, önerilen hamle sayısını ayarlayabildiğimi, yetenek seviyesi diye bir ayarı olduğunu gibi önemli bilgileri de ediniyorum. Şimdi kod kısmına geçelim.

## Node.js Stockfish Kütüphanesi
Öğrendiklerimizi kullanarak, Stockfish Node.js [kütüphanesi](https://www.npmjs.com/package/stockfish) ile hemen bir örnek yapalım.

```js
const stockfish = require('stockfish')();

stockfish.onmessage = console.log;
stockfish.postMessage('uci');
```

bu kodun çıktısı bize Stockfish motorumuzun opsiyonları ile ilgili bilgi veriyor. Örneğin:

```
...
option name Threads type spin default 1 min 1 max 1
option name MultiPV type spin default 1 min 1 max 500
option name Skill Level type spin default 20 min 0 max 20
option name Move Overhead type spin default 30 min 0 max 5000
option name Minimum Thinking Time type spin default 20 min 0 max 5000
...
```

Burada ilgimi çeken şeylerden biri, *Threads* opsiyonun olması, ama min ve max değerlerinin 1 olması. Muhtemelen Node.js ile alakalı bir şey. Kütüphanenin nasıl çalıştığını daha iyi anlamam lazım. Kütüphanenin açıklamasının başında `Stockfish.js is a pure JavaScript implementation of Stockfish` yazıyor. Şimdi *pure Javascript* denince kafam karıştı. Orjinali C++ olan bir motoru, sıfırdan Javascript ile mi yazmışlar? Öyle ise korkunç. Çünkü kim bilir ne eksikleri var? Ne kadar eski? Yüzlerce kişinin emeği olan C++ projesini, JS ile yeniden kaç kişi yazdı? 

Umutsuzluğa kapılmadan önce, projenin kaynak koduna bakıyorum. Neyse ki proje ağırlıklı C++ olarak görünüyor. [emscripten](https://github.com/emscripten-core/emscripten) ve WebAssembly kullanılarak, Javascript kütüphanesi haline getirilmiş. WebAssembly olduğu için *pure Javascript* dendiğini düşünüyorum. Sonuçta C++ kod ile FFI kullanılarak iletişim kurulmuyor. Compile olan kod gerçekten Node.js'in kullandığı Javascript motoru ile çalıştırılıyor. Aynı zamanda kütüphane hem Node.js hem de tarayıcıyı destekliyor. Bir önemli sorumuz daha cevaplandı. Bu kütüphane ile web tarayıcısı üzerinde istemci tarafında da analiz yapabiliriz(belki ileride React Native). Stockfish C++ olduğu için [IOS](https://github.com/akamai/stockfish-ios) ve [Android](https://github.com/evijit/material-chess-android/tree/master/app/src/main/jni/stockfish) üzerinde çalıştırmakta mümkün olabilir. Eğer Stockfish'i sunucu tarafında çalıştırırsak, kullanıcı sayısı arttıkca sunucu tarafındaki mühendislik ve kaynak gereksinimi artıcak. İstemci için aynı sorun yok. Belki arkadaşın ürünü için istemci daha mantıklı olabilir? Ben öyle olmadığını varsayıp, devam ediyorum :D

Kütüphane ile ilgili araştırmamı sonlandırmadan önce bir kaç şey daha test etmek istiyorum.

### Bootstrap Maliyeti
Bu kütüphanenin WebAssembly kullandığını öğrendik. Peki bu WebAssembly kütüphanesini *require* etmek ne kadar maliyetli? Ve *dispose* edebiliyor muyuz? Yani işimiz bittiğinde, kullandığı kaynağı teslim etmesini, ve bitmesini sağlayabiliyor muyuz? Hemen örnek bir kod ile deniyelim.

```js
const stockfishCreator = require('stockfish');
const start = Date.now();
const stockfish = stockfishCreator();

stockfish.onmessage = line => {
	if (line === 'readyok') console.log(Date.now() - start);
};
stockfish.postMessage('isready');
```
*UCI* protokolüne göre, *isready*'e *readyok* ile cevap verilmesi lazım. Motor ayağa kalkar kalkmaz bu cevabı vermeli. Bu kodu çalıştırdığımda, ekranda gördüğüm çıktı 120 ms civarlarında. Bilgisayarım normal bir sunucudan daha hızlı. Bu başlangıç zamanı korkutucu. Aynı zamanda github üzerinde açık bir [issue](https://github.com/nmrugg/stockfish.js/issues/14) görüyorum. Birisi her Stockfish instance'ı oluşturduğunda 30 MB bir memory kullanımı olduğunu, ve bu memory'i free edemediğini söylemiş. Bu da korkutucu. Nihai kararı vermeden önce bir şeyi daha araştırmak istiyorum.

### CPU blok kontrolü
Bu kütüphaneyi Node.js process'imize yüklediğimiz zaman, yaptığımız çağrılar, bizim kodumuz ile aynı çekirdekte mi çalışıyor? Eğer öyle ise çok tehlikeli. Çünkü Stockfish 1 saniyelik bir düşünme sürecine girerse, bizim Node processimiz kitlenicek demek. Bunu kontrol edicek kodu hemen yazalım.

```js
// her 500 ms'de bir ping yazdıralım
setInterval(() => console.log('ping'), 500);
// 500 ms sonra, 1 kez stockfish fonksiyonu çağıralım
setTimeout(() => stockfishCustom.getBestMove('...'), 500);
// 2 saniye sonra, 10.000 kez, stockfish fonksiyonu çağıralım
setTimeout(() => [...new Array(10000)].map(() => stockfishCustom.getBestMove('...')), 2000);
```

Burada gözlemlediğimiz şu. Ekrana 500 ms sonra bir kez *ping* yazılıyor, ve hemen ardından en iyi hamle yazılıyor. Bundan sonra 500 ms aralıklarla iki kez daha *ping* yazılıyor ve anlık bir kitlenme yaşıyoruz. Korktuğum şey başımıza geldi, WebAssembly kodu, bizim kodumuzu bloklayabiliyor. Yani bu kütüphaneyi kendi process'imizden bağımsız çalıştırmak için takla atmamız gerekicek. Bu [worker_threads](https://nodejs.org/api/worker_threads.html) kullanarak olabilir, [cluster](https://nodejs.org/api/cluster.html) veya [child_process](https://nodejs.org/api/child_process.html) kullanarak olabilir. Veya deployment sırasında kodumuzdan bağımsız bu process'i ayağa kaldırıp, IPC yapabiliriz. Bunların hepsinin üzerinden geçicez. İlk önce kütüphane ile ilgili son bir konu.

### Yardımcı Fonksiyonlar
İşlemler arası iletişim ve iş dağıtımı konularını konuşurken, *UCI* veya kütüphane ile ilgili çok düşünmek istemiyorum. O yüzden hemen kendimize yardımcı bir fonksiyon yazalım. Yapmak istediğimiz satranç tahtasının durumu (*FEN* string), motor yetenek seviyesi (0-20 arasında bir sayı) ve arama derinliği (sayı) verildiği zaman, bize en iyi hamleyi dönen bir fonksiyon. Bu fonksiyonun genel yapısı şöyle olucak:

```js
module.exports = () => {
	// aslında bunu fonksiyon parametresi olarak almak daha mantıklı(DI), ama oraya hiç girmiyorum.
	const stockfish = require('stockfish')();
	
	function getBestMove(fen, level, depth) {
		// ...
		stockfish.postMessage(`... ${fen}`);
		// ...
		return Promise(resolve => {
			// stockfish cevabını dinlemek için bir şeyler yap
		});
	}
};
```

Artık bu kodu şu şekilde kullanabiliriz:
```js
const { getBestMove } = require('./customStockfish.js')();
const bestMove = await getBestMove('...', 20, ...);
```

Tek aklımızda tutmamız gereken, bu kütüphaneyi her require edip çağırdığımız zaman, yeni bir Stockfish instance'ı oluşacak. O yüzden bunu process başına 1 kez yapmak mantıklı. Bu yardımcı kodun tam implementasyonunu [burada](https://github.com/Yengas/stockfish-cluster-example/blob/ec3258455a3c276b1a460232e8aab97d7c55a6d6/stockfish/index.js) bulabilirisiniz.

## Kütüphaneyi Kullanmanın Farklı Stratejileri
Şimdi eğlenceli kısma gelebiliriz. Bu kütüphaneyi farklı şekillerde nasıl kullanabiliriz. Ve mimarimiz nasıl olacak? Kütüphanenin implementasyonu yüzünden, aynı anda birden fazla fonksiyon çağrısı yapamıyoruz. Çünkü fonksiyon çağırma ve cevap alma birbirinden bağımsız. Yani `stockfish.postMessage()` yapıyorum ama cevap string olarak `stockfish.onmessage`'a verdiğim fonksiyona geliyor. Aynı Stockfish instance'ına paralelde birden fazla *postMessage* yaparsam, bir şeylerin birbirine karışacağı aşikar. O yüzden kodumuzu yazarken bunu da hesaba katmamız lazım. İlk önce Node.js işlemimiz ile aynı thread içerisinde neler yapabiliyoruz ona bakalım.

### Her İstek Başına Stockfish Motoru 
Eğer Stockfish motorunu aynı anda birden fazla kez çağıramıyorsak, birden fazla Stockfish motorunu aynı anda birer kez çağırabiliriz. Bunun kodu oldukça basit.

```js
const customStockfishCreator = require('../customStockfish');

module.exports = {
	async getBestMove(fen, level, depth) {
		const { getBestMove: realGetBestMove, quit } = customStockFishCreator();
		const result = await realGetBestMove(fen, level, depth);
		
		// bu quit fonksiyonu malesef modülü düzgün temizleyemiyor... 
		// yani işlevsiz. belki başka bir kütüphanede bunu çalıştırabiliriz?
		quit();
		return result;
	},
};
```

Daha önce oluşturduğumuz yardımcı fonksiyon kütüphanesi, her gelen istek için sıfırdan bir Stockfish motoru oluşturup, onun üzerinde tek istek yapabiliriz.

```js
await Promise.all([
	getBestMove(...),
	gttBestMove(...),
]);
```

Yaptığımız zaman, 2 tane stockfish motoru oluşur, ve aynı anda istekleri işlerler. Araştırmamızı yapmasaydık, bununla yetinebilirdik belki. Ama buradaki sorunları artık görebiliyoruz.

1. Stockfish motoru oluşturmak ucuz bir işlem değil, her istek min 120 ms sürecek.
2. Stockfish motoru her açıldığında Memory'de yer kaplıyor ve bunu free etmenin bir yolunu bulamadık.
3. Stockfish motoru bizim tek threadimizde çalışıyor. O yüzden bu iki motorun çalışması birbirini blocklayacak, birden fazla motor oluşturmak, avantajımıza olmayacak.

Bu yöntem az kullanıcı ile çalışacak olsa da, kullanıcı sayımız arttıkca patlayacak. Bunu [örnek projede](https://github.com/Yengas/stockfish-cluster-example) rahatlıkla görebiliriz. Örnek projeyi localinize çekin, ve `npm run per_call:benchmark` çalıştırın. Bu komut bu strateji ile, 1.000 tane hamle hesaplaması yapmaya çalışacak. Bilgisayarınızın RAM kullanımın artışını, ve bir süre sonra işlemin patlamasını seyredin. Bu kütüphane için, bu yöntemi kullanmak mantıklı değil gibi... Eğer Memory free etmenin bir yolunu bulsak, ve yavaşlık bizim için sıkıntı olmasa, belki kullanılabilirdi?

### Aynı İşlemde Tek Bir Stockfish Motoru
O zaman aynı işlem içinde uygulayabileceğimiz bir başka seçeneği deniyelim. Sadece bir tane Stockfish motorumuz olsun, ama her hamle hesaplama isteğini bir sıraya dizip, tek tek işleyelim. Bunu bir array'e tüm istekleri pushlayıp, array'deki işleri işleyen bir fonksiyon ile yapabiliriz. Ya da Promise chaining ile çok daha kolay ve temiz yapabiliriz. Promise chaining ile ilerleyelim.

```js
const { getBestMove: realGetBestMove } = require('../customStockfish')();
let work = Promise.resolve();

module.exports = {
	async getBestMove(fen, level, depth) {
		return work = work.then(() => realGetBestMove(fen, level, depth).catch(console.error));
	},
};
```

Bu kadar basit. Tek bir Promise'imiz var, her bir hamle hesaplama isteği geldiğinde, bu promise'in ucuna bu işlemi ekleyip, dönüyoruz. Biraz kafa karıştırabilir. Ama bu şekilde, her bir hamle hesaplama isteği, sıralı şekilde çalışıyor. Yani artık:

```js
await Promise.all([
	getBestMove(...),
	getBestMove(...),
]);
```

Yaptığımız zaman, bu iki işlem paralel gibi gözükse de, aslında tek bir Stockfish motoru tarafından, tek tek işleniyor. Bunun da eksileri var.

1. Kullanıcı sayımız arttıkca, Stockfish motoru üzerinde iş birikecek, ve cevap süremiz artacak.
2. Hala tek CPU üzerinde işlem yapıyoruz. Stockfish motoru CPU'yu kilitleyip, sunucumuzu kötü etkileyecek.

Bu strateji için örnek projede 10.000 hamle işlemi yapmak isterseniz, `BENCHMARK_ANALYSIS_COUNT=10000 npm run single:benchmark` komutunu çalıştırabilirsiniz. Benim bilgisayarımda yaklaşık 3 dakika sonra tüm işlemler bitiyor. Aynı işlemde bir kaç takla daha atarak iyileştirme yapmayı deneyebiliriz. Örneğin 1. sorunu "çözmek" için, yukarıdaki dosyayı 1 kere require etmek yerine, birden fazla require edilebilecek hale getiririz. Ve her istekte round robin şekilde o motorlardan birine istek göndeririz. Ama tek CPU çalıştığımız için bu bize fayda sağlamayacak. Artık tek işlemden kurtulup, birden fazla işlem ile çalışmamız lazım.

### Alt İşlemler ile Birden Fazla Stockfish Motoru
Artık kararımızı verdik. Stockfish motorumuz ana işlemimiz ile aynı CPU'yu kullanmayacak. Ve her işlem başına 1 Stockfish motorumuz olacak. Peki bu Stockfish işlemlerini nasıl başlatabiliriz? Ana işlem ile bu Stockfish motorları arasında nasıl iletişim kurabiliriz? Bunun en basit yollarından biri, Node.js ile sunulan [child_process](https://nodejs.org/api/child_process.html) kütüphanesini kullanmak. Bu kütüphane, istediğimiz bir Javascript dosyasını, kendi işlemimiz ile arasında bir haberleşme köprüsü olacak şekilde çalıştırmamızı sağlıyor. Ana işlemimize Master, alt işlemlerimize de Worker diyelim.

Burada işi Workerlar arasında dağıtmak için farklı yöntemler kullanabiliriz. Örneğin N adet Worker başlattık ve her birinde 1 adet Stockfish motoru çalışıyor. Bunların hangilerinin müsait olduğunun takibini yaparız(şu anda hamle hesaplaması yapmıyor). Ana işlemde tüm istekleri biriktirip, müsait olan motora işlemleri sırasıyla verebiliriz.

Ya da bize gelen tüm işleri bir dağıtma stratejisi kullanarak(round robin, random, en az kaynak kullanan vs.) Stockfish motoru çalışan işlemlere göndeririz, onlar sıralamayı kendi içerisinde yapar. Bu yöntemde kodumuz bir önceki bölüme çok benzeyecek. O yüzden bu yöntemi tercih edelim. Mastera gelen her istek, rastgele şekilde Workerlardan birine gönderilsin. Workerlar kendi içerisinde sıralı olarak işleyip, Mastera cevabı göndersin.

İlk önce *worker.js* kodumuz ile başlayalım:

```js
const { getBestMove } = require('../customStockfish')();

async function findBestMoveAndSendReply({ id: workId, params: { fen, level, depth } }) {
	try {
		const reply = await getBestMove(fen, level, depth);
		
		process.send({ workId, reply });
	} catch(err) {
		// eğer bu işleme spesifik bir hata olursa, Promsie chain'i durdurmak istemeyiz.
		console.error('there was an error when processing work:', err.message);
	}
}

let workChain = Promise.resolve();

process.on('message', workData => {
	workChain = workChain.then(() => (
		findBestMoveAndSendReply(workData)
	));
});
```

Burada kullanılan 2 fonksiyon Node.js'in bize sağladığı haberleşme köprüsünden geliyor. `process.on('message', ...)` ana işlem'den bir istek geldiğinde çalışan fonksiyonumuz. `process.send(...)` ise ana işleme mesaj göndermek için kullandığımız fonksiyon. Worker tarafı tamamdır. Bu *worker.js* dosyasını `child_process.fork` ile çalıştırdığımız zaman, hamle hesaplama için iş göndermeye başlayabiliriz. Bizim işlemimizden bağımsız şekilde, çalışmaya başlayacak. Örnek projede [worker.js](https://github.com/Yengas/stockfish-cluster-example/blob/ec3258455a3c276b1a460232e8aab97d7c55a6d6/child_process/worker.js) implementasyonuna da bakmak isteyebilirsiniz.

Şimdi işlem göndermek ile yükümlü olan kodumuza gelelim. *master.js* dosyasında workerlara iş dağıtmamız lazım. Worker işlem başlatma kodunu şimdilik dışarıda tutalım. *master.js* kodumuzda Workerların başka yerden geldiğini kabul edelim.

```js
const uuidv4 = require('uuid/v4');

const workWaitingMap = new Map();
const workers = [];

module.exports = {
	addWorker(worker) {
		workers.push(worker);
		
		// worker'dan cevap geldiği zaman, bekleyen fonksiyonları çağıralım
		worker.on('message', action => {
			const { workId, reply } = action;
			const waitingFunctions = workWaitingMap.get(workId);
			workWaitingMap.delete(workId);

			if (Array.isArray(waitingFunctions))
				waitingFunctions.forEach(func => func(reply));
		});
	},
	addWork(params) {
		const workId = uuidv4();
		const work = { id: workId, params };
		const worker = workers[Math.floor(Math.random() * workers.length)];

		worker.send(work);
		return workId;
	},
	waitWorkReply(workId) {
		return new Promise((resolve) => {
			if (!workWaitingMap.has(workId)) {
				workWaitingMap.set(workId, []);
			}

			workWaitingMap.get(workId).push(resolve);
		});
	},
};
```

Bu kod biraz daha karmaşık görünüyor. Ama oldukça basit:

- *addWorker* fonksiyonu ile alt işlem referansımızı alıyoruz. Bir dizi'de ileride kullanmak için saklıyoruz, ve gelen mesajlardan cevabımızı çıkarıp, cevap için beklemekte olan fonksiyonları çağırıyoruz.
- *addWork* verilen parametreler ile bir iş oluşturuyor. İş oluştururken, random bir id atıyor. Böylece Workerlardan cevap geldiği zaman, hangi işimizin cevabını olduğunu takip edebileceğiz(correlation id). Daha sonrada random bir Worker'a, bu işi gönderiyor ve işin id'sini bize dönüyor.
- *waitWorkReply* verilen id'li işin cevabını bekleyen bir promise döndürüyor. Böylece iş gönderdikten sonra, cevabını beklemek için yardımcı bir fonksiyonumuz olmuş oluyor.

Artık `const workId = master.addWork({ fen: xxx, level: y, depth: z })` diyerek bir iş oluşturabiliriz. Ve cevabını almak için de `const bestMove = await master.waitWorkReply(workId)` diyebiliriz. Bunu tek fonksiyona da indirgeyebiliriz.

```js
function getBestMove(fen, level, depth) {
	const workId = master.addWork({ fen, level, depth });
	
	return master.waitWorkReply(workId);
}
```

Şimdi yapmamız gereken, *master.js*'e ekleyeceğimiz Workerları oluşturmaya. Birden fazla alt işlem olarak çalıştırmamız lazım. Bunu uygulamamızın başlangıcında yapabiliriz. Kod şu şekilde: 

```js
const child_process = require('child_process');
const clusterMaster = require('../child_process/master');
const NUM_OF_WORKERS = 4;


for (let i = 0; i < NUM_OF_WORKERS; i++) {
	const worker = child_process.fork(require.resolve('../child_process/worker'));

	clusterMaster.addWorker(worker);
}
```

artık *clusterMaster*'ı kullanarak hamle hesaplaması yapabiliriz. Bu kodların çalışan halini [master.js](https://github.com/Yengas/stockfish-cluster-example/blob/ec3258455a3c276b1a460232e8aab97d7c55a6d6/child_process/master.js), [child_process/index.js](https://github.com/Yengas/stockfish-cluster-example/blob/ec3258455a3c276b1a460232e8aab97d7c55a6d6/child_process/master.js) ve [entrypoints/child_process.js](https://github.com/Yengas/stockfish-cluster-example/blob/ec3258455a3c276b1a460232e8aab97d7c55a6d6/entrypoints/child_process.js) dosyalarında, örnek projede görebilirsiniz. `BENCHMARK_ANALYSIS_COUNT=10000 npm run child_process:benchmark` komutunu çalıştırarak test ettiğimde, 10.000 hamlenin 20 saniyede hesaplandığını görüyorum.

Performans artışı sağladık, ve artık ana işlemimizi Stockfish motoru ile kitlemiyoruz. Bizim için bu kadar mühendislik yeterli olabilir. Ama bir kaç noktaya değinelim.

1. Her ana işlem (Selim'in durumunda WebSocket sunucusu) kendisi ile birlikte N tane Stockfish motoru başlatıyor. Ama aynı makinede, birden fazla process olarak çalışıyorlar. Hala birbirlerinin çalışmasını etkileyebilirler.
2. WebSocket sunucumuzu, Stockfish motorlarımızdan bağımsız ölçekleyemiyoruz. İkinci bir makinede Stockfish motoru çalıştırmak istersek, WebSocket sunucumuzu da orada çalıştırmamız gerekiyor.

Bunlar bizim için bir sorun olmayabilir. 4 çekirdekli bir sunucuda, 100 kullanıcıya hizmet verecek bir şeyler yapıyor olabiliriz. Bu kadar mühendislik bizim için yeterli olabilir. Ama uygulamamızı yayınladığımız ortam, kurulumumuz, veya kullanıcı sayımız böyle bir çözüm ile yetinmememize sebep olabilir. O yüzden bir strateji daha inceleyip, konuşmak istiyorum.

### Bağımsız İşlemler ile Birden Fazla Stockfish Motoru
[child_process](https://nodejs.org/api/child_process.html) kullanarak, ana işlemimiz ile aynı sunucuda, bağımlı işlemler başlatabiliyoruz. Ama Stockfish motorlarını bizim işlemimizden tamamen bağımsız yapmak istersek ne yapacağız? 100 kullanıcımız varken 3 motor, 1000 kullanıcımız varken 10 motor çalıştırmak istiyorsak? Stockfish motorlarının donanım gereksinimleri farklı ise ve onları özel makinelerde çalıştırmak istiyorsak? Bunları sağlamak için, araya ağ katmanı koymamız lazım. Örneğin Stockfish motorunu bir REST API haline getirebiliriz, veya herhangi bir queue teknolojisini, bu amaç için kullanabiliriz(örn. [RabbitMQ](https://www.rabbitmq.com), [Redis](https://redis.io/topics/pubsub)).

#### REST API ile
Stockfish motorunu ayrı bir işleme çıkartıp, bu işlemde REST API sunucusu ayağa kaldırabiliriz. Böylece ana işlemimiz Stockfish motoru çalıştıran sunuculardan birine REST isteği gönderebilir, ve en iyi hamle önerisini alabilir. Burada ana işlemimizin, birden fazla Stockfish REST API çalıştırdığımız zaman, hangisine istek atacağını nasıl bulacağı gibi bir sorun ortaya çıkıyor(Service Discovery). Bu sorunu deployment ortamımıza göre çözebiliriz. *Kubernetes* tarafında Service, veya nginx ile reverse proxy kullanmamız gerekebilir. Şimdilik bunu unutalım. 

Bu şekilde ilerlemeyi seçersek, [server.js](https://github.com/Yengas/stockfish-cluster-example/blob/260165f4bfac62b52858116b624417f7482034d9/rest/server.js) kodumuz:
```js
const fastify = require('fastify')();
const { getBestMove } = require('../stockfish')();

let work = Promise.resolve();

fastify.get('/chess', (request) => {
	const { fen, level, depth } = request.query;

	return work = work.then(() => getBestMove(fen, level, depth));
});

fastify.listen(process.argv[2]);
```

Ana [işlemimizden](https://github.com/Yengas/stockfish-cluster-example/blob/260165f4bfac62b52858116b624417f7482034d9/rest/index.js) bu sunucuya bir çağrı:

```js
const axios = require('axios');

module.exports = () => {
	return {
		async getBestMove(fen, level, depth) {
			let url = `http://localhost:8080/chess?fen=${encodeURIComponent(fen)}&level=${level}`;

			if (depth) url += `&depth=${depth}`;
			const { data: body } = await axios.get(url);
			return body;
		},
	}
};
```

REST API sunucusundan istediğimiz kadar, istediğimiz sayıda makinede çalıştırabiliriz. Tek yapmamız gereken ana işlemimize girdiğimiz `localhost:8080` url'sini, load balancer URL'i ile değiştirmek. Load balancer çalışmakta olan REST API sunucularına yükü dağıtmakla yükümlü olucak. Demo'da load balancer yerine, N tane sunucu farklı portlarda başlatılıyor, ve istek atılırken bu sunuculardan birinin portu rastgele seçiliyor. `BENCHMARK_ANALYSIS_COUNT=10000 npm run rest:benchmark` komutunu çalıştırarak bunu test edebilirsiniz. Benim makinemde 10.000 istekte, REST API sunucuları timeout vermeye başlıyor ve benchmark fail oluyor. 5.000 istekte 12 saniyede cevap alabiliyorum. Bunun sebebi REST API sunucularının tüm istekleri bir kerede üstüne alması, ve sıralı işlem yaparken bu isteklerin timeout alması. Bu sorunun *fastify* mı yoksa *axios* tarafından mı kaynaklandığını çözemedim. Pratikte her sunucunun eş zamanlı 1.000 istek almayacağını varsaydığımız için, bu yöntemi sıkıntılı olarak düşünmeyin. Güzel bir kurulum ile, sorun olmayacaktır.

#### Queue Teknolojisi ile
REST API'ye alternatif olarak, Queue teknolojisi de kullanabiliriz. Eğer kurulum ortamımız da REST API ile yük dağıtımı yapmak zor olacaksa, bu iş için Queue'da kullanabiliriz. Queue teknolojilerinin bize getirdiği [kolaylıklar](https://www.rabbitmq.com/features.html) ve [soyutlamalar](https://www.rabbitmq.com/tutorials/amqp-concepts.html) da işimize yarayabilir. Küçük bir araştırma sonucunda, Redis kullanan, ve ihtiyacımız olan her şeyi basit bir API ile sağlayan [bee-queue](https://github.com/bee-queue/bee-queue) adında kütüphane buldum.

Bu kütüphane ile [worker.js](https://github.com/Yengas/stockfish-cluster-example/blob/ec3258455a3c276b1a460232e8aab97d7c55a6d6/queue/worker.js) kodumuz:
```js
const { getBestMove } = require('../stockfish')();
const queue = require('./createQueue')();

// .process fonksiyonu default olarak, aynı anda max 1 tane işlem yapıyor.
queue.process(({ data: { fen, level, depth }}) => (
	getBestMove(fen, level, depth)
));
```

[master.js](https://github.com/Yengas/stockfish-cluster-example/blob/ec3258455a3c276b1a460232e8aab97d7c55a6d6/queue/index.js) kodumuz:

```js
const createQueue = require('./createQueue');

module.exports = () => {
	const queue = createQueue();

	return {
		getBestMove(fen, level, depth) {
			return new Promise((resolve, reject) => {
				const job = queue.createJob({ fen, level, depth });

				job.save();
				job.on('succeeded', result => resolve(result));
				job.on('error', err => reject(err));
			});
		},
		close() {
			return queue.close();
		}
	}
};
```

[createQueue.js](https://github.com/Yengas/stockfish-cluster-example/blob/ec3258455a3c276b1a460232e8aab97d7c55a6d6/queue/createQueue.js) kodumuz:
```js
const Queue = require('bee-queue');

module.exports = () => {
	// auto generated by the queue entrypoint
	const { ip: host, port } = require('./config.json');

	return new Queue('chess', { redis: { host, port } });
};
```

#### Sonuç
REST API veya Queue farketmez, tek yapmamız gereken, bu Master/Worker işlemleri birbirinden bağımsız şekilde çalıştırmak. Bunu [pm2](http://pm2.keymetrics.io/) kullanarak da yapabilirsiniz, [ECS](https://aws.amazon.com/ecs/) kullanarak da, [Kubernetes](https://kubernetes.io) kullanarak da. Örneğin Kubernetes üzerinde WebSocket sunucusunu ve Stockfish workerlarını farklı deployment yapabiliriz. Stockfish CPU kullanımı artınca otomatik ölçeklenmesini söyleyebiliriz. Böylelikle kullanıcı sayımız arttıkca, WebSocket sunucumuzdan bağımsız olarak, Stockfish motorlarımız ölçeklenebilir. Kod tarafı bu basitlikte iken, nasıl bir kurulum ortamımız olduğuna göre, istediğimiz gibi ölçeklenebiliriz.

Aynı zamanda araya Node.js'e spesifik bir iletişim protokolü (*child_process* IPC) değil de, ağ katmanı koyduğumuz için, Stockfish motorunu istediğimiz dilde çalıştırmak da oldukça kolay olacak. İstersek C++ ile bir proje yazarız ve ağ üzerinden erişiriz, istersek Rust ile. Böylelikle belki tek bir motorumuz ile birden fazla CPU kullanabiliriz? Kısaca faydalarından ve dezavantajlarından bahsedelim:

1. Ölçeklenmemizi limitleyecek şey Queue teknolojimiz veya Load Balancer'ımız haline geldi. İstediğimiz kadar makinede, istediğimiz kadar Worker çalıştırabiliriz. Queue/Load Balancer teknolojimiz, mesaj sayımızı kaldırmayana kadar. Bu çoğu proje için ulaşılamayacak bir limit.
2. Anlık yüke göre Stockfish motoru sayımızı değiştirebiliriz.
3. WebSocket sunucusu ve Stockfish motorunu tamamen farklı makinelarda çalıştırabiliriz.
4. İstersek Node.js modülünü kullanmaya devam edebiliriz, veya farklı bir dilde Stockfish motorunu kullanarak işlem yapabiliriz.

Eğer bilgisayarınızda docker yüklü ise `npm run queue:benchmark` çalıştırarak, bu queue stratejisini deneyebilirsiniz. REST API stratejisi için ise `npm run rest:benchmark` çalıştırabilirsiniz.

Bu maddelere bağımlı olarak, deployment sürecimiz kompleksleşti. Queue durumunda ise yeni bir veritabanı bağımlılığımız oldu. Bu dezavantajları kabullenip, böyle bir yapı kurmak, bazen bir seçenek olabilir. Ama bazen de zorunluluk haline gelebilir. Bu benim kişisel projem olsaydı, ben ne yapardım sorusuna aşağıda cevap veriyorum.

## Nasıl bir mimari yapalım?
Her zaman en iyi çözümü uygulamak, harcadığımız efora değmeyecek. O yüzden eğer kişisel projem olsaydı, çözümleri şu sırada denerdim:

1. Yükümün az olduğunu varsayıyorum. WebSocket projesi içinde Stockfish modülünü kullanırdım. [child_process](https://nodejs.org/api/child_process.html) veya [worker_threads](https://nodejs.org/api/worker_threads.html) kullanarak Stockfish modülünü çalıştırırdım. Ama mesajlaşma yönetimini kendim yapmak yerine, *bee-queue*'yu redis olmadan *worker_threads* ile çalıştırmayı denerdim.
2. Yüküm arttı veya Stockfish JS düzgün çalışmadığını farkettim. Araya REST API koyardım. Bunu yapınca, benim için Stockfish'i Node.js ile çalıştırmanın bir anlamı kalmıyor. Bir Docker imajı oluşturup, içine C++ Stockfish motorunu koyardım. Node.JS veya C++ ile bu motor ile konuşup, REST API sunan bir kod yazardım. Kubernetes ile çalıştırarak load balancing ve auto scaling sorunlarını çözerdim. Demo'da eş zamanlı 10.000 istek sıkıntı yaratsa da, deployment sırasında güzel bir kurulum ve daha iyi kod ile sorunun çözülebileceğini düşünüyorum.

Eğer ağ üzerinden gönderilen veriler ve gelen cevaplar büyük olsaydı(fen, level, depth çok küçük), o zaman REST API yerine GRPC deneyebilirdim? Veya yeniden deneme stratejileri, işlerin Worker üzerinde birikmesi gibi sorunlar olsaydı Queue koymayı deneyebilirdim. Ne olursa olsun ilk önce basit başlar, ondan sonra yaşadığım sorunlara göre çözümler ve yeni bir mimari üretirdim.

## Örnek Proje
Bahsettiğim stratejilerin hepsini [örnek proje'de](https://github.com/Yengas/stockfish-cluster-example/tree/ec3258455a3c276b1a460232e8aab97d7c55a6d6) inceleyebilirsiniz. `npm run child_process:benchmark` *child_process* stratejisine 1.000 eş zamanlı hamle öneri isteği gönderiyor, ve ne kadar sürdüğünü ekrana bastırıyor. 

Eğer `npm run child_process` komutunu çalıştırırsanız, *child_process* stratejisini kullanan bir WebSocket sunucusu başlatıyor. Bu WebSocket sunucusuna `npm run client` diyerek bağlanıp, *FEN* gönderebilirsiniz. Size o tahta durumu için, en iyi olduğunu düşündüğü hamleyi söylecek.
