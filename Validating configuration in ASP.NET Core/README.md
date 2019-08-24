# Validating configuration in ASP.NET Core
Configuration has been greatly improved in .NET Core, compared with the .NET framework. In this post, I extend this by adding validation of configuration in order to get information about ill-configured applications as soon as possible, and as clearly as possible. It will help ensure that application configuration is in a well-defined state.
## Configuration out of the box
There’s a lot to be said about configuration in ASP.NET core, much of it is discussed in this excellent blog post: [Deep Dive into Microsoft Configuration](https://www.paraesthesia.com/archive/2018/06/20/microsoft-extensions-configuration-deep-dive/). Let’s focus on one area, which is having strongly typed configuration. Here’s a brief introduction to how it works.
First, consider an *appsettings.json* file which have a section **AzureAdClientSettings**. This will represent configuration that an app would need to connect to Azure Active Directory:
```C#
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  },
  "AzureAdClientSettings": {
    "ClientId": "8117d6ba-c472-4319-bf60-452bd178a374",
    "ClientSecret": "KkKFsP7VPGLrhT/5+YAtkTvUfFQ+TJ0bveobMmiYN04=",
    "TenantId": "9cde71ba-4e99-413c-917c-3cbc32e8864d"
  }
}
```
Now, consider having the following class that will represent a strongly typed definition for this configuration:
```C#
public class AzureAdClientSettings
{
    public string ClientId { get; set; }
    public string ClientSecret { get; set; }
    public string TenantId { get; set; }
}
```
```C#
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.Configure<AzureAdClientSettings>(
            Configuration.GetSection(nameof(AzureAdClientSettings)));
    }

    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }
}
```
In our ASP.NET application Startup class, we have the class AzureAdClientSettings bound to our configuration section. Notice that if we introduce the convention that the class name is the same as the section name in configuration, we can use nameof(AzureAdClientSettings) for the name of the section to bind to. With this, we have a working configuration setup.

Now, let’s look at how we can use this configuration in our code. Let’s say we have a HomeController class, where we need to use it. We can use constructor injection to have passed an instance of IOptions<AzureAdClientSettings>. Then, we access it’s .Value property to get our configuration instance:

```C#
public class HomeController : Controller
{
    private readonly AzureAdClientSettings _settings;

    public HomeController(IOptions<AzureAdClientSettings> settings)
    {
        _settings = settings.Value;
    }
}
```
That’s it for the basic configuration wiring. Now, let’s improve on it.

## Adding validation to settings model
The first way to improve this, is to validate the configuration before we start using it. Let’s add some familiar System.ComponentModel.DataAnnotations to the configuration class:
```C#
using System.ComponentModel.DataAnnotations;

public class AzureAdClientSettings
{
    [Required]
    public string ClientId { get; set; }

    [Required]
    [MinLength(20)]
    public string ClientSecret { get; set; }

    [Required]
    [RegularExpression(@"^[0-9A-Fa-f]{8}[-]?([0-9A-Fa-f]{4}[-]?){3}[0-9A-Fa-f]{12}$")]
    public string TenantId { get; set; }
}
```
The DataAnnotations validation API is really awkward to work with, so I like to wrap this awkwardness in an extension method:
```C#
public static class ValidationExtensions
{
    public static IEnumerable<string> ValidationErrors(this object @this)
    {
        var context = new ValidationContext(@this, serviceProvider: null, items: null);
        var results = new List<ValidationResult>();
        Validator.TryValidateObject(@this, context, results, true);
        foreach (var validationResult in results)
        {
            yield return validationResult.ErrorMessage;
        }
    }
}
```
With the validation attributes and the extension method in place, we can add code to validate the configuration before we use it:
```C#
public class HomeController : Controller
{
    private readonly AzureAdClientSettings _settings;

    public HomeController(IOptions<AzureAdClientSettings> settings)
    {
        _settings = settings.Value;
    }

    public IActionResult Index()
    {
        var configurationErrors = _settings.ValidationErrors().ToArray();
        if (configurationErrors.Any())
        {
            throw new ApplicationException(
                $"Found {configurationErrors.Length} configuration error(s) in {_settings.GetType().Name}: {configurationErrors.Aggregate((a, b) => a + "," + b)}");
        }

        return View();
    }
}
```
## Fail fast — validating on first use
Having an easy way to validate the configuration is good, but still a little cumbersome to have to do this whenever the configuration is used in client code. Luckily, there is a PostConfigure<T>(Action<T>) method that we can use to register a callback which will be called before the configuration is used. So, let’s make an addition in our Startup class were we do the validation:
  
  ```C#
  public void ConfigureServices(IServiceCollection services)
{
    services.Configure<AzureAdClientSettings>(
        Configuration.GetSection(nameof(AzureAdClientSettings)));

    services.PostConfigure<AzureAdClientSettings>(settings =>
    {
        var configrrors = settings.ValidationErrors().ToArray();
        if (configErrors.Any())
        {
            var aggrErrors = string.Join(",", configErrors);
            var count = configErrors.Length;
            var configType = settings.GetType().Name;
            throw new ApplicationException(
                $"Found {count} configuration error(s) in {configType}: {aggrErrors}");
        }
    });
}
  ```
Then we can get rid of the explicit validation in our client code, and the PostConfigure callback will be called whenever we reference the .Value property of IOptions<T> :

```C#
public class HomeController : Controller
{
    private readonly AzureAdClientSettings _settings;

    public HomeController(IOptions<AzureAdClientSettings> settings)
    {
        _settings = settings.Value; // <-- FAIL HERE
    }
}
```
## Cleaning up
We now have our functional improvements in place, so let’s clean up the code a bit. Let’s add an extension method for IServiceCollection which combines Configure<T> and PostConfigure<T> :
  
```C#
public static class ServiceCollectionExtensions
{
    public static IServiceCollection ConfigureAndValidate<T>(this IServiceCollection @this,
        IConfiguration config) where T : class
        => @this
            .Configure<T>(config.GetSection(typeof(T).Name))
            .PostConfigure<T>(settings =>
            {
                var configErrors = settings.ValidationErrors().ToArray();
                if (configErrors.Any())
                {
                    var aggrErrors = string.Join(",", configErrors);
                    var count = configErrors.Length;
                    var configType = typeof(T).Name;
                    throw new ApplicationException(
                        $"Found {count} configuration error(s) in {configType}: {aggrErrors}");
                }
            });
}
```
Given that we uphold the convention that our config section name is the same as the class name, we end up with one line to register a configuration section and having it validated:
```C#
public void ConfigureServices(IServiceCollection services)
{
    services.ConfigureAndValidate<AzureAdClientSettings>(Configuration);
}
```
