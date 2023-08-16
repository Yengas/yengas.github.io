+++
type = "posts"
title = "Bir Satranç Oyunu, Node.JS ile İşlemler Arası İletişim ve İş Dağıtımı"
date = "2019-08-16T12:00:00+03:00"
Categories = ["Node.JS", "Architecture"]
Tags = ["node.js", "performance", "ipc", "webassembly", "ipc"]
+++

Bu makalenin konusu, Facebook gruplarından birinde karşılaştığım güzel bir sorudan geliyor. Soru hem temel konularda bilgi sahibi olmayı, hem de kullanılacak aracı iyi tanımayı gerektiriyor. Bu konuda fikirlerimi kısa bir cevap olarak yazarsam, haksızlık ederim diye düşündüm. Aynı zamanda bir şeyler öğrenmek ve kod maymunluğu yapmak için eğlenceli bir konu bulmuş oldum. Ortaya bu makale ve github üzerinde yayınladığım [örnek kodlar](https://github.com/Yengas/stockfish-cluster-example) çıktı. Soruya bakarak, esas konumuzu yavaş yavaş işleyelim.

## Bir Satranç Oyunu
<center><img src="/img/articles/selim-question.png" alt="drawing" style="width:70%; max-width: 600px; @media (min-width: 768px) { width: 50%; }"/></center>

Anlaşıldığı üzere, arkadaş bir satranç oyunu yapıyor ve kullanıcıyı bilgisayara karşı yarıştırmak istiyor. Stockfish motorunu(hamle önerisi yapan bir program), uygulamalarını yayınladığı ortama kurmuş ve şimdi bunu Node.js socket sunucusu ile entegre ederken kafasında soru işaretleri var. Bu soruya bakınca aklımda ilişkili sorular uçuşmaya başlıyor:

1. Bu motoru istemci tarafında çalıştırabilir miyim? Sunucu'da uğraşmamak, beni fazlaca mühendislikten kurtarır.
2. Stockfish motorunu kendi kontrolümde, Node.js işlemim içerisinde çalıştırabilir miyim?
3. Oldu da Node.js içerisinde çalıştırdım. Bunun bana CPU/Memory maliyeti ne olacak?
4. Bu motor herhangi bir state tutuyor mu? Örn. bir kullanıcı oyuna başladı, başlangıç itibari ile tüm hamleleri sırası ile mi motora vermem lazım? Yoksa her istekte tüm tahta durumunu mu vermeliyim? 
5. Ne kadar kullanıcım olacak? Anlık aktif satranç oyunu sayısı kaç olabilir?

Araştırmaya koyulmam lazım. Her sorunun cevabı dizayn ettiğim sistemi değiştirebilir. Ama ilk önce bir duraksayıp, bu sorunun neden makale yazmaya değer olduğunu anlatayım.

### Başka Birinin Kodu
Projelerimizde ihtiyacımız olan özellikleri edinmek için başkalarının kodunu hep kullanıyoruz. Resim küçültüp, büyütmek için olabilir. PDF bir dosya çıktısı almak için olabilir. QRCode oluşturmak için olabilir. Bu tarz ihtiyaçlarımızı NPM çoğu zaman karşılıyor. Ama bazı ihtiyaçlarımızı karşılayacak kodlar, kütüphane olarak bulunmayabilir. Böyle durumlarda artık kendi projemizin dışına adım atmamız ve farklı yazılımlar/işlemler ile konuşmamız gerekiyor.

Bu olaya işlemler arası iletişim ([IPC](https://stackoverflow.com/questions/40005935/which-kind-of-inter-process-communication-ipc-mechanism-should-i-use-at-which)) diyoruz. Eğer uygulamanın bir konsol arayüzü varsa, bu uygulama ile iletişim kurmak için ilk akla gelen şey; "komut çalıştırmak" oluyor. Evet bunu yapmak oldukca kolay, [child_process](https://nodejs.org/api/child_process.html) kullanılarak, istediğimiz programı, istediğimiz argümanlar ile başlatabiliriz. Başlayan programın stdin ve stdout "dosyalarını" kendi Node.js işlemimize bağlayarak da, istediğimiz gibi iletişim kurabiliriz.

Bu yöntemle birlikte düşünülmesi gereken şeyler artıyor. Bizim kodumuzun çalıştığı her yerde, bu uygulamanın kurulu olması gerekiyor. Hem de aynı versiyon ve konfigürasyon ile. Kullanmak istediğimiz tüm özelliklerin komut satırı üzerinden sağlanıyor olması gerekiyor. Eğer kendi kodumun içine bu fonksiyonaliteyi gömebiliyorsam, bunlarla uğraşmak istemem.

Neyse ki Stockfish durumunda, NPM üzerinde bir kütüphane [bulunuyor](https://www.npmjs.com/package/stockfish). Ama bu seferde başka sorunlar ortaya çıkıyor. Bu kütüphanenin nasıl çalıştığını anlamam lazım. Satranç hamlesi önermek CPU ucuz bir işlem olmamalı. Ben bu kütüphaneden bir fonksiyon çağırdığımda, CPU kullanımından dolayı sunucum kitlenecek mi? Sonuçta Node.js tek thread ve tek CPU demek. Bu kütüphanedeki bir fonksiyon 1 saniye beni kitlerse, o 1 saniye hiç bir kullanıcım hizmet alamayacak demek. Böyle bir duruma engel olmak için de kendi projem içindeki fonksiyonaliteleri birden fazla işlem olarak çalıştırmam ve IPC yapmam gerekebilir. 

Soruların hepsi çok güzel. Araştırma yapmam lazım.

## Stockfish ve Satranç Hamle Önerisi
Üç taş oyunu gibi oyunları bilgisayara oynatmaya çalıştığımızda, bilgisayarın tüm ihtimalleri değerlendirmesi çok kısa sürer. Kazanan strateji de önceden belirlidir. Ama satranç gibi çok fazla ihtimal içeren kompleks oyunlarda, tüm ihtimalleri değerlendirmek imkansız. Bu yüzden Stockfish gibi satranç motorları, tahtanın şu anki durumu üzerinden, potansiyel hamlelerin ne kadar mantıklı olduğunu(hangi hamlelerin kaybetme ihtimalini azaltacağını) hesaplamak için geleceğe dönük bir arama yapar. Problemin doğası gereği, bu ucuz bir işlem değil ve CPU'nun ısınmasına sebep olabilir. Stockfish'i kendi Node process'im içerisinde çalıştırmak istiyorum. O yüzden bunu aklımda tutmam lazım. 

Ama ilk önce bu motor ile nasıl konuşacağıma, hamle önermek için benden ne isteyeceğine bakıyorum. İnternette kısa bir araştırma sonucunda, [aradığımı](https://chess.stackexchange.com/a/12581) buldum. [Universal Chess Interface](http://wbec-ridderkerk.nl/html/UCIProtocol.html) (UCI) adı verilen bir protokol sayesinde, Stockfish'e tahtamın şu anki durumunu veriyorum ve istediğim sayıda hamle önerisi alabiliyorum. En basit örnek şuna indirgenebilir:

```
position fen r3kb1r/p2npppp/2p2n2/3N4/8/5N2/PPPP1PPP/R1B2RK1 b kq - 0 10
go
```

Kriptik görünen `r3kb1r/p2npppp/2p2n2/3N4/8/5N2/PPPP1PPP/R1B2RK1 b kq - 0 10` kısmının [açıklaması](http://kirill-kryukov.com/chess/doc/fen.html) aslında oldukca [basit](http://www.chessgames.com/fenhelp.html). *FEN* adı verilen bu notasyon, tahtanın konumunu, sıranın hangi oyuncuda olduğu, kaçıncı hamle olduğunu gibi bilgileri barındırıyor. Önemli sorularımdan biri cevaplandı. Her hamle önerisinde, tahtanın tüm pozisyonunu Stockfish'e verebiliyorum ve [önerilen](https://chess.stackexchange.com/a/12670) bu. Bu demek oluyor ki, 1 tane motor çalıştırıp, eş zamanlı birden fazla kullanıcıya hizmet verebilirim. Motor herhangi bir state tutmuyor ve arka arkaya iki farklı oyunun FEN notasyonu verildiği zaman şaşırmıyor.

Aynı zamanda araştırmalarımda Stockfish'in multi-thread çalışabildiği, düşünme zamanının kısıtlanabildiği, arama derinliğinin limitlenebildiği, önerilen hamle sayısının ayarlanabildiği, yetenek seviyesi ayarı bulunduğu gibi önemli bilgileri de edindim. Şimdi kod kısmına geçebilirim.

## Node.js Stockfish Kütüphanesi
Öğrendiklerimi kullanarak, Stockfish Node.js [kütüphanesi](https://www.npmjs.com/package/stockfish) ile hemen bir örnek yapıyorum.

```js
const stockfish = require('stockfish')();

stockfish.onmessage = console.log;
stockfish.postMessage('uci');
```

bu kodun çıktısı bana Stockfish motorunun opsiyonları ile ilgili bilgi veriyor. Örneğin:

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

Umutsuzluğa kapılmadan önce, projenin kaynak koduna bakıyorum. Neyse ki proje ağırlıklı C++ olarak görünüyor. [emscripten](https://github.com/emscripten-core/emscripten) ve WebAssembly kullanılarak, Javascript kütüphanesi haline getirilmiş. WebAssembly olduğu için *pure Javascript* dendiğini düşünüyorum. Sonuçta C++ kod ile FFI kullanılarak iletişim kurulmuyor. Compile olan kod gerçekten Node.js'in kullandığı Javascript motoru ile çalıştırılıyor. Aynı zamanda kütüphane hem Node.js hem de tarayıcıyı destekliyor. Bir önemli sorum daha cevaplandı. Bu kütüphane ile web tarayıcısı üzerinde istemci tarafında da analiz yapabilirim(belki ileride React Native). Stockfish C++ olduğu için [IOS](https://github.com/akamai/stockfish-ios) ve [Android](https://github.com/evijit/material-chess-android/tree/master/app/src/main/jni/stockfish) üzerinde çalıştırmakta mümkün olabilir. Eğer Stockfish'i sunucu tarafında çalıştırırsam, kullanıcı sayısı arttıkca sunucu tarafındaki mühendislik ve kaynak gereksinimi de artacak. İstemci için aynı sorun yok. Belki arkadaşın ürünü için istemci daha mantıklı olabilir? Ben öyle olmadığını varsayıp, devam ediyorum :D

Kütüphane ile ilgili araştırmamı sonlandırmadan önce bir kaç şey daha test etmek istiyorum.

### Bootstrap Maliyeti
Bu kütüphanenin WebAssembly kullandığını öğrendim. Peki bu WebAssembly kütüphanesini *require* etmek ne kadar maliyetli? Ve *dispose* edebiliyor muyum? Yani işim bittiğinde, kullandığı kaynağı teslim etmesini ve yok olmasını sağlayabiliyor muyum? Hemen örnek bir kod ile deniyorum.

```js
const stockfishCreator = require('stockfish');
const start = Date.now();
const stockfish = stockfishCreator();

stockfish.onmessage = line => {
	if (line === 'readyok') console.log(Date.now() - start);
};
stockfish.postMessage('isready');
```
*UCI* protokolüne göre, *isready*'e *readyok* ile cevap verilmesi lazım. Motor ayağa kalkar kalkmaz bu cevabı vermeli. Bu kodu çalıştırdığımda, ekranda gördüğüm çıktı 120 ms civarlarında. Bilgisayarım normal bir sunucudan daha hızlı. Bu başlangıç zamanı korkutucu. Aynı zamanda github üzerinde açık bir [issue](https://github.com/nmrugg/stockfish.js/issues/14) görüyorum. Birisi her Stockfish instance'ı oluşturduğunda 30 MB bir memory kullanımı olduğunu ve bu memory'i free edemediğini söylemiş. Bu da korkutucu. Nihai kararı vermeden önce bir şeyi daha araştırmak istiyorum.

### CPU blok kontrolü
Bu kütüphaneyi Node.js process'imize yüklediğim zaman, yaptığım çağrılar benim kodum ile aynı çekirdekte mi çalışıyor? Eğer öyle ise çok tehlikeli. Çünkü Stockfish 1 saniyelik bir düşünme sürecine girerse, Node process'im kitlenecek demek. Bunu kontrol edecek kodu hemen yazıyorum.

```js
// her 500 ms'de bir ping yazdır
setInterval(() => console.log('ping'), 500);
// 500 ms sonra, 1 kez stockfish fonksiyonu çağır
setTimeout(() => stockfishCustom.getBestMove('...'), 500);
// 2 saniye sonra, 10.000 kez, stockfish fonksiyonu çağır
setTimeout(() => [...new Array(10000)].map(() => stockfishCustom.getBestMove('...')), 2000);
```

Burada gözlemlediğim şu: Ekrana 500 ms sonra bir kez *ping* yazılıyor ve hemen ardından en iyi hamle yazılıyor. Bundan sonra 500 ms aralıklarla iki kez daha *ping* yazılıyor ve anlık bir kitlenme yaşıyorum. Korktuğum şey başıma geldi. WebAssembly kodu, benim kodumu bloklayabiliyor. Yani bu kütüphaneyi kendi process'imden bağımsız çalıştırmak için takla atmam gerekicek. Bu [worker_threads](https://nodejs.org/api/worker_threads.html) kullanarak olabilir, [cluster#fork](https://nodejs.org/api/cluster.html) kullanarak olabilir, [child_process#fork](https://nodejs.org/api/child_process.html) kullanarak olabilir veya deployment sırasında kodumdan bağımsız bi process'i ayağa kaldırıp IPC yaparak olabilir. Bunların hepsini deneyeceğim ancak kütüphane ile ilgili aklıma takılan son bir konu daha var.

### Yardımcı Fonksiyonlar
İşlemler arası iletişim ve iş dağıtımı konularını konuşurken, *UCI* veya kütüphane ile ilgili çok düşünmek istemiyorum. O yüzden hemen kendime yardımcı bir fonksiyon yazıyorum. Yapmak istediğim; satranç tahtasının durumu (*FEN* string), motor yetenek seviyesi (0-20 arasında bir sayı) ve arama derinliği (sayı) verildiği zaman, bana en iyi hamleyi dönen bir fonksiyon. Bu fonksiyonun genel yapısı şöyle olucak:

```js
module.exports = () => {
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

Artık bu kodu şu şekilde kullanabilirim:
```js
const { getBestMove } = require('./customStockfish.js')();
const bestMove = await getBestMove('...', 20, ...);
```

Tek aklımda tutmam gereken, bu kütüphaneyi her require edip çağırdığım zaman, yeni bir Stockfish instance'ı oluşacak. O yüzden bunu process başına 1 kez yapmak mantıklı. Bu yardımcı kodun tam implementasyonu [burada](https://github.com/Yengas/stockfish-cluster-example/blob/ec3258455a3c276b1a460232e8aab97d7c55a6d6/stockfish/index.js) bulunabilir.

## Kütüphaneyi Kullanmanın Farklı Stratejileri
Şimdi eğlenceli kısma gelebilirim. Bu kütüphaneyi farklı şekillerde nasıl kullanabilirim? Mimarim nasıl olacak? Kütüphanenin implementasyonu yüzünden, aynı anda birden fazla fonksiyon çağrısı yapamıyorum. Çünkü fonksiyon çağırma ve cevap alma birbirinden bağımsız. Yani `stockfish.postMessage()` yapıyorum ama cevap string olarak `stockfish.onmessage`'a verdiğim fonksiyona geliyor. Aynı Stockfish instance'ına paralelde birden fazla *postMessage* yaparsam, bir şeylerin birbirine karışacağı aşikar. O yüzden kodumu yazarken bunu da hesaba katmam lazım. İlk önce ana Node.js işlemim içerisinde neler yapabiliyorum ona bakıyorum.

### Her İstek Başına Stockfish Motoru 
Eğer Stockfish motorunu aynı anda birden fazla kez çağıramıyorsam, birden fazla Stockfish motorunu aynı anda birer kez çağırabilirim. Bunun kodu oldukça basit.

```js
const customStockfishCreator = require('../customStockfish');

module.exports = {
	async getBestMove(fen, level, depth) {
		const { getBestMove: realGetBestMove } = customStockFishCreator();
		const result = await realGetBestMove(fen, level, depth);
		
		return result;
	},
};
```

Daha önce oluşturduğum yardımcı fonksiyon kütüphanesini kullanarak, her gelen istek için sıfırdan bir Stockfish motoru oluşturup, bu motor üzerinde tek bir istek yapabilirim.

```js
await Promise.all([
	getBestMove(...),
	gttBestMove(...),
]);
```

Yaptığım zaman, 2 tane stockfish motoru oluşur ve aynı anda istekleri işlerler. Araştırmamı yapmasaydım, bununla yetinebilirdim belki. Ama buradaki sorunları artık görebiliyorum.

1. Stockfish motoru oluşturmak ucuz bir işlem değil, her istek min 120 ms sürecek.
2. Stockfish motoru her açıldığında Memory'de yer kaplıyor ve bunu free etmenin bir yolunu bulamadım.
3. Stockfish motoru ana Node.js thread'im üzerinde çalışıyor. O yüzden bu iki motorun çalışması birbirini blocklayacak, birden fazla motor oluşturmak, avantajıma olmayacak.

Bu yöntem az kullanıcı ile çalışabilir. Fakat kullanıcı sayısı artınca patlayacak. Bu [örnek projede](https://github.com/Yengas/stockfish-cluster-example) rahatlıkla görülebilir. Örnek projeyi localinize çekin ve `npm run per_call:benchmark` çalıştırın. Bu komut; bahsedilen strateji ile, eş zamanlı 1.000 tane hamle hesaplaması yapmaya çalışacak. Bilgisayarın RAM kullanımının artışı ve bir süre sonra işlemin patlaması seyredilebilir. Bu kütüphane için, bu yöntemi kullanmak mantıklı değil gibi... Eğer Memory free etmenin bir yolunu bulsam ve yavaşlık benim için sıkıntı olmasa... Belki kullanabilirdim?

### Aynı İşlemde Tek Bir Stockfish Motoru
O zaman aynı işlem içinde uygulayabileceğim bir başka seçeneği deniyorum. Sadece bir tane Stockfish motorum olsun, ama her hamle hesaplama isteğini bir sıraya dizip, tek tek işleyeyim. Bunu bir array'e tüm istekleri pushlayıp, array'deki işleri işleyen bir fonksiyon ile yapabilirim. Ya da Promise chaining ile çok daha kolay ve temiz yapabilirim. Promise chaining ile ilerliyorum.

```js
const { getBestMove: realGetBestMove } = require('../customStockfish')();
let workChain = Promise.resolve();

module.exports = {
	async getBestMove(fen, level, depth) {
		return workChain = workChain.then(() => realGetBestMove(fen, level, depth).catch(console.error));
	},
};
```

Bu kadar basit. Tek bir Promise'im var. Her bir hamle hesaplama isteği geldiğinde, bu promise'in ucuna hamle öneri işlemi ekleyip, dönüyorum. Biraz kafa karıştırabilir. Ama bu şekilde, her bir hamle hesaplama isteği, sıralı şekilde çalışıyor. Yani artık:

```js
await Promise.all([
	getBestMove(...),
	getBestMove(...),
]);
```

Yaptığım zaman, bu iki işlem paralel gibi gözükse de, aslında tek bir Stockfish motoru tarafından, tek tek işleniyor. Bunun da eksileri var.

1. Kullanıcı sayısı arttıkça, Stockfish motoru üzerinde iş birikecek ve cevap süresi artacak.
2. Hala tek CPU üzerinde işlem yapıyorum. Stockfish motoru CPU'yu kilitleyip, sunucumu kötü etkileyebilir.

Bu strateji için örnek projede eş zamanlı 10.000 hamle öneri işlemi yapmak için, `BENCHMARK_ANALYSIS_COUNT=10000 npm run single:benchmark` komutu çalıştırılabilir. Benim bilgisayarımda yaklaşık 3 dakika sonra tüm işlemler bitiyor. Aynı işlemde bir kaç takla daha atarak iyileştirme yapmayı deneyebilirim. Örneğin 1. sorunu "çözmek" için, yukarıdaki dosyayı 1 kere require etmek yerine, birden fazla require edilebilecek hale getirebilirim ve her istekte round robin şekilde o motorlardan birine istek gönderirim. Ama tek CPU çalıştığım için bu bana fayda sağlamayacak. Artık tek işlemden kurtulup, birden fazla işlem ile çalışmam lazım.

### Alt İşlemler ile Birden Fazla Stockfish Motoru
Artık kararımı verdim. Stockfish motorum ana işlemim ile aynı CPU'yu kullanmayacak ve her işlem başına bir Stockfish motorum olacak. Peki bu Stockfish işlemlerini nasıl başlatabilirim? Ana işlem ile bu Stockfish motorları arasında nasıl iletişim kurabilirim? Bunun en basit yollarından biri, Node.js ile sunulan [child_process](https://nodejs.org/api/child_process.html) kütüphanesini kullanmak. Bu kütüphane, istediğim bir Javascript dosyasını, kendi işlemim ile arasında bir haberleşme köprüsü olacak şekilde çalıştırmamı sağlıyor. Ana işlemime Master, alt işlemlere Worker diyorum.

Burada işi Workerlar arasında dağıtmak için farklı yöntemler kullanabilirim. Örneğin N adet Worker başlattım ve her birinde 1 adet Stockfish motoru çalıştırıyorum. Bunların hangilerinin müsait olduğunun takibini yaparım(şu anda hamle hesaplaması yapmıyor). Ana işlemde tüm istekleri biriktirip, müsait olan motora işlemleri sırasıyla veririm.

Ya da bana gelen tüm işleri bir dağıtma stratejisi kullanarak(round robin, random, en az kaynak kullanan vs.) Stockfish motoru çalışan işlemlere gönderirim. Onlar sıralamayı kendi içerisinde yapar. Bu yöntemde kodum bir önceki bölüme çok benzeyecek. O yüzden bu yöntemi tercih ediyorum. Master'a gelen her istek, rastgele şekilde Workerlardan birine gönderilsin. Workerlar kendi içerisinde sıralı olarak işleyip, Mastera cevabı göndersin.

İlk önce *worker.js* kodum ile başlıyorum:

```js
const { getBestMove } = require('../customStockfish')();

async function findBestMoveAndSendReply({ id: workId, params: { fen, level, depth } }) {
	try {
		const reply = await getBestMove(fen, level, depth);
		
		process.send({ workId, reply });
	} catch(err) {
		// eğer bu işleme spesifik bir hata olursa, Promsie chain'i durdurmak istemiyorum.
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

Burada kullanılan 2 fonksiyon Node.js'in sağladığı haberleşme köprüsünden geliyor. `process.on('message', ...)` ana işlem'den bir istek geldiğinde çalışan fonksiyon. `process.send(...)` ise ana işleme mesaj göndermek için kullandığım fonksiyon. Worker tarafı tamamdır. Bu *worker.js* dosyasını `child_process.fork` ile çalıştırdığım zaman, hamle önerisi hesaplama için iş göndermeye başlayabilirim. Ana işlemimden bağımsız şekilde çalışmaya başlayacak. Örnek projede *worker.js* dosyasına [buradan](https://github.com/Yengas/stockfish-cluster-example/blob/ec3258455a3c276b1a460232e8aab97d7c55a6d6/child_process/worker.js) erişilebilir.

Şimdi işlem göndermek ile yükümlü olan koduma geliyorum. *master.js* dosyasında workerlara iş dağıtmam lazım. Worker işlem başlatma kodunu şimdilik dışarıda tutuyorum. *master.js* dosyasına Workerların başka yerden geldiğini kabul ederek kodluyorum.

```js
const uuidv4 = require('uuid/v4');

const workWaitingMap = new Map();
const workers = [];

module.exports = {
	addWorker(worker) {
		workers.push(worker);
		
		// worker'dan cevap geldiği zaman, bekleyen fonksiyonları çağırıyorum
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

- *addWorker* fonksiyonu ile alt işlem referansını alıyorum. Bir dizi'de ileride kullanmak için saklıyorum ve gelen mesajlardan cevap çıkarıp, bu cevap için beklemekte olan fonksiyonları çağırıyorum.
- *addWork* fonksiyonuna verilen parametreler ile bir iş oluşturuyorum. İş oluştururken, random bir id atıyorum. Böylece Workerlardan cevap geldiği zaman, hangi işimin cevabı olduğunu takip edebileceğim(correlation id). Daha sonra da random bir Worker'a, bu işi gönderiyorum ve işin id'sini dönüyorum.
- *waitWorkReply* ise verilen id'li işin cevabını bekleyen bir promise döndürmek için kullanılıyor. Böylece iş gönderdikten sonra, cevabını beklemek için yardımcı bir fonksiyonum olmuş oldu.

Artık `const workId = master.addWork({ fen: xxx, level: y, depth: z })` diyerek bir iş oluşturabilirim ve cevabını almak için de `const bestMove = await master.waitWorkReply(workId)` diyebilirim. Bunu tek fonksiyona da indirgeyebilirim.

```js
function getBestMove(fen, level, depth) {
	const workId = master.addWork({ fen, level, depth });
	
	return master.waitWorkReply(workId);
}
```

Şimdi yapmam gereken, *master.js*'e ekleyeceğim Workerları oluşturmak. Birden fazla alt işlem olarak çalıştırmam lazım. Bunu uygulamamın başlangıcında yapabilirim. Kod şu şekilde: 

```js
const child_process = require('child_process');
const clusterMaster = require('../child_process/master');
const NUM_OF_WORKERS = 4;


for (let i = 0; i < NUM_OF_WORKERS; i++) {
	const worker = child_process.fork(require.resolve('../child_process/worker'));

	clusterMaster.addWorker(worker);
}
```

artık *clusterMaster*'ı kullanarak hamle hesaplaması yapabilirim. Bu kodların çalışan hali [master.js](https://github.com/Yengas/stockfish-cluster-example/blob/ec3258455a3c276b1a460232e8aab97d7c55a6d6/child_process/master.js), [child_process/index.js](https://github.com/Yengas/stockfish-cluster-example/blob/ec3258455a3c276b1a460232e8aab97d7c55a6d6/child_process/master.js) ve [entrypoints/child_process.js](https://github.com/Yengas/stockfish-cluster-example/blob/ec3258455a3c276b1a460232e8aab97d7c55a6d6/entrypoints/child_process.js) dosyalarında, örnek projede görülebilir. `BENCHMARK_ANALYSIS_COUNT=10000 npm run child_process:benchmark` komutu çalıştırılarak test edilebilir. Bilgisayarımda eş zamanlı 10.000 hamlenin 20 saniyede hesaplandığını görüyorum.

Performans artışı sağladım ve artık ana işlemimim Stockfish motoru ile kitlenmiyor. Bu kadar mühendislik yeterli olabilir. Ama bir kaç noktayı daha düşünüyorum.

1. Her ana işlem (Selim'in durumunda WebSocket sunucusu) kendisi ile birlikte N tane Stockfish motoru başlatıyor. Ama aynı makinede, birden fazla process olarak çalışıyorlar. Hala birbirlerinin çalışmasını etkileyebilirler.
2. WebSocket sunucumu, Stockfish motorlarımdan bağımsız ölçekleyemiyorum. İkinci bir makinede Stockfish motoru çalıştırmak istersem, WebSocket sunucumu da orada çalıştırmam gerekiyor.

Bunlar bir sorun olmayabilir. 4 çekirdekli bir sunucuda, 100 kullanıcıya hizmet verecek bir şeyler yapıyor olabilirim. Bu kadar mühendislik benim için yeterli olabilir. Ama uygulamamı yayınladığım ortam, kurulumum veya kullanıcı sayım böyle bir çözüm ile tatmin olmamama sebep olabilir. O yüzden bir strateji daha inceliyeceğim.

### Bağımsız İşlemler ile Birden Fazla Stockfish Motoru
[child_process](https://nodejs.org/api/child_process.html) kullanarak, ana işlemim ile aynı sunucuda, bağımlı işlemler başlatabiliyorum. Ama Stockfish motorlarını benim işlemimden tamamen bağımsız yapmak istersem ne ne yapacağım? 100 kullanıcım varken 3 motor, 1000 kullanıcım varken 10 motor çalıştırmak istiyorsam? Stockfish motorlarının donanım gereksinimleri farklı ise ve onları özel makinelerde çalıştırmak istiyorsam? Bunları sağlamak için, araya ağ katmanı koymam lazım. Örneğin Stockfish motorunu bir REST API haline getirebilirim veya herhangi bir queue teknolojisini, bu amaç için kullanabilirim(örn. [RabbitMQ](https://www.rabbitmq.com), [Redis](https://redis.io/topics/pubsub)). Bunlar seçeneklerimden sadece birkaçı.

#### REST API ile
Stockfish motorunu ayrı bir işleme çıkartıp, bu işlemde REST API sunucusu ayağa kaldırabilirim. Böylece ana işlemim Stockfish motoru çalıştıran sunuculardan birine REST isteği gönderebilir ve en iyi hamle önerisini alabilir. Burada ana işlemimin, birden fazla Stockfish REST API çalıştırdığım zaman, hangisine istek atacağını nasıl bulacağı gibi bir sorun ortaya çıkıyor(Service Discovery). Bu sorunu deployment ortamına göre çözebilirim. *Kubernetes* tarafında Service veya *nginx* ile reverse proxy kullanmak gerekebilir. Şimdilik bunu unutalım. 

Bu şekilde ilerlemeyi seçersem, [server.js](https://github.com/Yengas/stockfish-cluster-example/blob/260165f4bfac62b52858116b624417f7482034d9/rest/server.js) kodum:
```js
const fastify = require('fastify')();
const { getBestMove } = require('../stockfish')();

let workChain = Promise.resolve();

fastify.get('/chess', (request) => {
	const { fen, level, depth } = request.query;

	return workChain = workChain.then(() => getBestMove(fen, level, depth));
});

fastify.listen(process.argv[2]);
```

Ana [işlemimden](https://github.com/Yengas/stockfish-cluster-example/blob/260165f4bfac62b52858116b624417f7482034d9/rest/index.js) bu sunucuya bir çağrı:

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

REST API sunucusundan istediğim kadar, istediğim sayıda makinede çalıştırabilirim. Tek yapmam gereken ana işlemime girdiğim `localhost:8080` url'sini, load balancer URL'i ile değiştirmek. Load balancer çalışmakta olan REST API sunucularına yükü dağıtmakla yükümlü olucak. Demo'da load balancer yerine, N tane sunucu farklı portlarda başlatılıyor ve istek atılırken bu sunuculardan birinin portu rastgele seçiliyor.

`BENCHMARK_ANALYSIS_COUNT=10000 npm run rest:benchmark` komutu çalıştırılarak bu test edebilir. Benim makinemde 10.000 istekte, REST API sunucuları timeout vermeye başlıyor ve benchmark fail oluyor. 5.000 istekte 12 saniyede cevap alabiliyorum. Bunun sebebi REST API sunucularının tüm istekleri bir kerede üstüne alması ve sıralı işlem yaparken bu isteklerin timeout alması. Bu sorunun *fastify* mı yoksa *axios* tarafından mı kaynaklandığını çözemedim. Pratikte her sunucunun eş zamanlı 1.000 istek almayacağını varsaydığım için, bu yöntemi sıkıntılı olarak düşünmüyorum. Güzel bir kurulum ve daha iyi kod ile, sorun olmayacaktır.

#### Queue Teknolojisi ile
REST API'ye alternatif olarak, Queue teknolojisi de kullanabilirim. Eğer kurulum ortamımda REST API ile yük dağıtımı yapmak zor olacaksa, bu iş için Queue'da kullanabilirim. Queue teknolojilerinin getirdiği [kolaylıklar](https://www.rabbitmq.com/features.html) ve [soyutlamalar](https://www.rabbitmq.com/tutorials/amqp-concepts.html) da işime yarayabilir. Küçük bir araştırma sonucunda, Redis kullanan ve ihtiyacım olan her şeyi basit bir API ile sağlayan [bee-queue](https://github.com/bee-queue/bee-queue) adında bir kütüphane buldum.

Bu kütüphane ile [worker.js](https://github.com/Yengas/stockfish-cluster-example/blob/ec3258455a3c276b1a460232e8aab97d7c55a6d6/queue/worker.js) kodu:
```js
const { getBestMove } = require('../stockfish')();
const queue = require('./createQueue')();

// .process fonksiyonu default olarak, aynı anda max 1 tane işlem çalıştırıyor.
queue.process(({ data: { fen, level, depth }}) => (
	getBestMove(fen, level, depth)
));
```

[master.js](https://github.com/Yengas/stockfish-cluster-example/blob/ec3258455a3c276b1a460232e8aab97d7c55a6d6/queue/index.js) kodu:

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

[createQueue.js](https://github.com/Yengas/stockfish-cluster-example/blob/ec3258455a3c276b1a460232e8aab97d7c55a6d6/queue/createQueue.js) kodu:
```js
const Queue = require('bee-queue');

module.exports = () => {
	const { ip: host, port } = require('./config.json');

	return new Queue('chess', { redis: { host, port } });
};
```

#### Sonuç
REST API veya Queue farketmez, tek yapmam gereken, bu Master/Worker işlemleri birbirinden bağımsız şekilde çalıştırmak. Bunu [pm2](http://pm2.keymetrics.io/) kullanarak da yapabilirim, [ECS](https://aws.amazon.com/ecs/) kullanarak da, [Kubernetes](https://kubernetes.io) kullanarak da. Örneğin Kubernetes üzerinde WebSocket sunucusunu ve Stockfish workerlarını farklı deployment yapabilirim. Stockfish CPU kullanımı artınca otomatik ölçeklenmesini söyleyebilirim. Böylelikle kullanıcı sayısı arttıkça, WebSocket sunucumdan bağımsız olarak, Stockfish motorlarım ölçeklenebilir. Kod tarafı bu basitlikte iken, nasıl bir kurulum ortamım olduğuna göre, istediğim gibi ölçeklenebilirim.

Aynı zamanda araya Node.js'e spesifik bir iletişim protokolü (*child_process* IPC) değil de, ağ katmanı koyduğum için, Stockfish motorunu istediğim dilde çalıştırmak da oldukça kolay olacak. İstersem C++ ile bir proje yazarım ve ağ üzerinden erişirim, istersem Rust ile. Böylelikle belki tek bir motoru birden fazla CPU kullanacak hale getirebilirim? Kısaca faydalarından ve dezavantajlarından bahsetmek gerekirse;

1. Ölçeklenmemi limitleyecek şey Queue teknolojim veya Load Balancer haline geldi. İstediğim kadar makinede, istediğim kadar Worker çalıştırabilirim. Queue/Load Balancer teknolojim, mesaj sayısını kaldırmayana kadar. Bu çoğu proje için ulaşılamayacak bir limit.
2. Anlık yüke göre Stockfish motoru sayımı değiştirebilirim.
3. WebSocket sunucusu ve Stockfish motorunu tamamen farklı makinelerde çalıştırabilirim.
4. İstersem Node.js modülünü kullanmaya devam edebilirim, veya farklı bir dilde Stockfish motorunu kullanarak işlem yapabilirim.

Docker yüklü bilgisayarlarda `npm run queue:benchmark` çalıştırarak, bu queue stratejisi denenebilir. REST API stratejisi için ise `npm run rest:benchmark` çalıştırılabilir.

Bu maddelere bağımlı olarak, deployment sürecim kompleksleşti. Queue durumunda ise yeni bir veritabanı bağımlılığım oldu. Bu dezavantajları kabullenip, böyle bir yapı kurmak, bazen bir seçenek olabilir. Ama bazen de zorunluluk haline gelebilir. Bu benim kişisel projem olsaydı, ben ne yapardım sorusuna aşağıda cevap veriyorum.

## Nasıl bir mimari yaparım?
Her zaman en iyi çözümü uygulamak, harcadığımız efora değmeyebilir. O yüzden eğer kişisel projem olsaydı, çözümleri şu sırada denerdim:

1. Yükümün az olduğunu varsayıyorum. WebSocket projesi içinde Stockfish modülünü kullanırdım. [child_process](https://nodejs.org/api/child_process.html) veya [worker_threads](https://nodejs.org/api/worker_threads.html) kullanarak Stockfish modülünü çalıştırırdım. Ama mesajlaşma yönetimini kendim yapmak yerine, *bee-queue*'yu redis olmadan *worker_threads*/*cluster* ile çalıştırmayı denerdim.
2. Yüküm arttı veya Stockfish JS'in düzgün çalışmadığını farkettim. Araya REST API koyardım. Bunu yapınca, benim için Stockfish'i Node.js ile çalıştırmanın bir anlamı kalmıyor. Bir Docker imajı oluşturup, içine C++ Stockfish motorunu koyardım. Node.JS veya C++ ile bu motor ile konuşup, REST API sunan bir kod yazardım. Kubernetes ile çalıştırarak load balancing ve auto scaling sorunlarını çözerdim. Demo'da eş zamanlı 10.000 istek sıkıntı yaratsa da, deployment sırasında güzel bir kurulum ve daha iyi kod ile sorunun çözülebileceğini düşünüyorum.

Eğer ağ üzerinden gönderilen veriler ve gelen cevaplar büyük olsaydı(fen, level, depth çok küçük), o zaman REST API yerine GRPC deneyebilirdim? Veya yeniden deneme stratejileri, işlerin Worker üzerinde birikmesi gibi sorunlar olsaydı Queue koymayı deneyebilirdim. Ne olursa olsun ilk önce basit başlar, ondan sonra yaşadığım sorunlara göre çözümler ve yeni bir mimari üretirdim.

## Örnek Proje
Bahsettiğim stratejilerin hepsini [örnek proje'de](https://github.com/Yengas/stockfish-cluster-example/tree/ec3258455a3c276b1a460232e8aab97d7c55a6d6) inceleyebilirsiniz. `npm run child_process:benchmark` *child_process* stratejisine 1.000 eş zamanlı hamle öneri isteği gönderiyor ve ne kadar sürdüğünü ekrana bastırıyor. 

Eğer `npm run child_process` komutunu çalıştırırsanız, *child_process* stratejisini kullanan bir WebSocket sunucusu başlatıyor. Bu WebSocket sunucusuna `npm run client` diyerek bağlanıp, *FEN* gönderebilirsiniz. Size o tahta durumu için, en iyi olduğunu düşündüğü hamleyi söylecek.
