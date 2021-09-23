---
# Common-Defined params
title: "CQRS ( Command Query Responsibility Segregation) Nedir?"
date: "2021-01-21"
categories:
  - "dotnet"
  - "cqrs"

# Theme-Defined params
thumbnail: "img/command-query-responsibility-segregation-nedir/index.png" # Thumbnail image
clauthorbox: true # Enable authorbox for specific page
pager: true # Enable pager navigation (prev/next) for specific page
toc: true # Enable Table of Contents for specific page
---

Herkese merhaba,

Bu makalemde CQRS pattern kullanımına, ne olduğuna, nasıl kullanıldığına ve ne zaman kullanılması gerektiği gibi konuların üzerinde duracağız.

**Command Query Responsibility Segregation** yani kısaca **CQRS**, isminden de anlayacağımız üzere komut ve sorguların ayrıştırılması prensibine dayanıyor. Bu prensibe göre “ bir metot, bir nesnenin durumunu değiştirmelidir yada geriye değer döndürmelidir. İkisini de aynı anda yapmamalıdır. ”

**Command** : Bir nesnenin durumunu değiştiren işlemlerdir.
**Query** : Tetiklenen bir olay sonucu geriye bir değer döndüren işlemlerdir.

### Peki CQRS Ne Zaman Kullanılmalı?

- Büyük sistemlerde yazma ve okuma sayısı arasında büyük fark olması mümkündür. ( Sosyal medya ve e-ticaret bu sistemlere örnektir. ) Bu durumda iki tarafı da bağımsız olarak scale edebiliriz.
- Performans kritik düzeyde olduğunda CQRS ile okuma ve yazma kısımlarını optimize edebiliriz.
- Karışık iş kurallarının olduğu bir senaryoda problemi komut ve sorgulara bölerek işi kolaylaştırabiliriz.
- Geliştirme yaptığınız projede iki takım olarak çalışılacaksa yazma kısmına bir takım , okuma kısmına bir takım atayabiliriz. Ancak iki takımın da CQRS pattern hakkında bilgi sahibi olması beklenir.
- Zaten hali hazırda event odaklı uygulama geliştiriyorsanız yazma tarafı, mevcut kısmı bir dizi olay olarak kaydettiğinden CQRS ile birlikte kullanılabilir.

### CQRS Ne Zaman Kullanılmamalı?

- Basit bir kullanıcı arayüzüne sahip bir uygulama geliştiriyorsanız ( CRUD işlemleri yapan bir uygulama gibi )
- Eğer basit iş kurallarınız varsa CQRS implemente etmeniz kafa karıştırıcı olacaktır.
- Tüm sistem için kullanılmamalıdır. Sistemin her bir parçası için CQRS desenini kullanmak zorunda değilsiniz.

---

.Net Core ile bu mimari deseni uygulamak istediğimizde karşımıza daha önce kullananların tahmin edebileceği gibi **MediatR** kütüphanesi çıkıyor. **MediatR** sayesinde bu mimari deseni uygulayabileceğiz. Öncelikle MediatR kütüphanesinin nasıl kullanıldığına girmeden **mediator pattern** nedir, ondan bahsedelim.

### Mediator Pattern Nedir ?

Türkçesi arabulucu anlamına gelen bu pattern genel olarak nesnelerin yönetimi, aralarındaki iletişimin merkezi bir noktadan sağlanması ve yönetilmesi için kullanılır. En sık verilen örneklerden biri ise havaalanındaki uçakların birbiri arasındaki iletişimini sağlayan kule yapısıdır. Uçaklar birbirleri ile konuşmaz, kule ile iletişime geçer ve kule üzerinden gerekli işlemler yapılır.

### Net Core Web Api Oluşturma

Projemizi yapmaya başlayalım. Terminalimi açıp sırasıyla aşağıdaki komutları girelim.

```bash
dotnet new webapi -n NetCoreMediatRExample
cd NetCoreMediatRExample
dotnet add package MediatR -version 9.0.0
dotnet add package MediatR.Extensions.Microsoft.DependencyInjection -version 9.0.0
code .
```
NetCoreMediatRExample isminde bir proje oluşturdum ve gerekli kütüphaneleri indirdikten sonra visual studio code editöründe açtım.

```csharp
public void ConfigureServices(IServiceCollection services)
{
        services.AddControllers();
        services.AddMediatR(typeof(Startup));
        services
                .AddScoped<BaseRepository<Player>>()
                .AddScoped<BaseRepository<Game>>()
                .AddDbContext<MediatorContext>();
}
```
Startup.cs üzerinde gerekli değişikleri yaptık. Şimdi direk ApiController kısmına bakalım ve kodlar üzerinden açıklamalar yapalım. (Projenin geri kalan kısmını aşağıdaki github linkinden inceleyebilirsiniz)

Entity olarak Player ve Game sınıflarını kullanalım. Command ve Query yapılarının birbirinden ayrılacağını bildiğimize göre PlayerQuery, PlayerCommand, GameQuery ve GameCommand sınıflarını oluşturalım.

