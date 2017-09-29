
As easy as it sounds, this proved to be a difficult task. And that’s a good reason to detail here the steps required to make sure exceptions are being handled and logged in the Azure AppInsights.

Most of the time is handy to use the existing abstractions provided by framework, and [HttpResponseException](https://msdn.microsoft.com/en-us/library/system.web.http.httpresponseexception(v=vs.118).aspx) is just one of them. It helps translating an exception into a REST API  reponse with just a few lines of code. The difficulty comes from the fact that this is a special exception that unfortunately cannot be handled as the other web application exceptions described in [Exception Handling in ASP.NET Web API](https://docs.microsoft.com/en-us/aspnet/web-api/overview/error-handling/exception-handling).

Note that this applies to a classic WebApi project and makes use of the Dependency Injection provided by the WebApi or Autofac. Before starting, it’s also recommended to check out the official docs about how to [log to Azure AppInsights](https://docs.microsoft.com/en-us/azure/application-insights/app-insights-asp-net-exceptions).

It turns out that two exception "traps" have to be setup. First create a custom [ExceptionLogger](https://msdn.microsoft.com/en-us/library/system.web.http.exceptionhandling.exceptionlogger(v=vs.118).aspx) implementation that will do its logging in the Azure AppInsights. This will handle all exception, except HttpResponseException -- which has a special behavior dictated by the WebApi controllers.

```
using System.Web.Http.ExceptionHandling;
using Microsoft.ApplicationInsights;
using System;
using System.Web.Http;

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
                var ex = context.Exception ?? new Exception(context.ToString());
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
                var ex = new Exception(actionExecutedContext.Response.ToString());
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
