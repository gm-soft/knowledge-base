---
category: .net
tags: [.net, angular, spa, identity server, tutorial]
---

# Использование Identity Server 4 в Net Core 3.0

Источник: [Habr.com](https://habr.com/ru/post/461433/)

---

## Введение

На одном из моих поддерживаемых проектов недавно встала задача проанализировать возможность миграции с .NET фреймворка 4.5 на .Net Core по случаю необходимости рефакторинга и разгребания большого количества накопившегося технического долга. Выбор пал на целевую платформу .NET Core 3.0, так как, судя по утверждению разработчиков от Microsoft, с появлением релиза версии 3.0, необходимые шаги при миграции legacy кода уменьшатся в несколько раз. Особенно нас в нем привлекли планы выхода EntityFramework 6.3 для .Net Core т.е. большую часть кода, основанную на EF 6.2, можно будет оставить «как есть» в мигрированном проекте на net core.

С уровнем данных, вроде, стало понятно, однако, еще одной большой частью по переносу кода остался уровень безопасности, который, к сожалению, после беглых выводов аудита придется почти полностью выкинуть и переписать с нуля. Благо, на проекте уже использовалась часть ASP NET Identity, в виде хранения пользователей и других приделанных сбоку «велосипедов».

Тут возникает логичный вопрос: если в security часть придется вносить много изменений, почему бы сразу же не внедрить подходы, рекомендуемые в виде промышленных стандартов, а именно: подвести приложение под использование Open Id connect и OAuth посредством фреймворка [IdentityServer4](http://docs.identityserver.io/en/latest/).

## Проблемы и пути решения

Итак, нам дано: имеется JavaScript приложение на Angular (Client в терминах IS4), оно использует некоторое подмножество WebAPI (Resources), также есть база данных устаревшего ASP NET Identity с логинами пользователей, которые необходимо после обновления использовать заново (чтобы не заводить всех еще раз), плюс в некоторых случаях необходимо давать возможность входить в систему через Windows аутентификацию на стороне IdentityServer4. Т.е. бывают случаи, когда пользователи работают через локальную сеть в домене ActiveDirectory.

Основное решение миграции данных о пользователях состоит в том, чтобы вручную (или с помощью автоматизированных средств) написать скрипт миграции между старой и новой схемой данных Identity. Мы, в свою очередь, воспользовались автоматическим приложением сравнения схем данных и сгенерировали SQL скрипт, в зависимости от версии Identity целевой миграционный скрипт будет содержать разные инструкции по обновлению. Тут главное- не забыть согласовать таблицу EFMigrationsHistory, если до этого использовался EF и в дальнейшем планируется, например, расширять сущность IdentityUser на дополнительные поля.

А вот как правильно теперь сконфигурировать IdentityServer4 и настроить его совместно с Windows учетными записями будет описано ниже.

## План реализации

По причинам NDA я не стану описывать, как мы добились внедрения IS4 у себя на проекте, однако, в данной статье я на простом сайте ASP.NET Core, созданном с нуля, покажу, какие шаги нужно предпринять, чтобы получить полностью сконфигурированное и работоспособное приложение, использующее для целей авторизации и аутентификации IdentityServer4.
Чтобы реализовать желаемое поведение нам предстоит совершить следующие шаги:

- Создать пустой проект ASP.Net Core и сконфигурировать на использование IdentityServer4.
- Добавить клиента в виде Angular приложения.
- Реализовать вход через open-id-connect google
- Добавить возможность выбора Windows аутентификации

По соображениям краткости все три компонента (IdentityServer, WebAPI, Angular клиент) будут находиться в одном проекте. Выбранный тип взаимодействия клиента и IdentityServer (GrantType) – Implicit flow, когда access_token передается на сторону приложения в браузере, а затем используется при взаимодействии с WebAPI. Ближе к релизу, судя по изменениям в репозитории ASP.NET Core, Implicit flow будет заменена на Authorization Code + PKCE.)

В процессе создания и изменения приложения будет широко применяться интерфейс командной строки .NET Core, он должен быть установлен в системе в месте с последней версией preview Core 3.0 (на момент написание статьи 3.0.100-preview7-012821).

## Создание и конфигурирование web проекта

Выход IdentityServer версии 4, ознаменовался полным выпиливанием UI с этого фреймворка. Теперь у разработчиков появилось полное право самим определять главный интерфейс сервера авторизации. Тут есть несколько способов. Одним из популярных является использование UI из пакета QuickStart UI, его можно найти в официальном репозитории на github.

Другим, не менее удобным, способом является интеграция с ASP NET Core Identity UI, в данном случае разработчику необходимо правильно сконфигурировать соответствующие промежуточные ПО в проекте. Именно данный способ и будет описываться далее.

Начнем с создания простого web проекта, для этого выполним в командной строке следующую инструкцию:

```

dotnet new webapp -n IdentityServer4WebApp

```
После исполнения на выходе у нас будет каркас вэб приложения, который постепенно будет доводиться до нужного нам состояния. Тут нужно сделать оговорку, что в .Net Core 3.0 для Identity используются более легковесные RazorPages, в отличии от тяжеловесного MVC. Теперь необходимо добавить поддержку IdentityServer в наш проект. Для этого устанавливаем необходимые пакеты:

```

dotnet add package Microsoft.AspNetCore.ApiAuthorization.IdentityServer -v 3.0.0-preview7.19365.7
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore -v 3.0.0-preview7.19365.7
dotnet add package Microsoft.EntityFrameworkCore.Tools -v 3.0.0-preview7.19362.6
dotnet add package Microsoft.EntityFrameworkCore.Sqlite -v 3.0.0-preview7.19362.6

```

Помимо ссылок на пакеты сервера авторизации, здесь мы добавили поддержку Entity Framework для хранения информации о пользователях в экосистеме Identity. Для простоты будем использовать базу SQLite.

Для инициализации базы создадим модель нашего пользователя и контекст базы данных, для этого объявим два класса ApplicationUser, наследуемый от IdentityUser в папке Models и ApplicationDbContext, наследуемый от: ApiAuthorizationDbContext в папке Data. Далее необходимо сконфигурировать использование контекста EntityFramework и создать базу данных. Для этого прописываем контекст в метод ConfigureServices класса Startup:

```csharp

public void ConfigureServices(IServiceCollection services)
{
  services.AddDbContext<ApplicationDbContext>(options =>options.UseSqlite(Configuration.GetConnectionString("DefaultConnection")));
    services.AddRazorPages();
}

```

И добавляем строку подключения в appsettings.json:

```json

"ConnectionStrings": {
    "DefaultConnection": "Data Source=data.db"
},

```

Теперь можно создать первоначальную миграцию и проинициализировать схему базы данных. Тут стоит заметить, что необходим установленный tool для ef core (для рассматриваемого preview нужна версия 3.0.0-preview7.19362.6).

```

dotnet ef migrations add Init
dotnet ef database update

```

Если все предыдущие шаги были выполнены без ошибок, в вашем проекте должен появиться SQLite файл данных data.db.

На данном этапе мы уже можем полностью сконфигурировать и опробовать полноценную возможность использования Asp.Net Core Identity. Для этого внесем изменения в методы Startup. Configure и Startup.ConfigureServices.

```csharp

//Startup.ConfigureServices:
services.AddDefaultIdentity<ApplicationUser>()
                .AddEntityFrameworkStores<ApplicationDbContext>();

//Startup. Configure:
app.UseAuthentication();
app.UseAuthorization();

app.UseEndpoints(endpoints =>
{
    endpoints.MapRazorPages();
});

```

Этими строчками мы встраиваем возможность аутентификации и авторизации в конвейер обработки запросов. А также добавляем дефолтный пользовательский интерфейс для Identity.
Осталось только подправить UI, добавим в Pages\Shared новое представление Razor view с именем _LoginPartial.cshtml и следующим содержимым:

```html

@using IdentityServer4WebApp.Models
@using Microsoft.AspNetCore.Identity
@inject SignInManager<ApplicationUser> SignInManager
@inject UserManager<ApplicationUser> UserManager

<ul class="navbar-nav">
    @if (SignInManager.IsSignedIn(User))
    {
        <li class="nav-item">
            <a class="nav-link text-dark" asp-area="Identity" asp-page="/Account/Manage/Index" title="Manage">Hello @User.Identity.Name!</a>
        </li>
        <li class="nav-item">
            <form class="form-inline" asp-area="Identity" asp-page="/Account/Logout" asp-route-returnUrl="/">
                <button type="submit" class="nav-link btn btn-link text-dark">Logout</button>
            </form>
        </li>
    }
    else
    {
        <li class="nav-item">
            <a class="nav-link text-dark" asp-area="Identity" asp-page="/Account/Register">Register</a>
        </li>
        <li class="nav-item">
            <a class="nav-link text-dark" asp-area="Identity" asp-page="/Account/Login">Login</a>
        </li>
    }
</ul>

```

Вышеперечисленный код представления должен добавить в навигационную панель ссылки на область интерфейса Identity со встроенными элементами управления пользователями (ввод логина и пароля, регистрации и т.д.)

Чтобы добиться рендеринга новых пунктов меню, просто изменим файл _Layout.cshtml, добавив туда рендеринг этого частичного представления.

```html

        <ul class="navbar-nav flex-grow-1">
            <li class="nav-item">
                <a class="nav-link text-dark" asp-area="" asp-page="/Index">Home</a>
            </li>
            <li class="nav-item">
                <a class="nav-link text-dark" asp-area="" asp-page="/Privacy">Privacy</a>
            </li>
        </ul>
    </div>
    <partial name="_LoginPartial" /> <!––Здесь––> 
</div>

```

И теперь попробуем запустить наше приложение и перейти по появившимся ссылкам в головном меню, пользователю должна будет отобразиться страница с приветствием и просьбой ввести логин и пароль. При этом можно зарегистрироваться и залогиниться – все
должно работать.

![](/dot-net/2020-01-21-using-identityserver-and-net-core/yla2yln6zflkftldetnacko8trm.png)

Разработчики IdentityServer4 проделали превосходную работу по усовершенствованию процедуры интеграции ASP.NET Identity и самого фреймворка сервера. Чтобы добавить возможность использования токенов OAuth2, требуется дополнить наш проект некоторыми новыми инструкциями в коде.

В предпоследней строчке метода Startup.ConfigureServices добавить конфигурацию соглашений IS4 поверх ASP.NET Core Identity:

```csharp

services.AddIdentityServer()
    .AddApiAuthorization<ApplicationUser, ApplicationDbContext>();

```

Метод AddApiAuthorization указывает фреймворку использовать определенную поддерживаемую конфигурацию, главным образом через файл appsettings.json. На данный момент встроенные возможности по управлению IS4 не такие гибкие и следует относится к этому, как к некой отправной точке при построении своих приложений. В любом случае, можно воспользоваться перегруженной версией этого метода и более детально настроить параметры через callback.

​ Далее вызываем вспомогательный метод, который настраивает приложение для проверки токенов JWT, выданных фреймворком.

```csharp

services.AddAuthentication()
    .AddIdentityServerJwt();

```

И наконец, в методе Startup.Configure добавить промежуточное ПО для предоставления конечных точек Open ID Connect.

```csharp

app.UseAuthentication();
app.UseAuthorization();
app.UseIdentityServer(); // <-Сюда

```

Как говорилось выше по тексту, использованные вспомогательные методы читают конфигурацию в файле настроек приложения appsettings.json, в которых мы должны добавить новую секцию IdentityServer.

```json

"IdentityServer": {
    "Clients": {
        "TestIdentityAngular": {
        "Profile": "IdentityServerSPA"
        }
    }
}

```

В данной секции определяется клиента с именем TestIdentityAngular, которое мы присвоим будущему браузерному клиенту и определенным профилем конфигурации.

Профили приложений– это новое средство конфигурирования IdentityServer, предоставляющее несколько предопределенных конфигураций с возможностью уточнения некоторых параметров. Мы будем использовать профиль IdentityServerSPA, предназначенный для случаев, когда браузерный клиент и фреймворк расположены в одном проекте и имеют такие параметры:

- Ресурс redirect_uri, установленный на /authentication/login-callback.
- Ресурс post_logout_redirect_uri, установлен на /authentication/logout-callback.
- Набор областей включает openid, profile, для каждого ресурса API приложения.
- Набор разрешенных типов ответа OIDC — id_token token
- GrantType для клиента – Implicit

Другие возможные профили – SPA (приложение без IS4), IdentityServerJwt (API размещенное совместно с IS4), API (отдельное API). Помимо этого конфигурация регистрирует ресурсы:

- ApiResources: один API ресурс с именем << appname>>API с указанием properties для всех клиентов (*).
- IdentityServerResources: IdentityResources.OpenId() и IdentityResources.Profile()

Как известно, IdentityServer для подписи токенов использует сертификаты, их параметры также можно задать в конфигурационном файле, так на момент тестирования мы можем использовать тестовый x509-сертификат, для этого нужно указать его в секции «Key» файла appsettings.Development.json.

```json

"IdentityServer": {
    "Key": {
      "Type": "Development"
    }
  }

```

Теперь можно сказать, что бэкэнд, позволяющий использовать [IdentityServer готов](https://github.com/alexs0ff/IdentityServer4Core3Example/commit/51b62e4beb5347c9ced14e0e464850bb8052ec91) и можно приступать к реализации браузерного приложения.

## Реализация клиента на Angular

Наше браузерное SPA будет написано на платформе Angular. Приложение будет содержать две страницы, одна – для неавторизированных пользователей, а другая для пользователей, прошедших проверку. В примерах используется версия 8.1.2. Для начала создадим будущий каркас:

```
ng new ClientApp
```

В процессе создания нужно ответить “да” на предложение использовать роутинг. И немного стилизируем страницу через библиотеку bootstrap:

```
cd ClientApp
ng add bootstrap
```

Дальше необходимо добавить поддержку хостинга SPA в наше основное приложение. Сперва нужно подправить проект csproj – добавить информацию о нашем браузерном приложении.

```xml

<PropertyGroup>
    <TargetFramework>netcoreapp3.0</TargetFramework>

    <TypeScriptCompileBlocked>true</TypeScriptCompileBlocked>
    <TypeScriptToolsVersion>Latest</TypeScriptToolsVersion>
    <IsPackable>false</IsPackable>
    <SpaRoot>ClientApp\</SpaRoot>
    <DefaultItemExcludes>$(DefaultItemExcludes);$(SpaRoot)node_modules\**</DefaultItemExcludes>
    <BuildServerSideRenderer>false</BuildServerSideRenderer>
  </PropertyGroup>
…
<ItemGroup>
    <Content Remove="$(SpaRoot)**" />
    <None Remove="$(SpaRoot)**" />
    <None Include="$(SpaRoot)**" Exclude="$(SpaRoot)node_modules\**" />
  </ItemGroup>

<Target Name="DebugEnsureNodeEnv" BeforeTargets="Build" Condition=" '$(Configuration)' == 'Debug' And !Exists('$(SpaRoot)node_modules') ">
    <Exec Command="node --version" ContinueOnError="true">
      <Output TaskParameter="ExitCode" PropertyName="ErrorCode" />
    </Exec>
    <Error Condition="'$(ErrorCode)' != '0'" Text="Node.js is required to build and run this project. To continue, please install Node.js from https://nodejs.org/, and then restart your command prompt or IDE." />
    <Message Importance="high" Text="Restoring dependencies using 'npm'. This may take several minutes..." />
    <Exec WorkingDirectory="$(SpaRoot)" Command="npm install" />
  </Target>

```

После этого устанавливаем специальный пакет nuget для поддержки браузерных приложений.

```
dotnet add package Microsoft.AspNetCore.SpaServices.Extensions -v 3.0.0-preview7.19365.7
```

И применяем его вспомогательные методы:

```csharp

//Startup. ConfigureServices:
services.AddSpaStaticFiles(configuration =>
            {
                configuration.RootPath = "ClientApp/dist";
            });

//Startup. Configure:
app.UseSpa(spa =>
            {
                spa.Options.SourcePath = "ClientApp";

                if (env.IsDevelopment())
                {
                    spa.UseAngularCliServer(npmScript: "start");
                }
            });

```

Помимо вызова новых методов необходимо удалить страницы Razor Index.chtml и _ViewStart.chtml, чтобы контент теперь предоставляли сервисы SPA.

Если все было сделано в соответствии с инструкциями, при запуске приложения на экране появится дефолтная страница.

![](/dot-net/2020-01-21-using-identityserver-and-net-core/mvlvj6_2aqcutz9t0_9eozhoxk8.png)

Теперь необходимо настроить роутинг, для этого добавляем в проект 2 компонента:

```
ng generate component Home -t=true -s=true --skipTests=true
ng generate component Data -t=true -s=true --skipTests=true
```

И изменяем файл app.component.html, чтобы правильно отобразить пункты меню.

```html

<header>
    <nav class='navbar navbar-expand-sm navbar-toggleable-sm navbar-light bg-white border-bottom box-shadow mb-3'>
      <div class="container">
        <a class="navbar-brand" [routerLink]='["/"]'>Client App</a>
        <div class="navbar-collapse collapse d-sm-inline-flex flex-sm-row-reverse" [ngClass]='{"show": isExpanded}'>
          <ul class="navbar-nav flex-grow">
            <li class="nav-item" [routerLinkActive]='["link-active"]' [routerLinkActiveOptions]='{ exact: true }'>
              <a class="nav-link text-dark" [routerLink]='["/"]'>Home</a>
            </li>
            <li class="nav-item" [routerLinkActive]='["link-active"]'>
              <a class="nav-link text-dark" [routerLink]='["/data"]'>Web api data</a>
            </li>
          </ul>
        </div>
      </div>
    </nav>
  </header>
<div style="text-align:center">
  <h1>
    Welcome to {{ title }}!
  </h1>
</div>
<div class="router-outlet">
  <router-outlet></router-outlet>
</div>

```

На этом шаге можно завершить основную подготовку каркаса приложения для внедрения взаимодействия через токены, выданные IdentityServer.

Текущий этап подготовки каркаса нашего SPA можно назвать завершенным и теперь следует приступить к реализации модуля, отвечающего за взаимодействие с серверной частью по протоколам OpenID Connect и OAuth. К счастью, разработчики от Microsoft уже реализовали такой код и теперь можно просто позаимствовать этот модуль у них. Так как моя статья пишется, основываясь на предрелизе 7 ASP.NET Core 3.0, весь код мы будем брать по релизной метке «v3.0.0-preview7.19365.7» на [github](https://github.com/aspnet/AspNetCore/tree/v3.0.0-preview7.19365.7).

Перед импортом кода необходимо установить библиотеку oidc-client, которая предоставляет множество интерфейсов для браузерных приложений, а также поддерживает управление пользовательскими сессиями и токенами доступа. Для начала работы с ней нужно установить соответствующий пакет.

```
npm install oidc-client@1.8.0
```

Теперь в наше SPA необходимо внедрить модуль, инкапсулирующий полное взаимодействие по требуемым протоколам. Для этого нужно взять полностью модуль ApiAuthorizationModule из вышеуказанной метки репозитория ASP.NET Core и добавить в приложение все его файлы.

Помимо этого, необходимо импортировать его в главный модуль приложения AppModule:

```typescript

@NgModule({
  declarations: [
    AppComponent,
    HomeComponent,
    DataComponent
  ],
  imports: [
    BrowserModule,
    HttpClientModule,
    ApiAuthorizationModule,//<-Сюда
    AppRoutingModule
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }

```

Для отображения новых пунктов меню в импортированном модуле есть компонент app-login-menu, его можно полностью изменить в соответствии с вашими потребностями и добавить ссылку на него в секцию навигации представления app.component.html.

Модуль авторизации API для конфигурирования OpenID connect клиента SPA должен использовать специальный endpoint в бэке приложения, для его реализации мы должны выполнить такие шаги:

- Исправить ID клиента в соответствии с тем, что мы задали в конфигурационном файле appsettings.json в секции IdentityServer:Clients, в нашем случае это TestIdentityAngular, прописывается оно в первой строчке набора констант api-authorization.constants.ts.
- Добавить контроллер OidcConfigurationController, который будет непосредственно возвращать конфигурацию в браузерное приложение

Код создаваемого контроллера представлен ниже:

```csharp

[ApiController]
    public class OidcConfigurationController: ControllerBase
    {
        private readonly IClientRequestParametersProvider _clientRequestParametersProvider;

        public OidcConfigurationController(IClientRequestParametersProvider clientRequestParametersProvider)
        {
            _clientRequestParametersProvider = clientRequestParametersProvider;
        }

        [HttpGet("_configuration/{clientId}")]
        public IActionResult GetClientRequestParameters([FromRoute]string clientId)
        {
            var parameters = _clientRequestParametersProvider.GetClientParameters(HttpContext, clientId);
            return Ok(parameters);
        }
    }

```

Также нужно сконфигурировать поддержку API точек для приложения бэка.

```csharp

//Startup.ConfigureServices:
services.AddControllers();//<- Сюда
services.AddRazorPages();

//Startup. Configure:
app.UseEndpoints(endpoints =>
            {
                endpoints.MapRazorPages();
                endpoints.MapControllers();
            });

```

Теперь настало время запустить приложение. На главной странице в верхнем меню должны появиться два пункта – Login и Register. Также при старте импортированный модуль авторизации запросит у серверной стороны конфигурацию клиента, которую впоследствии и будет учитывать в протоколе. Примерный вывод конфигурации показан ниже:

```json

{
    "authority": "https://localhost:44367",
    "client_id": "TestIdentityAngular",
    "redirect_uri": "https://localhost:44367/authentication/login-callback",
    "post_logout_redirect_uri": "https://localhost:44367/authentication/logout-callback",
    "response_type": "id_token token",
    "scope": "IdentityServer4WebAppAPI openid profile"
}

```

Как видно, клиент при взаимодействии ожидает получить id токен и токен доступа, а также он сконфигурирован на область доступа к нашему API.

Теперь если же мы выберем пункт меню Login, нас должно редиректнуть на страницу нашего IdentityServer4 и тут мы можем ввести логин и пароль, и если они корректны, мы сразу будем перекинуты назад в браузерное приложение, которое в свою очередь получит id_token и access_token. Как видно ниже, компонент app-login-menu сам определил, что авторизация успешно завершилась, и отобразил «приветствие», а также кнопку для Logout.

![](/dot-net/2020-01-21-using-identityserver-and-net-core/crl-aa3ehwgdjybwkmb7nbg92pk.png)

При открытии в браузере «средств разработчика», можно увидеть в backstage все взаимодействие по протоколу OIDC/OAuth. Это получение информации об авторизующем сервере
через endpoint .well-known/openid-configuration и pooling активности сессии через точку доступа connect/checksession. Помимо этого, модуль авторизации настроен на механизм «тихого обновления токенов», когда при истекании времени действия токена доступа, система самостоятельно проходит шаги авторизации в скрытом iframe. Отключить автообновления токенов можно задав значение параметра includeIdTokenInSilentRenew равным «false» в файле authorize.service.ts.


Теперь можно заняться ограничением доступа неавторизированным пользователям со стороны компонентов SPA приложения, а также некоторым API контроллерам на бэке. В целях демонстрации некоторого API создадим в папке Models класс ExchangeRateItem, а так же контроллер в папке Controller, возвращающий некоторые случайные данные.

```csharp

//Controller:
    [ApiController]
    public class ExchangeRateController
    {
        private static readonly string[] Currencies = new[]
        {
            "EUR", "USD", "BGN", "AUD", "CNY", "TWD", "NZD", "TND", "UAH", "UYU", "MAD"
        };

        [HttpGet("api/rates")]
        public IEnumerable<ExchangeRateItem> Get()
        {
            var rng = new Random();
            return Enumerable.Range(1, 5).Select(index => new ExchangeRateItem
            {
                FromCurrency = "RUR",
                ToCurrency = Currencies[rng.Next(Currencies.Length)],
                Value = Math.Round(1.0+ 1.0/rng.Next(1, 100),2)
                })
                .ToArray();
        }
    }

//Models:
public class ExchangeRateItem
    {
        public string FromCurrency { get; set; }

        public string ToCurrency { get; set; }

        public double Value { get; set; }
    }

```

Далее на стороне фронтенда создаем новый компонент, который будет получать и отображать данные по курсам валют из только что созданного контроллера.

```
ng generate component ExchangeRate -t=true -s=true --skipTests=true
```

```typescript

import { Component, OnInit, Input } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, of, Subject } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Component({
  selector: 'app-exchange-rate',
  template: `
<div class="alert alert-danger" *ngIf="errorMessage | async as msg">
  {{msg}}
</div>
<table class='table table-striped'>
  <thead>
  <tr>
    <th>From currency</th>
    <th>To currency</th>
    <th>Rate</th>    
  </tr>
  </thead>
  <tbody>
  <tr *ngFor="let rate of rates | async">
    <td>{{ rate.fromCurrency }} </td>
    <td>{{ rate.toCurrency }}</td>
    <td>{{ rate.value }}</td>
  </tr>
  </tbody>
</table>

  `,
  styles: []
})
export class ExchangeRateComponent implements OnInit {
  public rates: Observable<ExchangeRateItem[]>;

  public errorMessage: Subject<string>;

  @Input() public apiUrl: string;

  constructor(private http: HttpClient) {
    this.errorMessage = new Subject<string>();
  }

  ngOnInit() {
    this.rates = this.http.get<ExchangeRateItem[]>("/api/"+this.apiUrl).pipe(catchError(this.handleError(this.errorMessage)) );
  }

  private handleError(subject: Subject<string>): (te:any) => Observable<ExchangeRateItem[]> {
    return (error) => {
      let message = '';
      if (error.error instanceof ErrorEvent) {
        message = `Error: ${error.error.message}`;
      } else {
        message = `Error Code: ${error.status}\nMessage: ${error.message}`;
      }
      subject.next(message);
      let emptyResult: ExchangeRateItem[] = [];
      return of(emptyResult);
    }
  }
}

interface ExchangeRateItem {
  fromCurrency: string;
  toCurrency: string;
  value: number;
}

```

Теперь осталось начать использовать его на странице app-data, просто в template записав строку `<app-exchange-rate apiUrl=«rates»></app-exchange-rate>` и можно снова запустить проект. Мы увидим при переходе по целевому пути, что компонент получил данные и вывел их в виде таблицы.

Далее попробуем добавить требование авторизации доступа к API контроллера, для этого у класса `ExchangeRateController` добавим атрибут `[Authorize]` и еще раз запустим SPA, однако, после того, как мы перейдем снова на компонент, вызывающий наше API, мы увидим ошибку, что свидетельствует об отсутствующих заголовках авторизации.

![](/dot-net/2020-01-21-using-identityserver-and-net-core/fokvovvxneeouxy0k30wuvlwj4u.png)

Для корректного добавления в исходящие запросы токена авторизации можно
задействовать Angular механизм перехватчиков Interceptors. К счастью, импортированный модуль уже содержит необходимый тип, нам нужно только зарегистрировать его в базовом модуле приложения.

```typescript

providers: [
    { provide: HTTP_INTERCEPTORS, useClass: AuthorizeInterceptor, multi: true }
  ],

```

После этих шагов все должно корректно отработать. Если снова посмотреть инструменты разработчика, в браузере будет виден новый заголовок авторизации Bearer access_token. На бэкенде данный токен будет провалидирован IdentityServer и он же даст разрешение на вызов защищенной точки API.

В окончание примера интеграции с сервером авторизации, можно на маршрут с данными по обменным курсам в SPA поставить Guard активации, он не даст пользователям переключиться на страницу, если они в данный момент не авторизованы. Данный защитник также представлен в импортированном ранее модуле, его нужно просто навесить на целевой маршрут.

```typescript
{ path: 'data', component: DataComponent, canActivate: [AuthorizeGuard]  }
```

Теперь в случае, когда пользователь не залогинился в приложении и выбрал ссылку на наш защищенный компонент, его сразу перебросит на страницу авторизации с просьбой ввести логин и пароль. Итоговый код доступен на [github](https://github.com/alexs0ff/IdentityServer4Core3Example/commit/36ce262ef84c0105adc0a28ddf3877c7678b3bfb).

## Подключение внешнего входа в систему через поставщика Google

Для подключения входа через учетные записи Google для ASP.NET core 1.1/2.0+ существует отдельный Nuget пакет Microsoft.AspNetCore.Authentication.Google, однако, в связи с изменениями в политике самой корпорации, у Microsoft есть планы для ASP.NET Core 3.0+ признать его устаревшим. И теперь подключать рекомендуется через вспомогательный метод OpenIdConnectExtensions и AddOpenIdConnect, его мы и будем использовать в данной статье.


Устанавливаем расширение OpenIdConnect:

```
dotnet add package Microsoft.AspNetCore.Authentication.OpenIdConnect -v 3.0.0-preview7.19365.7
```

Чтобы начать работу, нам нужно получить два ключевых значения от Google – Id Client и Client Secret, для этого предлагается выполнить следующие шаги:

- Перейти по [ссылке](https://developers.google.com/identity/sign-in/web/sign-in#before_you_begin) и выбрать кнопку « CONFIGURE A PROJECT».
- В появившемся диалоге указать Web server.
- Далее в текстовом поле «Authorized redirect URI» указать конечную точку на стороне создаваемого приложения для авторизационного редиректа Google (т.к. сейчас сайт находится по адресу https://localhost:44301/, то мы тут указываем https://localhost:44301/signin-google)
- После этого мастер отобразит значение ClientID и Client Secret.
- Если планируется развертывать сайт в интернете или другом сервере, отличном от локального, необходимо обновить URL указанный на предыдущем шаге.

Для хранения полученных ключей локально мы будем использовать Secret Manager. Чтобы сохраненные значения были доступны из приложения, необходимо выполнить следующие команды.

```

dotnet user-secrets init
dotnet user-secrets set "Authentication:Google:ClientId" "Должен быть ClientID"
dotnet user-secrets set "Authentication:Google:ClientSecret" "Должен быть ClientSecret"

```

Далее вызываем ранее указанный метод для добавления входа через поставщиков Google.

```csharp

services.AddAuthentication()
                .AddOpenIdConnect("Google", "Google",
                    o =>
                    {
                        IConfigurationSection googleAuthNSection =
                            Configuration.GetSection("Authentication:Google");
                        o.ClientId = googleAuthNSection["ClientId"];
                        o.ClientSecret = googleAuthNSection["ClientSecret"];
                        o.Authority = "https://accounts.google.com";
                        o.ResponseType = OpenIdConnectResponseType.Code;
                        o.CallbackPath = "/signin-google"; 
                    })
                .AddIdentityServerJwt();

```

Теперь можем запустить [проект](https://github.com/alexs0ff/IdentityServer4Core3Example/commit/6795d097a6da83baad8a79823eb50206237081ab) и зайти на страницу логина, там должен появиться
вариант со входом через сервисы Google. Можно его выбрать и проверить, что все нормально работает, после авторизации пользователя перекидывает снова на SPA и там он может запросить данные со страницы курсов валют.

![](dot-net\2020-01-21-using-identityserver-and-net-core\xirl5hobkgfaukb4wzp5sibosj4.png)

Аналогичными способами, описанными выше, можно подключить других поставщиков OAuth аутентификации, причем добавить их в наше приложение одновременно. Полный список Nuget пакетов со вспомогательными методами можно найти по [ссылке](https://www.nuget.org/packages?q=owners%3Aaspnet-contrib+title%3AOAuth).


## Доработка возможности входа под пользователями Windows

В некоторых случаях, когда SPA приложение предназначено для работы в интрасетях под управлением операционных систем Microsoft, может требоваться вход под учетными записями ActiveDirectory. В ситуации, если у нас выполняется серверный рендеринг Html в приложениях типа ASP.NET, WebForms и т.д., мы можем включить требование по Windows авторизации и работать в коде с типами WindowsIdentity, однако, с браузерными приложениями такой подход не сработает. Тут может нам помочь Identity Server, мы ему можем сказать чтобы он использовал пользователей Windows, как внешнего поставщика учетных записей и многие связанные с ними данные отображал в claims id_token и access_token. К счастью, разработчики IS4 уже нас снабдили примером, как можно добавить требуемую функциональность к разрабатываемым приложениям, этот пример можно найти на [github](https://github.com/damienbod/AspNetCoreWindowsAuth). Однако, его нужно будет адаптировать под наши нужды и связать с измененной инфраструктурой ASP.NET Core Identity 3.0.

В процессе внедрения нам необходимо доработать стандартный код шаблонов Identity, для этого затянем Razor шаблоны Login и ExternalLogin в наш проект (утилита CLI aspnet-codegenerator должна быть установлена глобально):

```

dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design

dotnet aspnet-codegenerator identity -dc IdentityServer4WebApp.Data.ApplicationDbContext --files "Account.Login;Account.ExternalLogin"

```

После выполнения вышеуказанных инструкций, в проекте появится новая Area с именем Identity и несколькими файлами внутри, в некоторые из них в дальнейшем мы и внесем изменения.

Чтобы наш новый пункт отобразился в меню внешних поставщиков, мы должны переписать фильтр возможных схем авторизации. Дело в том, что Identity берет внешних поставщиков аутентификации из IAuthenticationSchemeProvider. GetAllSchemesAsync() по предикату DisplayName != null, а вот у Windows поставщика свойство DisplayName = null. Для этого открываем модель LoginModel и в методе OnGetAsync заменяем следующий код:

```csharp

ExternalLogins = (await _signInManager.GetExternalAuthenticationSchemesAsync()).ToList();
// <<Заменяем на>>
ExternalLogins =(await _schemeProvider.GetAllSchemesAsync()).Where(x => x.DisplayName != null ||(x.Name.Equals(IISDefaults.AuthenticationScheme,StringComparison.OrdinalIgnoreCase))).ToList();

```

Одновременно с этим внедряем через конструктор новое поле private readonly AuthenticationSchemeProvider _schemeProvider. Далее заходим во View страницы и изменяем логику отображения имен списка поставщиков Login.cshtml:

```html

<button type="submit" class="btn btn-primary" name="provider" value="@provider.Name" title="Log in using your @provider.DisplayName account">@provider.DisplayName</button>
<!-- Заменяем на -->
<button type="submit" class="btn btn-primary" name="provider" value="@provider.Name" title="Log in using your @(provider.DisplayName ??provider.Name) account">@(provider.DisplayName ??provider.Name)</button>

```

И напоследок, включаем windows авторизацию при запуске в файле launchSettings.json (при деплое на IIS, необходимо включить соответствующие настройки в файлах web.config).

```json

"iisSettings": {
    "windowsAuthentication": true, 
    "anonymousAuthentication": true,
    "iisExpress": {
      "applicationUrl": "http://localhost:15479",
      "sslPort": 44301
    }
  },

```

Теперь на странице авторизации можно увидеть кнопку «Windows» в списке внешних поставщиков.

![](/dot-net/2020-01-21-using-identityserver-and-net-core/e_6s-m8ot5h52dv6dgvid8bgpky.png)

По причине наличия хостинга SPA приложения в одном проекте с IdentityServer анонимную аутентификацию нельзя полностью отключить. Поэтому воспользуемся некоторым «костылем» и заменим атрибут `[AllowAnonymous]` в классе страницы LoginModel на атрибут `[Authorize(AuthenticationSchemes = "Windows")]`, тем самым мы сможем гарантировать, что страница сможет работать с корректным WindowsIdentity.

Теперь доработаем страницу ExternalLogin, чтобы подсистема Identity смогла корректно обрабатывать Windows пользователей. Для начала создадим новый метод ProcessWindowsLoginAsync.

```csharp

private async Task<IActionResult> ProcessWindowsLoginAsync(string returnUrl)
        {
            var result = await HttpContext.AuthenticateAsync(IISDefaults.AuthenticationScheme);
            if (result?.Principal is WindowsPrincipal wp)
            {
                var redirectUrl = Url.Page("./ExternalLogin", pageHandler: "Callback", values: new { returnUrl });

                var props = _signInManager.ConfigureExternalAuthenticationProperties(IISDefaults.AuthenticationScheme, redirectUrl);
                props.Items["scheme"] = IISDefaults.AuthenticationScheme;

                var id = new ClaimsIdentity(IISDefaults.AuthenticationScheme);
                id.AddClaim(new Claim(JwtClaimTypes.Subject, wp.Identity.Name));
                id.AddClaim(new Claim(JwtClaimTypes.Name, wp.Identity.Name));
                id.AddClaim(new Claim(ClaimTypes.NameIdentifier, wp.Identity.Name));

                var wi = wp.Identity as WindowsIdentity;
                var groups = wi.Groups.Translate(typeof(NTAccount));
                var hasUsersGroup = groups.Any(i => i.Value.Contains(@"BUILTIN\Users", StringComparison.OrdinalIgnoreCase));

                id.AddClaim(new Claim("hasUsersGroup", hasUsersGroup.ToString()));

                await HttpContext.SignInAsync(IdentityConstants.ExternalScheme, new ClaimsPrincipal(id), props);

                return Redirect(props.RedirectUri);
            }

            return Challenge(IISDefaults.AuthenticationScheme);
        }

```

Новый метод подготавливает необходимую информацию из данных, полученных от операционной системы к объектам и свойствам вручную созданного принципала внешних провайдеров.

Далее переделаем метод ExternalLoginModel.OnPost на асинхронный и добавим в начала на проверку целевого провайдера:

```csharp

if (IISDefaults.AuthenticationScheme == provider)
            {
                return await ProcessWindowsLoginAsync(returnUrl);
            }

```

В процессе подготовки Claim для Windows пользователя мы использовали один нестандартный Claim «hasUsersGroup», чтобы он был доступен в токенах ID и access, необходимо его обработать отдельно. Для этого мы воспользуемся механизмами ASP.NET Identity и сохранением его в UserClaims. Начнем с написания вспомогательного метода в классе ExternalLoginModel.

```csharp

private async Task UpdateClaims(ExternalLoginInfo info, ApplicationUser user, params string[] claimTypes)
        {
            if (claimTypes == null)
            {
                return;
            }

            var claimTypesHash = new HashSet<string>(claimTypes);

            var claims = (await _userManager.GetClaimsAsync(user)).Where(c => claimTypesHash.Contains(c.Type)).ToList();

            await _userManager.RemoveClaimsAsync(user, claims);

            foreach (var claimType in claimTypes)
            {
                if (info.Principal.HasClaim(c => c.Type == claimType))
                {
                    claims = info.Principal.FindAll(claimType).ToList();
                    await _userManager.AddClaimsAsync(user, claims);
                }
            }
        }
```

Добавим вызов созданного кода в метод OnPostConfirmationAsync (который вызывается в момент первого захода пользователя в систему).

```csharp

var result = await _userManager.AddLoginAsync(user, info);
if (result.Succeeded)
{
    await _signInManager.SignInAsync(user, isPersistent: false);
    _logger.LogInformation("User created an account using {Name} provider.", info.LoginProvider);

    await UpdateClaims(info, user, "hasUsersGroup");//Сюда

    return LocalRedirect(returnUrl);
}

```

И в метод OnGetCallbackAsync, вызывающийся при последующих входах в систему.

```csharp

var result = await _signInManager
    .ExternalLoginSignInAsync(info.LoginProvider, info.ProviderKey, isPersistent: false, bypassTwoFactor : true);

if (result.Succeeded)
{
    var user = await _userManager.FindByLoginAsync(info.LoginProvider, info.ProviderKey);

    await UpdateClaims(info, user, "hasUsersGroup");//Сюда
}

```

Теперь предположим, что мы хотим ввести к нашему методу WebAPI требование наличия клейма «hasUsersGroup». Для этого определим новую политику «ShouldHasUsersGroup»

```csharp

services.AddAuthorization(options =>
{
    options.AddPolicy("ShouldHasUsersGroup", policy => { policy.RequireClaim("hasUsersGroup");});
});

```

Далее в контроллере ExchangeRateController создадим новую точку подключения и обозначим требование только что созданного Policy.

```csharp

[Authorize(Policy = "ShouldHasUsersGroup")]
[HttpGet("api/internalrates")]
public IEnumerable<ExchangeRateItem> GetInternalRates()
{
    return Get().Select(i=>{i.Value=Math.Round(i.Value-0.02,2);return i;});
}

```

Создадим новый view для отображения скорректированных данных.

```
ng generate component InternalData -t=true -s=true --skipTests=true
```

Заменим в нем template и зарегистрируем в таблице маршрутизации и в представлении верхнего меню.

```typescript

//internal-data.component.ts:
template: `<app-exchange-rate apiUrl="internalrates"></app-exchange-rate> `,

//app-routing.module.ts:
{ path: ' internaldata', component: InternalDataComponent, canActivate: [AuthorizeGuard]  }

//app.component.html:
<li class="nav-item" [routerLinkActive]='["link-active"]'>
    <a class="nav-link text-dark" [routerLink]='["/internaldata"]'>Internal api data</a>
</li>

```

После вышеперечисленных шагов мы можем запустить наше приложение и перейти по новой ссылке, однако, приложение вернет нам ошибку. Дело в том, что accsee_token на данный момент не содержит claim с именем hasUsersGroup, чтобы это исправить все нестандартные клеймы необходимо прописывать в конфигурацию ApiResources сервера авторизации. К сожалению, в момент написания статьи, такою настройку нельзя сделать декларативно через файл appsettings.json, и поэтому придется программным путем вносить необходимое изменение в методе Startup. ConfigureServices.

```csharp

services
    .AddIdentityServer()
    .AddApiAuthorization<ApplicationUser, ApplicationDbContext>(options =>
    {
        var apiResource = options.ApiResources.First();
        apiResource.UserClaims = new[] { "hasUsersGroup" };
    });

```

Теперь, если еще раз запустить приложение, заново зайти под windows пользователем и перейти по целевой ссылке, то новая информация должна без ошибок отобразиться в браузере.

Осталось добавить последний штрих к разрабатываемому примеру, а именно – реализовать Guard для проверки наличия claim с именем «hasUsersGroup» для маршрута роутера к «внутреннему обменному курсу». Для этого создадим Guard следующей командой:

```
ng generate guard AuthorizeWindowsGroupGuard --skipTests=true
```

И впишем в него такой код:

```typescript

import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';
import { map,tap} from 'rxjs/operators';
import { CanActivate, ActivatedRouteSnapshot, RouterStateSnapshot, Router, UrlTree } from '@angular/router';
import { AuthorizeService } from "./authorize.service";
import { ApplicationPaths, QueryParameterNames } from './api-authorization.constants';

@Injectable({
  providedIn: 'root'
})
export class AuthorizeWindowsGroupGuardGuard implements CanActivate{
  constructor(private authorize: AuthorizeService, private router: Router) {}

  canActivate(route: ActivatedRouteSnapshot, state: RouterStateSnapshot): Observable<boolean | UrlTree> |
                                                                          Promise<boolean | UrlTree> |
                                                                          boolean |
    UrlTree {
    return this.authorize.getUser().pipe(map((u: any) => !!u && !!u.hasUsersGroup)).pipe(tap((isAuthorized:boolean) => this.handleAuthorization(isAuthorized, state)));;
  }

  private handleAuthorization(isAuthenticated: boolean, state: RouterStateSnapshot) {
    if (!isAuthenticated) {
      window.location.href = "/Identity/Account/Login?" + QueryParameterNames.ReturnUrl + "=/";
    }
  }
}

```

И, конечно, заменим стандартный тип только что созданным в таблице маршрутизации.

```typescript
{ path: 'internaldata', component: InternalDataComponent, canActivate: [AuthorizeWindowsGroupGuardGuard]
```

Теперь осталось указать нашему IdentityServer, чтобы он возвращал помимо стандартных claims (sub, profile), еще и дополнительно созданный «hasUsersGroup». Это делается путем создания нового IdentityResource, опять-таки на момент написания статьи декларативный стиль указания IdentityResources не поддерживается и придется указывать явно через код в Startup.ConfigureServices.

```csharp

var identityResource = new IdentityResource
{
    Name = "customprofile",
    DisplayName = "Custom profile",
    UserClaims = new[] { "hasUsersGroup" },
};
identityResource.Properties.Add(ApplicationProfilesPropertyNames.Clients, "*");
options.IdentityResources.Add(identityResource); 

```

Наконец-то мы [достигли](https://github.com/alexs0ff/IdentityServer4Core3Example/commit/0d2736573379d2ef528b0ab9b01d7631c8a58760) всех поставленных целей, теперь после запуска программы необходимо залогиниться под пользователем windows и пройти по «внутреннем курсам» в меню SPA – все должно отобразиться, а если выбрать другого пользователя, только что созданный Guard перебросит на страницу авторизации.

## Заключение

Безусловно, разработчики ASP.NET Core 3.0 проделали огромную работу по упрощению внедрения некоторых стандартных сценариев интеграции с IdentityServer4, что способствует более легкой поддержке и простоте разрабатываемых решений. По причине использования в статье preview версии, некоторые вещи могут в дальнейшем быть изменены. В любом случае, постараюсь держать исходный код размещенный на github в актуальном состоянии.
