---
title:  "Swagger Nedir ? .NET Core Swagger Entegrasyonu"
date:   2020-10-7
tags:
- .Net Core
- Swagger
---
Merhabalar, bugün restful api üzerinde kullanacağımız swagger nedir, ne amaçla kullanılır ve nasıl kullanılır gibi soruların cevaplarını bu makalemde cevaplayacağım. Her şeyden önce API nedir ve neden ihtiyaç duyarız, bu soruları kendimize sormamız lazım.

## Application Programming Interface (API) Nedir?

Bir uygulamaya ait işlevlerin ve üretilen, depolanan verilerin başka tür uygulamalarda da kullanılması için geliştirilen yapı olarak özetleyebiliriz. Bir örnek verecek olursak web üzerinde iletişim programı yaptınız ve aynı şekilde kullandığınız veritabanindaki verileri mobilde de kullanmak istiyorsunuz. İşte burada API araya giriyor ve bizi büyük bir yükten kurtarıyor. Yalnız hem uygulamaların birbiri ile hem de server ile sağlıklı bir iletişim kurabilmesi için aynı dili konuşması gerektiğini unutmayalım. Burada sıklıkla REST ve SOAP mimarisi kullanılır.

<!-- more -->

## Swagger Nedir?

Web API geliştirirken en önemli ihtiyaç dökümantasyon ihtiyacıdır. API içerisindeki metodların ne iş yaptığı, ne için kullanıldığı, hangi parametreleri aldığı gibi birçok bilgi dökümantasyonda belirtilir. Dökümantasyonların elle yazılması güncel tutulma açısından zor ve oldukça vakit alıcı bir durum olacağı için geliştiricilere kullanıcı arayüzü sunan, hangi metodun ne işe yaradığını ve hangi parametrelerle gönderileceği gibi birçok bilgiyi aktarmamızı sağlayan Swagger teknolojisini kullanırız.

## Swagger Nasıl Kullanılır ?

Öncelikle bir solution project oluşturalım ve gerekli olan Business, DataAccess, Entities, UI ve son olarak en çok uğraşacağımız Web API katmanlarını ekleyelim. API katmanı dışındaki katmanları yazdığımızı varsayalım. API katmanı üzerine sağ tıklayalım ve Manage Nuget Packages seçeneğine tıklayalım.

![Nuget Swagger Paketi](/img/application-programming-interface-nedir/nuget-nswag.png)

NSwag.AspNetCore adlı kütüphanemizi indiriyoruz. Kullanıcı arayüzüne gitmeden önce startup.cs dosyasında birkaç ayar yapmamız gerekiyor.

```csharp
public class Startup
    {
        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        // This method gets called by the runtime. Use this method to add services to the container.
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddControllers();

            services.AddSwaggerDocument(); // Çeşitli ayarları yapabilmek için
        }

        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseHttpsRedirection();

            app.UseRouting();

            app.UseOpenApi(); // kullanabilmesi için
             
            app.UseSwaggerUi3(); // Görünüm ekranı için

            app.UseAuthorization();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllers();
            });
        }
}
```
Ayarlarımızı yaptık. Şimdi projemizi API üzerinden çalıştıralım ve localhost:PORT/swagger linkine gidelim. Karşımıza aşağıdaki gibi bir ekran gelecek. Şimdi bu ekrandakilerin neler olduğunu anlamaya çalışalım.

![Swagger Arayüzü](/img/application-programming-interface-nedir/swagger-ui.png)

Sayfamızın başında başlık, versiyon numarası gibi birkaç metin görebiliyoruz. Burayı makalenin sonlarına doğru beraber düzenleyeceğiz. Altında, API’de sahip olduğumuz controller isimlerini görüyoruz. Bizde şuanlık sadece Books controller var. İçerisinde de tanımlanmış metodları ve HTTP protokollerini görüyoruz lakin açıklaması olmadığını farkediyoruz. Altında Models kısmını görüyoruz.Burada API içerisinde kullanılan model sınıflarını görebiliyoruz . Model içerisinde de hangi kolonların hangi türde olduğunu da görebiliyoruz . Tabi normalde bu kadar az içerik görünmez, ben kısa ve öz olması açısından sadece BookController ile çalıştım ama siz daha fazla içerik ile uğraşabilirsiniz.

Alttaki fotoğrafta sol kısım henüz açıklama eklemediğimiz, sağ kısım ise açıklama eklememizden sonraki hali.

![Books Controller](/img/application-programming-interface-nedir/api-controllers.png)

Ancak tekrar aynı URL’e bağlanmaya çalıştığımızda açıklamamız görünmeyecektir. Bunu engellemek için Web API katmanı üzerinden Properties’e basıp Build sekmesine tıklayın. XML documentation file seçeneğine tik atın. (Fotoğraf aşağıda)

![Uygulama Ayarları](/img/application-programming-interface-nedir/app-settings.png)

Çalıştırmadan önce yukarıda da bahsettiğim gibi başlık versiyon numarası vb. kısımları düzenleyelim. Startup.cs dosyasında ConfigureServices metodunu tekrardan düzenledik.

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers();

    services.AddSwaggerDocument(config=>
        config.PostProcess=(settings =>
        {
            settings.Info.Title = "Swagger Nedir ve Nasıl Kullanılır";
            settings.Info.Contact = new NSwag.OpenApiContact
            {
                Email = "abdullah@ozturk.com",
                Name = "Abdullah Öztürk",
                Url = "abdullahozturk.github.io"
            };
            settings.Info.Version = "1.8.9";
        })); // Çeşitli ayarları yapabilmek için
}
```
Bu kısmı kendi dökümasyonundan da bakabilirsiniz . Birçok ayar bulunmakta. Artık tüm ayarları yapmış bulunuyoruz. Gelin beraber sonucu görelim.

![Swagger Arayüzünün Son Hali](/img/application-programming-interface-nedir/swagger-ui-last.png)

Swagger hakkında daha fazla bilgi edinmek için kendi dökümantasyonunu okuyabilirsiniz.