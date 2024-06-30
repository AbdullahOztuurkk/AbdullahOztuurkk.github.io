---
title:  "Efektif Dockerfile Kullanımı"
date:   2022-06-28
image: "/images/posts/efektif-dockerfile-kullanimi/index.jpg"
categories: ["Devops"]
tags: ["Devops", "Docker"]
---

Merhaba Arkadaşlar, bugün **dockerfile** yazarken nelere dikkat etmeliyiz, hangi kurallara dikkat etmeliyiz ki kaliteli bir dockerfile ortaya çıkabiliriz, bundan bahsedeceğim. Asıl konumuza geçmeden önce bilgi edinme açısından Docker ile ilgili önceki yazıma [buradan][docker-article] ulaşabilirsiniz.

Eğer Devops ile ilgileniyorsanız ya da Docker ile elinizi kirletmek istiyorsanız etkili bir dockerfile dosyasının öneminin farkındasınızdır. **Dockerfile** belli bir image oluşturmak için var olan tüm katmanların madde madde yazıldığı ve tüm işlemlerin detaylıca belirtildiği text dosyalarıdır. Son işlem gerçekleştirildiğinde elinizde güncel Docker image olur. Bu dockerfile, işlemlerinizi otomatize eder ve yaptığınız tüm değişiklikleri takip etmenize yardımcı olur.

Bir docker konteynerinin performansı dockerfile içerisinde belirtmiş olduğumuz birçok işlem maddelerine bağlıdır. Bundan dolayıdır ki dockerfile içerisinde kullanacağımız komutlara oldukça önem göstermeliyiz. Bu yazımızda az kaynak tüketerek efektif bir şekilde çalışan ve güncel Docker Image oluştururken kullanacağımız tekniklerini konuşuyor olacağız.

### 1. Gereksiz Paket Kurulumundan Kurtulun!
Eğer dockerfile içerisinde gereksiz paket kurarsanız bu durum dockerfile boyutunu ve build zamanını arttırır. Ayrıca dockerfile üzerinde yapacağınız her değişiklik sonucunda tüm komutlar baştan çalışacak ve dockerfile dosyamız daha da büyük hale gelecektir. Bu durum, performansa kötü yansıyacaktır. Bunu engellemek için sadece ihtiyacımız olan paketler include edilir ve aynı paketlerin defalarca kurulmasının önüne geçilir.

Sadece ihtiyacımız olan paketlerin indirilmesi için bir dosya oluşturup kullanabiliriz. Bunun için aşağıdaki komutu kullanabilirsiniz.

    `RUN pip3 install -r requirements.txt`

### 2. Tüm RUN Komutlarını Aynı Anda Çalıştırın!
Dockerfile içerisinde belirttiğimiz her bir RUN komutu ayrı bir katman oluşturur ve bellekte yer kaplar. Birçok RUN komutunun olması durumunda Dockerfile boyutu oldukça büyüyecektir. Tek bir komut halinde tüm RUN komutlarını yazabilirsiniz. Aşağıda bu ikisi arasındaki farkı görebilirsiniz.

![][multiple-run-statement]

### 3) Mümkün Oldukça .dockerignore Dosyası Kullanın!
Git tarafında .gitignore kullandığımız gibi, burada da Docker build kapsamında hariç tutmak istediğimiz dosyalar ve klasörlerin .dockerignore dosyası sayesinde üstesinden gelebiliriz. Bu durum, istenmeyen dosyaların docker konteyner içerisinde olmasını engelleyeceği için daha küçük boyut ve daha hızlı bir deployment ortamı sağlar.

### 4) En İyi Komut Sıralamasını Kullanın!
Dockerfile dosyanızın sonuna en sık değişen komutları ekleyin. Bunun altındaki sebep ise dockerfile içerisinde yapacağınız herhangi bir değişiklik, önbelleğin geçersiz hale gelmesi ve sonraki tüm komutların önbelleğinin de bozulmasıdır. Örneğin, en üste RUN komutlarını ve en alta COPY komutlarını ekleyin. Docker dosyasının sonuna CMD, ENTRYPOINT komutlarını ekleyin.

### 5) Gereksiz Paket Bağımlılıkları Kurmaktan Kaçının!
Docker Image’i oluştururken -no-install-recommends parametresini kullanabilirsiniz. Bu parametre ile apt paket yöneticisine gereksiz bağımlılıkları kurmamasını belirtiyoruz. Gereksiz paket yönetimi sadece Image boyutunu ve build zamanını arttırır ve performans düşmesine sebep olur.

### 6) Mümkünse Tüm Image’lar için Alpine Linux Kullanın!
Alpine Linux, oldukça küçük boyutlu (yaklaşık 5MB) bir işletim sistemidir. Docker konteyner çalıştırmak için linux ortamına ihtiyaç olunduğu zamanlarda ve production’a çıkacak uygulamalar için alpine-linux kullanılabilir.

### 7) Çok Katmanlı Dockerfile Dosyaları Kullanın!
Multi-stage docker yapıları, bize birden çok FROM ifadesiyle Dockerfile dosyası yazmamıza izin verir. Her FROM komutu farklı bir base image kullanarak Image’inizin son halinin boyutunu küçültmenizi sağlar.

![][multistage-dockerfile]

Özet olarak, dockerfile yazarken özellikle dikkat etmemiz gereken kısımlara değinmeye çalıştım. Okuduğunuz için teşekkür ederim. Bir sonraki yazıda görüşmek dileğiyle.

<!-- Links -->
[docker-article]: ../docker-nedir
[multistage-dockerfile]: /images/posts/efektif-dockerfile-kullanimi/multi-stage-dockerfile.png
[multiple-run-statement]: /images/posts/efektif-dockerfile-kullanimi/multiple-run-statement.png
