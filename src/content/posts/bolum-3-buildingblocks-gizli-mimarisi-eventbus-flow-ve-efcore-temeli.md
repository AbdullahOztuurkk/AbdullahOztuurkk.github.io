---
title:  "📚 Bölüm 3 - BuildingBlocks'ın Gizli Mimarisi: EventBus Akışı ve EF Core Temeli"
date:   2025-11-22
image: "/images/posts/bolum-3-buildingblocks-gizli-mimarisi-eventbus-flow-ve-efcore-temeli/index.jpg"
categories: ["Software"]
tags: ["Modular Monolith",".Net Core"]
---

Herkese Merhaba,

Serinin önceki bölümünde bir modular monolith uygulamasının dosya hiyerarşisi ve proje altyapısıyla alakalı bir taslak oluşturmuştuk. Bu bölümde ise bu taslak üzerinden devam ederek tüm modüllerde ortak olarak kullanılacak **EventBus** ve **EntityFrameworkCore** kütüphanelerini tasarlayacağız.

**EventBus** yapısını anlatmadan önce **Event-Driven Programming** felsefesini anlatalım. Bu felsefe hem modular monolith hem de mikroservis mimarisinde oldukça önemli bir rol oynar. Çoğunlukla kullanıcının bir işlem yapması ile bir event meydana gelir. (Örneğin; Kullanıcı kayıt olduğunda UserCreatedIntegrationEvent) Bu event bir **Publisher** aracılığıyla kamuoyuna duyurulur. Bu event ile ilgilenen **consumer** sınıfları bu eventi yakalar ve kendi iş akışlarını çalıştırırlar. Biz modular monolith serisinin bu bölümünde abstraction katmanı oluşturarak ilerleyen zamanda yeni bir pub-sub sistemine geçişimizi kolaylaştıracağız.

Seri boyunca ORM olarak **EntityFrameworkCore** kullanacağız. Bu süreçte repository pattern kullanmamayı tercih ediyorum. Ekstra bir soyutlama yapmamıza gerek yok. Aynı şekilde bu bölümde IQueryable için birkaç extension metot oluşturacağız.

---

Öncelikle Infrastructure katmanına **EventBus** adlı klasör oluşturalım. EventBus soyutlamalarımızı içerecek olan **BuildingBlocks.EventBus.Abstraction** adlı projeyi oluşturalım.

```cs
public record IntegrationEvent 
{
    public Guid CorrelationId { get; } = Guid.NewGuid();
}
```

- IntegrationEvent adlı record yapımızı oluşturduk. Tüm eventler bu recorddan türemek zorundadır. CorrelationId üzerinden de event süreç takibini sağlayabileceğiz.

```cs
public interface IIntegrationEventHandler { }

public interface IIntegrationEventHandler<in TIntegrationEvent> : IIntegrationEventHandler
    where TIntegrationEvent : IntegrationEvent
{
    Task Handle(TIntegrationEvent @event);
}
```

- IIntegrationEventHandler interface'i, hangi event handle edileceğini belirtir ve tüm consumer sınıfların implement etmek zorundadır.

```cs
public interface IModuleEventBusConfigurator
{
    Task ConfigureModuleSubscriptions(IEventBus eventBus, CancellationToken cancellationToken = default);
}
```

- IModuleEventBusConfigurator interface'i, consumer içeren her modül tarafından implemente edilmek zorundadır. Hangi event, hangi handler tarafından tüketileceği burada belirtilmelidir.

```cs
public interface IEventBus
{
    Task Publish<TIntegrationEvent>(TIntegrationEvent @event)
        where TIntegrationEvent : IntegrationEvent;

    Task Subscribe<TIntegrationEvent, TIntegrationEventHandler>()
        where TIntegrationEvent : IntegrationEvent
        where TIntegrationEventHandler : IIntegrationEventHandler<TIntegrationEvent>;

    Task Unsubscribe<TIntegrationEvent, TIntegrationEventHandler>()
        where TIntegrationEvent : IntegrationEvent
        where TIntegrationEventHandler : IIntegrationEventHandler<TIntegrationEvent>;
}
```

