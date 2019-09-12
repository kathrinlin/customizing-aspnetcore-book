# Part 05: IHostedService and BackgroundService

This fifth part does not really show a customization. This part is more about a feature you can use to create background services to run tasks asynchronously inside your application. Actually I use this feature to regularly fetch data from a remote service in a small ASP.NET Core application. 

## About IHostedService

Hosted service are a new thing in ASP.NET Core 2.0 and can be used to run tasks asynchronously in the background of your application. This can be used to fetch data periodically, do some calculations in the background or to do some cleanups. This can also be used to send preconfigured emails or whatever you need to do in the background.

Hosted services are basically simple classes, which implements the `IHostedService` interface.

```csharp
public class SampleHostedService : IHostedService
{
	public Task StartAsync(CancellationToken cancellationToken)
	{
	}
	
	public Task StopAsync(CancellationToken cancellationToken)
	{
	}
}
```

A `IHostedService` needs to implement a `StartAsync()` and a `StopAsync()` method. The `StartAsync()` is the place where you implement the logic to execute. This method gets executed once immediately after the application starts. The method `StopAsync()` on the other hand gets executed just before the application stops. This also means, to start a kind of a scheduled service you need to implement it by your own. You will need to implement a loop which executes the code regularly.

To get a `IHostedService` executed you need to register it in the ASP.NET Core dependency injection container as a singleton instance:

```csharp
services.AddSingleton<IHostedService, SampleHostedService>();
```

To see how a hosted service works, I created the next snippet. It writes a log message on start, on stop and every two seconds to the console:

```csharp
public class SampleHostedService : IHostedService
{
	private readonly ILogger<SampleHostedService> logger;
	
	// inject a logger
	public SampleHostedService(ILogger<SampleHostedService> logger)
	{
		this.logger = logger;
	}

	public Task StartAsync(CancellationToken cancellationToken)
	{
		logger.LogInformation("Hosted service starting");

		return Task.Factory.StartNew(async () =>
		{
			// loop until a cancalation is requested
			while (!cancellationToken.IsCancellationRequested)
			{
				logger.LogInformation("Hosted service executing - {0}", DateTime.Now);
				try
				{
					// wait for 3 seconds
					await Task.Delay(TimeSpan.FromSeconds(2), cancellationToken);
				}
				catch (OperationCanceledException) { }
			}
		}, cancellationToken);
	}

	public Task StopAsync(CancellationToken cancellationToken)
	{
		logger.LogInformation("Hosted service stopping");
		return Task.CompletedTask;
	}
}
```

To test this, I simply created a new ASP.NET Core application, placed this snippet inside, registered the `HostedService` and started the application by calling the next command in the console:

```shell
dotnet run
```

This results in the following console output:

![](images/customize-aspnetcore/hosted-service.png)

As you can see the log output is written to the console every two seconds.

## About BackgroundService

The `BackgroundService` class is new in ASP.NET Core 3.0 and is basically an abstract class that already implements the `IHostedService` Interface. It also provides an abstract method `ExecuteAsync()` that returns a `Task`.

To rewrite the hosted service from the last section it would look like this:

```csharp
public class SampleBackgroundService : BackgroundService
{
	private readonly ILogger<SampleHostedService> logger;
	
	// inject a logger
	public SampleBackgroundService(ILogger<SampleHostedService> logger)
	{
		this.logger = logger;
	}

	protected override async Task ExecuteAsync(CancellationToken cancellationToken)
	{
		logger.LogInformation("Hosted service starting");

		return Task.Factory.StartNew(async () =>
		{
			// loop until a cancalation is requested
			while (!cancellationToken.IsCancellationRequested)
			{
				logger.LogInformation("Hosted service executing - {0}", DateTime.Now);
				try
				{
					// wait for 3 seconds
					await Task.Delay(TimeSpan.FromSeconds(2), cancellationToken);
				}
				catch (OperationCanceledException) { }
			}
		}, cancellationToken);
	}

    public override async Task StopAsync(CancellationToken cancellationToken)
    {
        logger.LogInformation("Background service stopping");
        await Task.CompletedTask;
    }
}
```

Even the registration is new. In ASP.NET Core 3.0 the `ServiceCollection` has a new extension method to register hosted services or background worker:

~~~ csharp
services.AddHostedService<Worker>();
~~~

## The new worker service projects

The new worker services and the generic hosting in ASP.NET Core 3.0 makes it pretty easy to create simple service like applications that only do some stuff without the full blown ASP.NET stack and without a web server. This project is simply created with the following command:

~~~ shell
dotnet new worker -n BackgroundServiceSample -o BackgroundServiceSample
~~~

Basically this created a console application with a `Program.cs` and a `Worker.cs`. The `Worker.cs` is the `BackgrounService` and the program looks pretty familiar but without the `WebHostBuilder`.

~~~ csharp
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureServices((hostContext, services) =>
            {
                services.AddHostedService<Worker>();
            });
}
~~~

This creates a `IHostBuilder` with a dependency injection enabled. This means we are able to use dependency injection in any kind of application, not only in ASP.NET Core applications.

Than the `Worker` gets added to the service collection.

Where is this useful? You can run this app as a Windows service or as a background application in a docker container, which does not need a HTTP endpoint.

## Conclusion

You can now start to do some more complex things with the `IHostedServices` and the `BackgroundService`. Be careful with background services, because they run all in the same application. Do not use too much CPU or memory, this could slow down your application.

For bigger applications I would suggest to move such tasks in a separate application that is specialized to execute background tasks. A separate Docker container, a BackroundWorker on Azure, Azure Functions or something like this. However it should be separated from the main application in that case.

In the next chapter I am going to write about `Middlewares` and how you can use them to implement special logic to the request pipeline, or how you are able to serve specific logic on different paths.
