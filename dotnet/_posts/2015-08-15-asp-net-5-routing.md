---
layout: post
title: Маршрутизация в ASP.NET 5
tags: ASP-NET-5
---

>Эта статья была опубликована в корпоративном блоге Microsoft на habrahabr.ru: [Ссылка](http://habrahabr.ru/company/microsoft/blog/268037/).

Сегодня мы посмотрим на систему маршрутизации в ASP.NET 5.

## Как была организована система маршрутизации до ASP.NET 5
Маршрутизация до ASP.NET 5 осуществлялась с помощью ASP.NET модуля [UrlRoutingModule](https://msdn.microsoft.com/en-us/library/system.web.routing.urlroutingmodule(v=vs.100).aspx). Модуль проходил через [коллекцию](https://msdn.microsoft.com/en-us/library/system.web.routing.routecollection(v=vs.100).aspx) маршрутов (как правило объектов класса [Route](https://msdn.microsoft.com/en-us/library/system.web.routing.route(v=vs.110).aspx)) хранящихся в статическом свойстве `Routes` класса [RouteTable](https://msdn.microsoft.com/en-us/library/system.web.routing.routetable(v=vs.110).aspx), выбирал маршрут, который подходил под текущий запрос и вызывал обработчик маршрута, который хранился в свойстве `RouteHandler` класса `Route` - каждый зарегистрированный маршрут мог иметь собственный обработчик. В MVC-приложении этим обработчиком был [MvcRouteHandler](https://msdn.microsoft.com/en-us/library/system.web.mvc.mvcroutehandler(v=vs.118).aspx), который брал на себя дальнейшую работу с запросом.

Маршруты в коллекцию `RouteTable.Routes` мы добавляли в процессе настройки приложения. 

Типичный код настройки системы маршрутизации в MVC приложении:

```csharp
RouteTable.Routes.MapRoute(
    name: "Default",
    url: "{controller}/{action}/{id}",
    defaults: new { controller = "Home", action = "Index", id = UrlParameter.Optional });
```

Где [MapRoute](https://msdn.microsoft.com/ru-ru/library/Dd470521(v=VS.118).aspx) - extension-метод, объявленный в пространстве имен `System.Web.Mvc`, который добавлял в коллекцию маршрутов в свойстве `Routes` новый маршрут используя `MvcRouteHandler` в качестве обработчика.

<!--excerpt-->

Мы могли бы сделать это и самостоятельно:

```csharp
RouteTable.Routes.Add(new Route(
    url: "{controller}/{action}/{id}",
    defaults: new RouteValueDictionary(new { controller = "Home", action = "Index", id = UrlParameter.Optional }),
    routeHandler: new MvcRouteHandler()));
```

## Как организована система маршрутизации в ASP.NET 5: Короткий вариант
ASP.NET 5 больше не использует модули, для обработки запросов используются "middleware" введенные в рамках перехода на [OWIN](http://docs.asp.net/en/latest/fundamentals/owin.html) - "Open Web Interface" - позволяющей запускать ASP.NET 5 приложения не только на сервере IIS.

Поэтому сейчас маршрутизация осуществляется с помощью [RouterMiddleware](https://github.com/aspnet/Routing/blob/master/src/Microsoft.AspNet.Routing/RouterMiddleware.cs). Весь проект реализующий маршрутизацию [можно загрузить с github](https://github.com/aspnet/Routing). В рамках этой концепции запрос передается от одного middleware к другому, в порядке их регистрации при старте приложения. Когда запрос доходит до `RouterMiddleware` оно сравнивает подходит ли запрашиваемый Url адрес для какого-нибудь зарегистрированного маршрута, и если подходит, вызывает обработчик этого маршрута.

## Как организована система маршрутизации в ASP.NET 5: Длинный вариант

Для того, чтобы разобраться как система маршрутизации работает, давайте подключим ее к пустому проекту ASP.NET 5.

0. Cоздайте пустой проект ASP.NET 5 (выбрав Empty Template) и назовите его "AspNet5Routing".

1. Добавляем в зависимости ("dependencies") проекта в файле project.json "Microsoft.AspNet.Routing":

```json
"dependencies": {
    "Microsoft.AspNet.Server.IIS": "1.0.0-beta5",
    "Microsoft.AspNet.Server.WebListener": "1.0.0-beta5",
    "Microsoft.AspNet.Routing": "1.0.0-beta5"
},
```

2. В файле `Startup.cs` добавляем использование пространства имен `Microsoft.AspNet.Routing`:

```csharp
using Microsoft.AspNet.Routing;
```

3. Добавляем необходимые сервисы (сервисы, которые использует в своей работе система маршрутизации) в методе ConfigureServices() файла Startup.cs:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddRouting();
}
```

4. И наконец настраиваем систему маршрутизации в методе Configure() файла Startup.cs:

```csharp
public void Configure(IApplicationBuilder app)
{
    var routeBuilder = new RouteBuilder();
    routeBuilder.DefaultHandler = new ASPNET5RoutingHandler();
    routeBuilder.ServiceProvider = app.ApplicationServices;
    routeBuilder.MapRoute("default", "{controller}/{action}/{id}");
    app.UseRouter(routeBuilder.Build());
}
```

Взято из [примера в проекте маршрутизации](https://github.com/aspnet/Routing/blob/master/samples/RoutingSample.Web/Startup.cs).

Разберем последний шаг подробнее:

```csharp
var routeBuilder = new RouteBuilder();
routeBuilder.DefaultHandler = new ASPNET5RoutingHandler();
routeBuilder.ServiceProvider = app.ApplicationServices;
```

Создаем экземпляр `RouteBuilder` и заполняем его свойства. Интерес вызывает свойство `DefaultHandler` с типом [IRouter](https://github.com/aspnet/Routing/blob/master/src/Microsoft.AspNet.Routing/IRouter.cs) - судя по названию оно должно содержать обработчик запроса. Я помещаю в него экземпляр `ASPNET5RoutingHandler` - придуманного мною обработчика запросов, давайте создадим его:

```csharp
using Microsoft.AspNet.Routing;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNet.Http;

namespace AspNet5Routing
{
    public class ASPNET5RoutingHandler : IRouter
    {
        public VirtualPathData GetVirtualPath(VirtualPathContext context)
        {

        }

        public async Task RouteAsync(RouteContext context)
        {
            await context.HttpContext.Response.WriteAsync("ASPNET5RoutingHandler work");
            context.IsHandled = true;
        }
    }
}
```

Интерфейс `IRouter` требует от нас только два метода `GetVirtualPath` и `RouteAsync`.

Метод `GetVirtualPath` - нам знаком из предыдущих версий ASP.NET он был в интерфейсе класса [RouteBase](https://msdn.microsoft.com/en-us/library/system.web.routing.routebase(v=vs.110).aspx) от которого наследовался класс [Route](https://msdn.microsoft.com/en-us/library/system.web.routing.route(v=vs.110).aspx) представляющий собой маршрут. Этот метод отвечал за построение Url (например, когда мы вызывали метод ActionLink: `@Html.ActionLink("link", "Index")`).

А в методе `RouteAsync` - мы обрабатываем запрос и записываем результат обработки в `Response`.

Следующая строка метода `Configure`:

```csharp
routeBuilder.MapRoute("default", "{controller}/{action}/{id}");
```

Как две капли воды, похожа на использование метода `MapRoute` в MVC 5, его параметры - название добавляемого маршрута и шаблон с которым будет сопоставляться запрашиваемый Url.

Сам `MapRoute()` также как и в MVC 5 - [extension-метод](https://github.com/aspnet/Routing/blob/master/src/Microsoft.AspNet.Routing/RouteBuilderExtensions.cs), а его вызов в итоге сводится к Созданию экземпляра класса [TemplateRoute](https://github.com/aspnet/Routing/blob/master/src/Microsoft.AspNet.Routing/Template/TemplateRoute.cs) и добавлению его в коллекцию `Routes` нашего объекта `RouteBuilder`:

```csharp
routeBuilder.Routes.Add(new TemplateRoute(routeCollectionBuilder.DefaultHandler,
    name, // в нашем случае передается "default"
    template, // в нашем случае передается "{controller}/{action}/{id}"
    ObjectToDictionary(defaults),
    ObjectToDictionary(constraints),
    ObjectToDictionary(dataTokens),
    inlineConstraintResolver));
```

Что интересно свойство `Routes` - это коллекция `IRouter`, то есть `TemplateRoute` тоже реализует интерфейс `IRouter`, как и созданный нами `ASPNET5RoutingHandler`, кстати, он передается в конструктор `TemplateRoute`.

И наконец последняя строчка:

```csharp
app.UseRouter(routeBuilder.Build());
```

Вызов `routeBuilder.Build()` - создает экземпляр класса [RouteCollection](https://github.com/aspnet/Routing/blob/master/src/Microsoft.AspNet.Routing/RouteCollection.cs) и добавляет в него все элементы из свойства `Route` класса `RouteBuilder`.

А `app.UseRouter()` - [оказывается](https://github.com/aspnet/Routing/blob/master/src/Microsoft.AspNet.Routing/BuilderExtensions.cs) extension-методом, который на самом деле, подключает [RouterMiddleware](https://github.com/aspnet/Routing/blob/master/src/Microsoft.AspNet.Routing/RouterMiddleware.cs) в pipeline обработки запроса, передавая ему созданный и заполненный в методе `Build()` объект `RouteCollection`.

```csharp
public static IApplicationBuilder UseRouter([NotNull] this IApplicationBuilder builder, [NotNull] IRouter router)
{
    return builder.UseMiddleware<RouterMiddleware>(router);
}
```

И судя по конструктору `RouterMiddleware`:

```csharp
public RouterMiddleware(
    RequestDelegate next,
    ILoggerFactory loggerFactory,
    IRouter router)
```

Объект `RouteCollection` тоже реализует интерфейс `IRouter`, как и `ASPNET5RoutingHandler` c `TemplateRoute`.

Итого у нас получилась следующая матрешка:

Наш обработчик запроса `ASPNET5RoutingHandler` упакован в `TemplateRoute`, сам `TemplateRoute` или несколько экземпляров `TemplateRoute` (если бы мы несколько раз вызвали метод `MapRoute()`) упакованы в `RouteCollection`, а `RouteCollection` передан в конструктор `RouterMiddleware` и сохранен в нем.  

На этом процесс настройки системы маршрутизации завершен, можно запустить проект, перейти по адресу: "/Home/Index/1" и увидеть результат: "ASPNET5RoutingHandler work".
 
Ну и кратко пройдемся по тому, что происходит, с системой маршрутизации во время входящего запроса:

Когда очередь доходит до `RouterMiddleware`, в списке запускаемых middleware, оно вызывает метод `RouteAsync()` у сохраненного экземпляра `IRouter` - это объект класса `RouteCollection`.

`RouteCollection` в свою очередь проходит по сохраненным в нем экземплярам `IRouter` - в нашем случае это будет `TemplateRoute` и вызывает у них метод `RouteAsync()`. 

`TemplateRoute` проверяет соответствует ли запрашиваемый Url, его шаблону (передавали в конструкторе TemplateRoute: "{controller}/{action}/{id}") и если совпадает, вызывает хранящийся в нем экземпляр `IRouter` - которым является наш `ASPNET5RoutingHandler`.

##Подключаем систему маршрутизации к MVC приложению##

Теперь давайте посмотрим как связывается MVC Framework с системой маршрутизации.

Снова создадим пустой проект ASP.NET 5 используя Empty шаблон.

1. Добавляем в зависимости ("dependencies") проекта в файле project.json "Microsoft.AspNet.Mvc":

```json
"dependencies": {
    "Microsoft.AspNet.Server.IIS": "1.0.0-beta5",
    "Microsoft.AspNet.Server.WebListener": "1.0.0-beta5",
    "Microsoft.AspNet.Mvc": "6.0.0-beta5"
},
```

2. В файле `Startup.cs` добавляем использование пространства имен `Microsoft.AspNet.Builder`:
			
```csharp
using Microsoft.AspNet.Builder;
```

Нужные нам extensions-методы для подключения MVC находятся в нем.

3. Добавляем сервисы, которые использует в своей работе MVC Framework: в методе ConfigureServices() файла Startup.cs:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc();
}
```

4. Настраиваем MVC приложение в методе Configure() файла Startup.cs:

Нам доступны три разных метода:

1.

```csharp
public void Configure(IApplicationBuilder app)
{
    app.UseMvc()
}
```

2.

```csharp
public void Configure(IApplicationBuilder app)
{
    app.UseMvcWithDefaultRoute()
}
```
 
3.

```csharp
public void Configure(IApplicationBuilder app)
{
    return app.UseMvc(routes =>
    {
        routes.MapRoute(
            name: "default",
            template: "{controller=Home}/{action=Index}/{id?}");
    });
}
```

Давайте сразу посмотрим [реализацию этих методов](https://github.com/aspnet/Mvc/blob/master/src/Microsoft.AspNet.Mvc.Core/Builder/MvcApplicationBuilderExtensions.cs):

Первый метод:

```csharp
public static IApplicationBuilder UseMvc(this IApplicationBuilder app)
{
    return app.UseMvc(routes =>
    {
    });
}
```

Вызывает третий метод, передавая делегат `Action<IRouteBuilder>` который ничего не делает.

Второй метод:

```csharp
public static IApplicationBuilder UseMvcWithDefaultRoute(this IApplicationBuilder app)
{
    return app.UseMvc(routes =>
    {
        routes.MapRoute(
            name: "default",
            template: "{controller=Home}/{action=Index}/{id?}");
    });
}
```

Тоже вызывает третий метод, только в делегате `Action<IRouteBuilder>` добавляется дефолтный маршрут.

Третий метод:

```csharp
public static IApplicationBuilder UseMvc(
    this IApplicationBuilder app,
    Action<IRouteBuilder> configureRoutes)
{
    MvcServicesHelper.ThrowIfMvcNotRegistered(app.ApplicationServices);

    var routes = new RouteBuilder
    {
        DefaultHandler = new MvcRouteHandler(),
        ServiceProvider = app.ApplicationServices
    };

    configureRoutes(routes);

    // Adding the attribute route comes after running the user-code because
    // we want to respect any changes to the DefaultHandler.
    routes.Routes.Insert(0, AttributeRouting.CreateAttributeMegaRoute(
        routes.DefaultHandler,
        app.ApplicationServices));

    return app.UseRouter(routes.Build());
}
```

Делает тоже самое, что и мы в предыдущем разделе при регистрации маршрута для своего обработчика, только устанавливает в качестве конечного обработчика экземпляр `MvcRouteHandler` и делает вызов метода `CreateAttributeMegaRoute` - который отвечает за добавление маршрутов устанавливаемых с помощью атрибутов у контроллеров и методов действий (Attribute-Based маршрутизация).

Таким образом все три метода, будут включать в наше приложение Attribute-Based маршрутизацию, но кроме этого, вызов второго метода будет добавлять дефолтный маршрут, а третий метод, позволяет задать любые нужные нам маршруты передав их с помощью делегата (Convention-Based маршрутизация).

## Convention-Based маршрутизация

Как я уже писал выше - настраивается с помощью вызова метода `MapRoute()` - и процесс использования этого метода не изменился со времен MVC 5 - в метод `MapRoute()` мы можем передать имя маршрута, его шаблон, значения по-умолчанию и ограничения.

```csharp
routeBuilder.MapRoute("regexStringRoute", //name
    "api/rconstraint/{controller}", //template
    new { foo = "Bar" }, //defaults
    new { controller = new RegexRouteConstraint("^(my.*)$") }); //constraints
```

## Attribute-Based маршрутизация

В отличии от MVC 5, где маршрутизацию с помощью атрибутов, нужно было специально включать, в MVC 6 она включена по-умолчанию.

Следует также помнить, что маршруты определяемые с помощью атрибутов, имеют приоритет при поиске совпадений и выборе подходящего маршрута (по-сравнению с convention-based маршрутами).

Для задания маршрута нужно использовать атрибут `Route` как у методов действий, так и у контроллера (в MVC 5 для задания маршрута у контроллера использовался атрибут `RoutePrefix`).

```csharp
[Route("appointments")]
public class Appointments : ApplicationBaseController
{
    [Route("check")]
    public IActionResult Index()
    {
        return new ContentResult
        {
            Content = "2 appointments available."
        };
    }
}
```

В итоге данный метод действия будет доступен по адресу: "/appointments/check".

## Настройка системы маршрутизации
В ASP.NET 5 появился новый механизм настройки сервисов называющийся `Options` - [GitHub проекта](https://github.com/aspnet/Options). Он позволяет произвести некоторые настройки системы маршрутизации. 

Смысл его работы, сводится к тому, что при настройке приложения в файле `Startup.cs` мы передаем в систему регистрации зависимостей некий объект, с заданными определенным образом свойствами, а во время работы приложения этот объект достается и в зависимости от значений выставленных свойств приложение строит свою работу. 

Для настройки системы маршрутизации используется класс [RouteOptions](https://github.com/aspnet/Routing/blob/master/src/Microsoft.AspNet.Routing/RouteOptions.cs).

Для удобства нам доступен метод расширения `ConfigureRouting`:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.ConfigureRouting(
        routeOptions =>
        {
            routeOptions.LowercaseUrls = true; // генерация url в нижнем регистре
            routeOptions.AppendTrailingSlash = true; // добавление слеша в конец url
        });
}
```

"За кулисами" он просто делает вызов метода `Configure` передавая в него делегат `Action<RouteOptions>`:

```csharp
public static void ConfigureRouting(
    this IServiceCollection services,
    Action<RouteOptions> setupAction)
{
    if (setupAction == null)
    {
        throw new ArgumentNullException(nameof(setupAction));
    }

    services.Configure(setupAction);
}
```

## Шаблон маршрута
Принципы работы с шаблоном маршрута остались теми же, что были и в MVC 5:

- Сегменты адреса разделены слешем: `firstSegment/secondSegment`.
- Константной часть сегмента считается, если она не окаймлена фигурными скобками, соответствие такого маршрута запрашиваемому Url, происходит только если в адресе присутствуют точно такие же значения: `firstSegment/secondSegment` - такой маршрут соответствует только адресу вида: `siteDomain/firstSegment/secondSegment`.
- Переменные части сегмента берутся в фигурные скобки: `firstSegment/{secondSegment}` - шаблон будет соответствовать любым двух сегментным адресам, где первый сегмент: "firstSegment", а второй сегмент может быть любым набором символов (кроме слеша - так как это будет обозначать начало третьего сегмента):

	"/firstSegment/index"

	"/firstSegment/index-2"
- Ограничения для переменной части сегмента, как это следует из названия - ограничивают допустимые значения переменного сегмента и задаются после символа ":". На одну переменную часть можно наложить несколько ограничений, параметры передаются с использованием круглых скобок: `firstSegment/{secondSegment:minlength(1):maxlength(3)}`.
Строковое обозначение ограничений можно посмотреть в [методе `GetDefaultConstraintMap()` класса RouteOptions](https://github.com/aspnet/Routing/blob/master/src/Microsoft.AspNet.Routing/RouteOptions.cs).
- Для того, чтобы сделать последний сегмент "жадным", так что он будет поглощать всю оставшуюся строку адреса, нужно использовать символ `*`: `{controller}/{action}/{*allRest}`  - будет соответствовать как адресу: "/home/index/2", так и адресу: "/home/index/2/4/5".

Но в ASP.NET 5 шаблон маршрута получил некоторые дополнительные возможности: 

1. Возможность задавать прямо в нем значения по-умолчанию для переменных частей маршрута: `{controller=Home}/{action=Index}`.
2. Задавать не обязательность переменной части сегмента с помощью символа `?`: `{controller=Home}/{action=Index}/{id?}`.

Также при использовании шаблона маршрута в атрибутах, произошли изменения:

При настройке маршрутизации через атрибуты, к параметрам обозначающим контроллер и метод действия, теперь следует обращаться беря их в квадратные скобки и используя слова "controller" и "action": "[controller]/[action]" - и использовать их можно только в таком виде - не разрешаются ни значения по-умолчанию, ни ограничения, ни опциональность, ни жадность.

То есть, разрешается:

```csharp
Route("[controller]/[action]/{id?}")
Route("[controller]/[action]")
```

Можно использовать их по отдельности:

```csharp
Route("[controller]")
Route("[action]")
```

Не разрешаются:

```csharp
Route("{controller}/{action}")
Route("[controller=Home]/[action]")
Route("[controller?]/[action]")
Route("[controller]/[*action]")
```

Общая схема шаблона маршрута выглядит так:

```csharp
constantPart-{variablePart}/{paramName:constraint1:constraint2=DefaultValue?}/{*lastGreedySegment}
```

## Заключение:

В этой статье, мы пробежались по системе маршрутизации ASP.NET 5, посмотрели как она организована и подключили ее к пустому проекту ASP.NET 5, используя свой обработчик маршрута. Разобрали способы ее подключения к MVC приложению и настройке с помощью механизма Опций. Остановились на изменениях, которые произошли в использовании Attribute-Based маршрутизации и шаблоне маршрута.
