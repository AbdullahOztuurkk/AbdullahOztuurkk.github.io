---
title:  ".NET Core Ortamında Onion Architecture ile Dapper Uygulaması"
cover: /img/net-core-ortaminda-onion-architecture-ile-dapper-uygulamasi/index.jpg
date:   2022-07-15
tags:
- .Net Core
- Dapper
- Clean Architecture
- Swagger
- CQRS
---

Merhaba Arkadaşlar, bugün Micro-ORM denilince akla gelen Dapper nedir, nasıl kullanılır, neden tercih edilir gibi soruların cevaplarına değineceğim. Dapper tanımını yaptıktan sonra .NET Core ile Onion Architecture kullanarak bir uygulama geliştirerek pratik yapacağız.

## Dapper Nedir?
**Dapper**, bir sql sorgusunun sonucundaki verileri .NET sınıfı ile ilişkilendirmemizi sağlayan bir **Micro-ORM** aracıdır. **Stackoverflow** ekibi tarafından geliştirilmiş olan bu araç, **Entity Framework** performansıyla karşılaştırıldığında verileri sorgulamada kesinlikle daha hızlıdır. Dapper raw sql ile çalıştığı için zaman gecikmesi de bi o kadar azdır. Bu durum Dapper aracının performansını arttırmaktadır.

<!-- more -->
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

| ![][project-infrastructure] | 
|:--:| 
| Onion Architecture kullanarak oluşturduğumuz proje altyapısı |

Domain katmanına gelip entity sınıflarımızı oluşturmaya başlayabiliriz.

```cs
[AttributeUsage(AttributeTargets.Property | AttributeTargets.Field,AllowMultiple = true)]
public class DapperIgnoreAttribute:Attribute { }

public class BaseEntity
{
    /// <summary>
    /// Unique Identifier number for each all entities
    /// </summary>
    public int Id { get; set; }
}
		
public class Category : BaseEntity
{
    public Category()
    {
        Products = new List<Product>();
    }
    public string Name { get; set; }
    public string Description { get; set; }

    //Navigation property
    [DapperIgnore]
    public ICollection<Product> Products { get; set; }
}
		
public class Product : BaseEntity
{
    public string Name { get; set; }
    public int QuantityPerUnit { get; set; }
    public int UnitPrice { get; set; }
    public string Color { get; set; }
    public int UnitsInStock { get; set; }

    //Foreign Key Property
    public int CategoryId { get; set; }
}
```
Entity sınıflarımızda **DapperIgnore** adında bir Attribute kullandık. Bunun sebebi generic dapper repository tasarlarken reflection ile elde edeceğimiz property listesinde, tablolar arası ilişki için kullandığımız property değerlerinin kullanılmayacak olması. Bu kısma generic repository yazarken tekrardan değineceğiz.

---

Application katmanına gelelim ve Interfaces klasörü altındaki gerekli dosyaları oluşturalım.

```cs
public interface IDapperContext
{
    public SqlConnection GetConnection();
    public void Execute(Action<SqlConnection> @event);
}

public interface IGenericRepository<T> where T: BaseEntity
{
    T Get(int id);
    void Add(T entity);
    void Update(T entity);
    void Delete(T entity);
    List<T> GetAll();
}

public interface IProductRepository:IGenericRepository<Product>
{
    public List<Product> GetProductByCategoryId(int Id);
}

public interface ICategoryRepository:IGenericRepository<Category> { }
```

**DapperContext** adındaki sınıfımız bu IDapperContext interface’ini kullanarak bize generic repository tasarlarken temiz kod yazma imkanı verecek.

**Validations** klasörü altında Common adlı klasör oluşturalım ve ortak kurallar olmak üzere bakımı kolay, tekrar kullanılabilir bir yapı oluşturalım.

