---
title:  "📚 Bölüm 2 - Mimari Altyapı ve Proje Organizasyonu"
date:   2025-10-28
image: "/images/posts/bolum-2-mimari-altyapi-ve-proje-organizasyonu/index.jpg"
categories: ["Software"]
tags: ["Modular Monolith",".Net Core"]
---

Herkese merhaba,

Serinin 2.bölümünde Modüler Monolith mimarisinde önemli bir adım olan Bounded Context kavramlarının belirlenmesi ve en nihayetinde bir proje eşliğinde teknik kısma girmeyi planlıyorum. Bounded Context kavramından başlayalım.

**Bounded Context**, Domain Driven Design'ın (DDD) kalbi diyebileceğimiz, her bir kavramın sınırlarını net bir şekilde çizdiğimiz bir düşünce biçimi.
Her bir modül kendi bounded contextini oluşturur. Şayet bir modül, bir diğer modüle çok bağımlı halde ise bounded context kapsamını tekrardan değerlendirmek gerekir.

Serinin devam eden süreci boyunca Medium'u örnek alabileceğimiz bir proje geliştirmeyi planlıyorum. Burada temel birkaç modülümüz bulunacak.

- **UserModule**: Her bir kullanıcının sistem üzerindeki erişebileceği servisleri içerecek. Kullanıcı giriş yapabilecek, kendi url'i üzerinden bilgilerine erişebilecek, takip isteği atabilecek.

- **ResourceModule**: Sistem üzerindeki tüm konfigürasyonlar buradan tutulup daha dinamik bir yapı inşa edilecek. Hata mesajları, şifre kriterleri gibi konfigürasyonlar, Sıkça Sorulan Sorular ve Güvenlik soruları gibi birkaç alanda bilgiler tutulacak.

- **NotificationModule**: Her bir kullanıcıya ait bildirim ayarları kaydedilerek kullanıcının talep ettiği bildirim türü gönderilecek.
Şuanlık sadece Email gönderme şeklinde olsa da yeni bir notification türü kolaylıkla eklenebilir olacak.

- **InteractionModule**: Her bir kullanıcı bu modül üzerinden yazı oluşturabilecek, publication dediğimiz topluluk sayfaları oluşturabilecek ve bunlara katılabilecek. Ek olarak Trending ve Explore sayfaları da buradan beslenecek.

--- 

Proje içeriğinden de bahsettiğimize göre yavaştan proje yapısını oluşturmaya başlayalım.

```cs
src
├── Bootstrapper
│   └── Host.Web.Api
├── Infrastructure
│   └── BuildingBlocks
└── Modules
    ├── User
    │   ├── UserModule
    │   │   └── Host
    │   │       └── UserModuleHost.cs
    │   └── UserModule.Contracts
    ├── Interaction
    │   ├── InteractionModule
    │   │   └── Host
    │   │       └── InteractionModuleHost.cs
    │   └── InteractionModule.Contracts
    ├── Resource
    │   ├── ResourceModule
    │   │   └── Host
    │   │       └── ResourceModuleHost.cs
    │   └── ResourceModule.Contracts
    └── Notification
        ├── NotificationModule
        │   └── Host
        │       └── NotificationModuleHost.cs
        └── NotificationModule.Contracts
```

- **Bootstrapper** klasörü altında tek bir projemiz var. Bu bizim tüm modülleri yükleyip ayarlarını yapıp ayağa kaldıracağımız proje.

- **Infrastructure** klasörü altında tüm modüller tarafından kullanılabilecek ortak yapılar bulunacak. (AuditEntity, Module metotları vb.)

- **Modules** klasörü altında ise her bir modülün kendi klasörü olacak. Bu klasör içinde ise Modülün kendi projesi ve diğer modüllerin iletişim için kullanacakları sınıfları içeren **Contracts** projesi olacak.

---

Gelin, Infrastructure klasörü altındaki ilk projemiz olan BuildingBlocks projesini inşa edelim.