- IEventBus interface'i, ana interface olup geliştirilecek olan tüm EventBus implementasyonlarında kullanılmak durumundadır. Publish, Subscribe ve Unsubscribe işlevi bulunmaktadır.

```cs
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddIntegrationEventHandler<TIntegrationEventHandler>(this IServiceCollection services)
        where TIntegrationEventHandler : class, IIntegrationEventHandler
    {
        return services.AddTransient<TIntegrationEventHandler>();
    }
}
```

- Modüllerde tanımlanan event handler'ların IoC container'a otomatik olarak kaydedilebilmesi için **AddIntegrationEventHandler** extension metodu yazıldı.

```cs
public static class WebApplicationExtensions
{
    public static async Task ConfigureEventBusSubscriptionsAsync(this WebApplication app, List<IModule> modules,
        CancellationToken cancellationToken = default)
    {
        using var scope = app.Services.CreateScope();
        var serviceProvider = scope.ServiceProvider;
        var eventBus = serviceProvider.GetRequiredService<IEventBus>();
        var logger = serviceProvider.GetRequiredService<ILogger<WebApplication>>();

        var eventModules = modules.OfType<IModuleEventBusConfigurator>();

        foreach (var module in eventModules)
        {
            await module.ConfigureModuleSubscriptions(eventBus, cancellationToken);
        }

        app.Lifetime.ApplicationStarted.Register(() =>
        {
            logger.LogInformation("Event Bus subscriptions have been configured and are active.");
        });
    }
}
```

- Uygulama başlatılırken **ConfigureEventBusSubscriptionsAsync**, tüm handler sınıflarını keşfedip IoC container'a ekleyecek ve işlemle ilgili bilgilendirme logu yazacaktır.

---

Abstraction projesini tanımladığımıza göre **Infrastructure/EventBus** altında **BuildingBlocks.EventBus.InMemory** projesini oluşturabiliriz. Burada Channels yapısını baz alarak geliştirmelerimizi yapacağız.

> NET Channels; ChannelWriter<T> ve ChannelReader<T> ile çalışan asenkron, thread-safe bir publisher–consumer yapısıdır. FIFO(First IN First Out) sırada veri aktarır ve düşük lock/düşük allocation sayesinde yüksek **throughput** sağlar. Channel türü 2'ye ayrılır. **Unbounded channel** sınırsız kapasiteyle çalışır ve çok hızlı üretime izin verir ama bellek tüketimi kontrolü sana bırakır; **bounded channel** belirli kapasiteden sonra WriteAsync'i bekleterek back-pressure uygular ve sistemi kendi kendine dengeler. Bu sayede ne publisher alıp başını gider, ne de consumer verileriyle boğulur.

