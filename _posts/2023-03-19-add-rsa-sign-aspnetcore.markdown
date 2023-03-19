---
layout: post
title:  "Подписание тела ответа в ASP.NET Core"
date:   2023-03-19 15:00 +0300
categories: asp.net core
---

А что если надо обеспечить целостность ответов для asp.net core приложения? Самый очевидный вариант - добавить криптографическую подпись! А чтобы все было быстро - запуллировать ключи.

Сначала реализуем политику пуллирования:
{% highlight csharp %}
public class RsaKeyPoolPolicy : PooledObjectPolicy<RSA>
{
    private readonly string _certFile;
    private readonly string _certPwd;

    public RsaKeyPoolPolicy(string certFile, string certPwd)
    {
        _certFile = certFile;
        _certPwd = certPwd;
    }

    public override RSA Create() => 
        new X509Certificate2(_certFile, _certPwd).GetRSAPrivateKey() ?? throw new InvalidOperationException();

    public override bool Return(RSA obj) => true;
}
{% endhighligh %}

И зарегистрируем ее в DI:
{% highlight csharp %}
builder.Services.TryAddSingleton<ObjectPoolProvider, DefaultObjectPoolProvider>();
builder.Services.TryAddSingleton<ObjectPool<RSA>>(serviceProvider =>
{
    var provider = serviceProvider.GetRequiredService<ObjectPoolProvider>();
    var policy = new RsaKeyPoolPolicy("cert_file_with_private_key", "your_super_strong_password");
    return provider.Create(policy);
});
{% endhighligh %}

С обработкой тела ответа надо проделать небольшой трюк. Перед началом обработки запроса необходимо заменить stream, куда будет писаться ответ на контроллируемый этой middleware memory stream, чтобы запись шла туда. Затем необходимо скорпировать полученный ответ в исходный стрим и вернуть его на место.
{% highlight csharp %}
public class SigningMiddleware : IMiddleware
{
    private readonly ObjectPool<RSA> _keyPool;

    public SigningMiddleware(ObjectPool<RSA> keyPool) => _keyPool = keyPool;

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var key = _keyPool.Get();
        var hash = SHA256.Create();
        var rsaFormatter = new RSAPKCS1SignatureFormatter(key);
        rsaFormatter.SetHashAlgorithm(nameof(SHA256));
        
        var originalBody = context.Response.Body;
        try
        {
            // replace response body with controlled memory stream
            using var responseSubstitution = new MemoryStream();
            context.Response.Body = responseSubstitution;
            
            await next(context);
            
            // after all other middleware executed - compute hash and sign it 
            responseSubstitution.Position = 0;
            var bodyHash = hash.ComputeHash(responseSubstitution);
            var sign = rsaFormatter.CreateSignature(bodyHash);
            context.Response.Headers.TryAdd("X-RSA-Signature", Convert.ToBase64String(sign));
            
            // put body content to original body
            responseSubstitution.Position = 0;
            await responseSubstitution.CopyToAsync(originalBody);
        }
        finally
        {
            // replace response body back and return RSA key to pool
            context.Response.Body = originalBody;
            _keyPool.Return(key);
        }

    }
}
{% endhighligh %}

Подключение middleware тривиально:
{% highlight csharp %}
builder.Services.AddTransient<SigningMiddleware.Services.SigningMiddleware>();
var app = builder.Build();
app.UseMiddleware<SigningMiddleware.Services.SigningMiddleware>();
app.MapGet("/", () => "Hello World!");
app.MapGet("/time", () => $"{DateTime.Now}");
{% endhighligh %}

В итоге получаем заголовок с криптоподписью тела:
```
└─(15:28:46)──> curl -i http://localhost:5170
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Date: Sun, 19 Mar 2023 12:28:51 GMT
Server: Kestrel
Transfer-Encoding: chunked
X-RSA-Signature: CK5Xfg1v8YQem6k26h99h4vIKmk94OkRQ9QW0Hj7ltUrHGmOL8o61nlFuPT5sDv4MJQQhURTJFCbhPKpC4st7G7oU2szqiQsHWZRkeDNOlTPvoVF42q6TJIwv53Rhq8PULjkmkqAXlp9X/jfeIdrKqQtMw1h0l2pSmeY8QMawANJbPhlcv46KEm50EHib7mLIgwxrOPtW9KuzdRH/b3eoUqfW9Vz0XKm35CYkN3KVBRwHEYPVHgb+ICe4yyctl1c0R5m/kD7WQB7SCx/QoOqDVM5TFn5G9yIfalG0GN/NMxQ90qQR3wiDkE5HEQf184WPXQe3mNZNpXgORfYAlcXlA==

Hello World!% 
```

Или для динамических данных:
```
└─(15:28:52)──> curl -i http://localhost:5170/time
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Date: Sun, 19 Mar 2023 12:30:04 GMT
Server: Kestrel
Transfer-Encoding: chunked
X-RSA-Signature: akgj82Oj2WslJhDrxiz+gRb1sNuSCiPHVW6r43VvLTxySru0HJ98VSUVpXqOOcLlPD1mINk8Xe2gwPMW1DPXw7DyNu2DqcXp9rcJjg5G+Y2Cod5Ox91IE7FOBY3v0Ky4Af87ixAvCVzUXxjOv5IbjPW7lusYXYX7vZsso8gGYY1quIuNTHYtkAbLOd1QYYklUpPVdIAYOyK2B/ZgKmVgR3xU4Pq9MIumAX1Tp9CNRmLurPWX7TiaYP7PX9X5Jd81Ml2TTitxN3O3ERs41PVGNskJQR2p5ypbI/XReOXEVXGhIpT0gypNc1JdwVwaFyN0P+tFx7FCFcwP+4YbdfeZlQ==

3/19/2023 3:30:05 PM%   
```