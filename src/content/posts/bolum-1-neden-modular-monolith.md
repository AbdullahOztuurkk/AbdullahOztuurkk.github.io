---
title:  "📚 Bölüm 1 — Neden Modular Monolith?"
date:   2025-10-21
image: "/images/posts/bolum-1-neden-modular-monolith/index.png"
categories: [".Net Core"]
tags: ["Modular Monolith",".Net Core","Modulith"]
---

Herkese merhaba,

Hepimiz ilk yazılıma başladığımız zamanlarda her bir yapı taşını içeren tek bir proje geliştirmişizdir. Ancak zamanla değişen istekler doğrultusunda küçük bir değişiklik bile bizi tonlarca efor sarfetmeye zorlamıştır. Bunun önüne geçebilmek adına **katmanlı mimariye** geçtik. Peki katmanlı mimariyi neden kullandık, bize faydası neydi?

Katmanlı mimariyi kullanarak Clean Architecture (Onion, hexagonal architecture vb.) yapısını ve bunu efektif olarak nasıl kullanabileceğimizi öğrendik. Bu sayede veri erişim katmanımız ayrı, iş katmanımız ayrı, endpointlerimizin olduğu sunum katmanımız ayrı, veritabanındaki tabloları ifade eden Entity katmanımız ayrı hale geldi. Bu sayede ufak bir değişikliği çok daha az bir eforla yapabilmeye başladık.

Ancak zaman geçti, uzun süredir kullandığımız katmanlı mimari yetmemeye başladı. Bazı işleri diğer programlama dilleri ile yapmamız gerekti yada sistemimizde kullanılan bir endpoint diğer endpointlere kıyasla daha fazla trafik almaya başladı. Okuma / Yazma performansına göre sadece belli bir akış bazında farklı bir veritabanı seçilmesi gerekti. Test yapmak istediğimizde ise tam bir çileydi. Bu durum karşısında yıllardır sözü edilen **mikroservis mimarisine** geçtik. Peki biz neden bu mimariye geçtik? Bize faydası olacaktı elbette ama zararı da olabilir miydi?

> Mikroservis mimarisi, her bir servisin sadece tek bir amacının olduğu, ayrı bir veritabanı olan ve her bir servisin ayrı deployment süreçleri olan bir mimari türü.

Mikroservis mimarisine geçildi. BuildingBlocks dediğimiz yapıtaşlarını birer nuget package haline getirip tüm mikroservislerimizde birer standart haline getirildi. Her servis sistemdeki kendi rolüne bakarak en uygun olarak veritabanı ile ayağa kaldırıldı. Sistemin daima ayakta kalkması için birtakım önlemler alındı.(Retry-Fallpack Policies & Circuit Breaker).
Servislerin kendi aralarında event bazlı iletişimi için bir mesaj kuyruk sistemi kullanıldı. Senkron iletişimi için de internal api call yada grpc gibi bir takım altyapı oluşturuldu.

Ancak zamanla mikroservis mimarisinin bir takım problemleri meydana çıktı. Her bir servisin ayrı bir deployment süreci, ayrı bir bakımı gerekiyordu. Dağıtık sistemlerde debug yapabilmek zorlaşmış ve sistemin bir ucunda yapılan değişikliği diğer servislerin de algılaması için güncellenmesi gerekmesi işleri yokuşa sürmüştü. Günümüz gereği eğer mikroservise geçilmesi kaçınılmazsa bunların yapılması gerekiyor. Buna yapabileceğimiz birşey yok.

> Peki ya size projenizi hem düzenli, hem bakımı kolay, hem de gelecekteki büyümelere hazır hale getirecek üçüncü bir yol olduğunu söylesem?

**Modular Monolith** felsefesinden bahsediyorum.

**Modular Monolith** felsefesi, aslında bu iki projenin dengesidir. Uygulama tek bir proje olarak deploy edilmesine karşın, her bir domain modüllere ayrılmıştır. Mikroservis mimarisindeki her bir modül sadece kendi modülü ile ilgili kısımlardan sorumludur. Diğer modüller ile iletişim genelde contracts üzerinden kurulur, doğrudan modül iletişimi söz konusu değildir.
Bu sayede bağımlılıklar azalır ve daha clean bir yapı ortaya çıkar.

Özetle;

**Monolith Mimarisi:**
Aslında en klasik yapı diyebiliriz. Tüm uygulama tek bir **codebase** üzerinde yaşar, her şey birbirine **tightly coupled** durumdadır. Başlangıçta geliştirmesi kolay, deploy etmesi basittir. Ama sistem büyümeye başladıkça işler karışır; küçük bir değişiklik bile başka yerleri etkiler, **deployment** süreci riskli hale gelir, bakımı da gittikçe zorlaşır.

**Microservice Mimarisi:**
Burada işler tamamen farklı. Her servis kendi **codebase’ine**, kendi **database’ine** ve hatta kendi **deployment pipeline’ına** sahiptir. Yani her şey **loosely coupled** bir şekilde çalışır. Bu da ekiplerin bağımsız çalışmasını ve servislerin ayrı ayrı ölçeklenmesini sağlar. Fakat madalyonun diğer yüzü de var — sistem artık dağıtık hale geldiği için **communication, monitoring, logging, deployment orchestration** gibi konular ciddi bir efor ister.

**Modular Monolith Mimarisi:**
Bahsettiğimiz mimari biraz ortadaki bir noktada duruyor — iki dünyanın dengesini kurmaya çalışıyor diyebiliriz. Uygulama hâlâ tek bir **codebase’e** sahip ancak içeride bağımsız, sınırları net çizilmiş modüller var. Yani bir anlamda monolith’in sadeliğini korurken, microservice’in **modüler yapısını** ve **separation of concerns (Her katmanın veya bileşenin kendi işine odaklanması)** avantajlarını da elde ediyoruz. Bu sayede hem geliştirmesi kolay hem de gelecekte microservice’e evrilebilecek kadar esnek bir yapı ortaya çıkarmış oluyoruz.

---

Bu yazıda monolith ve mikroservis mimarilerinin sorunlarını ele alıp modular monolith felsefesinin nasıl çıktığını, ne tür sorunların önüne geçtiği ile alakalı bilgilendirmeyi sizlere sunuyorum. Bir seri olarak bu konu ile alakalı yazılar paylaşmaya devam etmeyi düşünüyorum.

Okuduğunuz için teşekkür ederim. Faydalı bulduysanız beğenilerinizi eksik görmeyin. Bir sonraki yazıda görüşmek dileğiyle.