```cs
public sealed class InMemoryEventBus : IEventBus
{
    private readonly ConcurrentDictionary<string, Channel<IntegrationEvent>> _channels = new();
    private readonly ConcurrentDictionary<string, List<Type>> _eventHandlerTypes = new();

    public IReadOnlyDictionary<string, Channel<IntegrationEvent>> Channels => _channels;
    public IReadOnlyDictionary<string, List<Type>> Handlers => _eventHandlerTypes;

    public async Task Publish<TIntegrationEvent>(TIntegrationEvent @event) where TIntegrationEvent : IntegrationEvent
    {
        var eventType = @event.GetType().Name;
        if (_channels.TryGetValue(eventType, out var channel))
            await channel.Writer.WriteAsync(@event);
    }

    public Task Subscribe<TIntegrationEvent, TIntegrationEventHandler>()
        where TIntegrationEvent : IntegrationEvent
        where TIntegrationEventHandler : IIntegrationEventHandler<TIntegrationEvent>
    {
        var eventType = typeof(TIntegrationEvent).Name;
        var handlerType = typeof(TIntegrationEventHandler);

        var handlers = _eventHandlerTypes.GetOrAdd(eventType, _ => new List<Type>());
        if (!handlers.Contains(handlerType))
            handlers.Add(handlerType);

        _channels.TryAdd(eventType, Channel.CreateUnbounded<IntegrationEvent>());

        return Task.CompletedTask;
    }

    public Task Unsubscribe<TIntegrationEvent, TIntegrationEventHandler>()
        where TIntegrationEvent : IntegrationEvent
        where TIntegrationEventHandler : IIntegrationEventHandler<TIntegrationEvent>
    {
        var eventType = typeof(TIntegrationEvent).Name;
        var handlerType = typeof(TIntegrationEventHandler);

        if (_eventHandlerTypes.TryGetValue(eventType, out var handlers))
        {
            if (handlers.Contains(handlerType))
                handlers.Remove(handlerType);

            if (handlers.Count == 0)
            {
                _eventHandlerTypes.TryRemove(eventType, out _);
                _channels.TryRemove(eventType, out _);
            }
        }

        return Task.CompletedTask;
    }
}
```

- Her event türü için bir channel tutulur. ÖrneğinOrderCreatedEvent varsa bunun event için özel bir kanal var. Producer publish ettiğinde bu kanala yazar, ilgili consumer bu kanaldan okur.

```cs
private readonly ConcurrentDictionary<string, Channel<IntegrationEvent>> _channels;
```

- Hangi event'i hangi handler'lar dinliyor bilgisini tutar. `OrderCreatedEvent → [OrderCreatedEmailHandler, OrderCreatedSmsHandler, ...]` gibi.

```cs
private readonly ConcurrentDictionary<string, List<Type>> _eventHandlerTypes;
```

- **Publish** metodu, event'in tip adına karşılık gelen kanalı bulur ve event'i kanalın yazıcısına (writer) yazar; böylece event handler'ların tüketebilmesi için event asenkron olarak kuyruğa bırakılmış olur.

- **Subscribe** metodu, belirli bir event türü için handler'ı kaydeder ve daha önce oluşturulmamışsa o event'e özel bir channel açarak ileride yayınlanacak event'lerin bu handler tarafından tüketilebilmesini sağlar.

- **Unsubscribe** metodu, belirtilen handler'ı event'in consumer listesinden çıkarır; o event'i dinleyen başka handler kalmamışsa event türüne ait handler kaydını ve channel'ı tamamen temizler.

Şuanda InMemoryEventBus sınıfımızı oluşturmuş olduk. İlgili eventlere erişebiliyor, abone olabiliyor yada publish edebiliyoruz. Ancak Handler metotlarının tetiklenme kısmında arka planda çalışacak bir sınıfa ihtiyacımız var. **InMemoryEventDispatcher** sınıfımızı oluşturmaya başlayalım.