```cs
public static class ProductRules
{
    public static IRuleBuilderOptions<T, string> CheckProductName<T>(this IRuleBuilder<T, string> ruleBuilder) where T: Product
    {
        return ruleBuilder
            .NotEmpty().WithMessage(ValidationMessages.Product_Name_Length_Error)
            .MinimumLength(3).WithMessage(ValidationMessages.Product_Name_Length_Error)
            .MaximumLength(70).WithMessage(ValidationMessages.Product_Name_Length_Error);

    }

    public static IRuleBuilderOptions<T, string> CheckProductColor<T>(this IRuleBuilder<T, string> ruleBuilder) where T : Product
    {
        return ruleBuilder
            .NotEmpty().WithMessage(ValidationMessages.Product_Color_Must_be_Known_Color)
            .IsEnumName(typeof(KnownColor)).WithMessage(ValidationMessages.Product_Color_Must_be_Known_Color);
    }

    public static IRuleBuilderOptions<T, int> CheckProductQuantity<T>(this IRuleBuilder<T, int> ruleBuilder) where T : Product
    {
        return ruleBuilder
            .NotEmpty().WithMessage(ValidationMessages.Product_Quantity_Must_Greater_Than_Zero)
            .GreaterThan(0).WithMessage(ValidationMessages.Product_Quantity_Must_Greater_Than_Zero);
    }

    public static IRuleBuilderOptions<T, int> CheckProductPrice<T>(this IRuleBuilder<T, int> ruleBuilder) where T : Product
    {
        return ruleBuilder
            .NotEmpty().WithMessage(ValidationMessages.Product_Price_Must_Greater_Than_Or_Equal_To_Zero)
            .GreaterThan(0).WithMessage(ValidationMessages.Product_Price_Must_Greater_Than_Or_Equal_To_Zero);
    }

    public static IRuleBuilderOptions<T, int> CheckProductStock<T>(this IRuleBuilder<T, int> ruleBuilder) where T : Product
    {
        return ruleBuilder
            .NotEmpty().WithMessage(ValidationMessages.Product_Stock_Must_Greater_Than_Zero)
            .GreaterThan(0).WithMessage(ValidationMessages.Product_Stock_Must_Greater_Than_Zero);
    }

    public static IRuleBuilderOptions<T, int> CheckProductCategoryId<T>(this IRuleBuilder<T, int> ruleBuilder) where T : Product
    {
        return ruleBuilder
            .NotEmpty().WithMessage(ValidationMessages.Product_Category_Id_Cannot_Be_Empty)
            .GreaterThan(0).WithMessage(ValidationMessages.Product_Category_Id_Cannot_Be_Empty);

    }
}
```

Bu sınıfta Product nesnemize ait her bir property için extension metot oluşturduk. Bu sayede gerek silme, gerek güncelleme işlemlerinde kullanacağımız validation sınıflarında daha temiz bir kod altyapısı oluşturabileceğiz. Örnek kullanım açısından **CreateProductValidator** sınıfı aşağıdaki gibi görünmektedir.

```cs
public class CreateProductValidator : AbstractValidator<Product>
{
    public CreateProductValidator()
    {
        RuleFor(p => p.Name).CheckProductName();
        RuleFor(p => p.Color).CheckProductColor();
        RuleFor(p => p.QuantityPerUnit).CheckProductQuantity();
        RuleFor(p => p.UnitPrice).CheckProductPrice();
        RuleFor(p => p.UnitsInStock).CheckProductStock();
        RuleFor(p => p.CategoryId).CheckProductCategoryId();
    }
}
```

**Features** adlı bir klasör oluşturalım ve burada CRUD işlemleri için Command ve Query sınıflarını MediatR kullanarak oluşturalım. CQRS kullanarak oluşturduğumuz handler ve request yapılarına uzaksanız buradaki [yazımdan][mediatr-article] bilgi edinebilirsiniz.Örnek olarak **CreateProductCommandHandler** sınıfımız aşağıdaki gibi görünmektedir.

