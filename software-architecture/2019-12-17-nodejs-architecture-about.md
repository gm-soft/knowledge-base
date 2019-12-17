---
category: software architecture
tags: [clean architecture, node js]
---

# Архитектура/паттерны организации кода Node.js приложений

Источник: [Gist](https://gist.github.com/zmts/6ac57301e2e8e8e9e059e9c087732c05) by [Sasha Zmts](https://gist.github.com/zmts)

![node.js](https://i.imgur.com/EGp4qfC.jpg)

## TL;DR
code: https://github.com/zmts/supra-api-nodejs

## Предисловие
Одной из болезней **Node.js** комьюнити это отсутствие каких либо крупных фреймворков, действительно крупных уровня Symphony/Django/RoR/Spring. Что является причиной все ещё достаточно юного возраста данной  технологии. И каждый кузнец кует как умеет ну или как в интернетах посоветовали. Собственно это моя попытка выковать некий свой подход к построению **Node.js** приложений.

### Несколько слов что такое архитектура (IMHO):
```
Архитектура - набор подходов для организации программно-аппаратного комплекса.
Описание компонентов системы и взаимосвязей между ними.
```

### Несколько слов про разделение приложения на слои:

Обычно в слое Контроллеров данные из запроса валидируются и приводятся к виду необходимому для последующего сервисного слоя. В свою очередь сервисный слой полностью изолирует бизнес логику и возвращает результат вычислений в контроллер. Или иначе ? Как вы считаете ? И так каждый программист делает как считает нужно либо так как привык. Давайте будем откровенны. Приводило ли подобная архитектура к тому ради чего задумывалась(к порядку и разделению логики) ? Зачастую все эти попытки сводятся к `big ball of mud`. Зачастую в контроллер просачивается бизнес логика, а в сервисный слой валидация, а то и вообще контекст запроса. Дабы так не происходило данный подход предлагает использовать единый слой для всей логики касаемо конкретного `use case`. Назовем этот слой `Action layer`. В итоге имеем:

1. Минималистичный контроллер - сугубо маппинг экшенов на роуты
2. Слой экшенов отвечающий за валидацию, всевозможные проверки доступа(владелец/не владелец/админ/не админ/аноним...) и бизнес логику
3. Data layer - прослойка к БД

```
Controller(роутинг, проверка прав по роли) >>
Action(проверка прав по id, схема валидации запроса, логика юзкейса) >>
Data Layer(биндинги к БД)
```

## Содержание

- [Assertion(type checking)](#assertiontype-checking)
- [Config](#config)
- [Server](#serverapp-initialization)
- [Middlewares](#middlewares)
- [Controllers](#controllers)
- [Actions](#actions)
- [Model](#model)
- [Clients](#clients)
- [Providers](#providers)
- [Helpers](#authhelpers)
- [DAO](#dao)
- [Errors](#errors)
- [Logging](#logging)
- [Policy/Roles/Permissions](#policyrolespermissions)

## Assertion(type checking)

Проверка типов это один из инструментов программиста который намного облегчает жизнь и помогает держать приложение в порядке. В этом мне помогает небольшой [Assertion class](https://github.com/zmts/supra-api-nodejs/tree/master/core/lib/assert). Из существующих библиотек есть [node-assert-plus](https://github.com/joyent/node-assert-plus).

```js

async function foo (id, options = {}) {
  assert.integer(id, { required: true })
  assert.object(options, { required: true, notEmpty: true })
  
  const data = await db.getById(id, options)
  return data
}

```

`Assertion` это набор статических методов для базовой проверки типов аргуметров и возможно некоторых дополнительных опций (аля `{ required: true, notEmpty: true, positive: true }`).

## Config

Все параметры неободимые для работы приложения должны хранится в едином конфиге. Параметры конфига делятся на два типа: констаннты и переменные. Первые храним как обычные поля, вторые извлекаем из переменных окружения (для примера это: логины/пароли доступа в БД, ключи для шифрования сессий или JWT токенов, ключи доступа к другим внешним ресурсам итд...).

```js

class AppConfig extends BaseConfig {
  constructor () {
    super()
    this.nodeEnv = this.set('NODE_ENV', v => ['development', 'production'].includes(v), 'development')
    this.port = this.set('APP_PORT', this.joi.number().port().required(), 5555)
    this.host = this.set('APP_HOST', this.joi.string().required(), 'localhost')
    this.name = this.set('APP_NAME', this.joi.string().required(), 'SupraAPI')
    this.foo = 'bar'
  }
}

```

`set` первым аргументом принимает название переменной окружения, вторым ф-цию валидатор (будь то `joi` правило или обычная ф-ция возвращающая `boolean`) и третий опциональный: аргумент по умолчанию. Таким образом система следит что бы все значения конфига были валидны. При условии невалидности приложение не стартует и выбросит исключение.

## Server(App initialization)

Жизнь приложения начинается с класса [Server](https://github.com/zmts/supra-api-nodejs/blob/master/core/Server.js). Класс отвечает за:
- Создание веб сервера(в данном случае `express.js`)
- Инициализию дефолтных мидлварей(`bodyParser`, `helmet`, etc.)
- Инициализию контроллеров(роутеров)
- Инициализию дефолтного обработчика ошибок (`errorMiddleware`)
- Отслеживание глобальных ошибок/исключений(`uncaughtException`, `unhandledRejection`, etc.)

Компоненты(мидлвари, роутеры, провайдеры) приложения обязаны инициализироватся асинхронно друг после друга. Последовательная инициализация позволяет подготовить зависимые/асинхронные данные для использования другими компонентами.

```

Server start initialization... 
InitMiddleware initialized ... 
CorsMiddleware initialized ...
SanitizeMiddleware initialized ... 
RootRouter initialized... 
RootProvider initialized..
UsersRouter initialized... 
UsersProvider initialized... 
DevErrorMiddleware initialized ... 
Server initialized... {"port":"5555","host":"localhost"}

```

## Middlewares

Прослойка промежуточных обработчиков запросов. Основаня задача дополниение контекста запроса/ответа. Например: 
- Получение мета данных пользователя из JWT и добавление их в `req.currentUser`
- Санитизация запросов
- Установка CORS в ответ(`res`)

Плохая практика навешивать в мидлвари какую либо бизнес логику или тяжолые блокирующие операции.

## Controllers

Основные задачи контроллера:
- Роутинг
- Взаимовоздействие с пользователем(принимает запрос, оправляет ответ)
- Вызывает Action
- Устанавливает хедеры

<details>
 <summary>За автоматизацию вызова экшена в контроллере и отправку ответа пользователю отвечает ф-ция actionRunner</summary>

```js
actionRunner (action) {
  assert.func(action, { required: true })

  return async (req, res, next) => {
    assert.object(req, { required: true })
    assert.object(res, { required: true })
    assert.func(next, { required: true })

    const ctx = {
      currentUser: req.currentUser,
      body: req.body,
      query: req.query,
      params: req.params,
      ip: req.ip,
      method: req.method,
      url: req.url,
      headers: {
        'Content-Type': req.get('Content-Type'),
        Referer: req.get('referer'),
        'User-Agent': req.get('User-Agent')
      }
    }

    try {
      await actionTagPolicy(action.accessTag, ctx.currentUser)
      
      if (action.validationRules && action.validationRules.notEmptyBody && !Object.keys(ctx.body).length) {
        return next(new ErrorWrapper({ ...errorCodes.EMPTY_BODY }))
      }

      if (action.validationRules) {
        await this.validate(ctx, action.validationRules)
      }
      
      const response = await action.run(ctx)

      if (response.headers) res.set(response.headers)

      return res.status(response.status).json({
        success: response.success,
        message: response.message,
        data: response.data
      })
    } catch (error) {
      error.req = ctx
      next(error)
    }
  }
}
```
</details>


<details>
 <summary>Таким образом контроллер представляет собой совсем небольшую прослойку мапинга экшенов на роуты</summary>

```js
const router = require('express').Router()
const actions = require('../actions/posts')
const BaseController = require('../core/BaseController')
const ErrorWrapper = require('../core/ErrorWrapper')
const { errorCodes } = require('../config')
const logger = require('../logger')

class PostsController extends BaseController {
  get router () {
    router.param('id', preparePostId)

    router.get('/', this.actionRunner(actions.ListPostsAction))
    router.get('/:id', this.actionRunner(actions.GetPostByIdAction))
    router.post('/', this.actionRunner(actions.CreatePostAction))
    router.patch('/:id', this.actionRunner(actions.UpdatePostAction))
    router.delete('/:id', this.actionRunner(actions.DeletePostAction))

    return router
  }

  async init () {
    logger.info(`${this.constructor.name} initialized...`)
  }
}

function preparePostId (req, res, next) {
  const id = Number(req.params.id)
  if (id) req.params.id = id
  next()
}

module.exports = new PostsController()
```
</details>

Рассмотрим более подробно что из себя представляет метод `actionRunner`. Если посмотреть на стандартное определение роута в Express.js.
```js
router.get('/api/users', function(req, res){
  res.send('hello world')
})
```

Мы увидим что для работы нам понадобится передать в ф-цию маппинга роута несколько параметров это сам путь(`'/api/users'`) и один или несколько обработчиков. У каждого обработчика есть доступ к двум(на самом деле их больше но в данный момент нас интересуют только первые два) аргументам: `req` - объект запроса и `res` - объект ответа.

За что и отвечает `actionRunner`:
- Передает данные из запроса в бизнес логику (обработчик экшена: метод `run`) и вызывает ее.
- Забирает результат вычисления и отправляет(`req.json(data)`) пользователю
- В случе ошибки: собираются метаданные и передаются дальше для логирования

__Выходит такой Lifecycle >> Запрос проходит дефолтные мидлвари >> Попадает в контроллер >> Попадает в `actionRunner` тот проверяет права доступа текущего юзера к экшену, валидирует запрос и стартует процесс обработки (статическая ф-ция `run`) >> В экшене выполняется бизнес логика и результат возращается в контроллер >> `actionRunner` получает результат и возвращает его клиенту.__

Что не стоит делать в контроллере:
- Валидировать параметры(`req.params`). Как показала практика это плохая идея. Проверку параметров лучше стоит делать непосредственно в экшене. Таким образом в дальнейшем будет более наглядно видно какие парметры в запросе доступны экшену.

## Actions
Основным ключевым моментом является использование отдельного класса для каждого эндпоинта. Я определяю понятие `Action` как класс инкапсулирующий всю логику работы эндпоинта. То есть для реализации круда у нас будет 5 файлов (`CreatePostAction`, `GetPostByIdAction`, `UpdatePostAction`, `DeletePostAction`, `ListPostsAction`) по экшену на каждый эндпоинт.

Каждый экшен обязан имплементировать такой контракт:
- Статический метод `run` в задачи которого входит выполнение бизнес логики. Его и вызывает `actionRunner`.
- Геттер `validationRules` - объект выполняющий роль схемы валидации входящих параметров экшена.
- Геттер `accessTag` - тег по которому специальный сервис проверяет права доступа.

Правила валидации (`validationRules`) импортируются из модели или в случае необходимости используются кастомные.

***Роль модели в экшене:***
Дабы не дублировать кучу подобных моделей(одна модель для создания сущности со всеми `required` полями, другая для обновления, третья для обновления конкретно указанного набора полей итд...) было принято решение не указывать в ф-ции валидации модели `required` требование. Валидаровать поля запроса в экшене помогает небольшой класс-хелпер `RequestRule` в его задачи входит предоставить [базовой ф-ции валидации](https://github.com/zmts/supra-api-nodejs/blob/master/controllers/BaseController.js#L96) интересующие ее [аргументы](https://github.com/zmts/supra-api-nodejs/blob/master/controllers/BaseController.js#L112) (ф-цию валидатор и `required` флаг).
```js
{
  validationRules: { 
    body: { 
      id: new RequestRule(PostModel.schema.id, { required: true })
    }
  }
}
```
Повторюсь таким образом мы избегаем черезмерного колличества однотипных моделей используя в экшене непосрественно те правила которые необходимы для конкретного юзкейса.

<details>
 <summary>В итоге имеем один класс(один файл) в котором сосредоточена все логика эндпоинта</summary>

```js
const joi = require('joi')

const BaseAction = require('../BaseAction')
const PostDAO = require('../../dao/PostDAO')
const PostModel = require('../../models/PostModel')

class CreateAction extends BaseAction {
  static get accessTag () {
    return 'posts:create'
  }

  static get validationRules () {
    return {
      params: {
        id: [PostModel.schema.id, true]
      },
      body: {
        title: [PostModel.schema.title, true],
        content: [PostModel.schema.content, true],
        someCustomField: [new Rule({
          validator: v => (typeof v === 'string') && v.length >= 10,
          description: 'string; min length 10 chars;'
        }), true],
      }
    }
  }

  static async run (ctx) {
    const { currentUser } = ctx
    const data = await PostDAO.BaseCreate({ ...ctx.body, userId: currentUser.id })

    return this.result({ data })
  }
}

module.exports = CreateAction
```
</details>

Все эшнены являются __framework agnostic__ это значит что в экшенах отсуствует код относящийся к веб-фреймворку. Что позволеят нам переиспользовать экшены как в других проектах так и с другими фреймворками.

__p.s.__
А еще разделение логики на отдельные экшены, облегчает командную работу над проектом. Меньше конфликтов при слиянии веток :)

## Model
В данном подходе модель представляет собой исключительно набор полей и правил валидации без какой либо бизнес логики и доп. ф-ционала.


<details>
 <summary>model example:</summary>

```js
const joi = require('@hapi/joi')
const { BaseModel, Rule } = require('supra-core')

const schema = {
  id: new Rule({
    validator: v => joi.validate(v, joi.number().integer().positive(), e => e ? e.message : true),
    description: 'number integer positive'
  }),
  userId: new Rule({
    validator: v => joi.validate(v, joi.number().integer().positive(), e => e ? e.message : true),
    description: 'number; integer; positive;'
  }),
  title: new Rule({
    validator: v => joi.validate(v, joi.string().min(3).max(20), e => e ? e.message : true),
    description: 'string; min 3; max 20;'
  }),
  content: new Rule({
    validator: v => joi.validate(v, joi.string().min(3).max(5000), e => e ? e.message : true),
    description: 'string; min 3; max 5000;'
  })
}

class PostModel extends BaseModel {
  static get schema () {
    return schema
  }
}

Каждое поле модели это инстанс `Rule` класса. `Rule` принимает в себя валидатор и описание онной простым текстом. Выполнение ф-ции валидации обязано вернуть либо булево значение либо строку(`error.message`)

module.exports = PostModel
```
</details>

Все последующие проверки полей в других компонентах приложения обязаны импортироваться из схемы модели. Например в cлое `DAO` в неком `getById(id)` вместо того что-бы делать:
```js
async function getById (id, options = {}) {
  assert.integer(id, { required: true })
  assert.object(options, { required: true, notEmpty: true })
  
  const data = await db.getById(id, options)
  return data
}
```

Стоит лучше взять правило из схемы модели:
```js
async function getById (id, options = {}) {
  assert.validate(Post.schema.id, { required: true })
  assert.object(options, { required: true, notEmpty: true })
  
  const data = await db.getById(id, options)
  return data
}
```

## Clients
`Clients` - это врапперы над внешними API/клиентами. Предназначение данной абстракции обеспечение единого контракта работы с внешними ресурсами. Допустим нам необходимо использовать в качестве хранилища файлов сервис от Амазон AWS S3. Мы создаем `S3Client` добавляем в него свои методы-обертки для работы с данным хранилищем. В случае критического изменения в клиенте амазона мы соотвественно меняем методы обертки без ущерба и переписывания остальной логики в остальных частях приложения. Единожды создав экземпляр клиент-враппера, можно переиспользовать его в разных местах вместо того что бы плодить инстансы голого клиента.

<details>
  <summary>client example:</summary>

```js
const AWS = require('aws-sdk')
const $ = Symbol('private scope')

class S3Client {
  constructor (options) {
    assert.object(options, { required: true, notEmpty: true })
    assert.string(options.access, { required: true, notEmpty: true })
    assert.string(options.secret, { required: true, notEmpty: true })
    assert.string(options.bucket, { required: true, notEmpty: true })

    AWS.config.update({
      accessKeyId: options.access,
      secretAccessKey: options.secret
    })

    this[$] = {
      client: new AWS.S3(),
      bucket: options.bucket
    }

    __logger.info(`${this.constructor.name} constructed...`)
  }

  async uploadImage (buffer, fileName) {
    if (!Buffer.isBuffer(buffer)) {
      throw new Error(`${this.constructor.name}: buffer param is not a Buffer type`)
    }

    assert.string(fileName, { required: true, notEmpty: true })

    return new Promise((resolve, reject) => {
      const params = {
        Bucket: this[$].bucket,
        Key: fileName,
        Body: buffer,
        ContentType: 'image/jpeg'
      }

      this[$].client.upload(params, (error, data) => {
        if (error) {
          __logger.error(`${this.constructor.name}: unable to upload objects`, error)
          return reject(error)
        }
        resolve(data.Location)
      })
    })
  }
}

module.exports = S3Client
```
</details>

## Providers
Как быть в ситуации когда необходимо использовать один и тот же клиент в нескольких экшенах ? Для этого случая предназанчен слой `Providers`. Дабы не плодить для каждого экшена свой клиент создаем необходимое единожды в провайдере, а дальше импортируем его в нужных местах.
<details>
  <summary>code:</summary>

```js
const S3Client = require('../core/clients/S3Client')
const config = require('../config')

class RootProvider {
  constructor () {
    this.s3Client = new S3Client({
      access: config.s3.access,
      secret: config.s3.secret,
      bucket: config.s3.bucket
    })
  }
  async init () {
    __logger.info(`${this.constructor.name} initialized...`)
  }
}

module.exports = new RootProvider()
```
</details>

## Auth(helpers)
Хелперы отвечают за всевозможный процессинговые и утилитарные ф-ции(шифрование, JWT) В своем большинстве реализованные через промис. 

## DAO
Как не трудно догадаться это слой работы с БД. Исключительно экшены имеют доступ к DAO. Ни в каких сервисах или миддлварях не должно быть методов из DAO. Для работы с БД я юзаю https://github.com/Vincit/objection.js

## Errors
Работа с ошибками организована через единственнный кастомный класс ошибки и список эррор кодов (хранящийся в виде конфига), это позволяет не создавать на каждый тип ошибки свой класс.
```js
class ErrorWrapper extends Error {
  constructor (options) {
    if (!options || !options.message) throw new Error('message param required')

    super()
    this.message = options.message
    this.status = options.status || 500
    this.code = options.code || 'SERVER_ERROR'
  }
}

module.exports = ErrorWrapper
```
```js
module.exports = {
  ACCESS: { message: 'Access denied', status: 403, code: 'ACCESS_ERROR' },
  BAD_ROLE: { message: 'Bad role', status: 403, code: 'BAD_ROLE_ERROR' },
  DB: { status: 500, code: 'DB_ERROR' }
}
```
```js
new ErrorWrapper({ ...errorCodes.ACCESS, message: 'Access denied dude !!!' })
```
В последствии на фронте выводим ошибку из поля `message` или то что фронтенд посчтитает нужным в зависимости от поля `code`.

### Обработка ошибок
<details>
<summary>Опишу частую ошибку встречающуюся во многих проектах на просторах интернета</summary>

```js
// IS BAD

testControllerHandler (req, res, next) {
  try {
    // my logic with throwed error
    await testService.getTest({ id: req.params.id })
  } catch (error) {
    res.send(error) // это и есть локальная обработка ошибки
  }
}
```
</details>

Все ошибки обязаны быть перехваченны в единственном месте, в глобальном обработчике ошибок(глобальная `error middleware`). Остальные `unexpected` ошибки аля `unhandledRejection` в соотвествующих им хендлерам. __Это означает никаких try/catch и локальной обработки ошибок в контроллерах__
https://github.com/zmts/supra-api-nodejs/blob/master/core/lib/Server.js#L71

<details>
<summary>Только в случае если нам как-то необходимо обработать/дополнить ошибку делаем так:</summary>

```js
// IS GOOD

testControllerHandler (req, res, next) {
  try {
    // my logic with throwed error
    await testService.getTest({ id: req.params.id })
  } catch (error) {
    if (error.code === 'SPECIFIC_ERROR_CODE') {
      error.message = 'Lets describe our specific error'
    }
    next(error)
  }
}
```
</details>

<details>
<summary>В остальных случаях хватит такого</summary>

```js
// IS GOOD

testControllerHandler (req) {
  // my logic with throwed error
  await testService.getTest({ id: req.params.id })
}
```
</details>


## Logging
Вместо `console.log` используем логгеры под каждый тип сообщения (`trace`, `warn`, `error`, `fatal`). Логи пишем в `Sentry` или что-то подобное. Ошибки и security issues логируем в первую очередь, дальше все предупреждения и [трассирующие логи](https://en.wikipedia.org/wiki/Tracing_(software))

## Policy/Roles/Permissions
Данный подход рассматривает использование статических прав(__hardcode__), тоесть необходимости менять их динамически (создавать/редактировать роли со своим уникальным списком прав через БД) отсуцтвует.
 
При формировании JWT, каждый токен получает в `payload` роль пользователя по которой в дальнейшем просиходит проверка прав через сопоставление роли списку доступных ей эксес-тегов. Любой запрос с невалидным токеном или без него система идентифицирует как `ROLE_ANONYMOUS`.

Как было сказано выше у каждого экшена есть свой `accessTag `. Это обычный стринговый ключ состоящий из двух частей название ресурса и название действия(например `posts:create`).

<details>
<summary>Определяем список прав каким ролям доступны какие эксес-теги.</summary>

```js
const roles = require('./roles')

const shared = [
  'users:list',
  'users:update',
  'users:get-by-id',
  'users:remove',
  'users:change-password'
]

module.exports = {
  [roles.admin]: [
    ...shared,
    'posts:all'
  ],

  [roles.user]: [
    ...shared,
    'posts:all'
  ],

  [roles.anonymous]: [
    'users:list',
    'users:get-by-id',
    'users:create',
    'users:send-reset-email',
    'users:reset-password',
    'user:get-posts-by-user-id',
    'posts:list',
    'posts:get-by-id'
  ]
}
```
</details>

Обязательная дефотлная проверка (`actionTagPolicy`): это проверка права на обращение к экшену, она происходит в контроллере (в методе `actionRunner`).

Все дальнейшие проверки происходят непосредственно в экшене: в зависимости от требований, будь то необходимость проверить является ли пользователь владельцем ресурса ли что-то еще дополнительное.

<details>
<summary>policy/actionTagPolicy.js</summary>

```js
return new Promise((resolve, reject) => {
  if (currentUser.role === roles.superadmin) return resolve()
  if (permissions[currentUser.role].includes(accessTagAll)) return resolve()
  if (permissions[currentUser.role].includes(accessTag)) return resolve()
  return reject(new ErrorWrapper({ ...errorCodes.ACCESS, message: 'Access denied, don\'t have permissions.' }))
})
```
</details>

При обращении к айтему(чтение/get by id) необходимо проверить айтем на приватность. В позитивном случае отдаем его только владелецу.

<details>
<summary>policy/privateItemPolicy.js</summary>

```js
return new Promise((resolve, reject) => {
  if (user.role === roles.superadmin) return resolve(model)
  if (user.id === model.userId) return resolve(model)
  if (!model.private) return resolve(model)
  if (model.private) {
    return reject(new ErrorWrapper({ ...errorCodes.ACCESS, message: `User ${user.id} don't have access to model ${model.id}` }))
  }
  return reject(new ErrorWrapper({ ...errorCodes.ACCESS }))
})
```
</details>

Проверка оборачивается в промис дабы избежать лишних `if-else` конструкций в экшене при использовании.

<details>
<summary>Пример использования:</summary>

```js
class GetPostByIdAction extends BaseAction {
  static get accessTag () {
    return 'posts:get-by-id'
  }

  static async run (req) {
    const { currentUser } = req
    const model = await PostDAO.baseGetById(+req.params.id) // получили модель
    await privateItemPolicy(model, currentUser) // проверили права
    return this.result({ data: model }) // отдали результат
  }
}

module.exports = GetPostByIdAction
```
</details>

При удалении или изменении проверяем является ли текущий юзер владельцем айтема.

<details>
<summary>policy/ownerPolicy.js</summary>

```js
return new Promise((resolve, reject) => {
  if (user.role === roles.superadmin) return resolve()
  if (user.id === model.userId) return resolve()
  return reject(new ErrorWrapper({ ...errorCodes.ACCESS }))
})
```
</details>

### Function declaration convention
```js
function foo (firstRequired, secondRequired, { someOptionalParam = false } = {}) {
 // code
}
```

### Bad practice
Во времена когда я впервые столкнулся с необходимостью проектирования API на **Node.js**, а точнее мне стало любопытно сделать это самому, делал я это не всегда правильнльным путем.

- Для начала основной моей ошибкой было чрезмерное использование миддлварей. Мидллвари использовались для сервисных целей(валидация запороса, проверка прав доступа) что загромождало роутер и вносило еще большую путаницу в код. Миддлвари должны использоватся в качестве неких глобальных обработчиков и не должны навешиватся на каждый эндпоинт гроздью (https://github.com/zmts/lovefy-api-nodejs/blob/master/api/controllers/postCtrl.js#L107).
- Контроллеры представляли из себя полотно кода из обработчиков плюс роутинг (которому в контроллере не место). На первый взгляд удобно все в одном файле - не нужно прыгать из файла в файл, но когда кода стало значительно больше ситуация поменялась (https://github.com/zmts/lovefy-api-nodejs/blob/master/api/controllers/postCtrl.js).
- Компоненты приложения инициализировались при первичном старте как попало, не в контролируемой последуемости.
- Отсуствие проверки типов.
- Отсуствие логирования.
 
## P.S.
Видео Виктора Турского про архитектуру 
- https://www.youtube.com/watch?v=Z08xL-oXMh0 (2017)
- https://www.youtube.com/watch?v=TjvIEgBCxZo (2019)