```cs
public sealed class InMemoryEventDispatcher(IServiceProvider sp, InMemoryEventBus bus, ILogger<InMemoryEventDispatcher> logger) : BackgroundService
{
    private readonly IServiceProvider _serviceProvider = sp;
    private readonly InMemoryEventBus _eventBus = bus;
    private readonly ILogger<InMemoryEventDispatcher> _logger = logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("In-memory event dispatcher started.");

        // Her bir event tipi için ProcessEventsAsync metotunu çağırıyoruz.
        var consumers = _eventBus.Channels.Select(kvp => ProcessEventsAsync(kvp.Key, kvp.Value.Reader, stoppingToken));

        await Task.WhenAll(consumers);
    }

    private async Task ProcessEventsAsync(string eventTypeName, ChannelReader<IntegrationEvent> reader, CancellationToken cancellationToken)
    {
        _logger.LogInformation("Started processing events for type '{EventTypeName}'.", eventTypeName);

        // Kanaldaki tüm mesajlar sırayla işleniyor
        await foreach (var @event in reader.ReadAllAsync(cancellationToken))
        {
            //cancellationtoken tetiklenmesi durumunda işlem durdurulur
            cancellationToken.ThrowIfCancellationRequested();
            _logger.LogDebug("Processing event '{EventTypeName}' (ID: {EventId}).", eventTypeName, @event.CorrelationId);

            //İlgili event için herhangi bir handler subscribe olmuş mu, onu kontrol ediyoruz.
            if (!_eventBus.Handlers.TryGetValue(eventTypeName, out var handlerTypes) || handlerTypes.Count == 0)
            {
                _logger.LogWarning("No active handlers found for event '{EventTypeName}' (ID: {EventId}).", eventTypeName, @event.CorrelationId);
                continue;
            }

            //Her bir event handler için
            foreach (var handlerType in handlerTypes.ToList())
            {
                using var scope = _serviceProvider.CreateScope();
                try
                {
                    var handler = scope.ServiceProvider.GetService(handlerType);

                    // İlgili handler sınıfı IoC'den erişiyoruz, bulunamaması durumunda pas geçiyoruz.
                    if (handler is null)
                        continue;

                    //Event Handler tipine bağlı olan IIntegrationEventHandler interface'ini buluyoruz.
                    var interfaceType = handlerType.GetInterfaces()
                            .FirstOrDefault(i => i.IsGenericType && i.GetGenericTypeDefinition() == typeof(IIntegrationEventHandler<>));

                    if (interfaceType is null)
                        continue;

                    // Interface içerisindeki Handle metotuna erişiyoruz.
                    var handleMethod = interfaceType.GetMethod("Handle");
                    if (handleMethod is null)
                        continue;

                    // İlgili event Handler'daki Handle metotunu çalıştırmış oluyoruz.
                    await (Task)handleMethod.Invoke(handler, new object[] { @event })!;
                }
                catch (Exception ex)
                {
                    // Hata alınması durumunda log basıyor ve hata fırlatıyoruz.
                    _logger.LogError(ex,
                                     "Error handling event '{EventTypeName}' (ID: {EventId}) with handler '{HandlerType}': {ErrorMessage}",
                                     eventTypeName,
                                     @event.CorrelationId,
                                     handlerType.Name,
                                     ex.Message);
                    throw;
                }
            }
        }
    }
}
```

- EventBus soyutlamasını kurduk ve Channels kullanarak **InMemoryEventBus** implementasyonunu hayata geçirdik. Artık geriye, bunu uygulamalarda kolayca kullanabilmek için gerekli extension metotlarını **DependencyInjection** klasöründe yazmak kaldı.

```cs
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddInMemoryEventBus(this IServiceCollection services)
    {
        services.AddSingleton<InMemoryEventBus>();

        services.AddHostedService<InMemoryEventDispatcher>();

        return services;
    }
}
```

---

Modüllerin veri erişim katmanında ORM olarak EntityFrameworkCore kullanacağımızdan bahsetmiştik. Yine aynı şekilde Infrastructure klasörü altında **BuildingBlocks.Database.EntityFrameworkCore** adlı bir proje oluşturalım.

```cs
public abstract class ModuleContext : DbContext
{
    public static IConfiguration? Configuration { get; private set; }
    public abstract string SchemaName { get; }

    public ModuleContext(DbContextOptions options) : base(options)
    {
        string environment = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ?? "Development";
        Configuration = new ConfigurationBuilder()
                .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
                .AddJsonFile($"appsettings.{environment}.json", optional: true)
#if DEBUG
                .AddJsonFile($"appsettings.Local.json", optional: true)
#endif
                .AddEnvironmentVariables()
                .Build();
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.HasDefaultSchema(SchemaName);
        base.OnModelCreating(modelBuilder);
    }
}
```

