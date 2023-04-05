---
layout: post
title:  "Interprocess communication с использованием GRPC"
date:   2020-10-15 17:39 +0300
categories: 
---

Сегодня хочу рассказать о нашем пути реализации межпроцессного взаимодействия между приложениями на NET Core и NET Framework при помощи протокола GRPC. Ирония заключается в том, что GRPC, продвигаемый Microsoft как замена WCF на своих платформах NET Core и NET5, в нашем случае случился именно из-за неполноценной реализации WCF в NET Core.

Я надеюсь эта статья найдется поиском когда кто-то будет рассматривать варианты организации IPC и позволит посмотреть на такое высокоуровневаое решение как GRPC с этой, низкоуровневой, стороны.

Вот уже более 7 лет моя трудовая деятельность связана с тем, что называют "информатизация здравоохранения". Это довольно интересная сфера, хотя и имеющая свои особенности. Некоторыми из них явяляются зашкаливающее количество легаси технологий (консервативность) и определнная закрытость для интеграции у большинства существующих решений (вендор-лок на экосистеме одного производителя). 

# Контекст
С комбинацией этих двух особенностей мы и столкнулись на текущем проекте: нам понадобилось инициировать работу и получать данные из некоего программно-аппаратного комплекса. Поначалу все выглядело очень неплохо: софтовая часть комплекса поднимает WCF службу, которая принимает команды на выполнение и выплёвывает результаты в файл. Более того, производитель предоставляет SDK с примерами! Что может пойти не так? Всё вполне технологично и современно. Никакого  ASTM с палочками-разеделителями, ни даже обмена файлами через общую папку.

Но по какой-то странной причине, WCF служба использует дуплексные каналы и привязку `WSDualHttpBinding`, которая недоступна под .NET Core 3.1, только в "большом" фреймворке (или уже в "старом"?). При этом дуплексность каналов никак не используется! Она просто есть в описании службы. Облом! Ведь остальной проект живет на NET Core и отказываться от этого нет никакого желания. Придется собирать этот "драйвер" в виде отдельного приложения на NET Framework 4.8 и каким то образом пытаться организовать хождение данных между процессами.

# Межпроцессное взаимодействие
В теме межпроцессного взаимодействия действительно есть из чего выбрать. Можно обмениваться файлами, кидать сигналы, использовать мьютексы, именованные каналы, поднять соединение через tcp-сокет, или даже воспользоваться каким-нибудь RPC протоколом верхнего уровня. Чтобы не потонуть в этом многообразии попробуем сформулировать требования к службе IPC:
* возможность передавать команды, в том числе параметризованные
* возможность получать ответы о результате выполнения команды
* возможность асинхронно получать ответы о ходе выполнения долгоиграющей задачи
* контроль получения сообщений второй стороной
* Поддержка в Windows (начиная с 7 версии)
* Поддержка в NET Framework и NET Core
* Строгая типизация сообщений
* Расширяемость
* Поменьше боли

Последний пункт самый важный, потому что всегда можно сгородить сколь угодно сложный и замороченный велосипед на базе самых низкоуровневых примитивов, например разделяемой памяти и мьютексов. Но зачем?

# Именованные каналы
Да, мы сделали это. В первой итерации были выбраны именно именованные каналы, по которым передавались команды и данные. У нас были абстракции для команды и для данных, и два однонаправленных канала в каждом "соединении". У нас не было контроля получения сообщений, зато была боль с поддержкой стейта - канал мог развалиться по каким то странным причинам. У нас не было связного асинхронного потока ответов о прогрессе долгоиграющей операци, зато были отдельные события получения сообщения и серверу надо было передавать исходную команду к которой относится ответ. У нас не было настоящей строгой типизации, а только надежда что на той стороне данные упаковали в правильную "команду" или "ответ". Боль? Боль была, постоянная: трудновоспроизводимые баги, никакущий дебаг, мрак и заброшенность.

На самом деле все было не так плохо. Это решение работало и проработало почти год. Мы потратили много времени на его тестирование, отлавливание багов, восстановление стейта, выработку workaround, чтобы его можно было использовать у конечных пользователей. Но мне кажется его никто не любил.

