
As easy as it sounds, this proved to be a difficult task. For that reason, I will log here the steps required to make sure exceptions are being logged in the Azure AppInsights. Note that this applies to a classic WebApi project and supports Dependency Injection provided by the WebApi or Autofac.

An API can make use heavily of the [HttpResponseException](https://msdn.microsoft.com/en-us/library/system.web.http.httpresponseexception(v=vs.118).aspx) which unfortunately cannot be handled as the other web application exceptions described in [Exception Handling in ASP.NET Web API](https://docs.microsoft.com/en-us/aspnet/web-api/overview/error-handling/exception-handling). 
Before starting, it's also recommended to check out the official docs about how to [log to Azure AppInsights](https://docs.microsoft.com/en-us/azure/application-insights/app-insights-asp-net-exceptions).

It turns out that two exception "traps" have to be created.
First create a custom [ExceptionLogger](https://msdn.microsoft.com/en-us/library/system.web.http.exceptionhandling.exceptionlogger(v=vs.118).aspx) implementation that will do its logging in the Azure AppInsights. This will handle all exception, except HttpResponseException -- which has a special behavior dictated by the WebApi controllers.

```
using System.Web.Http.ExceptionHandling;
using Microsoft.ApplicationInsights;
using System;
using System.Web.Http;
using Newtonsoft.Json;

namespace WebApi
{
    public class AppInsightsExceptionLogger : ExceptionLogger
    {
        private readonly TelemetryClient _telemetryClient;

        public AppInsightsExceptionLogger(TelemetryClient telemetryClient)
        {
            _telemetryClient = telemetryClient;
        }

        /// <inheritdoc />
        public override void Log(ExceptionLoggerContext context)
        {
            if (context != null)
            {
                var ex = context.Exception ?? new Exception(JsonConvert.SerializeObject(context));
                _telemetryClient.TrackException(ex);
            }

            base.Log(context);
        }
    }
}
```

Secondly, create a custom [ActionFilterAttribute](https://msdn.microsoft.com/en-us/library/system.web.mvc.actionfilterattribute(v=vs.118).aspx) implementation that will also log in the Azure AppInsights. This is the only known way to handle the erratic HttpResponseException.

```
using Microsoft.ApplicationInsights;
using Newtonsoft.Json;
using System;
using System.Web.Http;
using System.Web.Http.Filters;

namespace WebApi
{
    public class HttpResponseExceptionTrapAttribute : ActionFilterAttribute
    {
        private readonly TelemetryClient _telemetryClient;

        public HttpResponseExceptionTrapAttribute(TelemetryClient telemetryClient)
        {
            _telemetryClient = telemetryClient;
        }

        /// <inheritdoc />
        public override void OnActionExecuted(HttpActionExecutedContext actionExecutedContext)
        {
            if (actionExecutedContext != null &&
                actionExecutedContext.Response != null &&
                !actionExecutedContext.Response.IsSuccessStatusCode)
            {
                var ex = new Exception(JsonConvert.SerializeObject(actionExecutedContext.Response));
                _telemetryClient.TrackException(ex);
            }

            base.OnActionExecuted(actionExecutedContext);
        }
    }
}
```


Then configure them.

```
using System.Web.Http;
using System.Web.Http.ExceptionHandling;

namespace WebApi
{
    public static class WebApiConfig
    {
        public static void Register(HttpConfiguration config)
        {
            // Handle routing
            config.MapHttpAttributeRoutes();
            config.Routes.MapHttpRoute(
                name: "DefaultApi",
                routeTemplate: "api/{controller}/{action}/{id}",
                defaults: new { id = RouteParameter.Optional }
            );

            // Wire up global exception logger(s)
            config.Services.Add(typeof(IExceptionLogger), config.DependencyResolver.GetService(typeof(AppInsightsExceptionLogger)));
            GlobalConfiguration.Configuration.Filters.Add((HttpResponseExceptionTrapAttribute)config.DependencyResolver.GetService(typeof(HttpResponseExceptionTrapAttribute)));
        }
    }
}
```

I hope this helps.