- Her bir modül, **ModuleContext** sınıfından türetilmelidir. Şema adı belirtilerek veritabanı bazında ayrıştırma yapılacaktır. Bu sayede tüm modüllerin aynı yada farklı veritabanlarında olması bir sorun teşkil etmeyecektir. `(Yazılımsal ayrıştırmanın yapıldığını varsayıyorum)`

- Türetilen context'in connection string'lerine erişmesi için Configuration nesnesi oluşturuldu. Ek olarak sadece Debug ortamında **appsettings.local.json** dosyası eklendi.

Serinin önceki bölümünde AuditEntity sınıfı oluşturmuştuk. Bu sınıf içerisinde CreateDate ve CreateUserId gibi propertylerimiz vardı. Bunları her seferinde manuel doldurmaktansa Interceptor oluşturarak otomatize etmeyi sağlayacağız. Bu sayede ilgili veriyi veritabanına kaydetmeden güncelleme fırsatı bulacağız. 

Interceptors klasörünün altında ekleyeceğimiz adlı AuditEntityInterceptor sınıfının  içerisinde HttpContextAccessor üzerinden token okuyabilen bir servise ihtiyacımız var. O servisimizin adı **ICurrentUser**. Gelin, beraber **BuildingBlocks** projesinde services adlı bir klasör ekleyip bu servisin implementasyonunu yazalım.

```cs
public interface ICurrentUser
{
    long Id { get; }
    string UserName { get; }
    string Email { get; }
}

public sealed class CurrentUser(IHttpContextAccessor httpContextAccessor, IEncryptionService encryptionService) : ICurrentUser
{
    private readonly IHttpContextAccessor _httpContextAccessor = httpContextAccessor;
    private readonly IEncryptionService _encryptionService = encryptionService;

    public long Id => Convert.ToInt64(FindClaim("Uid"));
    public string UserName => FindClaim(ClaimTypes.Name);
    public string Email => FindClaim(ClaimTypes.Email);

    private string FindClaim(string type)
    {
        try
        {
            ClaimsPrincipal User = _httpContextAccessor.HttpContext?.User;
            //Check user is authenticated
            if (User is null)
                return string.Empty;

            // Get the claim
            var claim = User.FindFirst(x => x.Type == type);
            if (claim == null)
                return string.Empty;

            return _encryptionService.Decrypt(claim.Value);
        }
        catch (Exception)
        {
            return null;
        }
    }
}
```

- HttpContextAccessor üzerinden kullanıcının giriş yapmış olma şartı ile bilgilerine erişebiliyoruz. Ancak burada güvenlik icabı direkt bilgileri buraya koymayacağımız için şifreleme servisimiz ile bu bilgileri deşifre ederek istenen bilgiyi elde ediyoruz.

Şifreleme servisimizin içeriğinde de AES şifreleme yatıyor. Sırayla onu da oluşturalım ve nasıl şifreleyebileceğimizin üzerinden geçelim.