```cs
src
└── BuildingBlocks
    ├── Domain
    │   ├── Abstract
    │   │   ├── IConvertibleFrom.cs
    │   │   └── IEntity.cs
    │   ├── Concrete
    │   │   └── AuditEntity.cs
    │   ├── Constant
    │   │   └── ConfigurationKeys.cs
    │   └── Enums
    │       └── StatusType.cs
    └── Module
        ├── ModuleExtensions.cs
        └── ModuleLoader.cs
```

Her bir modüldeki entity sınıflarının türemesi için IEntity ve AuditEntity sınıflarımızı oluşturalım. Burada genel olarak Id alanını long olarak belirtiyorum. İsteyenler o kısmı değiştirip farklı bir identifier belirleyebilir.(Örneğin int yerine Guid kullanımı gibi)

```cs
public interface IEntity
{
    public long Id { get; set; }
    public StatusType Status { get; set; }
    public DateTime CreateDate { get; set; }
    public long CreateUserId { get; set; }
    public DateTime? UpdateDate { get; set; }
    public long? UpdateUserId { get; set; }
    public DateTime? DeleteDate { get; set; }
    public long? DeleteUserId { get; set; }
}

public class AuditEntity: IEntity
{
    public long Id { get; set; }
    public StatusType Status { get; set; } = StatusType.Active;
    public long CreateDate { get; set; }
    public long CreateUserId { get; set; }
    public DateTime? UpdateDate { get; set; }
    public long? UpdateUserId { get; set; }
    public DateTime? DeleteDate { get; set; }
    public long? DeleteUserId { get; set; }
}

public enum StatusType
{
    Passive = 0,
    Active = 1,
    Blocked = 2,
}

//Entity ve Response sınıfları arasında kolayca Map işlemi yapabilmek
//İçin bir helper interface (Tamamen Opsiyonel)
public interface IConvertibleFrom<in TSource, out TDestination>
    where TSource : new()
    where TDestination : new()
{
    TDestination Map(TSource entity);
}
```

**AuditEntity** sınıfımızı oluşturduk ve artık elimizde bir entity ne zaman oluşturuldu, kim tarafından oluşturuldu, kim tarafından değişikliğe uğranmış bilgisi var. **StatusType** enum yapısını da her bir kaydın güncel durumunu temsil etmek için kullanacağız. Soft-Delete işlemi yapacağımız zaman tekrardan bu alanı ele alacağız.

Şimdi ise modüllerin kullanacağı interfaceleri tanımlayalım. Özelleştirilebilir modül bazlı bir ortam oluşturabilmek istiyoruz. **ModuleExtensions** adlı sınıfı oluşturalım.

```cs
public interface IModule
{
    public string ModuleName { get; }
}

public interface IHaveService
{
    public IServiceCollection ConfigureServices(IServiceCollection services, IConfiguration configuration);
}

public interface IHaveEndpoint
{
    IMvcBuilder ConfigureEndpoints(IMvcBuilder mvcBuilder);
}

public interface IHaveMigration
{
    Task<WebApplication> MigrateDatabase(WebApplication app);
}

public interface IHaveSeeder
{
    Task SeedAsync(IServiceProvider serviceProvider);
}

public interface IHaveHealthCheck
{
    IHealthChecksBuilder CheckStatus(IHealthChecksBuilder builder, IConfiguration configurationManager);
}
```

- Her bir modül **IModule** interface'inden türemek zorundadır. Host uygulamamız ayağa kalkarken her bir modülde aradığı interface'dir.

- **IHaveService** interface'i, ilgili modüle dependency injection desteği sunar.

- **IHaveMigration** interface'i, uygulama ayağa kalkarken modülün kendi veritabanında bekleme durumunda bir migration işlemi varsa onu gerçekleştirir. 

- **IHaveSeeder** interface'i, kullanılan modülde başlangıç verilerinin eklenmesini sağlar.

- **IHaveHealthCheck** interface'i, her bir modüldeki tüm componentlerin(Db,WebHook vb.) health-check işlerinin izlenmesi ve monitörize edilmesini sağlar. 

