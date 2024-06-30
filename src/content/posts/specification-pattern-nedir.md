---
title:  "Specification Pattern Nedir?"
date:   2022-09-03
image: "/images/posts/specification-pattern-nedir/index.jpg"
categories: ["Software"]
tags: ["Dotnet", "Design Patterns"]
---

Herkese merhaba, bugün specification tasarım deseninin ne olduğunu ve hangi durumlarda kullanılacağı, bizi nasıl bir yükten kurtaracağını ve son olarak nasıl implemente edileceğinden bahsedeceğim.

**Domain Driven Design** tarafındaki makalelere bakarsak hepsinin ortak bir anlattığı konu, karışık iş süreçlerini ortak bir dil (**ubiqutious language**) aracılığıyla çözüme kavuşturmak. Bu süreçte
yazılımcıların daha kolay bir şekilde geliştirme yapmasını sağlayan desendir **Specification**.

**Specification**: Belirli bir domain kuralını tek bir birim -specification- olarak soyutlayıp farklı senaryolar için kullanmamıza ve farklı domain kuralları ile birleştirebilmemize olanak sağlayan bir yapı.

Örnek üzerinden gidelim ve neden ihtiyaç duyacağımızı anlayalım.

Bir proje yönetim sistemi üzerinde çalıştığınızda hizmet sağladığınız departman sizden inaktif durumdaki task listesini isteyebilir.

Task modeli aşağıdaki gibi property’lere sahip olsun.

```cs
public class Task
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Description { get; set; }
    public bool IsClosed { get; set; }
    public string? ClosedReason { get; set; } = null;
    public int? AssignedUserId { get; set; }
    public DateTime? LastCommentDate { get; set; }
    public DateTime CreationDate { get; set; }

    public bool IsInActive()
    {
        DateTime daysAgo30 = DateTime.Now.Subtract(TimeSpan.FromDays(30));
        return !IsClosed &&
                AssignedUserId == null &&
                CreationDate < daysAgo30 &&
                ( LastCommentDate < daysAgo30 || LastCommentDate == null );
    }
}
```
Genelde bu gibi bir işlemi, **IsInActive** metotu ile basitçe çözebiliriz.

Bu noktaya kadar her şey güzel.

Daha sonra farklı bir noktada ise taskların, aktif olup olmadıklarını ve aktif ise son yorum tarihinin 1 haftadan önce olanları kontrol etme ihtiyacımızın olduğunu düşünelim. Bu işlemi de aşağıdaki gibi çözebiliriz.

```cs
DateTime daysAgoOneWeek = Datetime.Now.Substract(TimeSpan.FromDays(7));
if(!task.IsInActive && task.CreationDate < daysAgoOneWeek) {
   //logic
}
```

Bu örnekte olduğu gibi birçok kez domain kuralını sürekli farklı yerlerde kullanmak zorunda kalacağız ve **DRY** prensibini uygulayamamış olacağız. İşte bu noktada **Specification Pattern** yardımıyla tüm domain kurallarımızı tek bir yerden tekrar kullanabilir hale getirebileceğiz. Ayrıca bu domain kurallarını birleştirip farklı bir domain kuralı da oluşturabileceğiz.

Bu sayede daha merkezi bir yapı oluşturarak daha az kod ile daha çok iş yapabileceğiz. Ayrıca ekibe yeni gelen birisinin de alışması bir o kadar kolay olacaktır.

--- 

# Kodlama Vakti

Öncelikle ISpecification adlı bir interface üretelim.
```cs
public interface ISpecification<T>{
    bool IsSatisfiedBy(T entity);
}
```
Tüm specification sınıfları için genel bir interface oluşturduk. Şimdi base specification sınıfı oluşturma vakti.

```cs
public abstract class CompositeSpecification<T> : ISpecification<T>     
{
    public abstract bool IsSatisfiedBy(T entity);

    public abstract Expression<Func<T, bool>> GetExpression();

    public ISpecification<T> And(ISpecification<T> specification)       
    {
        return new AndSpecification<T>(this, specification);
    }
    public ISpecification<T> Or(ISpecification<T> specification)        
    {
        return new OrSpecification<T>(this, specification);
    }
}
```

```cs
public class NotSpecification<T> : CompositeSpecification<T>
{
    public ISpecification<T> spec;
    public NotSpecification(ISpecification<T> spec)
    {
        this.spec = spec;
    }
    public override bool IsSatisfiedBy(T obj)
    {
        return !spec.IsSatisfiedBy(obj);
    }
}
```

Base sınıfımızı da oluşturduğumuza göre artık bu specification sınıflarından nasıl specification  oluşturabileceğinizi göstermek istiyorum.