```cs
public class CreateProductCommandRequest : IRequest<IResult>
{
    public string Name { get; set; }
    public int QuantityPerUnit { get; set; }
    public int UnitPrice { get; set; }
    public string Color { get; set; }
    public int UnitsInStock { get; set; }
    public int CategoryId { get; set; }
}

public class CreateProductCommandHandler : IRequestHandler<CreateProductCommandRequest, IResult>
{
    private readonly IProductRepository productRepository;
    private readonly IMapper mapper;
    private readonly CreateProductValidator validator;
    public CreateProductCommandHandler(
        IProductRepository productRepository,
        CreateProductValidator validator,
        IMapper mapper)
    {
        this.productRepository = productRepository;
        this.validator = validator;
        this.mapper = mapper;
    }
    public Task<IResult> Handle(CreateProductCommandRequest request, CancellationToken cancellationToken)
    {
        Product product = mapper.Map<Product>(request);
        var result = validator.Validate(product);
        if (result.Errors.Any())
            return Task.FromResult<IResult>(new ErrorResult(result.Errors.First().ErrorMessage));
        productRepository.Add(product);
        return Task.FromResult<IResult>(new SuccessResult(ResultMessages.Product_Added));
    }
}
```
Request tipinde alacağımız nesneyi product sınıfına map ettikten sonra gerekli validate işlemlerini gerçekleştiriyoruz. Bir hata olması durumunda **ErrorResult** döndürüyor, işler tıkırında gittiğinde ise nesnemizi veritabanına ekliyor ve bununla ilgili bir mesaj içeren **SuccessResult** tipinde bir veri döndürüyoruz.

```cs
public static void AddApplicationDependencies(this IServiceCollection services)
{
    services.AddMediatR(Assembly.GetExecutingAssembly());
    services.AddAutoMapper(Assembly.GetExecutingAssembly());
    services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());
}
```

Bu katmanda son olarak yukarıdaki gibi servisler arası bağlantımızı sağlayacak extension metot oluşturalım.

---

Artık yavaştan **Persistence** katmanına geçelim ve bizim için oldukça önemli olan dapper generic repository sınıfımızı tasarlayalım. Öncelikle repository sınıfımızda oldukça kullanacağımız **DapperContext** sınıfını, Context klasörü altında oluşturarak başlayalım.

```cs
public class DapperContext : IDapperContext
{
    IConfiguration configuration;
    public DapperContext(IConfiguration configuration)
    {
        this.configuration = configuration;
    }
    public void Execute(Action<SqlConnection> @event)
    {
        using (var connection = GetConnection())
        {
            connection.Open();
            @event(connection);
        }
    }

    public SqlConnection GetConnection()
    {
        return new SqlConnection(configuration.GetConnectionString("DefaultConnection"));
    }
}
```

Constructor içerisine **IConfiguration** nesnesi alıyoruz. Bu sayede **GetConnection** metodundan yeni bir SqlConnection bağlantısı oluşturabiliyoruz. Dapper tarafında oluşturacağımız her bir komut öncesi bağlantı açmamız gerektiği için **Execute** adında bir metot oluşturup parametre olarak oluşturduğumuz bağlantı nesnesini kullanan bir fonksiyon kabul ettiğini belirtiyoruz. Generic Repository sınıfını açıklarken bu dediğimi daha iyi anlayacaksınız.

```cs
public abstract class DapperGenericRepository<T> : IGenericRepository<T> where T : BaseEntity
{
    public IDapperContext dapperContext;
    private string tableName;
    public DapperGenericRepository(IDapperContext dapperContext, string tableName)
    {
        this.dapperContext = dapperContext;
        this.tableName = tableName;
    }
    
    private IEnumerable<string> GetColumns()
    {
        return typeof(T)
                .GetProperties()
                .Where(e => e.Name != "Id" 
                && !e.PropertyType.GetTypeInfo().IsGenericType 
                && !Attribute.IsDefined(e,typeof(DapperIgnoreAttribute)))
                .Select(e => e.Name);
    }

    public void Add(T entity)
    {
        var columns = GetColumns();
        var stringOfColumns = string.Join(", ", columns);
        var stringOfParameters = string.Join(", ", columns.Select(e => "@" + e));
        var query = $"insert into {tableName} ({stringOfColumns}) values ({stringOfParameters})";
        
        dapperContext.Execute((conn) => {
            conn.Execute(query, entity);
        });
    }

    public void Delete(T entity)
    {
        var query = $"delete from {tableName} where Id = @Id";

        dapperContext.Execute((conn) => {
            conn.Execute(query, entity);
        });
    }

    public void Update(T entity)
    {
        var columns = GetColumns();
        var stringOfColumns = string.Join(", ", columns.Select(e => $"{e} = @{e}"));
        var query = $"update {tableName} set {stringOfColumns} where Id = @Id";

        dapperContext.Execute((conn) => {
            conn.Execute(query, entity);
        });
    }

    public T Get(int id)
    {
        var query = $"select * from {tableName} where Id = @Id ";

        using(var conn = dapperContext.GetConnection()) {
            conn.Open();
            return conn.QueryFirst<T>(query);
        }
    }

    public List<T> GetAll()
    {
        var query = $"select * from {tableName}";

        using (var conn = dapperContext.GetConnection()) {
            conn.Open();
            return (List<T>)conn.Query<T>(query);
        }
    }
}
```
- 5.satırda constructor tarafında iki parametre alıyoruz. Bunlardan biri IoC tarafınca oluşturulacak IDapperContext instance, diğeri ise entity sınıflarının veritabanındaki tablo ismi.

