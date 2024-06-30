---
title:  ".NET Core Ortamında Onion Architecture ile Dapper Uygulaması"
date:   2022-07-15
image: "/images/posts/net-core-ortaminda-onion-architecture-ile-dapper-uygulamasi/index.jpg"
categories: ["Software"]
tags: ["Dapper", "Clean Code", "ORM"]
---

Merhaba Arkadaşlar, bugün Micro-ORM denilince akla gelen Dapper nedir, nasıl kullanılır, neden tercih edilir gibi soruların cevaplarına değineceğim. Dapper tanımını yaptıktan sonra .NET Core ile Onion Architecture kullanarak bir uygulama geliştirerek pratik yapacağız.

## Dapper Nedir?
**Dapper**, bir sql sorgusunun sonucundaki verileri .NET sınıfı ile ilişkilendirmemizi sağlayan bir **Micro-ORM** aracıdır. **Stackoverflow** ekibi tarafından geliştirilmiş olan bu araç, **Entity Framework** performansıyla karşılaştırıldığında verileri sorgulamada kesinlikle daha hızlıdır. Dapper raw sql ile çalıştığı için zaman gecikmesi de bi o kadar azdır. Bu durum Dapper aracının performansını arttırmaktadır.

## Dapper Özellikleri

- Performans açısından hızlı
- Daha az kod ile daha çok iş
- Statik nesne eşleştirme
- Dinamik nesne eşleştirme
- Stored Procedure desteği
- Çoklu sorgu desteği
- Toplu veri ekleme desteği
- Kolaylıkla SQL sorgusu kullanımı
- Kolaylıkla stored prodecure kullanımı

## Ne Zaman Dapper Kullanılmalı?

Var olan yada yeni oluşturacağınız projede Dapper kullanıp kullanılmayacağı kararı, performans ele alınarak verilmelidir. Öyle ki Dapper’ın geliştiricileri Entity framework kullandıklarında Stackoverflow sitesinin trafiği ele alındığında yeteri kadar iyi olmadığını fark edip kendi Micro-ORM aracını geliştirdiler.

Ayrıca Dapper ADO.NET IDbConnection interface’ini kullanmaktadır. Bu sayede ADO.NET’in desteklediği herhangi bir veritabanı ile sorunsuz çalışabilmektedir.

```cs
var sql = "select * from products";
var products = new List<Product>();
using (var connection = new SqlConnection(connString))
{
    connection.Open();
    using (var command = new SqlCommand(sql, connection))
    {
        using (var reader = command.ExecuteReader())
        {
            var product = new Product
            {
                ProductId = reader.GetInt32(reader.GetOrdinal("ProductId")),
                ProductName = reader.GetString(reader.GetOrdinal("ProductName")),
                QuantityPerUnit = reader.GetString(reader.GetOrdinal("QuantityPerUnit")),
                UnitPrice = reader.GetDecimal(reader.GetOrdinal("UnitPrice")),
                UnitsInStock = reader.GetInt16(reader.GetOrdinal("UnitsInStock")),
                CategoryId = reader.GetInt32(reader.GetOrdinal("CategoryId")),
            };
            products.Add(product);
        }
    }
}
```
En basit düzeyde Dapper, 5.satır ile 21.satır arasındaki kodları aşağıdaki şekilde elde etmemizi sağlıyor.

```cs
products = connection.Query<Product>(sql);
```

Hadi o zaman gelin, yavaştan proje altyapısını oluşturalım ve konu anlatımına oradan devam edelim. **DapperORM** adında bir **.Net Core 5** uygulaması oluşturalım ve katmanlarımızı, klasör yapımızı aşağıdaki gibi düzenleyelim.

![](/images/posts/net-core-ortaminda-onion-architecture-ile-dapper-uygulamasi/project-structure.png)

Domain katmanına gelip entity sınıflarımızı oluşturmaya başlayabiliriz.