```cs
namespace NetCoreMediatRExample.Data.Queries
{
    public class GetPlayerQuery : IRequest<Player>
    {
        public int Id { get; private set; }
        public GetPlayerQuery(int id)
        {
            Id = id;
        }
    }
}

namespace NetCoreMediatRExample.Data.Commands
{
    public class CreatePlayerCommand:IRequest
    {
        public CreatePlayerCommand(Player request)
        {
            Player = request;
        }

        public Player Player { get; set; }
    }
}
```
> GetPlayerQuery ve CreatePlayerCommand Sınıfları

```cs
namespace NetCoreMediatRExample.Data.Queries
{
    public class GetGameQuery:IRequest<Game>
    {
        public int Id { get; private set; }

        public GetGameQuery(int id)
        {
            Id = id;
        }
    }
}

namespace NetCoreMediatRExample.Data.Commands
{
    public class CreateGameCommand:IRequest
    {
        public Game Game { get; set; }

        public CreateGameCommand(Game game)
        {
            this.Game = game;
        }
    }
}
```
> GetGameQuery ve CreateGameCommand Sınıfları

Yukarıda sınıflarımızı tanımladık. Şimdi teker teker açıklayalım. Command ve Query sınıflarımız gördüğünüz gibi IRequest arayüzünden türetilmiştir. Bu arayüz ile beraber, eğer geriye bir değer döndürülecekse, sınıfımız için dönüş tipini belirtiyoruz.

Şimdi ise Handler sınıflarımızı yazacağız.

```cs
namespace NetCoreMediatRExample.Data.Handlers
{
    public class GetPlayersHandler:IRequestHandler<GetPlayersQuery,List<Player>>
    {
        private BaseRepository<Player> playerRepository;
        public GetPlayersHandler(BaseRepository<Player> playerRepository)
        {
            this.playerRepository = playerRepository;
        }
        public async Task<List<Player>> Handle(GetPlayersQuery request, CancellationToken cancellationToken)
        {
            var playerList = await playerRepository.GetAllAsync();
            return playerList;
        }
    }
}

namespace NetCoreMediatRExample.Data.Handlers
{
    public class CreatePlayerHandler : IRequestHandler<CreatePlayerCommand>
    {
        private BaseRepository<Player> playerRepository;

        public CreatePlayerHandler(BaseRepository<Player> playerRepository)
        {
            this.playerRepository = playerRepository;
        }

        public async Task<Unit> Handle(CreatePlayerCommand request, CancellationToken cancellationToken)
        {
            await playerRepository.AddAsync(request.Player);
            return await Task.FromResult(Unit.Value);
        }
    }
}
```
> CreatePlayerHandler ve GetPlayersHandler Sınıfları

Tüm Handler sınıfları IRequestHandler arayüzünden türetilir ve ilk parametresi tetiklenecek model sınıfını, ikinci parametremiz ise dönüş tipini belirtmektedir. Örnek olması açısından sadece PlayerHandler sınıfını gösterdim. Son olarak Web API üzerinde kullanımını görerek sistemin işleyişini anlayalım.

```cs
namespace NetCoreMediatRExample.Controllers
{
    [ApiController]
    [Route("[controller]")]
    public class PlayerController : ControllerBase
    {

        private readonly ILogger<PlayerController> _logger;
        private IMediator mediator;

        public PlayerController(ILogger<PlayerController> logger, IMediator mediator)
        {
            _logger = logger;
            this.mediator = mediator;
        }

        [HttpGet]
        public async Task<IActionResult> GetPlayers()
        {
            var PlayerQuery=new GetPlayersQuery();
            var result = await mediator.Send(PlayerQuery);
            return result.Count != 0 ? (IActionResult) Ok(result) : BadRequest();
        }

        [HttpGet]
        [Route("{id}")]
        public async Task<IActionResult> GetPlayer(int id)
        {
            var PlayerQuery = new GetPlayerQuery(id);
            var result = await mediator.Send(PlayerQuery);
            return result != null ? (IActionResult)Ok(result) : BadRequest();
        }

        [HttpPost]
        public async Task<IActionResult> AddPlayer([FromBody]Player player)
        {
            var PlayerCommand = new CreatePlayerCommand(player);
            var result = await mediator.Send(PlayerCommand);
            return result != null ? (IActionResult)Ok(result) : BadRequest();
        }
    }
}
```

IMediator arayüzünü enjekte ettikten sonra localhost:44347/api/player endpoint’ine gidildiğinde GetPlayersHandler, localhost:44347/api/players/1 endpoint’ine gidildiğinde ise GetPlayersHandler sınıfları tetiklenir ve Handle metodu içerisindeki işlemler gerçekleşir. Send metodu eğer getPlayersQuery tipinde bir request gönderirse bu durumda GetPlayersHandler sınıfı tetiklenecektir çünkü GetPlayersHandler handler sınıfının request tipi GetPlayersQuery’dir.

Örnek projeyi inceleyerek MediatR ve CQRS deseni arasındaki ilişkiyi daha iyi anlayabilirsiniz.

Örnek Proje Linki : [.Net-Core-MediatR-CQRS](https://github.com/AbdullahOztuurkk/.Net-Core-MediatR-CQRS)

#### Kaynaklar :

- https://www.objectivity.co.uk/blog/when-to-use-and-not-to-use-cqrs/
- https://gcifguvercin.medium.com/cqrs-nedir-muhte%C5%9Fem-i%CC%87kili-ayr%C4%B1l%C4%B1yor-mu-a2b2f4032d34