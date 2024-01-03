#### Cola.ColaGrpc 框架

##### 1. 注入

注入可以通过代码直接注入

```csharp
builder.Services
    .AddGrpcClient<Greeter.GreeterClient>(o =>
    {
        o.Address = new Uri("https://localhost:5005");
        o.Interceptors.Add(new ClientGrpcInterceptor(builder.Services.BuildServiceProvider().GetService<IColaLogs>()!,
            config));
    })
    .AddCallCredentials((context, metadata) =>
    {
        if (!string.IsNullOrEmpty(_token))
        {
            metadata.Add("Authorization", $"odinsam Bearer {_token}");
        }
        return Task.CompletedTask;
    })
    .ConfigureChannel(options =>
    {
        options.CreateGrpcClientChannelOptions(config);
    });
```

##### 2. 链接 
```csharp
var client = builder.Services.BuildServiceProvider().GetService<Greeter.GreeterClient>();
```

##### 3. 调用

```csharp
// IPC进程内调用grpc
// net core 3.x 显式的指定HTTP/2不需要TLS支持
AppContext.SetSwitch("System.Net.Http.SocketsHttpHandler.Http2UnencryptedSupport", true);

var credentials = CallCredentials.FromInterceptor((context, metadata) =>
{
    if (!string.IsNullOrEmpty(_token))
    {
        metadata.Add("Authorization", $"Bearer {_token}");
    }
    return Task.CompletedTask;
});
var channel = new ColaGrpcHelper(new ColaWindowsGrpc()).CreateChannel("https://localhost:5005",config,null,credentials);
var invoker =
    channel.Intercept(new ClientGrpcInterceptor(builder.Services.BuildServiceProvider().GetService<IColaLogs>()!,
        config));
var client = new Greeter.GreeterClient(invoker);

// 一元调用
ConsoleHelper.WriteInfo("一元调用 start");
var headers = new Metadata();
headers.Add("customHeader","odin-custom-header");
var helloReply = client.SayHelloAsync(new HelloRequest
{
    Name = "odin-sam"
},headers);
var resultOnce = await helloReply;
Console.WriteLine(resultOnce.Message);
ConsoleHelper.WriteInfo("一元调用 over");
Console.WriteLine();Console.WriteLine();

// 客户端流调用
ConsoleHelper.WriteInfo("客户端流调用 start");
var clientStreamMethodAsync = client.ClientStreamMethodAsync();
var requestStream = clientStreamMethodAsync.RequestStream;
for (int k = 1; k < 11; k++)
{
    await requestStream.WriteAsync(new ClientStreamMethodParam { Par = k });
    Console.WriteLine($"send par {k}");
    await Task.Delay(500);
}
await requestStream.CompleteAsync();
var resultClientStream = await clientStreamMethodAsync;
Console.WriteLine($"server response result： {resultClientStream.Result}");
ConsoleHelper.WriteInfo("客户端流调用 over");
Console.WriteLine();Console.WriteLine();

//服务端流调用
ConsoleHelper.WriteInfo("服务端流调用 start");
var param = new ClientMethodParam();
param.Lst.Add(new RepeatedField<int>() { 1, 2, 3, 4, 5 });
var clientServerStreamMethodAsync = client.ServerStreamMethodAsync(param);
var responseStream = clientServerStreamMethodAsync.ResponseStream;
// Define the cancellation token.
CancellationTokenSource source = new CancellationTokenSource();
CancellationToken token = source.Token;
while (await responseStream.MoveNext(token))
{
    Console.WriteLine($"server response result { responseStream.Current.Result }");
}
ConsoleHelper.WriteInfo("服务端流调用 over");
Console.WriteLine();Console.WriteLine();

//双向流调用
ConsoleHelper.WriteInfo("双向流调用 start");
var clientServerStream = client.ClientServerStreamMethodAsync();
var clientServerRequestStream = clientServerStream.RequestStream;
for (int k = 0; k < 10; k++)
{
    await clientServerRequestStream.WriteAsync(new ClientStreamMethodParam { Par = k });
    Console.WriteLine($"send par {k}");
    await Task.Delay(500);
}
await clientServerRequestStream.CompleteAsync();
var clientServerResponse = clientServerStream.ResponseStream;
while (await clientServerResponse.MoveNext(token))
{
    Console.WriteLine($"{ clientServerResponse.Current.Result }");
}
ConsoleHelper.WriteInfo("双向流调用 over");
Console.WriteLine();Console.WriteLine();

// 一元复杂调用
ConsoleHelper.WriteInfo("一元复杂调用 start");
var par = new ServiceMethodParamts
{
    Id = 100,
    Name = "ali yun",
    IsExists = true,
};
par.Roles.Add(new []{"刘备","关羽","张飞"});
par.Attributes.Add(new Dictionary<string, string>()
{
    {"董卓","吕布"},
    {"王允","貂蝉"}
});
par.Detail = Any.Pack(new Person{ Start = Timestamp.FromDateTimeOffset(DateTime.Now)});
var testResult = await client.ServiceMethodAsync(par);
if (testResult.ResultCase == ServiceMethodResponse.ResultOneofCase.Person)
{
    Console.WriteLine(JsonConvert.SerializeObject(testResult.Person).ToJsonFormatString());
}
ConsoleHelper.WriteInfo("一元复杂调用 over");
Console.WriteLine();Console.WriteLine();
```

##### 2. 使用

详见 ConsoleApp1Test 
