Her bir modül için özelleştirilebilir interfacelerimizi de oluşturduğumuza göre bu modülleri Program.cs tarafında register ve load edecek **ModuleLoader** sınıfımızı yazmaya başlayalım.

```cs
namespace BuildingBlocks.Module;

public static class ModuleLoader
{
    public static WebApplicationBuilder RegisterModules(this WebApplicationBuilder builder, List<IModule> modules)
    {
        var mvcBuilder = builder.Services.AddControllers();

        var healthCheckBuilder = builder.Services.AddHealthChecks();

        var logger = LoggerFactory.Create(loggingBuilder =>
        {
            loggingBuilder.AddConsole();
        }).CreateLogger(nameof(ModuleLoader));

        foreach (var module in modules)
        {
            if (module is IHaveService serviceModule)
            {
                serviceModule.ConfigureServices(builder.Services, builder.Configuration);
            }

            if (module is IHaveEndpoint endpointModule)
            {
                endpointModule.ConfigureEndpoints(mvcBuilder);
            }

            if (module is IHaveHealthCheck healthCheckModule)
            {
                healthCheckBuilder = healthCheckModule.CheckStatus(healthCheckBuilder, builder.Configuration);
            }

            logger.LogInformation($"{module.ModuleName} Module has been registered.");
        }

        return builder;
    }

    public static async Task LoadModulesAsync(this WebApplication app, List<IModule> modules)
    {
        var seedEnabled = bool.TryParse(app.Configuration[ConfigurationKeys.SeedEnabled], out _);
        var migrationEnabled = bool.TryParse(app.Configuration[ConfigurationKeys.MigrationEnabled], out _);

        if(!seedEnabled || !migrationEnabled)
        {
            app.Lifetime.ApplicationStarted.Register(() =>
            {
                var logger = app.Services.GetRequiredService<ILoggerFactory>()
                    .CreateLogger(nameof(ModuleLoader));

                logger.LogInformation("Loading Modules part has been skipped: Seed or migration not enabled.");
            });

            return;
        }

        using var scope = app.Services.CreateScope();
        var serviceProvider = scope.ServiceProvider;

        var seederModules = modules.OfType<IHaveSeeder>();
        var migrationModules = modules.OfType<IHaveMigration>();

        if (seedEnabled)
        {
            foreach (var seeder in seederModules)
            {
                await seeder.SeedAsync(serviceProvider);
            }
        }

        if (migrationEnabled)
        {
            foreach (var migration in migrationModules)
            {
                await migration.MigrateDatabase(app);
            }
        }
    }
}
```

- **RegisterModules** metotunda her bir modül foreach ile teker teker gezinip kullandığı interface'e göre kendisine düşen sorumluluğu yerine getirmektedir. Tüm bu işlemler bittiğinde ise konsola ilgili modülün işlemlerinin bittiğini belirten bir yazı yazmaktadır.

- **LoadModulesAsync** metotunda ise konfigürasyondan SeedEnabled ve MigrationEnabled değerlerini alıyoruz. Bu iki değerin de false olması durumunda akışı sonlandırıyor, uygulamanın başlangıçtan hemen sonrası log basmasını sağlıyoruz. İkisinden birinin true olması durumunda akış devam ederek seeding yada migration işlemi gerçekleştiriliyor.

---

Şuanlık BuildingBlocks tarafında işimiz bitti. Şimdi örnek olması açısından her bir modülün Host sınıfını hazırlayalım. Örnek olarak UserModuleHost sınıfını göstereyim.

```cs
namespace UserModule.Host;

public sealed class UserModuleHost : IModule, IHaveService, IHaveEndpoint
{
    public string ModuleName => "User";

    public IMvcBuilder ConfigureEndpoints(IMvcBuilder mvcBuilder)
    {
        mvcBuilder.AddApplicationPart(typeof(UserModuleHost).Assembly);
        return mvcBuilder;
    }

    public IServiceCollection ConfigureServices(IServiceCollection services, IConfiguration configuration)
    {
        return services;
    }
}
```

