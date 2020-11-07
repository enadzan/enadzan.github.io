### What is MassiveJobs.NET?

MassiveJobs.NET is an open-source library for simple, fast distributed background processing for .NET applications. Distributed processing is implemented using the RabbitMQ server. The library allows easy scale-out by deploying multiple instances of your application on multiple machines.

### Why?

#### Queue isolation

The published jobs are stored in multiple RabbitMQ queues. That means, even if a long-running job blocks one queue, it will not affect the other queues. When you publish a job, you can optionally specify the timeout value for the job execution (the default is 5 seconds). If you specify a value above 10 seconds, the job will be routed to a separate long-running queue to prevent it from blocking the regular queues. There are also separate queues for periodic jobs, delayed jobs, and jobs that are waiting to be retried because an exception was raised during job execution.

#### Low latency

In most cases, when a job is published, it will be immediately picked up and executed by a running instance of your application. However, this may be affected by the network connection between your application instances and the RabbitMQ server, the size of jour job's arguments, and the utilization of the machine running the RabbitMQ server...

#### Job Safety

If your application is stopped or if it crashes, the unfinished jobs are returned to the original queue and will be picked up by another instance of your application (or the same instance when it restarts). This means that, in some situations, your jobs may be triggered twice, so __it is imperative that your jobs are idempotent__ if executing the same job twice has negative effects on your application.

#### Speed

MassiveJobs.NET allows running a large number of jobs, fast. An [automated test publishing and performing 100,000 jobs](https://github.com/enadzan/massivejobs-rabbitmq/blob/master/MassiveJobs.RabbitMqBroker.Tests/RabbitMqPublisherTest.cs#L43) runs under 10 seconds on a developer machine (i5 7th Gen, 2c/4t), with a local RabbitMQ instance. This includes serializing 100,000 jobs to JSON, publishing them as messages to RabbitMQ, consuming them from RabbitMQ, deserializing them, and invoking the `Perform` method on each job (which only does an interlocked counter increase). 

Of course, depending on what your jobs are doing, network latencies, etc., you may have a different experience. But, the library itself will not get in your way and it gives you the option to distribute both publishers and consumers across multiple machines.

__WARNING: MassiveJobs.NET is still in pre-release. However, we invite you to try it out and report any issues you may find.__

MassiveJobs.NET is inspired by [Sidekiq library for Ruby](https://sidekiq.org/) but has no association with it.

### Features

#### Simple job definition and publishing
A job is a piece of work that can be performed in the background without waiting for it to finish. The simplest way to define a job is to create a class that inherits from `Job<TJob, TArgs>`. For example, if you need to send a welcome email upon customer registration, you would create a class similar to this:
```csharp
public class WelcomeEmailSender: Job<WelcomeEmailSender, string>
{
  public override void Perform(string customerEmail) 
  {
    // TODO: Send welcome email to the customer
  }
}
```
Then, in your customer registration procedure, after the customer is registered, you would call:
```csharp
WelcomeEmailSender.Publish(customerEmail);
```
And that's it.
  
When a job is published, it will be picked up and executed in the background __by any of the running instances our your application__. If the job throws an exception, it will be retried after an increasing number of seconds. After 25 retries (about 21 days), the job will be moved to the __failed queue__ in RabbitMQ, where it can be examined and possibly republished.

### Delayed jobs

A job can be scheduled to be performed at a later point by specifying the delay time on publish:
```csharp
// perform the job after 5 seconds
WelcomeEmailSender.Publish(customerEmail, TimeSpan.FromSeconds(5));
```
### Periodic jobs

Jobs can be performed periodically, every specified number of seconds. For example, if you have a `StatusChecker` job:
```csharp
// Inherit from Job<T> (no second type parameter) 
// if your Perform method does not have any parameters.
public class StatusChecker: Job<StatusChecker> 
{
  public override void Perform() 
  {
    // TODO: Check some status
  }
}
```
then, on your application start you would call:
```csharp
// run every 60 seconds
StatusChecker.PublishPeriodic("status_check", 60);
```
The first parameter is the "periodic job-id" and it must be unique across all periodic jobs. This id is used to de-duplicate periodic jobs in scenarios where you have multiple instances of the application running, such as web farms. The periodic job will be executed every 60 seconds by only one of the instances of your application.  As long as there is at least one instance running, the job will be performed.

### Cron jobs
Periodic jobs can be published using cron expression syntax. For example, this will publish a job that runs every weekday at 6 am:
```csharp
SummaryJob.PublishPeriodic("daily_summary", "0 0 6 * * MON-FRI")
```

### Publishing jobs in batches
Publishing jobs in batches, with the help of the `JobBatch` class, can significantly increase the publishing speed:
```csharp
JobBatch.Do(() =>
{
    for (var i = 0; i < 100_000; i++)
    {
        YourJob.Publish(jobArguments);
    }
});
```
__Note that batches are not transactions - they serve purely for improving publishing performance.__

### Support for .NET Framework 4.6.1+ and .NET Core 2.0+
MassiveJobs.NET is distributed as a .net standard 2.0 library, which means that it supports both .NET Framework (version 4.6.1 and above), and .NET Core (2.0 and above).

### Support for .NET Core hosted environments

MassiveJobs.NET is easy to integrate with a .NET Core hosted environment (ASP.NET Core, Worker Services) by installing `MassiveJobs.RabbitMqBroker.Hosting` package in your application. Then, in your startup class, when configuring services, call services.AddMassiveJobs().
```csharp
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        //...
        services.AddMassiveJobs();
    }

    //...
}
```
This will start MassiveJobs workers within a hosted background service.

### Support for Dependency Injection

In a .NET Core hosted environment, your jobs can use constructor dependency injection. This works without the need to register your jobs with the service container (of course, services that your job is using must be registered). This works by simply declaring a constructor with services needed as parameters. Your jobs are executed within a scope, so scope level services, such as DbContext, can be injected in the constructor of your jobs.

```csharp
public class WelcomeEmailSender: Job<WelcomeEmailSender, string>
{
  private readonly IEmailServer _emailServer;
  
  public WelcomeEmailSender(IEmailServer emailServer) 
  {
    _emailServer = emailServer;
  }
  
  public override void Perform(string customerEmail) 
  {
    string welcomeText;
    // TODO: Create welcome text
    
    _emailServer.Send(customerEmail, welcomeText);
  }
}
```

## Quick Start
For a quick start visit MassiveJobs.RabbitMqBroker github repository:
- [MassiveJobs.RabbitMqBroker](https://github.com/enadzan/massivejobs-rabbitmq)
