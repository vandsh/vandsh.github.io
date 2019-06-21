---
layout: post
title: "Dependency Injection in Kentico"
category: Kentico
tags: Kentico DependencyInjection
---

If you have been in development long enough you have likely heard of [Dependency Injection](https://en.wikipedia.org/wiki/Dependency_injection) and also are familiar with some of the benefitial side effects like testability, reuseability/replaceability and overall code quality.  When developing a _custom development_ solution your options are pretty endless, but when you are working within a CMS based solution like Kentico you usually are confined to the bounds of the framework.  Thankfully, Kentico's services [are injectable](https://docs.kentico.com/k12/developing-websites/initializing-kentico-services-with-dependency-injection) which means if done correctly, you shouldn't have to initialize your calls to out-of-box services but instead be lazy (like me) and have them be injected.

For the sake of this blog post, I am going to try and lay out the base implementation details needed to both inject the Kentico services and your own custom ones. Our solution will be Kentico 12 using [Autofac](https://www.nuget.org/packages/Autofac.Mvc5) as our DI container. Before anything, we must get the application to pick up and register the services at application start:

### DependencyInjectionConfig.cs

```csharp
namespace YourNamespace.App_Start
{
    public static class DependencyInjectionConfig
    {
        public static void RegisterDependencies()
        {
            // Initializes the Autofac builder instance
            var builder = new ContainerBuilder();

            // Adds a custom registration source (IRegistrationSource) that registers Kentico Services
            builder.RegisterSource(new CMSRegistrationSource());
            builder.RegisterControllers(typeof(MvcApplication).Assembly);

            // Find all `Service` classes by their interface and resolve the attribute/class types
            var serviceIntefaces = Assembly.GetExecutingAssembly().GetTypes()
					.Where(x => x.IsInterface && x.Name.EndsWith("Service"));

            var serviceClasses = Assembly.GetExecutingAssembly().GetTypes()
					.Where(x => !x.IsInterface && x.Name.EndsWith("Service"))
					.Select(x => 
					    new ServiceRegistration { 
					        Service = x, 
					        Interface = x.GetInterfaces().FirstOrDefault(i => serviceIntefaces.Contains(i)) 
					    });
            
	    // Find all Service that is assignable from IService and register them
            foreach(var serviceClass in serviceClasses.Where(x => x.Interface != null))
            {
                builder.RegisterType(serviceClass.Service).As(serviceClass.Interface);
            }

            // Autowire Property Injection for controllers (in case they aren't using constructor injection)
            var allControllers = Assembly.GetExecutingAssembly().GetTypes()
					.Where(type => typeof(BaseController).IsAssignableFrom(type));
            foreach (var controller in allControllers)
            {
                builder.RegisterType(controller).PropertiesAutowired();
            }

            // Resolves the dependencies
            var container = builder.Build();
            DependencyResolver.SetResolver(new AutofacDependencyResolver(container));

            var allRegs = container.ComponentRegistry.Registrations;
        }
    }

    public class ServiceRegistration
    {
        public Type Service { get; set; }
        public Type Interface { get; set; }
    }

    public class CMSRegistrationSource : IRegistrationSource
    {
        /// <summary>
        /// Gets whether the registrations provided by this source are 1:1 adapters on top of other components (I.e. like Meta, Func or Owned.)
        /// </summary>
        public bool IsAdapterForIndividualComponents => false;

        /// <summary>
        /// Retrieves registrations for an unregistered service, to be used by the container.
        /// </summary>
        /// <param name="service">The service that was requested.</param>
        /// <param name="registrationAccessor">A function that will return existing registrations for a service.</param>
        public IEnumerable<IComponentRegistration> RegistrationsFor(Service service, Func<Service, IEnumerable<IComponentRegistration>> registrationAccessor)
        {
            // Checks whether the container already contains an existing registration for the requested service
            if (registrationAccessor(service).Any())
            {
                return Enumerable.Empty<IComponentRegistration>();
            }

            // Checks that the requested service carries valid type information
            var swt = service as IServiceWithType;
            if (swt == null)
            {
                return Enumerable.Empty<IComponentRegistration>();
            }

            // Gets an instance of the requested service using the CMS.Core API
            object instance = null;
            if (CMS.Core.Service.IsRegistered(swt.ServiceType))
            {
                instance = CMS.Core.Service.Resolve(swt.ServiceType);
            }

            if (instance == null)
            {
                return Enumerable.Empty<IComponentRegistration>();
            }

            // Registers the service instance in the container
            return new[] { RegistrationBuilder.ForDelegate(swt.ServiceType, (c, p) => instance).CreateRegistration() };
        }
    }
}
``` 

Now let's invoke the config class to register those `Services` at startup:

### Global.asax.cs

```csharp
namespace YourNamespace
{
    public class MvcApplication : HttpApplication
    {
        protected void Application_Start()
        {
            // Register services both CMS and custom
            DependencyInjectionConfig.RegisterDependencies();

	    /// ... all the other things
        }
    }
}
```

The above will do two very key things:

 - Register all `Kentico Services` for injection
 - Register all custom `Services` we create
   - This keys specficially on naming conventions, if you have a `Service` end it with the word `Service`. `ExampleService` will implement `IExampleService` (below).

Doing this on `Application_Start` will take a little time on initial load, but no time during runtime.

Alright, let's go over an example of a service. Below will be a service `Interface` with an `Implementation`:

### IExampleService.cs

```csharp
namespace YourNamespace.Services.Interfaces
{
    public interface IExampleService
    {
        ITreeNode GetCurrentNode();
        IEnumerable<SubNav> GetSubNavFromAliasPath(string nodeAliasPath, CultureInfo cultureInfo, ISiteInfo siteInfo = null);
    }
}
```

### ExampleService.cs

```csharp
namespace YourNamespace.Services.Implementation
{
    /// <summary>
    /// This class is an example of how to use Dependency Injection to get common classes and services without having instantiating them on your own.
    /// Some high level reasons that may prove useful: reusability/replaceability (forces modular development)
    /// and testability (you can mock interfaces in order to properly test your written code)
    /// </summary>
    public class ExampleService : IExampleService
    {
        // Below are Kentico classes that are coded to an intercace and can be injected via Constructor Injection
        public ISiteService _siteService;
        public IEventLogService _eventLogService;
        public IHttpContextAccessor _httpContextAccessor;

        /// <summary>
        /// Any parameters in the signature should be injected via constructor injection, 
        /// if invoking this service direclty you will need to instantiate your own
        /// </summary>
        /// <param name="siteService"></param>
        /// <param name="eventLogService"></param>
        /// <param name="httpContextAccessor"></param>
        public ExampleService(ISiteService siteService, IEventLogService eventLogService, IHttpContextAccessor httpContextAccessor)
        {
            _siteService = siteService;
            _eventLogService = eventLogService;
            _httpContextAccessor = httpContextAccessor;
        }

        /// <summary>
        /// Simply gets the current node using the absolute path of the HttpContext.
        /// </summary>
        /// <param name="context"></param>
        /// <returns></returns>
        public ITreeNode GetCurrentNode()
        {
	    /// Using the injected _httpContextAccessor to get the request path
            var absolutePath = _httpContextAccessor.HttpContext.Request.Url.AbsolutePath;
            ITreeNode FoundNode = DocumentQueryHelper.GetNodeByAliasPath(absolutePath);
            return FoundNode;
        }

        /// <summary>
        /// Gets a list of `SubNav` items under the `Node` passed in.
        /// </summary>
        /// <param name="node"></param>
        /// <returns></returns>
        public IEnumerable<SubNav> GetSubNavFromAliasPath(string nodeAliasPath, CultureInfo cultureInfo = null, ISiteInfo siteInfo = null)
        {
            if (siteInfo == null)
            {
                // If site is not provided, get the current site from the injected SiteService
		// and log (may be overkill but using it as an example)
                siteInfo = _siteService.CurrentSite;
                _eventLogService.LogEvent("GetSubNavFromAliasPath", "ExampleService", "GET", "Using current site");
            }
            else
            {
                _eventLogService.LogEvent("GetSubNavFromAliasPath", "ExampleService", "GET", "Using passed in site");
            }

            if(cultureInfo == null)
            {
                cultureInfo = LocalizationContext.GetCurrentCulture();
                _eventLogService.LogEvent("GetSubNavFromAliasPath", "ExampleService", "GET", "Using current culture");
            }
            else
            {
                _eventLogService.LogEvent("GetSubNavFromAliasPath", "ExampleService", "GET", "Using passed in culture");
            }

            var subnavList = new List<SubNav>();
            try
            {
                foreach (ITreeNode Node in DocumentQueryHelper.RepeaterQuery(
                Path: nodeAliasPath + "/%",
                CultureCode: cultureInfo.CultureCode,
                SiteName: siteInfo.SiteName,
                ClassNames: "CMS.MenuItem",
                OrderBy: "NodeLevel, NodeOrder",
                Columns: "MenuItemName,NodeAliasPath"
                ))
                {
                    subnavList.Add(new SubNav()
                    {
                        LinkText = Node.GetValue("MenuItemName").ToString(),
                        // You have to decide what your URL will be, for us our URLs = NodeAliasPath
                        LinkUrl = Node.NodeAliasPath
                    });
                }
            }
            catch (Exception ex)
            {
                _eventLogService.LogException("ExampleService", "GET", ex);
            }
            return subnavList;
        }
    }
}
```

Above is a trimmed down (for the sake of demonstration) version of what you will see in [here](https://github.com/KenticoDevTrev/KenticoBoilerplate_v12). 
Using the `Service` naming convention, we can pick it up in the `DependencyInjection.RegisterDependencies()` (line 15) method and register it with its similarly named Interface. 
I generally use this approach as a form of lazy-registering-by-naming to get off the ground fast within development, but when it comes to replace an implementation I usually will create a new `Service` and manually register it until it is fully fleshed out and then potentially rename it so it continues to get picked up by this method.  

Now, back to the `ExampleService`, this service leverages the following `Kentico` Services:

 - `CMS.Base.ISiteService` to get the current site info
 - `CMS.Core.IEventLogService` to allow us to log events
 - `CMS.Base.IHttpContextAccessor` to access the http context

The `ExampleService` itself illustrates using `Constructor Injection` where already instantiated services are passed in thru the constructor. This `ExampleService` is ultimately used in a controller, `ExampleController`:

### ExampleController.cs

```csharp
namespace YourNamespace.Controllers.Examples
{
    public class ExamplesController : BaseController
    {
        public IExampleService _exampleService;
        public ExamplesController(IExampleService exampleService)
        {
            // Use constructor injection to get a handle on our ExampleService
            _exampleService = exampleService;
        }

        // GET: Examples
        public ActionResult Index()
        {
            return View();
        }


        /// <summary>
        /// This Repeater "Webpart" Relies on just a path that would be provided through the View's context.  Does not rely on passing
        /// the ViewBag like the NavigationByContext, but does then require the calling View to provide the properties, and if ever more
        /// properties are needed, would need to adjust both Controller and View alike.
        /// </summary>
        /// <returns></returns>
        public ActionResult NavigationByPath(string Path, string Culture, string SiteName)
        {
            // Build the actual Partial View's model from the data provided by the parent View
            ExampleMVCWebPartsSubNavs Model = new ExampleMVCWebPartsSubNavs();
            // Get the Sub Nav Items from the ExampleService
            Model.SubNavigation = _exampleService.GetSubNavFromAliasPath(Path, CultureInfoProvider.GetCultureInfo(Culture));
            return View("Navigation", Model);
        }
    }
}
``` 

So at this point, we have a `Controller` with an injected dependency, `IExampleService`, that is registered at startup and that class itself uses some injected `Kentico` services. So one thing that may be apparent, this example is not perfect as some of the dependencies like `DocumentQueryHelper` are both static and not injected, but that is something we hope to have ironed out in the future.  The only thing left that I would love to illstrate and, in my opinion, is one of the biggest benefits to leveraging DI is testing (but that will have to be for a later blog post!)

And like I mentioned before, this is all [publicly accessible](https://github.com/KenticoDevTrev/KenticoBoilerplate_v12) and brought to you from the beautiful Kentico minds at [Heartland Business Sytems](https://twitter.com/HBSTech).

I am hoping this post helps you put some of the pieces together on how to use `Dependency Injection` in your next `Kentico` project and reap the benefits in all their glory!

</message>
