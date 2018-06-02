---
layout: post
title: Погружение в ASP.NET 5 Runtime
tags: ASP-NET-5 DNX
---

>Последняя версия этой статьи опубликована на [habrahabr.ru](http://habrahabr.ru/post/273509/)

## Вступление от переводчика

Данная статья является переводом [ASP.NET 5 - A Deep Dive into the ASP.NET 5 Runtime](https://msdn.microsoft.com/en-us/magazine/dn913182.aspx) - введения в архитектуру DNX Runtime и построенного на нем ASP.NET 5. Так как оригинальная статья была написана в марте 2015 года, во время, когда ASP.NET 5 был еще в стадии активной разработки (примерно beta 3), многое в ней устарело. Поэтому при переводе вся информация была актуализирована до текущей версии ASP.NET 5 (RC1), также были добавлены ссылки на связанные ресурсы (в основном на docs.asp.net) и исходный код на GitHub (смотрите только в случаях, если вам интересна реализация). Приятного погружения!

## .NET Runtime Environment (DNX)

ASP.NET базируется на гибком, кроссплатформенном runtime, который может работать с разными .NET CLR (.NET Core CLR, Mono CLR, .NET Framework CLR). Вы можете запустить ASP.NET 5 используя полный .NET Framework или можете запустить используя новый .NET Core [(docs.asp.net: Introducing .NET Core)](https://docs.asp.net/en/latest/conceptual-overview/dotnetcore.html), который позволяет вам просто копировать все необходимые библиотеки вместе с приложением в существующее окружение, без изменения чего-либо еще на вашей машине. Используя .NET Core вы также можете запустить ASP.NET 5 кроссплатформенно на Linux [(docs.asp.net: Installing ASP.NET 5 On Linux)](https://docs.asp.net/en/latest/getting-started/installing-on-linux.html) и Mac OS [(docs.asp.net: Installing ASP.NET 5 On Mac OS X)](https://docs.asp.net/en/latest/getting-started/installing-on-mac.html).

<!--excerpt-->

Инфраструктура позволяющая запускать и исполнять приложения ASP.NET 5 называется .NET Runtime Environment или кратко DNX [(docs.asp.net: DNX Overview)](https://docs.asp.net/en/latest/dnx/overview.html). DNX предоставляет все что необходимо для работы .NET приложений: host process, CLR hosting логику, обнаружение управляемой Entry Point и т.д.

Логически архитектура DNX имеет пять слоев. Я опишу каждый из этих слоев вместе с их обязанностями.

![Архитектура ASP.NET 5 и DNX](/images/asp-net-5/runtime/dnxDiagram2.jpg)

Изображение взято из статьи [DNX-structure](https://github.com/aspnet/Home/wiki/DNX-structure)

## Слой первый: Нативный процесс

Нативный процесс (имеется в виду процесс операционной системы) - это очень тонкий слой с обязанностью найти и вызвать нативный CLR host, передав в него аргументы, переданные в сам процесс. В Windows - это dnx.exe (находится в %YOUR_PROFILE%/.dnx/runtimes/%CHOOSEN_RUNTIME%). В Mac и Linux - это запускаемый bash script (тоже с именем dnx). Запуск на IIS [(docs.asp.net: Publishing to IIS)](https://docs.asp.net/en/latest/publishing/iis.html) происходит с помощью устанавливаемого на IIS нативного HTTP-модуля: [HTTPPlatformHandler](https://azure.microsoft.com/en-us/blog/announcing-the-release-of-the-httpplatformhandler-module-for-iis-8/) (который в итоге тоже запустит dnx.exe). Использование HTTPPlatformHandler позволяет запускать веб-приложение на IIS без любых зависимостей от .NET Framework (естественно, при запуске веб-приложений нацеленных на .NET Core, а не на полный .NET Framework).

>Примечание: DNX приложения (и консольные и веб-приложения ASP.NET 5) исполняются в адресном пространстве этого нативного процесса. С этого момента и далее под "нативным процессом" я буду подразумевать dnx.exe и его аналоги в других операционных системах.

## Слой второй и третий: Нативные CLR host и CLR

Имеют три главных обязанности:

1. Запустить CLR. Способы достижения этого отличаются в зависимости от используемой версии CLR, но результат будет тот же самый.
2. Начать выполнение кода четвертого слоя (Управляемая Entry Point) в CLR.
3. Когда нативный CLR host возвращает управление, он будет "убирать за собой" и выключать CLR.

## Слой четвертый: Управляемая Entry Point

> Примечание:  В целом логика этого слоя находится в сборке Microsoft.DNX.Host [(github.com/aspnet/dnx: Microsoft.DNX.Host)](https://github.com/aspnet/dnx/tree/1.0.0-rc1-final/src/Microsoft.Dnx.Host). Entry Point этого слоя можно считать RuntimeBootstrapper [(github.com/aspnet/dnx: Microsoft.DNX.Host/RuntimeBootstrapper.cs](https://github.com/aspnet/dnx/blob/1.0.0-rc1-final/src/Microsoft.Dnx.Host/RuntimeBootstrapper.cs).

Этот первый слой, в котором работа DNX приложения переходит к выполнению управляемого кода (отсюда и его название). Он ответственен:

1. За создание LoaderContainer [(github.com/aspnet/dnx: Microsoft.Dnx.Host/Bootstrapper.cs#L29)](https://github.com/aspnet/dnx/blob/1.0.0-rc1-final/src/Microsoft.Dnx.Host/Bootstrapper.cs#L29), контейнер для ILoader'ов. ILoader'ы ответственны за загрузку сборок. Когда CLR будет просить LoaderContainer предоставить какую-либо сборку, он будет делать это используя его ILoader'ы.
2. Создание корневого ILoader'а [(github.com/aspnet/dnx: Microsoft.Dnx.Host/Bootstrapper.cs#L32)](https://github.com/aspnet/dnx/blob/1.0.0-rc1-final/src/Microsoft.Dnx.Host/Bootstrapper.cs#L32), который будет загружать требуемые сборки из папки bin выбранного dnx runtime: %YOUR_PROFILE%/.dnx/runtimes/%CHOOSEN_RUNTIME%/bin/ и предоставленных во время запуска нативного процесса дополнительных путей, с помощью параметра `--lib`: `dnx --lib <LIB_PATHS>`).
3. Настройку IApplicationEnvironment и ядра инфраструктуры системы Dependency Injection [(github.com/aspnet/dnx: Microsoft.Dnx.Host/Bootstrapper.cs#L59-L70)](https://github.com/aspnet/dnx/blob/dev/src/Microsoft.Dnx.Host/Bootstrapper.cs#L59-L70).
4. Вызов entry point [(github.com/aspnet/dnx: Microsoft.Dnx.Host/Bootstrapper.cs#L80)](https://github.com/aspnet/dnx/blob/dev/src/Microsoft.Dnx.Host/Bootstrapper.cs#L80) конкретного приложения или Microsoft.DNX.ApplicationHost [(github.com/aspnet/dnx: Microsoft.Dnx.ApplicationHost/Program.cs)](https://github.com/aspnet/dnx/blob/1.0.0-rc1-final/src/Microsoft.Dnx.ApplicationHost/Program.cs), в зависимости от параметров переданных в нативный процесс во время запуска приложения.

## Слой пятый: Application Host
 
Microsoft.DNX.ApplicationHost [(github.com/aspnet/dnx: Microsoft.Dnx.ApplicationHost)](https://github.com/aspnet/dnx/tree/1.0.0-rc1-final/src/Microsoft.Dnx.ApplicationHost) - это хост приложения поставляемый вместе с DNX. В его обязанности входит:

1. Добавление дополнительных загрузчиков сборок (ILoader'ы) в LoaderContainer [(github.com/aspnet/dnx: Microsoft.Dnx.ApplicationHost/Program.cs#L72)](https://github.com/aspnet/dnx/blob/1.0.0-rc1-final/src/Microsoft.Dnx.ApplicationHost/Program.cs#L72), которые могут загружать сборки из различных источников, таких как установленные NuGet пакеты и исходники компилируемые в runtime, используя Roslyn, и т.д.
2. Просмотреть зависимости указанные в project.json и загрузить их. Логика обхода зависимостей описана более детально в статье [Dependency-Resolution](https://github.com/aspnet/Home/wiki/Dependency-Resolution).
3. Вызов entry point [(github.com/aspnet/dnx: Microsoft.Dnx.ApplicationHost/Program.cs#L230-L240)](https://github.com/aspnet/dnx/blob/1.0.0-rc1-final/src/Microsoft.Dnx.ApplicationHost/Program.cs#L230-L240) вашей сборки, в случае консольного приложения или сборки Microsoft.AspNet.Hosting [(github.com/aspnet/Hosting: Microsoft.AspNet.Hosting/Program.cs)](https://github.com/aspnet/Hosting/blob/1.0.0-rc1/src/Microsoft.AspNet.Hosting/Program.cs) в случае веб-приложения. Сборка может быть чем угодно имеющим entry point, которую ApplicationHost знает как вызвать [(github.com/aspnet/dnx: Microsoft.Dnx.Runtime.Sources/Impl/EntryPointExecutor.cs#L20)](https://github.com/aspnet/dnx/blob/1.0.0-rc1-final/src/Microsoft.Dnx.Runtime.Sources/Impl/EntryPointExecutor.cs#L20). Приходящий вместе с DNX ApplicationHost знает как найти [(github.com/aspnet/dnx: Microsoft.Dnx.Runtime.Sources/Impl/EntryPointExecutor.cs#L70-L110)](https://github.com/aspnet/dnx/blob/1.0.0-rc1-final/src/Microsoft.Dnx.Runtime.Sources/Impl/EntryPointExecutor.cs#L70-L110) `public static void Main` метод.

## Кроссплатформенныe SDK Tools

DNX поставляется вместе c SDK, содержащим все необходимое для построения кросс-платформенных .NET приложений.

[**DNVM** - DNX Version Manager](https://github.com/aspnet/Home/wiki/Version-Manager). Позволяет просмотреть установленные на компьютере DNX, установить новые и выбрать тот, который вы будете использовать [(docs.asp.net: Install the .NET Version Manager (DNVM))](https://docs.asp.net/en/latest/getting-started/installing-on-windows.html#install-the-net-version-manager-dnvm).

После установки вы сможете использовать DNVM из командной строки, просто наберите `dnvm`

DNVM устанавливает DNX'ы из NuGet feed настроенного на использование DNX_FEED переменной среды. DNX'ы не являются NuGet пакетами в традиционном смысле - пакеты на которые вы можете ссылаться в качестве зависимостей. NuGet - это удобный способ доставки и управления версиями DNX. По умолчанию DNX устанавливается копированием и распаковкой архива с DNX в "%USERPROFILE%\.dnx\runtimes".

[**DNU** - DNX Utility](https://github.com/aspnet/Home/wiki/DNX-utility). Инструмент для установки, восстановления и создания NuGet пакетов. По умолчанию пакеты устанавливаются в "%USERPROFILE%\.dnx\packages", но вы можете изменить это, установив другой путь в в вашем global.json файле (Находится в папке Solution Items вашего проекта).

Для использования DNU наберите `dnu` в командной строке

## Кроссплатформенное консольное приложение на .NET

Сейчас я покажу вам как, используя DNX, создать простое кроссплатформенное консольное приложение на .NET.

Нам необходимо создать DLL с entry point и мы можем сделать это, используя "Console Application (Package)" шаблон в Visual Studio 2015.

Код нашего приложения:

```csharp
namespace ConsoleApp
{
    public class Program
    {
        public static void Main(string[] args)
        {
            Console.WriteLine("Hello World");
            Console.ReadLine();
        }
    }
}
```

Код выглядит абсолютно идентично обычному консольному приложению.

Вы можете запустить это приложение из Visual Studio или из командной строки, набрав в командной строке `dnx run`, находясь в папке с файлом project.json (корень проекта), запускаемого приложения.

`dnx run` - это просто сокращение, нативный процесс, фактически, развернет его в команду:

`dnx.exe --appbase . Microsoft.DNX.ApplicationHost run`

Что будет соответствовать шаблону:

`dnx.exe --appbase <путь до папки проекта> <сборка, имеющая entry point> <параметры, для этой сборки >`

Разберем развернутую команду и шаблон:

`dnx.exe` - нативный процесс. 

Путь до папки проекта задан параметром `--appbase`, его значение "`.`" - обозначает текущую директорию. 

В качестве сборки имеющей entry point, по умолчанию, выбирается `Microsoft.DNX.ApplicationHost` - именно его entry point вызовет четвертый слой (Слой четвертый: Управляемая Entry Point, четвертая ответственность). Четвертый слой не имеет специальных знаний о Microsoft.DNX.ApplicationHost. Он просто ищет entry point в предоставленной ему сборке и вызывает ее, передавая ей параметры (в нашем примере в качестве параметра будет передано "`run`"). 

Далее в работу вступает `Microsoft.DNX.ApplicationHost`. Команда `run`, переданная ему в качестве параметра, говорит найти entry point проекта по предоставленному через параметр `--appbase` пути и вызвать ее. Вместо команды `run` можно передать имя какой-либо сборки имеющей entry point и ApplicationHost вызовет ее, ниже в разделе Хостинг веб-приложений, мы рассмотрим этот пример.

Если вы не хотите использовать ApplicationHost. Вы можете вызвать нативный слой, передав ему ваше приложение напрямую.

Для того, чтобы сделать это:

1. Сгенерируйте DLL, запустив build вашего приложения (убедитесь, что вы поставили галочку около пункта "Produce outputs on build" в разделе "Build" свойств вашего проекта). Вы можете найти результаты в директории artifacts в корне папки вашего решения (вы также можете использовать не Visual Studio, а команду `dnu build`, набрав ее находясь в папке с файлом project.json вашего проекта).
2. Наберите команду: `dnx <путь_до_библиотеки_и_ее_имя_включая_расширение>`.

Вызов DLL напрямую - это довольно низкоуровневый подход написания приложений. Вы не используете `Microsoft.DNX.ApplicationHost`, поэтому вы отказываетесь и от использования файла project.json и улучшенного NuGet-based механизма управления зависимостями. Вместо этого любые библиотеки, от которых вы зависите, будут загружаться из указанных при запуске нативного процесса с помощью параметра `--lib` директорий. До окончания этой статьи я буду использовать Microsoft.DNX.ApplicationHost.

Microsoft.DNX.ApplicationHost - последний слой DNX, все что находится выше можно считать ASP.NET 5.

## Хостинг веб-приложений

В веб-приложениях ASP.NET 5 поверх `Microsoft.DNX.ApplicationHost` работает слой хостинга [(docs.asp.net: Hosting)](https://docs.asp.net/en/latest/fundamentals/hosting.html). Он представлен сборкой `Microsoft.AspNet.Hosting` [(github.com/aspnet/Hosting: Microsoft.AspNet.Hosting)](https://github.com/aspnet/Hosting/tree/1.0.0-rc1/src/Microsoft.AspNet.Hosting). Этот слой ответственен за поиск веб-сервера, запуск веб-приложения на нем и "уборку за собой" во время выключения веб-приложения. Он также предоставляет приложению некоторые дополнительные, связанные со слоем хостинга сервисы [(github.com/aspnet/Hosting: Microsoft.AspNet.Hosting/WebHostBuilder.cs#L69)](https://github.com/aspnet/Hosting/blob/1.0.0-rc1/src/Microsoft.AspNet.Hosting/WebHostBuilder.cs#L69).

Для запуска веб-приложения `Microsoft.DNX.ApplicationHost` должен вызывать entry point метод `Microsoft.AspNet.Hosting` [(github.com/aspnet/Hosting: Microsoft.AspNet.Hosting/Program.cs)](https://github.com/aspnet/Hosting/blob/1.0.0-rc1/src/Microsoft.AspNet.Hosting/Program.cs). Используемый хостингом веб-сервер можно выбрать, указав опцию `--server` или используя другие способы настройки хостинга, такие как файл hosting.json или environment variables. Слой хостинга загружает выбранный веб-сервер и стартует его. Обычно используемый веб-сервер должен быть перечислен в списке зависимостей в файле project.json, чтобы его можно было загрузить.

Обычно запуск веб-приложения из командной строки производится командой `dnx web`.

Шаблон веб-приложения ASP.NET 5 включает набор команд [(docs.asp.net: Using Commands)](https://docs.asp.net/en/latest/dnx/commands.html), определенных в project.json файле и команда `web` одна из них:

```json
"commands": {
    "web": "Microsoft.AspNet.Server.Kestrel",
    "ef": "EntityFramework.Commands"
},
```

Команды, на самом деле, только устанавливают дополнительные аргументы для `dnx.exe` и когда вы набираете `dnx web` для запуска веб-приложения, в реальности это преобразуется в:

`dnx.exe --appbase . Microsoft.DNX.ApplicationHost Microsoft.AspNet.Server.Kestrel`

В свою очередь, вызов entry point `Microsoft.AspNet.Server.Kestrel` [(github.com/aspnet/KestrelHttpServer: Microsoft.AspNet.Server.Kestrel/Program.cs)](https://github.com/aspnet/KestrelHttpServer/blob/dev/src/Microsoft.AspNet.Server.Kestrel/Program.cs) преобразуется в вызов: 

`Microsoft.AspNet.Hosting --server Microsoft.AspNet.Server.Kestrel`

Так что итоговая команда будет:

`dnx.exe --appbase . Microsoft.DNX.ApplicationHost Microsoft.AspNet.Hosting --server Microsoft.AspNet.Server.Kestrel`

В результате выполнения которой, `Microsoft.DNX.ApplicationHost` вызовет entry point метод `Microsoft.AspNet.Hosting`.

> Пока статья про хостинг в документации docs.asp.net не готова, прочитать про ключи, используемые для настройки хостинга, можно [здесь](https://github.com/aspnet/Announcements/issues/108)

## Стартовая логика веб-приложения

Слой хостинга также ответственен за запуск стартовой логики [(docs.asp.net: Application Startup)](https://docs.asp.net/en/latest/fundamentals/startup.html) веб-приложения [(github.com/aspnet/Hosting: Microsoft.AspNet.Hosting/WebApplication.cs#L56)](https://github.com/aspnet/Hosting/blob/1.0.0-rc1/src/Microsoft.AspNet.Hosting/WebApplication.cs#L56). Раньше она находилась в файле Global.asax, теперь по умолчанию находится в классе Startup и состоит из Configure метода, используемого для построения конвейера обработки запросов и ConfigureServices метода, используемого для настройки сервисов веб-приложения.

```csharp
namespace WebApplication1
{
    public class Startup
    {
        public void ConfigureService(IServiceCollection services)
        {
          // Добавьте сервисы для вашего приложения здесь
        }
        public void Configure(IApplicationBuilder app)
        {
          // Настройте конвейер обработки запросов здесь
        }
    }
}
```

Для построения конвейера обработки запросов в Configure методе используется интерфейс IApplicationBuilder. IApplicationBuilder позволяет зарегистрировать request delegate ("Use" метод) и зарегистрировать middleware ("UseMiddleware" метод) в конвейере обработки запросов.

Request delegate - это ключевая концепция ASP.NET 5. Request delegate - это обработчик входящего запроса, он принимает HttpContext и асинхронно делает нечто полезное с ним:

`public delegate Task RequestDelegate(HttpContext context);`

Обработка запроса в ASP.NET 5 - это вызов по цепочке зарегистрированных request delegate. Но принятие решения о вызове следующего request delegate в цепочке, остается за их автором.

В качестве упрощения регистрации в конвейере обработки запросов request delegate не вызывающего следующий request delegate, вы можете использовать Run extension метод IApplicationBuilder.

```csharp
public void Configure(IApplicationBuilder app)
{
    app.Run(async context => await context.Response.WriteAsync("Hello, world!"));
}
```

Того же самого можно достичь используя Use extension метод и не вызывая следующий request delegate:

```csharp
public void Configure(IApplicationBuilder app)
{
    app.Use(next => async context => await context.Response.WriteAsync("Hello, world!"));
}
```

И пример с вызовом следующего в цепочке request delegate:

```csharp
public void Configure(IApplicationBuilder app)
{
    app.Use(next => async context =>
        {
            await context.Response.WriteAsync("Hello, world!");
            await next.Invoke(context);
        });
}
```

Для того, чтобы request delegate было удобно переиспользовать, можно оформить его в виде ASP.NET 5 middleware [(docs.asp.net: Middleware)](https://docs.asp.net/en/latest/fundamentals/middleware.html).

Middleware ASP.NET 5 - это обычный класс, следующий определенному соглашению:

1. Следующий в цепочке request delegate (а также необходимые сервисы и дополнительные параметры) передается в конструктор middleware. 
2. Логика обработки HttpContext должна быть реализована в асинхронном Invoke методе.

Пример middleware:

```csharp
using Microsoft.AspNet.Builder;
using Microsoft.AspNet.Http;
using System.Threading.Tasks;
public class XHttpHeaderOverrideMiddleware
{
    private readonly RequestDelegate _next;
    
    public XHttpHeaderOverrideMiddleware(RequestDelegate next)
    {
        _next = next;
    }
    
    public Task Invoke(HttpContext httpContext)
    {
        var headerValue =
          httpContext.Request.Headers["X-HTTP-Method-Override"];
        var queryValue =
          httpContext.Request.Query["X-HTTP-Method-Override"];
        
        if (!string.IsNullOrEmpty(headerValue))
        {
            httpContext.Request.Method = headerValue;
        }
        else if (!string.IsNullOrEmpty(queryValue))
        {
            httpContext.Request.Method = queryValue;
        }

        return _next.Invoke(httpContext);
    }
}
```

Вызов следующего (если вы хотите вызвать следующий) в цепочке request delegate должен осуществляться внутри Invoke метода. Если вы разместите какую-нибудь логику ниже вызова следующего request delegate, то она будет выполнена после того, как все следующие за вашим обработчики входящего запроса отработают.

В конвейер обработки запросов вы можете включить middleware следующее этому соглашению с помощью `UseMiddleware<T>` extension метода IApplicationBuilder:

```csharp
public void Configure(IApplicationBuilder app)
{
    app.UseMiddleware<XHttpHeaderOverrideMiddleware>();
}
```

Любые параметры, переданные в этот метод, будут внедрены в конструктор middleware после `RequestDelegate next` и запрошенных сервисов:

Конструктор middleware, принимающий дополнительно сервисы и параметры:

```csharp
public XHttpHeaderOverrideMiddleware(RequestDelegate next, SomeServise1 service1, 
    SomeServise2 service2, string param1, bool param2)
{
    _next = next;
}
```
	
Включение middleware в конвейер обработки запросов и передача ему параметров:

```csharp
public void Configure(IApplicationBuilder app)
{
    app.UseMiddleware<XHttpHeaderOverrideMiddleware>(param1, param2);
}
```

По-соглашению, включение middleware в цепочку вызовов следует оформлять в виде "Use..." extension метода у IApplicationBuilder:

```csharp
public static class BuilderExtensions
{
    public static IApplicationBuilder UseXHttpHeaderOverride(
        this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<XHttpHeaderOverrideMiddleware>();
    }
}
```

Чтобы включить этот middleware в конвейер обработки запросов, вам необходимо вызвать этот extension метод в Configure методе:

```csharp
public void Configure(IApplicationBuilder app)
{
    app.UseXHttpHeaderOverride();
}
```

ASP.NET 5 поставляется с большим набором встроенных middleware. Есть middleware для работы с файлами [(docs.asp.net: Working with Static Files)](https://docs.asp.net/en/latest/fundamentals/static-files.html), маршрутизации [(docs.asp.net: Routing)](https://docs.asp.net/en/latest/fundamentals/routing.html), обработки ошибок, диагностики [(docs.asp.net: Diagnostics)](https://docs.asp.net/en/latest/fundamentals/diagnostics.html) и безопасности. Middleware поставляются как NuGet пакеты через nuget.org.

## Сервисы

В ASP.NET 5 вводится понятие Сервисов - "общих" компонентов, доступ к которым может требоваться в нескольких местах приложения. Сервисы доступны приложению через систему внедрения зависимостей. ASP.NET 5 поставляется с простым IoC-контейнером, поддерживающим внедрение зависимостей в конструктор [(docs.asp.net: Dependency Injection)](https://docs.asp.net/en/latest/fundamentals/dependency-injection.html), но вы легко можете заменить его на любой другой контейнер [(docs.asp.net: Replacing the default services container)](https://docs.asp.net/en/latest/fundamentals/dependency-injection.html#replacing-the-default-services-container).

Startup класс также поддерживает внедрение зависимостей, для этого достаточно запросить их в качестве параметров конструктора.

По умолчанию вам доступны следующие сервисы:

`Microsoft.Extensions.PlatformAbstractions.IApplicationEnvironment` - информация о приложении (физический путь до папки приложения, его имя, версия, конфигурация (Release, Debug), используемый Runtime фреймворк).

`Microsoft.Extensions.PlatformAbstractions.IRuntimeEnvironment` - информация о DNX runtime и ОС.

`Microsoft.AspNet.Hosting.IHostingEnvironment` - доступ к Web-root вашего приложения (обычно папка "wwwroot"), а также информация о текущей среде (dev, stage, prod).

`Microsoft.Extensions.Logging.ILoggerFactory` - фабрика для создания логгеров [(docs.asp.net: Logging)](https://docs.asp.net/en/latest/fundamentals/logging.html).

`Microsoft.AspNet.Hosting.Builder.IApplicationBuilderFactory` - фабрика для создания IApplicationBuilder (используется для построения request pipeline).

`Microsoft.AspNet.Http.IHttpContextFactory` - фабрика для создания Http-контекста.

`Microsoft.AspNet.Http.IHttpContextAccessor` - предоставляет доступ к текущему Http-контексту.

Вы можете добавить сервисы в приложение в ConfigureServices методе класса Startup, используя интерфейс IServiceCollection.

Обычно фреймворки и библиотеки предоставляют "Add..." extension метод у IServiceCollection для добавления их сервисов в IoC-контейнер. Например, добавление сервисов используемых ASP.NET MVC 6 производится так:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // Добавление сервисов MVC
    services.AddMvc();
}
```

Вы можете добавлять собственные сервисы в IoC-контейнер. Добавляемые сервисы могут быть одними из трех типов: transient (AddTransient метод IServiceCollection ), scoped (AddScoped метод IServiceCollection ) или singleton (AddSingleton метод IServiceCollection ). Transient сервисы создаются при каждом их запросе из контейнера. Scoped сервисы создаются, только если они еще не создавались в текущем scope. В веб-приложениях scope-контейнер создается для каждого запроса, поэтому можно думать о них как о сервисах, создаваемых для каждого http-запроса. Singleton сервисы создаются только один раз за цикл жизни приложения.

>В консольном приложении, где доступ к dependency injection отсутствует, для доступа к сервисам: IApplicationEnvironment и IRuntimeEnvironment необходимо использовать статический объект `Microsoft.Extensions.PlatformAbstractions.PlatformServices.Default`.

## Конфигурация приложения

Web.config и app.config файлы больше не поддерживаются. Вместо них ASP.NET 5 использует новое, упрощенное Configuration API [(docs.asp.net: Configuration)](https://docs.asp.net/en/latest/fundamentals/configuration.html). Оно позволяет получать данные из разных источников. Используемые по умолчанию configuration-провайдеры поддерживают JSON, XML, INI, аргументы командной строки, environment variables, а также установку параметров прямо из кода (in-memory collection). Вы можете указать несколько источников, и они будут использоваться в порядке их добавления (добавленные последними будут переопределять настройки добавленных ранее). Также вы можете иметь разные настройки для каждой среды [(docs.asp.net: Working with Multiple Environments)](https://docs.asp.net/en/latest/fundamentals/environments.html): test, stage, prod. Что облегчает публикацию приложения в разные среды.

Пример файла appsettings.json:

```json
{
    "Name": "Stas",
    "Surname": "Boyarincev"
}
```

Пример получения конфигурации приложения, используя Configuration API:

```csharp
var builder = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json")
    .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true)
    .AddEnvironmentVariables();

var Configuration = builder.Build();
```

Запросить данные, вы можете используя метод GetSection и имя настройки:

```csharp
var name = Configuration.GetSection("name");
var surname = Configuration.GetSection("surname");
```

Работать с Configuration API рекомендуется в Startup классе, а в дальнейшем разделять настройки на небольшие наборы данных, соответствующие какой-либо функциональности и передавать другим частям приложения с помощью механизма Options [(docs.asp.net: Using Options and configuration objects)](https://docs.asp.net/en/latest/fundamentals/configuration.html#using-options-and-configuration-objects).

Механизм Options позволяют использовать Plain Old CLR Object (POCO) классы в качестве объектов с настройками. Вы можете добавить его в ваше приложение, вызвав AddOptions extension-метод у IServiceCollection в ConfigureServices методе:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    //добавляем механизм Options в приложение
    services.AddOptions();
}
```

Фактически, вызов AddOptions добавляет `IOptions<TOption>` в систему внедрения зависимостей. Этот сервис может быть использован для получения Options разных типов везде, где внедрение зависимостей доступно (достаточно лишь запросить из нее `IOption<TOption>`, где TOption POCO класс с нужными вам настройками).

Для регистрации вашей options вы можете использовать `Configure<TOption>` extension-метод IServiceCollection:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.Configure<MvcOptions>(options => options.Filters.Add(
      new MyGlobalFilter()));
}
```

В пример выше MvcOptions - это класс [(github.com/aspnet/Mvc: Microsoft.AspNet.Mvc.Core/MvcOptions.cs)](https://github.com/aspnet/Mvc/blob/6.0.0-rc1/src/Microsoft.AspNet.Mvc.Core/MvcOptions.cs), который MVC-фреймворк использует для получения от пользователя своих настроек.

Вы также можете легко передать часть конфигурационных настроек в options:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    //Configuration - конфигурация приложения полученная в примерах выше
    services.Configure<MyOptions>(Configuration);
}
```

В этом случае ключи настроек из конфигурации будут мапиться на имена свойств POCO класса настроек.

А для получения MyOptions с установленными настройками внутри приложения, достаточно запросить из системы внедрения зависимостей `IOption<MyOptions>`, например, через конструктор контроллера.

Внутренне механизм Options работает через добавление `IConfigureOptions<TOptions>` в сервис-контейнер, где TOptions - класс с настройками. Стандартная реализация `IOptions<TOption>` будет собирать все `IConfigureOptions<TOptions>` одного типа и "суммировать их свойства", а затем предоставлять конечный экземпляр - это происходит потому что вы можете множество раз добавлять объект с настройками одного и того же типа в сервис-контейнер, переопределяя настройки.

## Веб-сервер

Как только веб-сервер стартует, он начинает ожидать входящие запросы и запускать процесс обработки	 для каждого из них. Уровень веб-сервера, поднимает запрос на уровень хостинга, отправляя ему набор feature интерфейсов. Есть feature интерфейсы для отправки файлов, веб-сокетов, поддержки сессий, клиентских сертификатов и многих других [(docs.asp.net: Request Features)](https://docs.asp.net/en/latest/fundamentals/request-features.html).

Например feature интерфейс для Http request:

```csharp
namespace Microsoft.AspNet.Http.Features
{
    public interface IHttpRequestFeature
    {
        string Protocol { get; set; }
        string Scheme { get; set; }
        string Method { get; set; }
        string PathBase { get; set; }
        string Path { get; set; }
        string QueryString { get; set; }
        IHeaderDictionary Headers { get; set; }
        Stream Body { get; set; }
    }
}
```

Веб-сервер использует feature интерфейсы для раскрытия низкоуровневой функциональности уровню хостинга. А он в свою очередь делает доступными их всему приложению через `HttpContext`. Это позволяет разорвать тесные связи между уровнем веб-сервера и хостинга и разместить приложение на различных веб-серверах. ASP.NET 5 поставляется с встроенной поддержкой IIS, оберткой над HTTP.SYS ([Microsoft.AspNet.Server.Web­Listener](https://www.nuget.org/packages/Microsoft.AspNet.Server.WebListener)) и новым кроссплатформенным веб-сервером под названием Kestrel [(github.com/aspnet/KestrelHttpServer)](https://github.com/aspnet/KestrelHttpServer).

Open Web Interface for .NET (OWIN) стандарт, разделяющий эти же цели. OWIN стандартизирует как .NET сервера и приложения должны общаться друг с другом. ASP.NET 5 поддерживает OWIN [(docs.asp.net: OWIN)](https://docs.asp.net/en/latest/fundamentals/owin.html) с помощью [Microsoft.AspNet.Owin](https://github.com/aspnet/HttpAbstractions/tree/1.0.0-rc1/src/Microsoft.AspNet.Owin) пакета. Вы [можете хостить](https://github.com/aspnet/Entropy/tree/dev/samples/Owin.Nowin.HelloWorld) ASP.NET 5 приложения на OWIN-based веб-серверах и вы можете [использовать OWIN middleware](https://github.com/aspnet/Entropy/tree/dev/samples/Owin.HelloWorld) в ASP.NET 5 pipeline.

[Katana Project](http://katanaproject.codeplex.com) была первой попыткой Microsoft реализовать поддержку OWIN на стеке ASP.NET и многие идеи и концепты были перенесены из нее в ASP.NET 5. Katana имеет похожую модель построения pipeline из middleware и хостинга на различных веб-серверах. Однако, в отличие от Katana, открывающей ниже лежащий уровень OWIN приложению, ASP.NET 5 переходит к более удобным абстракциям. Но вы до сих пор можете использовать Katana middleware в ASP.NET 5 с помощью [OWIN моста](https://github.com/aspnet/Entropy/tree/dev/samples/Owin.IAppBuilderBridge)

## Итоги

ASP.NET 5 runtime построен с нуля для поддержки кроссплатформенных веб-приложений. ASP.NET 5 имеет гибкую, многослойную архитектуру, которая может запускаться на полном .NET Framework, .NET Core и даже на Mono. Новая хостинг модель позволяет легко компоновать приложения, используя middleware, хостить их на различных веб-серверах, а также поддерживает внедрение зависимостей и новые улучшенные возможности по конфигурированию приложений.

## Cсылки

[docs.asp.net](https://docs.asp.net/en/latest/index.html) - документация ASP.NET 5.

[github.com/aspnet/Home/wiki](https://github.com/aspnet/Home/wiki) - статьи про DNX Runtime.
