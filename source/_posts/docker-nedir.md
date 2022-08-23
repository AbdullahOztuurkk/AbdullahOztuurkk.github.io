---
title:  "Docker Nedir?"
cover: /img/docker-nedir/index.jpg
date:   2022-06-19
tags:
- Devops
- Docker
---

Merhaba Arkadaşlar, uzun bir aradan sonra tekrardan sizinleyim. Bugün yazılım dünyasında oldukça önemli bir rol oynayan “**Docker**” nedir, nasıl kullanılır, sanal makinelerden farkı nedir gibi sorulara değiniyor olmakla beraber “**Docker**” denilince akla gelen kavramları da sizlere anlatıyor olacağım.

**Docker**, uygulamalarınızı hızla derlemenize, test etmenize ve dağıtmanıza imkan tanıyan bir yazılım platformudur. **Docker**, yazılımların düzgün çalışması için gerekli olan kütüphane ve diğer gerekli her şeyi içeren “**container**” teknolojisini kullanır. İlk sürümü 2013 yılında yayınlanmış olup tamamen açık kaynak bir projedir.

> “*Sanal Makinelerde de konteyner oluşturabiliyorduk. O zaman ikisi de aynı işlevi görüyor.”* şekilde düşünüyor olabilirsiniz.

<!-- more -->

| ![][docker-vs-virtual-machine] | 
|:--:| 
| Docker ve Sanal Makine karşılaştırması |

>> Sanal Makine üzerinde çalışan işletim sistemi, altta çalışan Hypervisor’un varlığından haberdar değildir.

**Cevap:** Hayır, sanal makineler konteyner oluşturabilmeyi sağlamasına rağmen uygulamaların çalışabilmesi için bir işletim sistemini zorunlu hale getiriyor. Maliyet açısından büyük bir problemle karşı karşıya geliyoruz. Docker ise var olan işletim sistemi üzerinde işlem izolasyonu sağlıyor.

Aşağıda konteyner mimarisi ile sanal makine mimarisi karşılaştırmasını görebilirsiniz.

| ![][container-arch-vs-virtual-arch] | 
|:--:| 
| Container Mimarisi ile Sanal Mimari karşılaştırması |

<hr>

Docker bileşenlerine göz atalım ve teker teker bu bileşenler ne yapıyor, açıklayalım.

**Docker Client:** Docker’ı yapılandırmak için kullanılır. Asıl görevi kullanıcının Docker Daemon ile etkileşime girmesini sağlamaktır.

**Docker Daemon:** Tüm Docker işlemlerini bu servis gerçekleştirmekte olup client üzerinden gelen api isteklerini dinlemekte ve yönetmektedir.

| ![][docker-components] | 
|:--:| 
| Docker Bileşenleri |

Bu zamana kadar Docker’ın sanal makine ile karşılaştırılması ve kendi bileşenlerinden bahsettik. Gelin, Docker kullanırken en çok rastlayacağımız kavramları açıklayalım.

## Docker Image

Docker Image, birden çok katmandan oluşan ve Docker konteynerda işlem yapmak için oluşturulan dosyalardır. Özelliklerine geçelim.

- Her image dosyası uygulama kod, runtime, ortam değişkenleri, proje dosyaları ve bağımlılıkları içerir.
- Dockerfile kullanılarak oluşturulur. (Birazdan değineceğiz)
- Konteyner silinmeden image silinemez. (Konteyner oluştururken image kullanıyoruz)
- Birden çok katmanı bulunur ve her katman önceki katmanın uzantısı durumundadır.

## Docker Container
Konteyner, bir uygulamanın ihtiyaç duyulan tüm bağımlılıklar, kütüphaneler ve diğer tüm nesnelerle birlikte paketlenerek standart hale getirilmesidir. Özelliklerine geçelim.

- Docker Image’ın çalışan örneğidir.
- Bir Image’in birden fazla konteynerı oluşturulabilir.
- Uygulamayı çalıştırmak için gereken tüm paketi konteynerler bünyesinde tutmaktadırlar.

## Docker Volume
Konteyner içerisinde kalıcı veri depolamak için kullanılır.

- Konteynerler silinse dahi Docker Volume içerisindeki veriyi tutmaya devam eder.
- Docker Volume içerisindeki veriler diğer konteynerler içerisinde paylaştırılabilir ve kullanılabilir.
- Konteyner boyutunu arttırmaz.
- Docker Volume içerisindeki değişiklikler doğrudan yapılabilir.

