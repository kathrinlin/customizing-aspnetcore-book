# Part 11: WebHostBuilder

When I wrote the blog post about HTTPS, a reader asked how to configure the HTTPS settings using user secrets.

> "How would I go about using user secrets to pass the password to `listenOptions.UseHttps(...)`? I can't fetch the configuration from within `Program.cs` no matter what I try. I've been Googling solutions for like a half hour so any help would be greatly appreciated."
> [https://github.com/JuergenGutsch/blog/issues/110#issuecomment-441177441](https://github.com/JuergenGutsch/blog/issues/110#issuecomment-441177441)

In this post I'm going to answer this question. 

## WebHostBuilderContext

It is about this Kestrel configuration in the `Program.cs`. In that post I wrote that you should use user secrets to configure the certificates password:

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        CreateWebHostBuilder(args).Build().Run();
    }

    public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
        WebHost.CreateDefaultBuilder(args)
        	.UseKestrel(options =>
            {
                options.Listen(IPAddress.Loopback, 5000);
                options.Listen(IPAddress.Loopback, 5001, listenOptions =>
                {
                    listenOptions.UseHttps("certificate.pfx", "topsecret");
                });
            })
        	.UseStartup<Startup>();
}
```

The reader wrote that he couldn't fetch the configuration inside this code. And he is true, if we are only looking at this snippet. You need to know that the method `UseKestrel()` is overloaded:

```csharp
.UseKestrel((host, options) =>
{
    // ...
})
```

This first argument is a `WebHostBuilderContext`. Using this you are able to access the configuration.

So lets rewrite the lambda a little bit to use this context:

```csharp
.UseKestrel((host, options) =>
{
    var filename = host.Configuration.GetValue("AppSettings:certfilename", "");
    var password = host.Configuration.GetValue("AppSettings:certpassword", "");
    
    options.Listen(IPAddress.Loopback, 5000);
    options.Listen(IPAddress.Loopback, 5001, listenOptions =>
    {
        listenOptions.UseHttps(filename, password);
    });
})
```

In this sample I chose to write the keys using the colon divider because this is the way you need to read nested configurations from the `appsettings.json`:

```json
{
    "AppSettings": {
        "certfilename": "certificate.pfx",
        "certpassword": "topsecret"
    },
    "Logging": {
        "LogLevel": {
            "Default": "Warning"
        }
    },
    "AllowedHosts": "*"
}
```

You are also able to read from the user secrets store with this keys set up:

```shell
dotnet user-secrets init
dotnet user-secrets set "AppSettings:certfilename" "certificate.pfx"
dotnet user-secrets set "AppSettings:certpassword" "topsecret"
```

As well as environment variables:

```shell
SET APPSETTINGS_CERTFILENAME=certificate.pfx
SET APPSETTINGS_CERTPASSWORD=topsecret
```

## Why does it work?

Do you remember the days back where you needed to configure app configuration in the `Startup.cs` ASP.NET Core? That was configured in the constructor of the Startup class and looked similar like this, if you added user secrets:

```csharp
 var builder = new ConfigurationBuilder()
        .SetBasePath(env.ContentRootPath)
        .AddJsonFile("appsettings.json")
        .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true);

    if (env.IsDevelopment())
    {
        builder.AddUserSecrets();
    }

    builder.AddEnvironmentVariables();
    Configuration = builder.Build();
```

This code now is wrapped inside the `CreateDefaultBuilder` Method ([see on GitHub](https://github.com/aspnet/AspNetCore/blob/3c09d644cccdb21801f7a79e1188a1a1212de5d9/src/DefaultBuilder/src/WebHost.cs)) and looks like this:

```csharp
builder.ConfigureAppConfiguration((hostingContext, config) =>
{
    var env = hostingContext.HostingEnvironment;

    config.AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
        .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true, reloadOnChange: true);

    if (env.IsDevelopment())
    {
        var appAssembly = Assembly.Load(new AssemblyName(env.ApplicationName));
        if (appAssembly != null)
        {
            config.AddUserSecrets(appAssembly, optional: true);
        }
    }

    config.AddEnvironmentVariables();

    if (args != null)
    {
        config.AddCommandLine(args);
    }
})
```

It is almost the same code and it is one of the first things that gets executed when building the `WebHost`. It needs to be one of the first things because the Kestrel is configurable via the app configuration. Maybe you know that you are able to specify ports and URLs and so on using environment variables or the `appsettings.json`:

I found this lines in the [WebHost.cs](https://github.com/aspnet/AspNetCore/blob/3c09d644cccdb21801f7a79e1188a1a1212de5d9/src/DefaultBuilder/src/WebHost.cs): 

```csharp
builder.UseKestrel((builderContext, options) =>
{
    options.Configure(builderContext.Configuration.GetSection("Kestrel"));
})

```

That means you are able to add this lines to the `appsettings.json` to configure Kestrel endpoints:

```json
"Kestrel": {
  "EndPoints": {
  "Http": {
  "Url": "http://localhost:5555"
 }}}

```

Or to use environment variables like this to configure the endpoint:

```shell
SET KESTREL_ENDPOINTS_HTTP_URL=http://localhost:5555

```

Also this configuration isn't executed 

## Update on ASP.NET Core 3.0

Since the hosting model in ASP.NET Core 3.0 is more generic, it looks a little bit different, but works the same way as in version 2.x. 

In ASP.NET Core 3.0 a generic `IHostBuilder` is created using the method `CreateDefaultBuilder()`. That `IHostBuilder` could also be an `IWebHostBuilder` to configure hosting of a web application, or a `IBlazorHostBuilder` to configure hosting of an application on WebAssembly, or even a simple `HostBuilder` to configure a Worker Service host. On an empty web application the Program.cs looks like this: 

~~~ csharp
 public class Program
 {
     public static void Main(string[] args)
     {
         CreateHostBuilder(args).Build().Run();
     }

     public static IHostBuilder CreateHostBuilder(string[] args) =>
         Host.CreateDefaultBuilder(args)
         .ConfigureWebHostDefaults(webBuilder =>
         {
             webBuilder.UseStartup<Startup>();
         });
 }
~~~

Now let's adopt the concepts of configuration to the `IHostBuilder`. You are able to configure the `IHostBuilder` as well as the `IWebHostBuilder` in the Program class using the `appsettings.json`, the user secrets store as well as the environment variables:

~~~ csharp
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureLogging((context, config) => {
                var logLevel = context.Configuration
                    .GetValue("Logging:LogLevel", "");
                
            })
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder
                    .UseKestrel((host, options) =>
                    {
                        var filename = host.Configuration
                            .GetValue("AppSettings:certfilename", "");
                        var password = host.Configuration
                            .GetValue("AppSettings:certpassword", "");
                        
                        options.Listen(IPAddress.Loopback, 5000);
                        options.Listen(IPAddress.Loopback, 5001, listenOptions =>
                        {
                            listenOptions.UseHttps(filename, password);
                        });
                    })
                    .UseStartup<Startup>();
            });
}
~~~

## Conclusion

Inside the `Program.cs` you are able to use app configuration inside the lambdas of the configuration methods, if you have access to the `WebHostBuilderContext`. This way you can use all the configuration you like to configure the `WebHostBuilder`.