Öncelikle Task modelimizin aktif olup olmadığını kontrol eden specification sınıfını oluşturalım.

```cs
public class ActiveTaskSpecification : CompositeSpecification<Models.Task>
{
    public override bool IsSatisfiedBy(Models.Task obj)
    {
        DateTime daysAgo30 = DateTime.Now.Subtract(TimeSpan.FromDays(30));
        return !obj.IsClosed &&
                obj.CreationDate > daysAgo30 &&
                (obj.LastCommentDate > daysAgo30 || obj.LastCommentDate == null );
    }
}
```
Kullanıcıya ait taskları getirmemizi sağlayan specification da oluşturalım.

```cs
public class GetAssignedUserTaskSpecification : CompositeSpecification<Models.Task>
{
    private int UserId { set; get; }

    public GetAssignedUserTaskSpecification(int userId) => UserId = userId;

    public override bool IsSatisfiedBy(Models.Task obj) => obj.AssignedUserId == UserId;
}
```
Bir Task nesnesinin aktif olup olmadığını kontrol eden bir spesifikasyon ve bir kullanıcıya assign edilen taskları getiren bir spesifikasyonumuz var. Gelin bu ikisini birleştirerek kullanıcıya assign edilen aktif taskları bulalım.

```cs
public class GetActiveUserTaskSpecification : CompositeSpecification<Models.Task>
{
    private readonly int userId;
    public GetActiveUserTaskSpecification(int userId)
    {
        this.userId = userId;
    }
    public override bool IsSatisfiedBy(Models.Task obj)
    {
        var activeSpec = new ActiveTaskSpecification();
        var userTaskSpec = new GetAssignedUserTaskSpecification(userId);

        return (activeSpec.And(userTaskSpec)).IsSatisfiedBy(obj);
    }
}
```
Elimizdeki bu specification sınıfını kullanmadan önce modelimizi yeni haliyle güncelleyelim.

```cs
public class Task
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Description { get; set; }
    public bool IsClosed { get; set; }
    public string? ClosedReason { get; set; } = null;
    public int? AssignedUserId { get; set; }
    public DateTime? LastCommentDate { get; set; }
    public DateTime CreationDate { get; set; }

    public bool IsInActive()
    {
        var spec = new InActiveTaskSpecification();
        return spec.IsSatisfiedBy(this);
    }
}
```
Gördüğünüz gibi specification kullanarak DRY prensibini çiğnemeyerek daha merkezi bir yapı oluşturmuş olduk. Şimdi ise elimizde List<Models.Task> olsun ve bunun üzerinde işlem yapalım.

```cs
[ApiController]
[Route("task")]
public class TaskController : ControllerBase
{
    public List<Models.Task> Tasks = new List<Models.Task>
    {
        new Models.Task
        {
            Title = "Test Task 1",
            CreationDate = DateTime.Now.Subtract(TimeSpan.FromDays(15)),
            Description = "Test Description 1",
            AssignedUserId = 1,
        },
        new Models.Task
        {
            Title = "Test Task 2",
            CreationDate = DateTime.Now.Subtract(TimeSpan.FromDays(35)),
            Description = "Test Description 2",
            AssignedUserId = 1,

        },
        new Models.Task
        {
            Title = "Test Task 3",
            CreationDate = DateTime.Now.Subtract(TimeSpan.FromDays(25)),
            Description = "Test Description 3",
            AssignedUserId = 2,

        },
        new Models.Task
        {
            Title = "Test Task 4",
            CreationDate = DateTime.Now.Subtract(TimeSpan.FromDays(45)),
            Description = "Test Description 4",
            AssignedUserId = 3,

        },
    };

    [HttpGet]
    public List<Models.Task> Get()
    {
        List<Models.Task> inActiveTasks = new List<Models.Task>();
        var spec = new GetActiveUserTaskSpecification(1);
        foreach (var item in Tasks)
        {
            if (spec.IsSatisfiedBy(item))
                inActiveTasks.Add(item);
        }
        return inActiveTasks;
    }
}
```
Dummy bir liste oluşturdum ve bunun üzerinden işlem yapacağız. IIS Express üzerinden ayağa kaldırdıktan sonra swagger üzerinden istek yapıyorum ve response modelimiz aşağıdaki şekilde dönüyor.

<p align="center">
    <img src="/images/posts/specification-pattern-nedir/postman-output.png"/>
</p>

---

Bu yazımda specification nedir, tam olarak ne işe yarar ve zincirleme specification implementasyonu gibi konuları anlatmaya çalıştım.

Okuduğunuz için teşekkür ederim. Bir sonraki yazıda görüşmek dileğiyle.