## Dockerfile
Belli bir image oluşturmak için var olan tüm katmanların madde madde yazıldığı ve tüm işlemlerin detaylıca belirtildiği text dosyalarıdır.

- Uzantısı yoktur.
- İnsanların kolaylıkla okuyabileceği “YAML” formatı kullanılır.

Örnek olarak bir Dockerfile inceleyelim ve satır satır ne yapılmak istendiğini açıklayalım.

```docker
FROM python:2.7
RUN mkdir -p /app
WORKDIR /app
COPY ./requirements.txt /app/
RUN pip install -r requirements.txt
COPY . /app
CMD["python","main.py"]
```

- FROM komutu ile python’ın 2.7 sürümünü kullanacağımızı belirtiyoruz. Lokalimizde python 2.7 sürümünü bulmaya çalışacak eğer yoksa hub.docker.com üzerinden indirme yapacak.
- RUN komutu app adında bir klasör oluşturacağımızı belirtiyoruz
- WORKDIR komutu ile app klasörünü çalışma dizinimiz olarak belirtiyoruz.
- COPY komutu ile Dockerfile’ın olduğu klasör içerisindeki requirements.txt dosyasına mevcut app klasörüne yani çalışma dizinimize kopyalıyoruz.
- RUN komutu ile python paket yöneticisini (pip) kullanarak requirements.txt dosyası içerisindeki bağımlılıkları indiriyoruz.
- COPY komutu ile yüklenmiş bağımlılıklarla beraber tüm dosyaları app klasörüne kopyalıyoruz.
- CMD komutu ile python uygulamasını kullanacağımızı ve parametre olarak da main.py dosyasını belirterek uygulamamızı çalıştırıyoruz.

Başka bir Dockerfile daha inceledikten sonra Docker Compose kavramına geçelim ve yazımızı sonlandıralım.

```docker
FROM ubuntu:18.04
RUN apt-get update && apt-get install -y nginx
WORKDIR var/www/html
COPY . .
EXPOSE 80/tcp
CMD ["nginx","-g daemon off;"]
```

- FROM komutu ile Ubuntu’nun 18.04 sürümünü kullanacağımızı belirtiyoruz. Lokalimizde Ubuntu’nun 18.04 sürümünü bulmaya çalışacak eğer yoksa hub.docker.com üzerinden indirme yapacak.
- RUN komutu ile gerekli kütüphanelerin güncellenmesi yapılıp nginx paketinin indirilmesini sağlıyoruz.
- WORKDIR komutu ile nginx’in varsayılan çalışma dizini olan var/www/html yolunu çalışma dizinimiz yapıyoruz.
- COPY komutu ile nginx tarafında ne çalıştıracak isek onları çalışma dizinimize kopyalıyoruz.
- EXPOSE komutu ile TCP protokolünü seçip 80 portunda çalıştıracağımızı belirtiyoruz.
- CMD komutu ile çalıştırılacak uygulamayı nginx olarak belirtip parametre olarak “-g daemon off” göndererek çalıştırıyoruz.

## Docker Compose
Docker Compose, birden çok container tek bir YAML dosyası üzerinden yönetmemizi sağlayan orkestrasyon aracıdır. Bu ayar dosyası “docker-compose.yml” olarak adlandırılır ve aşağıdakine benzer bir yapısı vardır.
```yaml
version: "3.8"
services: 
    identity.api:
        image: mssql
        build: 
            context: .
            dockerfile: src/your-project/services/identity-api/Dockerfile
    catalog.api: 
        image: postgres
        build: 
            context: .
            dockerfile: src/your-project/services/catalog-api/Dockerfile
    order.api: 
        image: mssql
        build: 
            context: .
            dockerfile: src/your-project/services/order-api/Dockerfile
    rabbitmq: 
        image: rabbitmq:3.8.14-management
    rabbitmqmass: 
        image: masstransit/rabbitmq
    gateway.api:
        image: gateway_api_image
        build: 
            context: .
            dockerfile: src/your-project/Gateway.WebApi/Dockerfile
    pgAdmin4:
        image: dpage/pgadmin4
```

Örnek bir E-Ticaret sitesi baz alınarak hazırlanmıştır.

> Okuduğunuz için teşekkür ederim. Bir sonraki yazıda görüşmek dileğiyle.

<!-- Links -->
[docker-vs-virtual-machine]: /img/docker-nedir/docker_vs_virtual_machine.png
[container-arch-vs-virtual-arch]: /img/docker-nedir/container_arch_vs_virtual_arch.png
[docker-components]: /img/docker-nedir/docker_components.png