```cs
[AttributeUsage(AttributeTargets.Property | AttributeTargets.Field,AllowMultiple = true)]
public class DapperIgnoreAttribute:Attribute { }

public class BaseEntity
{
    /// <summary>
    /// Unique Identifier number for each all entities
            settings.Info.Description = "Dapper Tutorial with Clean Architecture";
            settings.Info.Contact = new OpenApiContact
            {
                Email = "oabdullahozturk@yandex.com",
                Name = "Abdullah Öztürk",
                Url = "https://abdullahozturk.live",
            };
            settings.Info.Version = "v1";
        }));
    services.AddPersistenceDependencies();
    services.AddApplicationDependencies();
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }

    app.UseOpenApi();

    app.UseSwaggerUi3();

    app.UseHttpsRedirection();

    app.UseRouting();

    app.UseAuthorization();

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

API dökümantasyonunu oluşturacağımız Swagger için de çeşitli ayarlar yapmış bulunmaktayız. Swagger hakkında kendinizi eksik hissediyorsanız [buradaki](dotnet-core-swagger-entegrasyonu) yazımdan bilgilenebilirsiniz. Extension metotlarımızı da kullandığımıza göre son olarak Controller tarafına geçelim ve yazımızı sonlandıralım.

```cs
[Route("api/products")]
[ApiController]
public class ProductController : ControllerBase
{
    private readonly IMediator mediator;
    public ProductController(IMediator mediator)
    {
        this.mediator = mediator;
    }


    /// <summary>
    /// Get all products
    /// </summary>
    /// <param name="request">Empty request body</param>
    /// <returns>List of products</returns>
    [ProducesResponseType(StatusCodes.Status200OK)]
    [HttpGet("get-all")]
    public async Task<IActionResult> GetAll([FromQuery] GetAllProductQueryRequest request)
    {
        var result = await mediator.Send(request);
        return Ok(result);
    }

    /// <summary>
    /// Get all products by category
    /// </summary>
    /// <param name="request">Category identifier number</param>
    /// <returns>List of products</returns>
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [HttpGet("get-all-by-category")]
    public async Task<IActionResult> GetAllByCategory([FromQuery] GetProductByCategoryQueryRequest request)
    {
        var result = await mediator.Send(request);
        if (result.IsSuccess == false)
            return BadRequest(result.Message);
        return Ok(result);
    }

    /// <summary>
    /// Get product by Id
    /// </summary>
    /// <param name="request">Product identifier number</param>
    /// <returns>A product</returns>
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [HttpGet(":Id")]
    public async Task<IActionResult> Get([FromQuery] GetProductQueryRequest request)
    {
        var result = await mediator.Send(request);
        if (result.IsSuccess == false)
            return BadRequest(result.Message);
        return Ok(result);
    }

    /// <summary>
    /// Add Product to System
    /// </summary>
    /// <param name="request">Product body</param>
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateProductCommandRequest request)
    {
        var result = await mediator.Send(request);
        if (result.IsSuccess == false)
            return BadRequest(result.Message);
        return Ok(result);
    }

    /// <summary>
    /// Delete product from System
    /// </summary>
    /// <param name="request">Product identifier number</param>
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [HttpDelete]
    public async Task<IActionResult> Delete([FromQuery] DeleteProductCommandRequest request)
    {
        var result = await mediator.Send(request);
        if (result.IsSuccess == false)
            return BadRequest(result.Message);
        return Ok(result);
    }

    /// <summary>
    /// Update product in System
    /// </summary>
    /// <param name="request">Product features</param>
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [HttpPut]
    public async Task<IActionResult> Update([FromBody] UpdateProductCommandRequest request)
    {
        var result = await mediator.Send(request);
        if (result.IsSuccess == false)
            return BadRequest(result.Message);
        return Ok(result);
    }
}
```

---

Bu yazımda dapper nedir, özellikleri nedir ve nasıl kullanılır gibi soruların cevaplarını vermeye çalıştık.

Projeyi tam anlamıyla incelemek için github linkine **[buradan](https://github.com/AbdullahOztuurkk/CleanDapper)** ulaşabilirsiniz.

> Okuduğunuz için teşekkür ederim. Bir sonraki yazıda görüşmek dileğiyle.