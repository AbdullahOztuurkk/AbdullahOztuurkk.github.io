---
title:  "Unit Of Work Pattern Nedir?"
cover: /img/unit-of-work-pattern-nedir/index.png
date:   2020-12-27
tags:
- .Net Core
- Unit Of Work
---

Merhaba Arkadaşlar, bu makalemde generic repository ile Unit of Work tasarım deseni nasıl kullanılır , ne işe yarar gibi konularını işleyeceğiz.

**Generic Repository** : Oluşturduğumuz bu sınıf , veritabanında karşılığı olan tüm tablolar için genel bir CRUD işlemlerini yapabilmemizi sağlar. Bu sayede kod okunabilirliğini arttırmışken DRY (Don’t Repeat Yourself) yazılım geliştirme ilkesini de çiğnememiş oluruz.

**Unit Of Work** : Veritabanı ile ilgili tüm işlemlerin tek kanaldan yapılmasını sağlayan ve yapılan tüm işlemlerin hafızada tutularak toplu halde gerçekleştirilmesini sağlayan bir tasarım desenidir.

O zaman gelin , beraber altyapıyı kuralım. Öncelikle çalışmak istediğimiz klasörde terminali başlatıp aşağıdaki komutu girerek “UnitOfWorkApp” adlı bir konsol uygulaması oluşturuyoruz.

```bash
    dotnet new console -n UnitOfWorkApp
```
<!-- more -->
Solution içerisinde 2 adet klasörümüz bulunmakta. Bunlardan biri model sınıflarımızın olduğu “Entities” klasörü , diğeri de veritabanı ile ilgili sınıfların olduğu “Data” klasörümüz. Makale bitiminde aşağıdaki gibi bir hiyerarşik yapı oluşturmuş olacağız.

![Klasör Yapısı](/img/unit-of-work-pattern-nedir/solution-structure.png)

“Entities” klasörüne gelip model sınıflarımızı oluşturmaya başlayalım.

```csharp
namespace UnitOfWorkApp.Entities
{
    public class BaseEntity
    {
        public int Id { get; set; }
        public DateTime CreatedAt  => DateTime.Now;
    }
}

public class Department:BaseEntity
{
    public Department()
    {
        Employees = new List<Employee>();
    }
    public string Name { get; set; }
    public string Location { get; set; }

    public List<Employee> Employees { get; set; }
}

namespace UnitOfWorkApp.Entities
{
    public class Employee:BaseEntity
    {
        public string FirstName { get; set; }
        public string LastName { get; set; }
        public int Salary { get; set; }
        public string PhoneNumber { get; set; }
        public string EmailAddress { get; set; }
        public string Location { get; set; }

        public int DepartmentId { get; set; }
        public Department Department { get; set; }
    }
}
```
“Data” klasörüne geçelim ve context sınıfımızı yazalım.

```csharp
namespace UnitOfWorkApp.Data
{
    public class UoWContext:DbContext
    {
        public DbSet<Employee> Employees { get; set; }
        public DbSet<Department> Departments { get; set; }

        protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        {
            optionsBuilder.UseSqlServer("Data Source=DESKTOP-2QF0S4K;Initial Catalog=UoWDb;Integrated Security=True;");
            base.OnConfiguring(optionsBuilder);
        }
    }
}
```
“Data” klasörünün içerisinde repository sınıfımızı yazalım.

```csharp
namespace UnitOfWorkApp.Data
{
    public interface IRepository<T>
    {
        IEnumerable<T> GetAll();
        T Get(Expression<Func<T, bool>> predicate);

        void Add(T entity);
        void Update(T entity);
        void Delete(T entity);
        void Delete(int id);
    }

    public class Repository<T>:IRepository<T> where T: BaseEntity // Generic repository sınıfımızda sadece BaseEntity sınıfından kalıtılanlar kullanılabilir
    {
        private UoWContext context;
        public Repository(UoWContext context)
        {
            this.context = context;
        }
        public IEnumerable<T> GetAll()
        {
            return context.Set<T>().ToList();
        }

        public T Get(Expression<Func<T, bool>> predicate)
        {
            return context.Set<T>().SingleOrDefault(predicate);
        }

        public void Add(T entity)
        {
            var currentEntity = context.Entry(entity);
            currentEntity.State = EntityState.Added;
        }

        public void Update(T entity)
        {
            var currentEntity = context.Entry(entity);
            currentEntity.State = EntityState.Modified;

        }

        public void Delete(T entity)
        {
            var currentEntity = context.Entry(entity);
            currentEntity.State = EntityState.Deleted;
        }

        public void Delete(int id)
        {
            var currentEntity = context.Set<T>().SingleOrDefault(z => z.Id == id);
            context.Set<T>().Remove(currentEntity);
        }
    }
}
```
Aynı klasörün içerisine “UnitOfWork” klasörü açalım ve gerekli sınıfları yazalım.

```cs
namespace UnitOfWorkApp.Data.UnitOfWork
{
    public interface IUnitOfWork
    {
        public int Commit();
    }

    public class UnitOfWork:IUnitOfWork
    {
        private UoWContext context;

        public IRepository<Employee> EmployeeRepository { get; private set; }

        public IRepository<Department> DepartmentRepository { get; private set; }

        public UnitOfWork(UoWContext _context)
        {
            this.context = _context;
            EmployeeRepository=new Repository<Employee>(context);
            DepartmentRepository=new Repository<Department>(context);
        }
        public int Commit()
        {
            return context.SaveChanges();
        }
    }
}
```
Şimdi ise veritabanını oluşturması için Package Manager Console’u açıyorum ve sırasıyla aşağıdaki komutları giriyorum.

```bash
Enable-Migrations 
Add-Migration Init
Update-Database -verbose
```
> **Dikkat** : Migration işlemi yaparken hata alırsanız EntityFrameworkCore.Tools ve EntityFramework.Design paketlerinin yüklü olduğundan emin olunuz.

Artık genel altyapıyı kurduk. Gelin beraber ana sınıfımızda testimizi yapalım.

```cs
namespace UnitOfWorkApp
{
    class Program
    {
        static void Main(string[] args)
        {
            UnitOfWork unitOfWork=new UnitOfWork(new UoWContext());

            unitOfWork.DepartmentRepository.Add(new Department
            {
                Location = "İstanbul",
                Name = "Bilgi İşlem"
            });//Ram'de tutuluyor

            unitOfWork.EmployeeRepository.Add(new Employee()
            {
                DepartmentId = 1,
                FirstName = "Abdullah",
                LastName = "Öztürk",
                EmailAddress = "oabdullahozturk@yandex.com.tr",
                Location = "İstanbul",
            });//Ram'de tutuluyor

            unitOfWork.Commit();//Veritabanına kaydedildi.

            Console.ReadKey();
        }
    }
}
```
Bu makalemde temel düzeyde unit of work tasarım deseninin kurumsal mimaride nasıl implemente edileceğini gördük.

Projenin github linkine [buradan](https://github.com/AbdullahOztuurkk/UnitOfWorkTutorial) ulaşabilirsiniz.