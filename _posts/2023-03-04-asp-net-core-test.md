---
layout: post
title:  "Еще раз про интеграционное тестирование ASP.NET Core c testserver и testcontainers"
date:   2023-03-03 14:45 +0300
categories: 
---
Сегодня я предлагаю совершить небольшое исследование на тему "как нам обустроить интеграционное тестирование и встроить его в сиайку".
Написать эту заметку меня сподвигла дискуссия, случившаяся недавно на работе. Инициативная группа "четырехглазых в свитерах" пыталась родить меры по улучшению качества нашего изделия и снижения трудозатрат QA-инженеров на проведение рутинного регрессионного тестирования. Как это часто бывает, разработчики если и писали тесты, то только модульные, оставляя интеграционные и end-to-end для тестировщиков. Для выполнения интеграционного тестирования QA-инженеры используют "тестовый стенд", на котором развернуты компоненты приложения (еще около 40, с позволения сказать, "микросервисов"), сервер базы данных (с не всегда ясным наполнением этой самой базы), брокер сообщений (RabbitMQ) и все остальное, что может потребоваться для запуска приложения. На этот тестовый стенд натравливаются автотесты, которые шатают приложение за все доступные снаружи конечные точки, таблицы БД и элементы UI пытаясь проверить максимальное количество тестовых сценариев в границах (и за ними!) возможных входных данных.

Схема работы не отличается оригинальностью, возможно я не сильно ошибусь, если скажу, что такой подход можно легко встретить в большом числе проектов, команды которых пока находятся в стадии взращивания своих практик. Со всеми очевидными плюсами (простота реализации, понятность и прозрачность для пользователей, многофункциональность стенда - можно же не только автотесты позапускать, но и для демо поиспользовать) этот подход имеет и существенные минусы:
- разработчики отрываются от процессов тестирования, ведь происходящее на тестовом стенде это забота QA
- общие зависимости (БД, Redis, брокеры) наполняются остатками данных с предыдущих тестовых сценариев, что приводит к их раздуванию или в худшем случае может вызывать нежелательные побочные эффекты 
- QA тратят много сил на поддержание тестового стенда в актуальном состоянии (когда твое приложение состоит из более 40 компонентов с запутанными связями между ними, а команд больше чем одна - это становится действительно непростой задачей) 
Всё это способствует росту количества false-positive срабатываний, что в итоге приводит к тому что в горячие моменты некоторые релизы могут поехать в прод без должного тестирования.
Ну да хватит слов, скорее к делу. Попробуем сделать интеграционное тестирование веселым снова.