```cs
public sealed class EncryptionService : IEncryptionService
{
    private string encryptionKey;

    public EncryptionService(IConfiguration configuration)
    {
        encryptionKey = configuration[ConfigurationKeys.EncryptionKey];

        // Şifreleme/çözme anahtarı yoksa sistem çalışamaz, hata fırlatılır.
        ArgumentException.ThrowIfNullOrWhiteSpace(encryptionKey, nameof(encryptionKey));
    }

    public string Encrypt(string flatData)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(flatData);

        using Aes aes = Aes.Create();

        // AES her şifrelemede rastgele bir IV üretir (aynı veriyi 2 kez şifrelesek bile sonuç farklı olur)
        using ICryptoTransform cryptoTransform = aes.CreateEncryptor(Encoding.UTF8.GetBytes(encryptionKey), aes.IV);

        using MemoryStream memoryStream = new();
        using CryptoStream cryptoStream = new(memoryStream, cryptoTransform, CryptoStreamMode.Write);
        using (StreamWriter streamWriter = new(cryptoStream))
        {
            streamWriter.Write(flatData); // veriyi burada şifreliyoruz.
        }

        byte[] iv = aes.IV; // daha sonra çözebilmek için IV'yi saklamamız gerekiyor
        byte[] encryptedContent = memoryStream.ToArray();

        // Final sonucumuz: [IV] + [şifreli veri] tek bir
        byte[] result = new byte[iv.Length + encryptedContent.Length];
        Buffer.BlockCopy(iv, 0, result, 0, iv.Length);
        Buffer.BlockCopy(encryptedContent, 0, result, iv.Length, encryptedContent.Length);

        return Convert.ToBase64String(result); // string formatında döndürüyoruz
    }

    public string Decrypt(string cipherText)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(cipherText);

        byte[] fullCipher = Convert.FromBase64String(cipherText);

        // Şifre çözerken ilk kısım IV, kalan kısmı şifreli veri
        byte[] iv = new byte[16];
        byte[] cipher = new byte[fullCipher.Length - iv.Length];
        Buffer.BlockCopy(fullCipher, 0, iv, 0, iv.Length);
        Buffer.BlockCopy(fullCipher, iv.Length, cipher, 0, cipher.Length);

        using Aes aes = Aes.Create();

        // Aynı key + kaydedilen IV ile decrypt ediyoruz.
        using ICryptoTransform cryptoTransform = aes.CreateDecryptor(Encoding.UTF8.GetBytes(encryptionKey), iv);

        string flatData;
        using (MemoryStream memoryStream = new(cipher))
        {
            using CryptoStream cryptoStream = new(memoryStream, cryptoTransform, CryptoStreamMode.Read);
            using StreamReader streamWriter = new(cryptoStream)
            {
                flatData = streamWriter.ReadToEnd(); // şifre çözüldü, artık dönebiliriz.
            }
        }

        return flatData;
    }
}
```

- Encryption ve CurrentUser servislerini altyapıya ekledik; biri veriyi güvende tutmak için, diğeri de isteklerin kimden geldiğini bilmek için.

- İleride yetkilendirme, audit ve güvenli veri saklama senaryolarında aranan servisler olacaklardır.

Bu servislerimizi uygulamanın her bir tarafında kullanabilmek için IoC Container'a kayıt ettirecek bir extension metota ihtiyacımız var. Bunun için BuildingBlocks projesinde DependencyInjection klasörü oluşturalım ve içerisinde ServiceExtensions sınıfını ekleyelim.

```cs
public static class ServiceExtensions
{
    public static IServiceCollection AddInfrastructuralServices(this IServiceCollection services)
    {
        services.AddScoped<ICurrentUser, CurrentUser>();

        services.AddScoped<IEncryptionService, EncryptionService>();

        return services;
    }
}
```

---

Tekrardan **BuildingBlocks.Database.EntityFrameworkCore** projemize odaklanalım. Interceptors klasörü altına **AuditEntityInterceptor** adlı sınıfımızı oluşturalım.