Şuanlık buranın boş olduğuna bakmayalım, henüz User modülüne tam anlamıyla giriş yapmadık. Serinin ilerleyen zamanlarda User modülüne girdiğimizde burayı tekrardan güncelleyeceğiz.

Host.Web.Api projesine geçelim ve Infrastructure klasörü açarak HostModule sınıfımızı Oluşturalım.

```cs
public sealed class HostModule : IModule, IHaveHealthCheck, IHaveService
{
    public string ModuleName => "Host";

    public IHealthChecksBuilder CheckStatus(IHealthChecksBuilder builder, IConfiguration configurationManager)
    {
        return builder.AddApplicationStatus("Application Status");
    }

    public IServiceCollection ConfigureServices(IServiceCollection services, IConfiguration configuration)
    {
        services.AddHttpContextAccessor();

        var connectionString = configuration.GetConnectionString(ConfigurationKeys.ConnectionStrings.HealthCheckDb);
        if (string.IsNullOrEmpty(connectionString))
        {
            throw new InvalidOperationException("HealthCheckDb connection string is not configured.");
        }

        var baseUrl = configuration[ConfigurationKeys.BaseUrl];

        services.AddHealthChecksUI(opt =>
        {
            opt.AddHealthCheckEndpoint("Modular Medium", $"{baseUrl}/healthz");
            opt.SetEvaluationTimeInSeconds(60);
        })
        .AddSqlServerStorage(connectionString, (optionsBuilder) =>
        {
            optionsBuilder.ConfigureWarnings(builder => builder.Ignore(RelationalEventId.MultipleCollectionIncludeWarning));
            optionsBuilder.EnableSensitiveDataLogging(false);
            optionsBuilder.EnableDetailedErrors(false);
        });

        return services;
    }
}
```

- HostModule sınıfında daha çok HealthCheck mekanizmasını ayağa kaldırmak için ilgili kodlarımız bulunuyor. Her health-check atıldığında elde edilen verilerin kaydedileceği bir SqlServer belirttik.
- Son olarak Infrastructure klasöründe hem Scalar hem de HealthCheck-UI konfigüre etmek için ApplicationBuilderExtensions sınıfımızı oluşturuyoruz.

> Scalar, OpenAPI dökümanlarını oldukça güzel bir arayüz ile kullanıcıya sunan ve etkileşimli bir API dokümantasyon sitesine dönüştüren bir platformdur.

```cs
public static class ApplicationBuilderExtensions
{
    public static IApplicationBuilder UseCustomHealthChecks(this IApplicationBuilder app)
    {
        Dictionary<HealthStatus, int> statusCodes = new()
        {
            [HealthStatus.Healthy] = StatusCodes.Status200OK,
            [HealthStatus.Degraded] = StatusCodes.Status500InternalServerError,
            [HealthStatus.Unhealthy] = StatusCodes.Status503ServiceUnavailable,
        };

        app
            .UseHealthChecks("/health", new HealthCheckOptions
            {
                Predicate = _ => true,
                ResultStatusCodes = statusCodes,
            })
            .UseHealthChecks("/healthz", new HealthCheckOptions
            {
                Predicate = _ => true,
                ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
                ResultStatusCodes = statusCodes,
            });

        return app;
    }

    public static IApplicationBuilder UseHealthCheckDashboard(this IApplicationBuilder app)
    {
        app.UseHealthChecksUI(options =>
         {
             options.UIPath = "/ui/health";
             options.ApiPath = "/api/health";

             options.UseRelativeApiPath = false;
             options.UseRelativeResourcesPath = false;
             options.UseRelativeWebhookPath = false;
         });

        return app;
    }

    public static IApplicationBuilder ConfigureScalarApi(this WebApplication app)
    {
        app.MapScalarApiReference(opt =>
        {
            opt.WithDownloadButton(true)
                .WithTestRequestButton(true)
                .WithTitle("Modular Medium API")
                .WithSidebar(true)
                .WithDarkModeToggle(false)
                .WithLayout(ScalarLayout.Modern)
                .WithModels(false)
                .WithDefaultHttpClient(ScalarTarget.CSharp,ScalarClient.HttpClient);
        });

        return app;
    }
}
```

