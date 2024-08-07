---
title:  ".Net Core Ortamında MediatR Pipeline Geliştirelim"
date:   2022-08-19
image: "/images/posts/net-core-ortaminda-mediatr-pipeline-gelistirelim/index.jpg"
categories: ["Software"]
tags: ["Dotnet", "CQRS"]
---

Merhaba arkadaşlar,

Bugün MediatR kapsamında gerçekleştirilen her bir istek için oluşturduğumuz genel yapıları (validation, logging vb) daha merkezi bir sisteme nasıl taşırız gibi soruların cevaplarına değineceğim. Genel tanımı yaptıktan sonra .NET Core ortamında bir uygulama geliştireceğiz. 

MediatR kütüphanesinin ne işe yaradığını bilmiyorsanız [buradaki](../command-query-responsibility-segregation-nedir) yazımı okuyabilirsiniz.

## Pipelines
Bir uygulamaya istek attığımız her isteğimize(request) karşın bir yanıt (response) döner. Yalnız bu request - response döngüsünde farkında olunmayan bir kısım var, **pipelines**. **Pipelines** sayesinde gönderilen istek birtakım operasyonlardan geçirilir ve eğer bir sıkıntı yoksa döngüye devam edilir. Bir problem meydana gelirse istek geri çevrilir ve response döndürülür. Aşağıda .NET Core pipeline mimarisinin yapısını görmektesiniz.

| ![ .Net Core Pipeline Mimarisi ](/images/posts/net-core-ortaminda-mediatr-pipeline-gelistirelim/net-core-pipeline-architecture.png)| 

.Net Core Middleware PipelineRequest ile gönderilen objeyi validasyondan geçirmek istediğimizi varsayalım. Bunun üstesinden nasıl gelirdik? Muhtemelen RequestHandler sınıfımız içerisinde aşağıdakine benzer bir kod yazar ve bu şekilde işlemin üstesinden gelirdik. Ancak çok fazla sınıfınızın olduğunu ve her seferinde bu kodu yazmak zorunda kaldığınızı düşünün. 

```cs
...
var result = validator.Validate(entity);
if (result.Errors.Any())
	return Task.FromResult<IResult>(new ErrorResponse(result.Errors.First().ErrorMessage));
...
```

Bunun yerine Pipelines ile her request için belli operasyonları geliştiren bir merkezi yapı oluşturarak daha temiz, daha güvenli, daha verimli bir  çalışma ortamı elde edebiliyoruz.

---

Tanımı yaptığımıza göre yavaştan yapacağımız uygulama için hazırlıklara başlayalım. Öncelikle Domain katmanına gidelim ve Entities klasörü altında veritabanındaki tabloları temsil eden sınıfları oluşturalım.

Ayrıca asıl konudan sapmamak adına ayrıntılı bir şekilde oluşturduğum yapıları göstermeyeceğim. Yazının sonunda projenin linkini bulabilirsiniz.

```cs
public class BaseEntity
{
    public int Id { get; set; }
}

public class Note : BaseEntity
{
    public string Title { get; set; }
    public string Content { get; set; }
    public bool IsStarred { get; set; }

    //Relationship
    public Tag Tag { get; set; }

    public Note(string title, string content, bool isStarred)
    {
        Title = title;
        Content = content;
        IsStarred = isStarred;
    }
}

public class Tag : BaseEntity
{
    public string Name { get; set; }
    public string Description { get; set; }

    //Relationship
    public ICollection<Note> Notes { get; set; }
}
```

MediatR tarafında kullanacağımız request’e yanıt niteliğindeki response sınıfını oluşturalım.

```cs
        //Get request values
        IList<PropertyInfo> props = new List<PropertyInfo>(myType.GetProperties());
        foreach (PropertyInfo prop in props)
        {
            object propValue = prop.GetValue(request, null);
            _logger.LogInformation("{Property} : {@Value}", prop.Name, propValue);
        }

        var response = await next();
        //Response
        _logger.LogInformation($"Response Type : {typeof(TResponse).Name}");
        return response;
    }
}
```

- Pipeline olması için IPipelineBehaviour interface’inden türetiyoruz
- 2. ve 3. satırda bize gelen response ve request için kısıtlamalar getirerek dönecek ve gelecek veriyi belirtiyoruz.
- Constructorda tüm operasyonlara loglara dökmek için bir dependency injection uyguluyoruz.
- 13. satırda gelen request’in typeof aracılığıyla tipini bulup metot adını logluyoruz.
- 14. ve 22. satırlar arasında Reflection ile gelen request’e ait property’lerin içeriğini ve türlerini loglayabiliyoruz.
- Son olarak geri dönüş değeri olarak belirttiğimiz response modelimizi logluyoruz ve operasyonun devam etmesini sağlıyoruz.