```cs
//SaveChangesInterceptor sınıfından türetiyoruz.
public class AuditEntityInterceptor(ICurrentUser currentUser) : SaveChangesInterceptor
{
    private readonly ICurrentUser _currentUser = currentUser;

    //Kaydetmeden hemen önce SetAuditFields adlı metotumuzun çalışmasını sağlıyoruz.
    public override InterceptionResult<int> SavingChanges(
        DbContextEventData eventData,
        InterceptionResult<int> result)
    {
        var context = eventData.Context;
        if (context == null)
            return base.SavingChanges(eventData, result);

        SetAuditFields(context);

        return base.SavingChanges(eventData, result);
    }

    //Kaydetmeden hemen önce SetAuditFields adlı metotumuzun çalışmasını sağlıyoruz.
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        var context = eventData.Context;
        if (context == null)
            return base.SavingChangesAsync(eventData, result, cancellationToken);

        SetAuditFields(context);

        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }

    private void SetAuditFields(DbContext context)
    {
        var entries = context.ChangeTracker.Entries<AuditEntity>();

        foreach (var entry in entries)
        {
            var now = DateTime.UtcNow.AddHours(3);

            switch (entry.State)
            {
                case EntityState.Added:
                    entry.Entity.CreateDate = now;
                    entry.Entity.CreateUserId = _currentUser.Id;
                    break;

                case EntityState.Modified:
                    entry.Entity.UpdateUserId = _currentUser.Id;
                    entry.Entity.UpdateDate = now;
                    break;

                case EntityState.Deleted:
                    entry.State = EntityState.Modified;
                    entry.Entity.DeleteDate = now;
                    entry.Entity.DeleteUserId = _currentUser.Id;
                    entry.Entity.Status = StatusType.Passive;
                    break;
            }
        }
    }
}
```

- Bu interceptor içinde soft-delete işlemini uyguladık; proje akışı gereği veriler fiziksel olarak silinmeyecek, bunun yerine Status alanı üzerinden global filtre çalışacak ve böylece silinmiş görünen kayıtlar aslında korunmuş olacak.

Interceptor da oluşturduğumuza göre EntityFrameworkCore tarafında sıkça kullanacağımız extension metotları yazma vaktimiz geldi. Extensions klasörü açalım ve içerisine **IQueryableExtensions** adlı sınıfımızı ekleyelim.

```cs
namespace BuildingBlocks.Database.EntityFrameworkCore.Extensions;

public static class IQueryableExtensions
{
    public static PagedResult<T> ToPagedResult<T>(this IQueryable<T> query, ListRequest request)
        where T : notnull, new()
    {
        int totalCount = query.Count();

        if (!string.IsNullOrEmpty(request.SortBy))
        {
            var propertyInfo = typeof(T).GetProperty(request.SortBy);

            // Eğer ilgili property bulunmuyorsa boş bir liste dönüyoruz.
            if (propertyInfo == null)
                return new PagedResult<T>();

            //  lambda expression oluşturuyoruz: x => x.PropertyName
            var parameter = Expression.Parameter(typeof(T), "x");
            var property = Expression.Property(parameter, propertyInfo);
            var lambda = Expression.Lambda(property, parameter);

            string methodName = request.IsDescending ? "OrderByDescending" : "OrderBy";

            // Reflection yardımıyla ilgili LINQ sorgusunu buluyoruz.
            var method = typeof(Queryable).GetMethods()
                .First(m => m.Name == methodName && m.GetParameters().Length == 2)
                .MakeGenericMethod(typeof(T), property.Type);

            // Sıralamayı uygulamak için metodu manuel çağırıyoruz.
            query = (IQueryable<T>)method.Invoke(null, [query, lambda]);
        }

        int skip = (request.PageNumber - 1) * request.PageSize;

        var items = query
            .Skip(skip)
            .Take(request.PageSize)
            .ToList();

        return new PagedResult<T>
        {
            Items = items,
            TotalCount = totalCount,
            PageNumber = request.PageNumber,
            PageSize = request.PageSize
        };
    }

    // Filtrelemeyi yaptıktan sonra otomatik olarak ToList yapmamızı sağlayan yardımcı metot.
    public static Task<List<T>> QueryListAsync<T>(this IQueryable<T> dbSet,
                                                                       Expression<Func<T, bool>> expression,
                                                                       CancellationToken cancellationToken = default) where T : class
    {
        return dbSet.Where(expression).ToListAsync(cancellationToken);
    }

    //Koşullu filtrelememizi kolaylaştıran extension metot.
    public static IQueryable<T> WhereIf<T>(this IQueryable<T> query,
        bool condition,
        Expression<Func<T, bool>> predicate)
        where T : class
    {
        if (condition)
        {
            return query.Where(predicate);
        }
        return query;
    }
}
```