- 10.satırda GetColumns metodu, T olarak gönderilen Entity sınıfının Id kolonu ve DapperIgnore attribute’ü tarafından işaretlenmiş kolonların haricinde kalan property listesini elde etmemizi sağlıyor.

- Add ve Update fonksiyonlarında gördüğünüz üzere GetColumns metodundan elde ettiğimiz her bir property, sql cümleciğimize parametre olarak ekleniyor.

- Get ve GetAll fonksiyonlarında sql cümleciğini düzenleme sonrasında elde edilen değer geriye döndürülüyor. Burada DapperContext sınıfının Execute metodunu kullanmadık çünkü diğer fonksiyonların aksine burada geri dönüş değişkenimizin olması gerekmekte.

```cs
public class DapperCategoryRepository: DapperGenericRepository<Category>,ICategoryRepository
{
    public DapperCategoryRepository(IDapperContext dapperContext):base(dapperContext,"Categories") { }
}

public class DapperProductRepository : DapperGenericRepository<Product>, IProductRepository
{
    public DapperProductRepository(IDapperContext dapperContext) : base(dapperContext, "Products") { }

    public List<Product> GetProductByCategoryId(int Id)
    {
        var query = $"select * from Products where CategoryId = {Id} ";

        using (var conn = dapperContext.GetConnection())
        {
            conn.Open();
            return (List<Product>)conn.Query<List<Product>>(query);
        }
    }
}
```

Product ve Category sınıflarımız için iki repository sınıfımızı da elde etmiş olduk. Bu katmanda son olarak aşağıdaki gibi servisler arası bağlantımızı sağlayacak extension metot oluşturalım ve artık API tarafına geçerek projemizi sonlandıralım.

```cs
public static void AddPersistenceDependencies(this IServiceCollection services)
{
    services.AddScoped<IProductRepository, DapperProductRepository>();
    services.AddScoped<ICategoryRepository, DapperCategoryRepository>();
    services.AddScoped<IDapperContext, DapperContext>();
}
```

---

API tarafında Startup.cs dosyasını açıyoruz ve aşağıdaki gibi düzenliyoruz.

```cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers();
    services.AddSwaggerDocument(config =>
        config.PostProcess = ( settings => {
            settings.Info.Title = "DapperORM.WebApi";
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

API dökümantasyonunu oluşturacağımız Swagger için de çeşitli ayarlar yapmış bulunmaktayız. Swagger hakkında kendinizi eksik hissediyorsanız [buradaki][swagger-article] yazımdan bilgilenebilirsiniz. Extension metotlarımızı da kullandığımıza göre son olarak Controller tarafına geçelim ve yazımızı sonlandıralım.

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

Projeyi tam anlamıyla incelemek için github linkine **[buradan][project-link]** ulaşabilirsiniz.

> Okuduğunuz için teşekkür ederim. Bir sonraki yazıda görüşmek dileğiyle.

<!-- Links -->
[project-infrastructure]: /img/net-core-ortaminda-onion-architecture-ile-dapper-uygulamasi/project-structure.png
[project-link]: https://github.com/AbdullahOztuurkk/CleanDapper
[swagger-article]: /dotnet-core-swagger-entegrasyonu
[mediatr-article]: /command-query-responsibility-segregation-nedir