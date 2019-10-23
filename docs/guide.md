
# Гайд

  В этом руководстве рассматриваются темы Koa, которые не имеют прямого отношения к API, такие как рекомендации по написанию промежуточного программного обеспечения и предложения по структуре приложения. В этих примерах мы используем асинхронные функции в качестве промежуточного программного обеспечения - вы также можете использовать commonFunction или generatorFunction, которые будут немного отличаться.

## Содержание

- [НАписание Middleware](#writing-middleware)
- [Middleware Best Practices](#middleware-best-practices)
  - [Middleware опции](#middleware-options)
  - [Именование middleware](#named-middleware)
  - [Комбинирование multiple middleware с koa-compose](#combining-multiple-middleware-with-koa-compose)
  - [отклик Middleware](#response-middleware)
- [Асинхронные операции](#async-operations)
- [Отладка Koa](#debugging-koa)

## Написание Middleware

  Koa middleware простые функции, которые возвращают `MiddlewareFunction` с подписью (ctx, next). когда
  промежуточное программное обеспечение запущено, оно должно вызываться вручную `next()` для запуска "downstream" middleware.

  Например, если вы хотите отследить, сколько времени потребуется для распространения запроса через Коа, добавив
  `X-Response-Time` поле заголовка промежуточное программное обеспечение будет выглядеть следующим образом:

```js
async function responseTime(ctx, next) {
  const start = Date.now();
  await next();
  const ms = Date.now() - start;
  ctx.set('X-Response-Time', `${ms}ms`);
}

app.use(responseTime);
```

  Если вы являетесь разработчиком фронтэнда, вы можете придумать любой код перед `next();` как фаза "capture",
  в то время как любой код после является фазой "bubble". Эта crude gif иллюстрирует, как асинхронная функция позволяет нам правильно использовать поток стека для реализации потоков запросов и ответов:

![Koa middleware](/docs/middleware.gif)

   1. Создать дату, чтобы отслеживать время ответа
   2. Ожидайте управления следующим промежуточным ПО
   3. Создать другую дату для отслеживания продолжительности
   4. Ожидайте управления следующим промежуточным ПО
   5. Установите тело ответа "Hello World"
   6. Рассчитать продолжительность времени
   7. Вывод строки в лог
   8. Рассчитать время отклика
   9. Задать `X-Response-Time` поле заголовка
   10. Вручите Коа, чтобы обработать ответ

 Далее мы рассмотрим лучшие практики для создания Koa middleware.

## Middleware Лучшие практики

  Этот раздел охватывает лучших практик разработка middleware, таких как middleware
  принятие опций, именование middleware для отладки, и другие.

### Middleware опции

  При создании public middleware полезно соответствовать конвенции
  обертывание промежуточного программного обеспечения в функцию, которая принимает параметры,
  позволяя пользователям расширять функциональность. 
  Даже если ваше промежуточное ПО принимает _no_ опций, это все еще хорошая идея, чтобы держать вещи единообразно.

  Здесь наше искусственное промежуточное программное обеспечение `logger` принимает строку `format` для настройки,
  и возвращает само промежуточное ПО:

```js
function logger(format) {
  format = format || ':method ":url"';

  return async function (ctx, next) {
    const str = format
      .replace(':method', ctx.method)
      .replace(':url', ctx.url);

    console.log(str);

    await next();
  };
}

app.use(logger());
app.use(logger(':method :url'));
```

### Именнованное middleware

  Именование промежуточного п.о. не является обязательным, 
  однако для отладки полезно назначить имя.

```js
function logger(format) {
  return async function logger(ctx, next) {

  };
}
```

### Объединение нескольких middleware с koa-compose

  Иногда вы хотите "compose" нескольких middleware в один middleware для легкого повторного использования или экспорта. Вы можете использовать [koa-compose](https://github.com/koajs/compose)

```js
const compose = require('koa-compose');

async function random(ctx, next) {
  if ('/random' == ctx.path) {
    ctx.body = Math.floor(Math.random() * 10);
  } else {
    await next();
  }
};

async function backwards(ctx, next) {
  if ('/backwards' == ctx.path) {
    ctx.body = 'sdrawkcab';
  } else {
    await next();
  }
}

async function pi(ctx, next) {
  if ('/pi' == ctx.path) {
    ctx.body = String(Math.PI);
  } else {
    await next();
  }
}

const all = compose([random, backwards, pi]);

app.use(all);
```

### Ответное промежуточное ПО

  Промежуточное программное обеспечение, которое решает ответить на запрос и хочет обойти нижестоящее промежуточное ПО, может просто опустите `next ()`. Как правило, это будет в маршрутизации middleware, но это может быть выполнено
  Любым. Например, следующее ответит «два», однако все три будут выполнены, давая
  нижестоящему «три» промежуточного программного обеспечения шанс манипулировать ответом.

```js
app.use(async function (ctx, next) {
  console.log('>> one');
  await next();
  console.log('<< one');
});

app.use(async function (ctx, next) {
  console.log('>> two');
  ctx.body = 'two';
  await next();
  console.log('<< two');
});

app.use(async function (ctx, next) {
  console.log('>> three');
  await next();
  console.log('<< three');
});
```

  Следующая конфигурация пропускает `next()` во втором middleware, и все равно ответит
  с "two", Однако третий (и любой другой нижестоящий middleware) будут игнорироваться:

```js
app.use(async function (ctx, next) {
  console.log('>> one');
  await next();
  console.log('<< one');
});

app.use(async function (ctx, next) {
  console.log('>> two');
  ctx.body = 'two';
  console.log('<< two');
});

app.use(async function (ctx, next) {
  console.log('>> three');
  await next();
  console.log('<< three');
});
```

  Когда самый дальний downstream middleware исполняет `next();`, это действительно уступает
  функция, позволяющая промежуточному программному обеспечению правильно составлять в любом месте стека.

## Асинхронные операции

  Async function и promise образует фундамент Коа, позволяющий
  вам написать последовательный неблокирующий код. Например, это промежуточное ПО считывает имена файлов из `./docs`,
  а затем читает содержимое каждого markdown параллельный файл перед назначением Body для совместного результата.


```js
const fs = require('mz/fs');

app.use(async function (ctx, next) {
  const paths = await fs.readdir('docs');
  const files = await Promise.all(paths.map(path => fs.readFile(`docs/${path}`, 'utf8')));

  ctx.type = 'markdown';
  ctx.body = files.join('');
});
```

## Отладка Koa

  Koa along with many of the libraries it's built with support the __DEBUG__ environment variable from [debug](https://github.com/visionmedia/debug) which provides simple conditional logging.

  For example
  to see all Koa-specific debugging information just pass `DEBUG=koa*` and upon boot you'll see the list of middleware used, among other things.

```
$ DEBUG=koa* node --harmony examples/simple
  koa:application use responseTime +0ms
  koa:application use logger +4ms
  koa:application use contentLength +0ms
  koa:application use notfound +0ms
  koa:application use response +0ms
  koa:application listen +0ms
```

  Since JavaScript does not allow defining function names at
  runtime, you can also set a middleware's name as `._name`.
  This is useful when you don't have control of a middleware's name.
  For example:

```js
const path = require('path');
const serve = require('koa-static');

const publicFiles = serve(path.join(__dirname, 'public'));
publicFiles._name = 'static /public';

app.use(publicFiles);
```

  Now, instead of just seeing "serve" when debugging, you will see:

```
  koa:application use static /public +0ms
```