- Extension metotlarımızı yazdık ancak elimizde ListRequest ve PagedResult sınıfları yok. Hemen onları oluşturalım. **BuildingBlocks** projesine dönüyoruz ve Domain/Dto klasörü altında bu 2 sınıfımızı ekliyoruz.

```cs
public class ListRequest
{
    public int PageNumber { get; set; } = 1;
    public int PageSize { get; set; } = 10;
    public string SortBy { get; set; }
    public bool IsDescending { get; set; } = false;
}

public sealed record PagedResult<T> where T : notnull, new()
{
    public IReadOnlyList<T> Items { get; set; } = new List<T>();
    public int TotalCount { get; set; }
    public int PageNumber { get; set; }
    public int PageSize { get; set; }
    public int TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);
}
```

Artık Asenkron şekilde Event fırlatabileceğimiz bir EventBus yapımız ve veritabanı seviyesinde ortak kullanılan bir EntityFrameworkCore kütüphanemiz bulunuyor.

Yazmış olduğumuz extension metotları Bootstrapper uygulamasında kullanalım ve serimizin bu bölümünü sonlandıralım.

```cs
...

builder.Services.AddInfrastructuralServices();

builder.Services.AddInMemoryEventBus();
...

var app = builder.Build();

...

await app.ConfigureEventBusSubscriptionsAsync(Modules);

app.Run();
```

Uygulamamızı çalıştıralım. Konsolda EventBus yapısının çalıştığını görelim ve serinin bu bölümünü sonlandıralım.

```yaml
info: ModuleLoader[0]
      Host Module has been registered.
info: ModuleLoader[0]
      Resource Module has been registered.
info: ModuleLoader[0]
      Interaction Module has been registered.
info: ModuleLoader[0]
      User Module has been registered.
info: ModuleLoader[0]
      Notification Module has been registered.
info: BuildingBlocks.EventBus.InMemory.InMemoryEventDispatcher[0]
      In-memory event dispatcher started.
info: HealthChecks.UI.Core.HostedService.UIInitializationHostedService[0]
      Initializing UI Database
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:8000
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Debug
info: Microsoft.Hosting.Lifetime[0]
      Content root path: D:\repositories\Modular.Medium\src\Bootstrapper\Host.Web.Api
info: System.Net.Http.HttpClient.health-checks.LogicalHandler[100]
      Start processing HTTP request GET http://localhost:8000/healthz
info: System.Net.Http.HttpClient.health-checks.ClientHandler[100]
      Sending HTTP request GET http://localhost:8000/healthz
warn: Microsoft.AspNetCore.HttpsPolicy.HttpsRedirectionMiddleware[3]
      Failed to determine the https port for redirect.
info: System.Net.Http.HttpClient.health-checks.ClientHandler[101]
      Received HTTP response headers after 170.2278ms - 200
```

Gördüğünüz üzere ilgili tüm modüller kendilerini register etti ve In-Memory Event Dispatcher, şuanda kendisinin çalışır vaziyette beklediğini duyurdu.

Projenin son haline [buradaki](https://medium.com/r/?url=https%3A%2F%2Fgithub.com%2FAbdullahOztuurkk%2FModular.Medium%2Ftree%2FEpisode-2) linkten erişebilirsiniz.

--- 

Bu yazıda, modular monolith yapısında modüllerin iletişimde önemli bir rol oynayan EventBus yapısını ve veritabanı tarafında yoğunlıkla kullanacağımız EntityFrameworkCore temelleri içeren kütüphanelerin nasıl geliştirilmesi gerektiğini anlattım. 

Okuduğunuz için teşekkür ederim. Faydalı bulduysanız beğenilerinizi eksik görmeyin. Bir sonraki yazıda görüşmek dileğiyle.