HealthCheck UI ve scalar altyapımızı ayarladık. Scalar'da Layout olarak Modern tercih ediyorum ancak swagger arayüzüne alışık biri iseniz Classic olarak da güncelleyebilirsiniz.

Ana projemize indirilen paket bilgileri şu şekildedir;

|Paket Adı|Version|
|---|---|
|AspNetCore.HealthChecks.UI|9.0.0| 
|AspNetCore.HealthChecks.UI.Client|9.0.0|
|AspNetCore.HealthChecks.UI.SqlServer.Storage|9.0.0|
|AspNetCore.HealthChecks.HealthChecks|9.0.0|
|Scalar.AspNetCore|1.2.56|

```cs
var builder = WebApplication.CreateBuilder(args);
builder.Configuration.AddEnvironmentVariables();

#if DEBUG
builder.Configuration.AddJsonFile("appsettings.Debug.json", optional: true, reloadOnChange: true);
builder.Logging.AddDebug();
#endif

#region Module Definitions
List<IModule> Modules = 
[
    new HostModule(),
    new ResourceModuleHost(),
    new InteractionModuleHost(),
    new UserModuleHost(),
    new NotificationModuleHost(),
];
#endregion

builder.RegisterModules(Modules);

builder.Services.AddOpenApi();

var app = builder.Build();

if (!app.Environment.IsProduction())
{
    app.MapOpenApi();
    app.ConfigureScalarApi();
}

app.UseHttpsRedirection()
    .UseCustomHealthChecks()
    .UseHealthCheckDashboard();

await app.LoadModulesAsync(Modules);

app.Run();
```

- Sadece Debug ortamında konfigürasyon bilgilerini tutacağım için bu json dosyasını program.cs tarafında ekliyorum ve git tarafında ignore ediyorum. 

- Modüllerimizi buradaki gibi bir liste halinde tutarak yazdığımız extension metotlara parametre olarak gönderebiliriz.

Uygulamamızı çalıştıralım. Hem konsolda hem de healthCheck-ui ekranında bizi neler bekliyor, görelim.

![](/images/posts/bolum-2-mimari-altyapi-ve-proje-organizasyonu\scalar-ui.png)

![](/images/posts/bolum-2-mimari-altyapi-ve-proje-organizasyonu\hc-ui.png)

> Herhangi bir endpoint olmadığından dolayı boş bir Scalar Sayfası & Modüllerde herhangi bir komponent ekli olmadığı için boş olan HealthCheck Sayfası

![](/images/posts/bolum-2-mimari-altyapi-ve-proje-organizasyonu\console-output.png)

> SeedEnabled ve MigrationEnabled olduğundan dolayı hata vermeyen konsol çıktısı

Uygulamamızı ayağa kaldırdık ve olabildiğince dinamik bir şekilde bu süreci yönettik. Serinin ilerleyen süreçlerinde bu sayfaları çok daha dolu göreceğiz diyebilirim :)

> Projenin son haline [buradaki](https://github.com/AbdullahOztuurkk/Modular.Medium/tree/Episode-1) linkten erişebilirsiniz.

---

Bu yazıda, modular monolith yapısında modüllerin birbirinden bağımsız çalışmasını kolaylaştıran klasör altyapısını, `IModule`, `IHaveSeeder` ve `IHaveService` gibi arayüzlerin kullanımını, **ModuleLoader** aracılığıyla bu yapıların nasıl otomatik keşfedilip yönetildiğini anlattım. Ayrıca, Scalar ile API dokümantasyonu ve **HealthCheck UI** ile sistem durumunun izlenmesi süreçlerini paylaştım.

Okuduğunuz için teşekkür ederim. Faydalı bulduysanız beğenilerinizi eksik görmeyin. Bir sonraki yazıda görüşmek dileğiyle.