Artık Behaviour sınıflarımızı yazdığımıza göre bunları inject edelim ve API tarafındaki testlerimizi yapalım.

```cs
public static void AddApplicationServices(this IServiceCollection services)
{
    services.AddAutoMapper(typeof(GeneralProfile));
    services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());
    services.AddMediatR(Assembly.GetExecutingAssembly());

    services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehaviour<,>));
    services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehaviour<,>));
}
```

---

Presentation katmanında yani bizim projedeki API katmanında startup.cs üzerinde birkaç ayar yapmamız gerekiyor.

```cs
public Startup(IConfiguration configuration)
{
    Configuration = configuration;
}

public IConfiguration Configuration { get; }

public void ConfigureServices(IServiceCollection services)
{
    ...
    services.AddPersistenceServices(Configuration);
    services.AddApplicationServices();
    ...
}

// This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    ...
    app.UseOpenApi();
    app.UseSwaggerUi3();
    ...
}
```

API dökümantasyonunu oluşturacağımız **Swagger** için de çeşitli ayarlar yapmış bulunmaktayız. Swagger hakkında kendinizi eksik hissediyorsanız [buradaki](../dotnet-core-swagger-entegrasyonu) yazımdan bilgilenebilirsiniz. Startup tarafındaki ayarlamalarımızı yaptığımıza göre Controller tarafını kodlamaya başlayabiliriz. Aşağıda asıl konudan kopmamak adına HttpPost metotlarını görmektesiniz.

```cs
[ApiController]
[Route("api/notes")]
public class NoteController : ControllerBase
{
    public readonly IMediator mediator;
    public NoteController(IMediator mediator)
    {
        this.mediator = mediator;
    }

    /// <summary>
    /// Create a Note
    /// </summary>
    /// <param name="request">Note Model</param>
    /// <returns></returns>
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateNoteCommandRequest request)
    {
        var result = await mediator.Send(request);
        if (result.IsSucceed == false)
            return BadRequest(result.Message);
        return Ok(result);
    }
}

[Route("api/tags/")]
[ApiController]
public class TagController : ControllerBase
{
    public readonly IMediator mediator;

    public TagController(IMediator mediator)
    {
        this.mediator = mediator;
    }
    
    /// <summary>
    /// Create a Tag
    /// </summary>
    /// <param name="request">Tag Model</param>
    /// <returns></returns>
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateTagCommandRequest request)
    {
        var result = await mediator.Send(request);
        if (result.IsSucceed == false)
            return BadRequest(result.Message);
        return Ok(result);
    }
}
```

Şimdi programımızı logları görmek açısından IIS Express üzerinden değil de CleanArch.API üzerinden başlatalım ve bir tane geçerli request objesi gönderelim ve logları gözlemleyelim. Sonrasında ise geçersiz bir request objesi gönderelim ve **ValidationBehaviour** sınıfını gözlemleyelim.

| ![Postman Çıktısı](/images/posts/net-core-ortaminda-mediatr-pipeline-gelistirelim/postman_output.png) | 

Veri başarıyla eklendi, şimdi loglarımızı inceyelim. Bakalım nasıl bir çıktı verdi?

| ![Log Çıktısı](/images/posts/net-core-ortaminda-mediatr-pipeline-gelistirelim/logbehaviour_output.png) | 

Gördüğünüz gibi hangi request’in gönderildiği, hangi property’nin hangi değere sahip olduğu ve response türü gibi özellikleri yazdırabildik. Şimdi geçersiz bir request objesi gönderelim ve response modelini inceleyelim ve yazımızı sonlandıralım.

| ![ Validation Çıktısı ](/images/posts/net-core-ortaminda-mediatr-pipeline-gelistirelim/validationbehaviour_output.png) | 

Evet, **RequestHandler** sınıfımızda herhangi bir validasyon kodu yazmasak da **ValidationBehaviour** sınıfı request objesi için bizim yerimize response döndü.

---

Bu yazımda pipeline nedir, tam olarak ne işe yarar ve MediatR implementasyonu nasıl gerçekleştirilir gibi soruların cevaplarını vermeye çalıştım.

Projeyi tam anlamıyla incelemek için github linkine [buradan](https://github.com/AbdullahOztuurkk/CleanArchitecture) ulaşabilirsiniz.

Okuduğunuz için teşekkür ederim. Bir sonraki yazıda görüşmek dileğiyle.