Объектом сегодняшних исследований станет простое asp.net core web api приложение, с одним контроллером с набором CRUD методов. В качестве out-of-process звисимости, которую было бы сложно замокать, будем использовать БД (мы говорим БД - подразумеваем Postgres).
Демо-приложение отличается от шаблона получаемого при помощи `dotnet new webapi` только наличием ef core, поэтому здесь я весь листинг приводить не будут - версия приложения до начала тестирования [отмечена тегом](https://github.com/zetroot/TestHostTestContainers/tree/v0) `v0` в репозитории.
В следующих разделах я пройду путь от наивного подхода к интеграционным тестам до stateless тестов с реальными зависимостями, запускаемыми в контейнерах.

## "Наивный" подход
С самого начала, с момента, когда я задумал написать эту заметку, я хотел назвать этот подход "наивным" или "в лоб". Но, должен отметить, что в лоб ничего не вышло и мне потребовалось около двух часов ломиться в открытую дверь, потому что я не мог заставить тесты работать. Все вызовы к контроллерам завершались неуспешно и возвращали 404.  Чтобы приложение нашло свои контроллеры в код настройки DI пришлось вносить изменения, которые бы заставили задуматься, о разумности действий. Так что для ясности я возьму слово "наивный" в кавычки.
В чем суть подхода: приложение запускатеся в test-runner "как есть" и использует реальную БД доступную из агента CI (это может быть как раз БД тестового стенда или выделенный инстанс специально для CI). Иные out-of-process зависимости так же используются реальные с тестового стенда.
В связи с тем, что во время прогона тестов исполняемой сборкой является сборка с тестами, а не с приложением, необходимо при настройке DI явно указать, что контроллеры следует искать в сборке приложения:
```cs
builder.Services.AddMvc()
    .AddApplicationPart(typeof(Program).Assembly)
    .AddControllersAsServices();
builder.Services.AddControllers();
        
```

Теперь можно написать немного тестов. Для проведения тестирования приложение надо запустить, что даст возможность получить досутп к его DI и выдернуть из него `DbContext`, который можно использовать для проверки side-эффектов (изменение состояния БД).
```cs
private WebApplication _app = null!;
private DataContext _context = null!;
private HttpClient _client = null!;
private IDataClient _refitClient = null!;
private IServiceScope _scope = null!;

[SetUp] public async Task Setup()
{
    var builder = WebApplication.CreateBuilder()
        .ConfigureServices();
    _app = builder.CreateApplication();
    _app.Urls.Add("http://*:8080");
    await _app.StartAsync();
    _scope = _app.Services.CreateScope();
    _context = _scope.ServiceProvider.GetRequiredService<DataContext>();
    _client = new HttpClient { BaseAddress = new Uri("http://localhost:8080") };
    _refitClient = RestService.For<IDataClient>(_client);
}
```
Так же при настройке тестового класса создается http-клиент (`_client`), нацеленный на локальное приложение и типизированный refit-клиент (`_refitClient`), для вызова контроллеров по существу.
После завершения тестов необходимо остановить приложение и освободить ресурсы выделенные для http-клиента:
```cs
[TearDown] public async Task TearDown()
{
    _scope.Dispose();
    await _app.StopAsync();
    _client.Dispose();
}

```

Когда вся инфраструктура для тестов поднята, можно поделать запросы и проверить функционирование логики приложения:
```cs
[Test] public async Task PostData_WhenCalled_Returns200()
{
    //act
    var response = await _client.PostAsJsonAsync(new Uri("data", UriKind.Relative), "test");
    //assert
    response.StatusCode.Should().Be(HttpStatusCode.OK);
}

[Test] public async Task PostData_WhenCalled_ReturnsIdOfAddedRecord()
{
    //arrange
    var cntBefore = await _context.Set<UserData>().CountAsync();
    //act
    var id = await _refitClient.Create("test creation");
    //assert
    _context.Set<UserData>().Count().Should().BeGreaterThan(cntBefore);
    _context.Set<UserData>().Any(x => x.Id == id).Should().BeTrue();
    _context.Set<UserData>().Single(x => x.Id == id).Data.Should().Be("test creation");
}
```
Собственно цель достигнута: тесты проходят, можно делать реальные http запросы, база данных наполняется, есть возможность получить доступ к ней из тестовых классов. Но есть и сложности:
    - используется реальная БД, креды к ней необходимо хранить в репозитории или подкладывать на этапе тестирования
    - так же необходимо обеспечить наполнение БД исходными данными и очистку после выполнения всех тестов
    - надо быть уверенным, что используемый порт приложения на test-runner будет доступен
Код приложения с этими тестами [отмечен тегом](https://github.com/zetroot/TestHostTestContainers/tree/v1) `v1`.

## Testserver
Следующий шаг - использование возможностей ASP.NET Core для выполнения интеграционного тестирования. Заменим kestrel на test server!
Для удобного доступа к DI и осуществления манипуляций с контейнером внедрения зависимостей созданим наследника `WebApplicationFactory<>`:
```cs
public class CustomAppFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureTestServices(services =>
        {
            // Удалим зарегистрированный DataContext
            var descriptor = services.SingleOrDefault(d => d.ServiceType == typeof(DbContextOptions<DataContext>));
            if (descriptor != null)
                services.Remove(descriptor);

            // Зарегистрируем снова с указанием на тестовую БД
            services.AddDbContextPool<DataContext>(opts => opts.UseNpgsql("Host=localhost;Database=test_ci_db;Username=postgres;Password=;"));

            // Обеспечим создание БД
            var serviceProvider = services.BuildServiceProvider();
            using var scope = serviceProvider.CreateScope();
            var scopedServices = scope.ServiceProvider;
            var context = scopedServices.GetRequiredService<DataContext>();
            context.Database.EnsureDeleted();
            context.Database.EnsureCreated();
            // Здесь можно выполнить код "наполняющий" БД тестовыми данными...
        });
    }
}
```

Эта фабрика и будет обеспечивать запуск приложения. Метод `ConfigureTestServices` вызывается после настройки DI выполняемого приложением, поэтому в нем можно переопределить настройку внедрения зависимостей и нацелить приложение на конкретный инстанс сервера БД, используемый для тестов выполняемых в CI.
Код тестов несколько упрощается. При создании тестового класса создается фабрика приложения, из которой можно получить сервисы из DI-контейнера и готовый http-клиент, нацеленный на теситруемое приложение:
```cs
private CustomAppFactory _factory = new();
private DataContext _context = null!;
private HttpClient _client = null!;
private IDataClient _refitClient = null!;
private IServiceScope _scope = null!;

[SetUp] public void Setup()
{
    _scope = _factory.Services.CreateScope();
    _context = _scope.ServiceProvider.GetRequiredService<DataContext>();
    _client = _factory.CreateClient();
    _refitClient = RestService.For<IDataClient>(_client);
}

[TearDown] public void TearDown()
{
    _scope.Dispose();
    _client.Dispose();
}
```

Код самих тестов не меняется. Этот этап развития тестового приложения [отмечен тегом](https://github.com/zetroot/TestHostTestContainers/tree/v2) `v2` в репозитории.
Что мы получили на текущем этапе:
    - не запускается kestrel
    - есть контроль над используемой БД 
    - есть контроль над сервисами приложения, можно использовать моки вместо внешних зависимостей

Тем не менее, тестам все еще надо иметь внешнюю БД и другие out-of-process зависимости. Так что выполнение на CI все еще нельзя назвать полностью автономным. 

## Тестовые контейнеры
Что ж, и для этой проблемы есть решение. Существует проект [Testcontainers](https://dotnet.testcontainers.org/), предоставляющий, по их собственным словам, легковесные, одноразовые экземляры внешних зависимостей. Библиотека построена поверх Docker remote API и фактически позволяет запускать контейнеры из любых образов для использования их в тестах.

Чтобы не бороться за порты и hostnames позволим передавать их в фабрику приложения через параметры:
```cs
public class CustomAppFactory : WebApplicationFactory<Program>
{
    private readonly string _dbConnStr;

    public CustomAppFactory(string host, int port, string password)
    {
        var sb = new NpgsqlConnectionStringBuilder
        {
            Host = host, Port = port, Database = "test_ci_database", Username = "postgres", Password = password
        };
        _dbConnStr = sb.ConnectionString;
    }

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureTestServices(services =>
        {
            // ...
            services.AddDbContextPool<DataContext>(opts => opts.UseNpgsql(_dbConnStr));
            // ...
        });
    }
}

```

И в тестовом классе создадим и запустим контейнер с постгресом:
```cs
[OneTimeSetUp] public async Task SetupContainer()
{
    const string postgresPwd = "pgpwd";
    
    _pgContainer = new ContainerBuilder()
        .WithName(Guid.NewGuid().ToString("N"))
        .WithImage("postgres:15")
        .WithHostname(Guid.NewGuid().ToString("N"))
        .WithExposedPort(5432)
        .WithPortBinding(5432, true)
        .WithEnvironment("POSTGRES_PASSWORD", postgresPwd)
        .WithEnvironment("PGDATA", "/pgdata")
        .WithTmpfsMount("/pgdata")
        .WithWaitStrategy(Wait.ForUnixContainer().UntilCommandIsCompleted("psql -U postgres -c \"select 1\""))
        .Build();
    await _pgContainer.StartAsync();
    
    _factory = new(_pgContainer.Hostname, _pgContainer.GetMappedPublicPort(5432), postgresPwd);
}
```
Из особенностей: имя контейнера и имя хоста выбираются случайно (насколько случайным может быть `Guid.NewGuid()`), порт привязывается к случайному внешнему порту. Все это делается, чтобы избежать проблем с другими экзеплярами приложения и другими запусками тестов на той же машине.
Сгенерированные имена и порты легко извлечь и передать в фабрику для настройки SUT.
Так же обращу внимание на лайфхак - `.WithEnvironment("PGDATA", "/pgdata")` указывает субд хранить данные баз данных по пути `/pgdata` который маппится в память при помощи `.WithTmpfsMount("/pgdata")`. Так что даже если тестов будет много, или в ходе тестов будут использоваться тяжелые тестовые данные - место на диске не пострадает, БД будет существовать только in-memory.
Второй лайфхак - перед тем как запускать тесты надо дождаться когда PG полностью поднимется и будет инициализирована. Можно добиться этого прописывая хелсчеки в кастомном dockerfile, а можно воспользоваться вызовами testcontainers: `.WithWaitStrategy(Wait.ForUnixContainer().UntilCommandIsCompleted("psql -U postgres -c \"select 1\""))`. Здесь вызывающее приложение дождется, пока БД полностью оживет и выполнится команда `select 1`, что и будет означать готовность БД к нашим последующим запросам.
После завершения тестов в тестовом классе контейнер необходимо выбросить:
```cs
[OneTimeTearDown] public async Task DisposeContainer() =>
        await _pgContainer.DisposeAsync();
```

Вот теперь приложение тестируется в полностью stateless манере, не требуется никакой настройки окружения. Все что нужно для прогона тестов - это dotnet sdk и docker.
Код этого состояния [доступен под тегом](https://github.com/zetroot/TestHostTestContainers/tree/v3) `v3`

### Запуск в CI
До этого все разговоры были о CI, агенты которого выполняются в контролируемом, доступном для модификации окружении. Но Github Actions - не такой. Он бесплатный (по мере воможностей), популярный, но его агенты живут где то там далеко и возможностей поднять рядом с нашим приложением какую то БД (ну хотя бы БД) - нет.
С тестконтейнерами это не проблема!
Добавим шаблонный github action:
```yaml
name: .NET

on:
  push:
    branches: [ "master" ]
  pull_request:
    branches: [ "master" ]
  workflow_dispatch:

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 7.0.x
    - name: Restore dependencies
      run: dotnet restore
    - name: Build
      run: dotnet build --no-restore
    - name: Test
      run: dotnet test --no-build --verbosity normal

```
И все, билд проходит, [тесты зеленые](https://github.com/zetroot/TestHostTestContainers/actions/runs/4330729004/jobs/7562132598), мы запустили настоящие интеграционные тесты в окружении, которое мы не можем формировать.

Это состояние я [отметил тегом](https://github.com/zetroot/TestHostTestContainers/tree/v3.1) `v3.1`.

## Вместо заключения
Ну что, все всё сами видели, можно писать полноценные интеграционные тесты для asp.net core приложений использующих реальные базы данных и реальные внешние зависимости практически не касаясь yaml магии и не внося существенных изменений в привычный CI/CD пайплайн. Надеюсь эта заметка была хорошей иллюстрацией и поможет кому-то начать использовать интеграционные тесты в ежедневной работе.

[Оригинал статьи размещен на хабре](https://habr.com/ru/articles/720420/)