# GRPC
Уныло поглядев на в очередной раз глюканувший софт, рассказывающий о сломанных трубах, было принято решение все переписать. И переписать на GRPC. Почему GRPC? Ну, мы используем его в хвост и гриву для взаимодействия между клиентом и основным сервером приложения. Он произвел впечатление достаточно быстрого и богатого возможностями решения.

Что касается чеклиста с требованиями, то получается примерно так:
* возможность передавать команды, в том числе параметризованные - да, это то что называется Unary call
* возможность получать ответы о результате выполнения команды - то же
* возможность асинхронно получать ответы о ходе выполнения долгоиграющей задачи - да, это server streaming rpc
* контроль получения сообщений второй стороной - обеспечивается транспортным уровнем HTTP/2
* Поддержка в Windows (начиная с 7 версии) - да, но в семёрке есть определённые ограничения
* Поддержка в NET Framework и NET Core - да
* Строгая типизация сообщений - да, обеспечивается форматом protobuf
* Расширяемость - и расширяемость
* Поменьше боли - больно почти не будет ~~, как комарик укусит~~

# Затаскиваем GRPC за 5 минут
Я позволю себе написать очередной tutorial. Туториалов по GRPC много, но они все одинаковые и не об этом, а в начале своего пути по дороге IPC я бы хотел чтобы что-то такое попалось мне на глаза. Все примеры кода есть в репозитории [тут](https://github.com/zetroot/IpcGrpcSample).

## Подготовка
Наше решение будет состоять из трёх сборок:
* `IpcGrpcSample.CoreClient` - консольное приложение под NET Core 3.1, будет играть роль клиента RPC
* `IpcGrpcSample.NetServer` - консольное приложение под NET Framework 4.8, будет за сервер RPC
* `IpcGrpcSample.Protocol` - библиотека, нацеленная на NET Standard 2.0. В ней будет определен контракт RPC

Для удобства работы отредактируем формат файла проекта на NET Framework под новый лад и снесем `Properties\AssemblyInfo.cs`
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net48</TargetFramework>
  </PropertyGroup>
  <Import Project="$(MSBuildExtensionsPath)\$(MSBuildToolsVersion)\Microsoft.Common.props" Condition="Exists('$(MSBuildExtensionsPath)\$(MSBuildToolsVersion)\Microsoft.Common.props')" />
  <PropertyGroup>...</PropertyGroup>
  <PropertyGroup Condition=" '$(Configuration)|$(Platform)' == 'Debug|AnyCPU' ">...</PropertyGroup>
  <PropertyGroup Condition=" '$(Configuration)|$(Platform)' == 'Release|AnyCPU' ">...</PropertyGroup>
  <ItemGroup>
    <Reference Include="System" />
    <Reference Include="System.Core" />
    <Reference Include="System.Xml.Linq" />
    <Reference Include="System.Data.DataSetExtensions" />
    <Reference Include="Microsoft.CSharp" />
    <Reference Include="System.Data" />
    <Reference Include="System.Net.Http" />
    <Reference Include="System.Xml" />
  </ItemGroup>
  <ItemGroup>
    <None Include="App.config" />
  </ItemGroup>
  <Import Project="$(MSBuildToolsPath)\Microsoft.CSharp.targets" />
</Project>
```
Настало время тащить NuGet!
* В проект `IpcGrpcSample.Protocol` подключаем пакеты `Google.Protobuf`, `Grpc` и `Grpc.Tools`
* В сервер затащим `Grpc`, `Grpc.Core`, `Microsoft.Extensions.Hosting` и `Microsoft.Extensions.Hosting.WindowsServices`. 
* В клиента тащим `Grpc.Net.Client` и `OneOf` - он нам пригодится.

## Описываем gRPC службу
Наверно всем уже надоел пример с `GreeterService`? Давайте что-нибудь повеселее. Сделаем две службы. Одна будет управлять термоциклером-амплификатором, другая будет запускать протокол экстракции на автомтаизированной станции экстракции.

Службы описываются в виде `.proto` файлов в разделяемой сборке `IpcGrpcSample.Protocol`. Protobuf-компилятор потом соберет из них нужные нам классы и заготовки под сервисы и клиентов.

Описание службы работы с роботизированной станцией
```protobuf
//указание синтаксиса описания
syntax = "proto3"; 
// импорт типа Empty
import "google/protobuf/empty.proto";
// классы будут генерироваться в этом пространстве имен
option csharp_namespace = "IpcGrpcSample.Protocol.Extractor"; 
// описание методов RPC службы управления экстрактором
service ExtractorRpcService {  
  // унарный вызов "запускающий" операцию
  rpc Start (google.protobuf.Empty) returns (StartResponse);  
}

// ответ на старт
message StartResponse {
	bool Success = 1;
}
```
Описание службы работы с термоциклером
```protobuf
//указание синтаксиса описания
syntax = "proto3"; 
// классы будут генерироваться в этом пространстве имен
option csharp_namespace = "IpcGrpcSample.Protocol.Thermocycler"; 
// описание методов RPC службы управления термоциклером
service ThermocyclerRpcService {  
  // server-streaming вызов "запускающий эксперимент". На один запрос отправит множество сообщений-ответов, которые будут доступны асинхронно
  rpc Start (StartRequest) returns (stream StartResponse);  
}

// описание сообщения - запроса на запуск эксперимента
message StartRequest {
  // поля запроса - это поле будет названием эксперимента
  string ExperimentName = 1;
  // а числовое поле - количество циклов, в ходе которых прибор будет "снимать показания" и отправлять назад
  int32 CycleCount = 2;
}

// сообщение из стрима после старта
message StartResponse {
  // номер цикла
  int32 CycleNumber = 1;
  // поле в виде конструкции oneof - сообщение может содержать объект одного из типов. 
  // Что-то вроде discriminated union, но попроще
  oneof Content {
	// прочитанные данные реакционного блока
	PlateRead plate = 2;
	// сообщение статуса прибора
	StatusMessage status = 3;
  }
}

message PlateRead {
  string ExperimentalData = 1;
}

message StatusMessage {
  int32 PlateTemperature = 2;
}
```
Все proto-файлы надо пометить как компилируемые protobuf компилятором. Ручками по одному или добавить в csproj эти строки:
```xml
  <ItemGroup>
    <Protobuf Include="**\*.proto" />
  </ItemGroup>
```

## Строим сервер
На дворе 2020 год и поэтому сервер соберем на базе Hosting абстракций из NET Core. Сразу вставляем такой сниппет в Program.cs:
{% highlight csharp %}
class Program
{
    static Task Main(string[] args) => CreateHostBuilder(args).Build().RunAsync();

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
        .UseWindowsService()
        .ConfigureServices(services =>
        {
            services.AddLogging(loggingBuilder =>
            {
                loggingBuilder.ClearProviders();
                loggingBuilder.SetMinimumLevel(LogLevel.Trace);                    
                loggingBuilder.AddConsole();
            });
            services.AddTransient<ExtractorServiceImpl>(); // регистрация зависимостей - сервисов с логикой управления приборами
            services.AddTransient<ThermocyclerServiceImpl>();
            services.AddHostedService<GrpcServer>(); // регистрация GRPC сервера как HostedService
        });
}
{% endhighlight %}

Обойдемся в примере без конфигурации и логгирования. Реализация служб (контроллеров) будет внедряться в сервер через конструктор.

Код сервера достаточно прямолинеен - в методе старта службы сервер поднимается, в методе остановки - останавливается. Здесь можно определить будет ли сервер использовать TLS (тогда ему нужен сертификат для шифрования) или трафик будет ходить без шифрования - и тогда необходимо использовать `ServerCredentials.Insecure`. Зачем это может пригодиться и что еще необходимо сделать чтобы гонять голый http/2 трафик - в конце.

{% highlight csharp %}
internal class GrpcServer : IHostedService
{
    private readonly ILogger<GrpcServer> logger;
    private readonly Server server;
    private readonly ExtractorServiceImpl extractorService;
    private readonly ThermocyclerServiceImpl thermocyclerService;

    public GrpcServer(ExtractorServiceImpl extractorService, ThermocyclerServiceImpl thermocyclerService, ILogger<GrpcServer> logger)
    {
        this.logger = logger;
        this.extractorService = extractorService;
        this.thermocyclerService = thermocyclerService;
        var credentials = BuildSSLCredentials(); // строим креды из сертификата и приватного ключа. 
        server = new Server //создаем объект сервера
        {
            Ports = { new ServerPort("localhost", 7001, credentials) }, // биндим сервер к адресу и порту
            Services = // прописываем службы которые будут доступны на сервере
            {
                ExtractorRpcService.BindService(this.extractorService),
                ThermocyclerRpcService.BindService(this.thermocyclerService)
            }
        };            
    }

    /// <summary>
    /// Вспомогательный метод генерации серверных кредов из сертификата
    /// </summary>
    private ServerCredentials BuildSSLCredentials()
    {
        var cert = File.ReadAllText("cert\\server.crt");
        var key = File.ReadAllText("cert\\server.key");

        var keyCertPair = new KeyCertificatePair(cert, key);
        return new SslServerCredentials(new[] { keyCertPair });
    }

    public Task StartAsync(CancellationToken cancellationToken)
    {
        logger.LogInformation("Запуск GRPC сервера");
        server.Start();
        logger.LogInformation("GRPC сервер запущен");
        return Task.CompletedTask;
    }

    public async Task StopAsync(CancellationToken cancellationToken)
    {
        logger.LogInformation("Останов GRPC сервера");
        await server.ShutdownAsync();
        logger.LogInformation("GRPC сервер остановлен");
    }
}
{% endhighlight %}

Осталось закинуть какую нибудь логику в контроллеры и стартовать!
У экстрактора очень простой сервис. Будет через раз отвечать что все получилось или нет:
{% highlight csharp %}
internal class ExtractorServiceImpl : ExtractorRpcService.ExtractorRpcServiceBase
{
    private static bool success = true;
    public override Task<StartResponse> Start(Empty request, ServerCallContext context)
    {
        success = !success;
        return Task.FromResult(new StartResponse { Success = success });
    }
}
{% endhighlight %}

А у термоциклера организуем что-нибудь повеселее:
{% highlight csharp %}
internal class ThermocyclerServiceImpl : ThermocyclerRpcService.ThermocyclerRpcServiceBase
{
    private readonly ILogger<ThermocyclerServiceImpl> logger;

    public ThermocyclerServiceImpl(ILogger<ThermocyclerServiceImpl> logger)
    {
        this.logger = logger;
    }

    public override async Task Start(StartRequest request, IServerStreamWriter<StartResponse> responseStream, ServerCallContext context)
    {
        logger.LogInformation("Эксперимент начинается");
        var rand = new Random(42);
        for(int i = 1; i <= request.CycleCount; ++i)
        {
            logger.LogInformation($"Отправка цикла {i}");
            var plate = new PlateRead { ExperimentalData = $"Эксперимент {request.ExperimentName}, шаг {i} из {request.CycleCount}: {rand.Next(100, 500000)}" };
            await responseStream.WriteAsync(new StartResponse { CycleNumber = i, Plate = plate });
            var status = new StatusMessage { PlateTemperature = rand.Next(25, 95) };
            await responseStream.WriteAsync(new StartResponse { CycleNumber = i, Status = status });
            await Task.Delay(500);
        }
        logger.LogInformation("Эксперимент завершен");
    }
}
{% endhighlight %}
Сервер готов. Можно позапускать консольку и убедиться что GRPC сервер поднимается и останавливается по нажатию на `Ctrl-C`:
```
dbug: Microsoft.Extensions.Hosting.Internal.Host[1]
      Hosting starting
info: IpcGrpcSample.NetServer.GrpcServer[0]
      Запуск GRPC сервера
info: IpcGrpcSample.NetServer.GrpcServer[0]
      GRPC сервер запущен
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Production
info: Microsoft.Hosting.Lifetime[0]
      Content root path: C:\Users\user\source\repos\IpcGrpcSample\IpcGrpcSample.NetServer\bin\Debug
dbug: Microsoft.Extensions.Hosting.Internal.Host[2]
      Hosting started
info: Microsoft.Hosting.Lifetime[0]
      Application is shutting down...
dbug: Microsoft.Extensions.Hosting.Internal.Host[3]
      Hosting stopping
info: IpcGrpcSample.NetServer.GrpcServer[0]
      Останов GRPC сервера
info: IpcGrpcSample.NetServer.GrpcServer[0]
      GRPC сервер остановлен
dbug: Microsoft.Extensions.Hosting.Internal.Host[4]
      Hosting stopped
```
Соответственно в контроллерах может быть все что угодно: маппинг с транспортной модели на модель бизнеслогики которую нужно дергать из NET Framework, вызов WCF etc. И не надо тащить никакого Kestrel!

Сервер без клиента можно подергать например при помощи [grpcurl](https://github.com/fullstorydev/grpcurl), но все городилось для межпроцессного взаимодействия. Нужен клиент на NET Core.

## Клиентская часть на NET Core
Здесь все проще. Реализуем пару клиентов для сервисов и будем слушать ввод от пользователя.

Код сервиса экстрактора прямолинеен и тривиален. В конструкторе настраивается канал и создается gRPC клиент. Единственный метод дергает RPC метод и разворачивает результат.
{% highlight csharp %}
class ExtractorClient
{
    private readonly ExtractorRpcService.ExtractorRpcServiceClient client;

    public ExtractorClient()
    {
        //AppContext.SetSwitch("System.Net.Http.SocketsHttpHandler.Http2UnencryptedSupport", true); //без этого кода невозможна коммуникация через http/2 без TLS        
        var httpClientHandler = new HttpClientHandler
        {
            ServerCertificateCustomValidationCallback = HttpClientHandler.DangerousAcceptAnyServerCertificateValidator // заглушка для самоподписанного сертификата
        };
        var httpClient = new HttpClient(httpClientHandler);
        var channel = GrpcChannel.ForAddress("https://localhost:7001", new GrpcChannelOptions { HttpClient = httpClient });
        client = new ExtractorRpcService.ExtractorRpcServiceClient(channel);
    }

    public async Task<bool> StartAsync()
    {
        var response = await client.StartAsync(new Empty());
        return response.Success;
    }
}
{% endhighlight %}

В клиенте термоциклера используем `IAsyncEnumerable<>` возвращающим последовательность из монад `OneOf<,>` - эмулируется асинхронная последовательность данных от прибора.
{% highlight csharp %}
public async IAsyncEnumerable<OneOf<string, int>> StartAsync(string experimentName, int cycleCount)
{
    var request = new StartRequest { ExperimentName = experimentName, CycleCount = cycleCount };
    using var call = client.Start(request, new CallOptions().WithDeadline(DateTime.MaxValue)); // настройка времени ожидания
    while (await call.ResponseStream.MoveNext())
    {
        var message = call.ResponseStream.Current;
        switch (message.ContentCase)
        {
            case StartResponse.ContentOneofCase.Plate:
                yield return message.Plate.ExperimentalData;
                break;
            case StartResponse.ContentOneofCase.Status:
                yield return message.Status.PlateTemperature;
                break;
            default:
                break;
        };
    }
}
{% endhighlight %}

Остальной код клиента тривиален и позволяет только позапускать клиентов в цикле и вывести результаты на консоль.

# HTTP/2 и Windows 7
Оказалось, что в седьмой версии Windows нет поддержки TLS в HTTP/2. Поэтому сервер надо бинить немного иначе, используя небезопасные креды:
{% highlight csharp %}
server = new Server //создаем объект сервера
{
    Ports = { new ServerPort("localhost", 7001, ServerCredentials.Insecure) }, // биндим сервер к адресу и порту
    Services = // прописываем службы которые будут доступны на сервере
    {
        ExtractorRpcService.BindService(this.extractorService),
        ThermocyclerRpcService.BindService(this.thermocyclerService)
    }
};            
{% endhighlight %}

И у клиентов необходимо указать в качестве протокола `http`, а не `https`. Но это не все. Необходимо так же сообщить среде, что будет осознанно использоваться небезопасное соединение по http/2:
{% highlight csharp %}
AppContext.SetSwitch("System.Net.Http.SocketsHttpHandler.Http2UnencryptedSupport", true);
{% endhighlight %}

В коде проекта специально сделано много упрощений - не обрабатываются исключения, не ведется нормально логгирование, параметры захардкожены в код. Это не production-ready, а заготовка для решения проблем. Надеюсь, было интересно, задавайте вопросы!

Это репост [оригинальной статьи с хабра](https://habr.com/ru/articles/523638/). Ну так, для истории.