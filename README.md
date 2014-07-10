fist [![NPM version](https://badge.fury.io/js/fist.svg)](http://badge.fury.io/js/fist) [![Build Status](https://travis-ci.org/fistlabs/fist.png?branch=master)](https://travis-ci.org/fistlabs/fist) [![Dependency Status](https://david-dm.org/fistlabs/fist.svg)](https://david-dm.org/fistlabs/fist) [![devDependency Status](https://david-dm.org/fistlabs/fist/dev-status.svg)](https://david-dm.org/fistlabs/fist#info=devDependencies)
=========
```Fist``` - это nodejs-фреймворк для написания серверных приложений. ```Fist``` предлагает архитектуру, поддержка которой одинаково проста как для простых так и для сложных web-серверов.
```js
var fist = require('fist');
var app = fist();

app.unit({
    path: 'time.appstart', 
    data: new Date()
});

app.unit({
    path: 'time.uptime',
    deps: ['time.appstart'],
    data: function (track, ctx) {
        return new Date() - ctx.getRes('time.appstart')
    }
});

app.unit({
    path: 'uptimeController', 
    deps: ['time.uptime'], 
    data: function (track, ctx) {
        return track.send(ctx.getRes('time.uptime'));
    }
});

app.route('/uptime/', 'uptimeController');

app.listen(1337);
```

Приложение фреймворка представляет собой плагинизируемый веб-сервер, состоящий из множества взаимосвязанных, зависящих друг от друга узлов, один из которых может обработать поступивший запрос, принятый роутером.

Приходящий в сервер запрос матчится первый подходящий локейшен, описанный в роутере. Каждый локейшн должен быть ассоциирован с узлом, операция разрешения которого запускается при успешном матчинге. Как правило, узел, ассоциированный с локейшеном является контролером и может выполнить ответ приложения. Но если он этого не сделает, то матчинг продолжится и операция повторится уже на другом узле, до тех пор пока один из них не выполнит ответ.

```js
//  при любом запросе проверять права на просмотр страницы
//  может отправить например 403 если прав нет
app.route('/ e', {
    // название роута
    name: 'checkRights',
    //  название узла с которым роут ассоциирован
    unit: 'rightChecker'
});

//  отображает главную страницу если есть права
app.route('/', {
    name: 'indexPage',
    unit: 'indexPageController'
});

//  настройки сайта
app.route('/settings/', {
    name: 'settingsPage',
    unit: 'settingsPageController'
});

//  далее идет декларация узлов
//  ***
```
Узлом приложения является инстанс класса ```fist/unit/_unit```. Каждый узел должен иметь некоторый идентификатор и имплементирать метод ```data```, который отвечает за разрешение узла. Узлы могут зависеть друг от друга, что должно быть указано в декларации. Это значит что до того как выполнится текущий узел, будут выполнены его зависимости, результаты которых будут доступны в методе ```data```.

```js
app.unit({
    //  идентификатор узла
    path: 'content.news',
    //  тело узла
    data: function () {
        var defer = vow.defer();
        doRequestForNews(function (err, res) {
            if ( err ) {
                defer.reject(err);
            } else {
                defer.resolve(res);
            }
        });
        return defer.promise();
    }
});

app.unit({
    path: 'indexPage',
    //  Зависимости узла
    deps: ['content.news'],
    data: function (track, ctx) {
        return track.send(doTemplate(ctx.getRes('content.news')));
    }
});
```
В каждый узел при его выполнении передается ```track``` и ```ctx```.
Объект ```track``` создается на каждый запрос и аггрегирует возможности ```request``` и ```response```. 
Объект ```ctx``` - это контекст вызова узла, в нем содержатся результаты зависимостей узла.

Результатом разрешения узла является возвращенное из него значение или брошенное исключение. Если преполагается асинхронное выполнение узла, то можно возвратить ```promise```. Узел считается разрешенным когда будет разрешено возвращенное из него значение.


```fist``` работает на nodejs >= 0.10

#Приложение
##API
###```fist([params])```
Инстанцирует приложение
```js
var fist = require('fist');
var configs = require('./configs');
//  Инстанцирую приложение
var app = fist(configs);
```

Несколько параметров поддерживаются из коробки:
####```params.units```
Один или несколько glob-шаблонов, указывающих файлы-декларации узлов.
Таким образом можно никогда не писать app.unit, так как каждый узел приложения будет описан в отдельном файле, и будет продекларирован автоматически

```js
var app = fist({ 
    units: [
        './controllers/*.js',
        './models/*.js'
    ]
});
```
####```params.routes```
Массив деклараций маршрутов приложения.
С момощью этого параметра можно не связывать вручную маршруты с узлами, достаточно лишь передать в конфигурацию массив деклараций роутов.

```js
var app = fist({
    routes: [
        {
            pattern: '/about/',
            name: 'aboutPage',
            unit: 'aboutPageController'
        },
        {
            pattern: '/',
            name: 'indexPage',
            unit: 'indexPageController'
        }
    ]
});
```

###```app.listen()```
Запускает сервер приложения
```js
app.listen(1337);
```
###```app.plug(plugin...)```
Добавляет в приложение плагин.
```js
app.plug(function (done) {
    this.myFeature = 42;
    done();
});
```
###```app.route(pattern, data)```
Линкует роут с узлом.
```js
app.route('/', {
    name: 'index',
    unit: 'indexController'
});
```
###```app.unit(decl)```
Добавляет в приложение функциональный узел
```js
app.unit({
    path: 'indexController',
    data: function () {
        doSomething()
    }
});
```
###```app.ready([force])```
Запускает инициализацию приложения и возвращает ```promise```, статус которого является статусом инициализации приложения.
Если инициализация уже была выполнена то снова она выполнена не будет. Но можно передать параметр ```force```, тогда инициализация будет запущена в принудительном порядке и все узлы и плагины будут проинстанцированы снова. Это действие выполняется автоматически при старте сервера приложения.
###```app.params```
Все параметры приложения, которые были переданы в конструктор
###```app.renderers```
Объект, ключами которого являются имена шаблонов, а значениями - функции, которые вызываются в контексте ```track```. Используется в методе ```track.render``` для шаблонизации данных.

###```app._createCache(params)```
_protected_

Этот метод вызывается при инстанцировании. Возвращаемый объект реализует механизм кэширования и должен имплементировать некоторый интерфейс
####```cache.set(key, value, maxAge, callback)```
####```cache.get(key, callback)```

##События
Приложение обладает свойствами ```EventEmitter```, поэтому на нем можно слушать некоторые автоматические события.
###```sys:ready```
Приложение готово обрабатывать запросы
###```sys:eready```
Ошибка инициализации приложения
###```ctx:pending```
Начинается разрешение узла
###```ctx:accept```
Узел разрешен без ошибки
###```ctx:reject```
Узeл разрешен с ошибкой
###```ctx:notify```
Узел послал уведомление
###```sys:request```
В приложение поступил запрос
###```sys:match```
Роутер сматчил запрос
###```sys:ematch```
Роутер не нашел подходящего шаблона для запроса
###```sys:response```
Приложение выполнило ответ

#Плагины
Плагином приложения является обычная функция, которая вызывает резолвер по завершении своей работы. Плагины могут использоваться для конфигурирования и расширения возможностей приложения.
Когда запускается сервер, сначала выполняются плагины, и когда они все отработают приложение начинает отвечать на запросы. Пока приложение инициализируется, запросы откладываются на обработку после инициализации. Если хотя бы один плагин был разрешен с ошибкой, то приложение проинициализируется, но начнет отвечать  ```500 Internal Server Error```.
Для того чтобы отклонить выполнение плагина нужно передать любой аргумент в резолвер.
```js
app.plug(function (done) {
    doSomethingAsync(function (err) {
        if ( err ) {
            done(err);
        } else {
            done();
        }
    });
});
```
Если есть возможность устранить проблему, то по факту ее устранения можно перезапустить приложение вызвав ```app.ready(true)```
#Узлы
Узлом приложения является инкапсулированая логическая часть приложения, динамичность поведения которой зависит от контекста вызова и параметров запроса. Узлы декларируются методом ```app.unit```. Необходимо обязательно указать ```path``` узла и имплементировать метод ```data```. Узел может зависеть от результатов других узлов, тогда нужно указать массив ```deps``` с идентификаторами узлов.

```js
app.unit({
    path: 'fortyTwo',
    deps: ['someUnit'],
    data: function () {
        
        return 42;
    }
});
```
Объект передаваемый в метод ```app.unit``` является расширением прототипа узла. По умолчанию каждый узел наследует от ```fist/unit/_unit```, но можно наследовать от любого узла. Для этого необходимо указать ```base```, что является именем узла, от которого нужно унаследовать. В приложении могут быть абстрактные узлы, декларация которых не требуется, но от которых нужно унаследовать. Чтобы создать такой узел в его ```path``` первый символ должен быть не буквенный. Подобные узлы нет смысла добавлять в зависимости, потому что они не участвуют в операции разрешения.
```js

app.unit({
    path: '_model',
    data: function () {
        return doAuth();
    }
})
app.unit({
    base: '_model',
    path: 'users',
    data: function (track, ctx) {
        return this.__base(track, ctx).then(getUsers);
    }
});

```
###```unit.addDeps(deps)```
Добавляет зависимости в узел.
```js
app.unit({
    base: 'users',
    path: 'extendedUsers',
    __constructor: function (params) {
        this.__base(params);
        this.addDeps('userExtensions');
    },
    data: function (track, ctx) {
        return doSomethingWithUsers(this.__base(track, ctx));
    }
});
```

###```unit.params```
Параметры узла. Все узлы инстанцируются со всеми параметрами приложения.
```js
var config = {a: 42};
var app = new Framework(config);
app.unit({
    path: 'test',
    __constructor: function (params) {
        assert.deepEqual(params, config);
    }
});
```

###Кэширование
У узлов есть возможность кэширования результатов выполнения. Для этого нужно указать в декларации свойство ```_maxAge``` и имплементировать метод ```_getCacheKeyParts``` который должен вернуть массив строк - токенов ключа для кэширования.
```js
app.unit({
    path: 'newsPost',
    _maxAge: 500, //ms
    _getCacheKeyParts: function (track, ctx) {
        return [track.arg('blogId'), track.arg('postId')];
    },
    data: function () {
        return loadPost();
    }
})
```

Механизм кэширования можно [заменить](#app_createcacheparams) на свой.
#Track
Этот объект является контекстом запроса. В нем содержатся средства для чтения из ```request``` и для записи в ```response```.
###```track.arg(name[, only])```
Возвращает параметр запроса. Если передать только ```name```, то будет возвращено значение из ```pathname``` или из ```query``` если в ```pathname``` он не будет найден. Параметр ```only``` означает что значения из query не интересуют, и нужно искать параметр только в ```pathname```.
```js
app.route('/(<pageName>/)', {name: 'anyPage', unit: 'universalController'});
//  ***
//  GET /index/
track.arg('pageName'); // -> index
//  GET /?pageName=foo
track.arg('pageName'); // -> foo
track.arg('pageName', true); // -> undefined
```
###```track.header(name[, value])```
Устанавливает заголовок в ```response``` или читает его из ```request```
```js
//  Поставить шапку в ответ
track.header('Content-Type', 'text/html');

track.header('Cookie'); // -> name=value
```
###```track.cookie(name[, value[, opts]])```
Читает куку из ```request``` или ставит ее в ```response```
```js
track.cookie('name'); // -> value

//  Выставить куку
track.cookie('name', 'value', {
    path: '/'
});
```
###```track.send([status[, body]])```
Выполняет ответ приложения
```js
track.send(200, ':)');
```
Этот метод не выполняет непосредственной записи в ```response```, но возвращает специальный объект, разрешение узла которым игнорирует вызов зависимых узлов, и по завершении разрешения узла-контроллера запись в ```response``` произойдет автоматически
```js
var resolver = track.send();
//  записи в response не было, лишь был создан объект ответа приложения

return resolver;
```

###```track.body()```
Скачивает тело запроса и парсит его, возвращает ```promise```
```js
track.body().then(function (body) {

    //  в поле body.type содержится тип тела multipart/json/urlencoded/raw
    //  в поле body.input содержится само тело, во всех случаях кроме raw это ключ-значение,
    //  если тело - raw, то это будет Buffer
    //  Если тело - multipart, то можно обнаружит специальное поле files
});
```
###```track.redirect([status, ]url)```
Создает перенаправление на клиенте, возвращает объект ответа приложения
```js
return track.redirect(301, 'http://www.yandex.ru');
```
###```track.buidlPath(routeName[, opts])```
Создает ```url``` из шаблона запроса
```js
app.route('/(<pageName>/)', {name: 'anyPage', unit: 'universalController'});
//  ***
track.buildPath('anyPage', {pageName: 'test', x: 42}); // -> /test/?x=42
```

###```track.url```
Объект распаршенного ```url``` запроса
###```track.match```
Объект параметров запроса, сматчившихся на шаблон ```url```-а
###```track.route```
Имя маршрута, на который сматчился запрос
###```track.render([code, ]name[, arg...])```
Выполняет ответ приложения, предварительно отшаблонизировав данные по одному из заданных шаблонов.
Возвращает объект ответа приложения
```js
return track.render(404, 'not-found');
```

Как уже говорилось выше, объект ```track``` - это по сути, полиморфная обертка над ```request``` и ```response```, в которой собраны самые популярные возможности, но с помощью него можно сделать не все. Например, нельзя прочитать уже установленный заголовок в ```response``` или узнать ```statusCode```. Поэтому в ```track``` есть доступ к безопасным оберткам вокруг этих оригинальных объектов.

###```track.req```
Содержит API для работы c ```request```.
###```track.res```
Содержит API для работы с ```response```

#Контекст
Второй аргумент, который передается в тело узла - ```ctx```. Этот объект является контекстом вызова узла.

###```ctx.result```
Объект, содержащий результаты зависимостей, разрешенных без ошибки
###```ctx.errors```
Объект, содержащий результаты зависимостей, разрешенных с ошибкой
###```ctx.getRes(path)```
Возвращает вложенное в ```ctx.result``` значение
```js
//  нет гарантии что существует ctx.res.info.news.posts
var posts = ((ctx.result.info || {}).news || {}).posts; // так писать неудобно
// а так удобно и не страшны ReferenceError-ы
var posts = ctx.getRes('info.news.posts'); 
```
###```ctx.getErr(path)```
Аналогично ```ctx.getRes``` только работает с объектом ```ctx.errors```

###```ctx.trigger(eventName, data)```
Иногда нужно сообщить приложению о каком либо событии, но это не будет иметь смысла если не передать id запроса и какие-то контекстные данные, хотя бы имя узла, в котором было подожжено событие.
Этот метод поджигает событие на экземпляре вашего приложения, обернув переданные данные в контекстную обертку.
```js
app.on('my-event', function (event) {
    // id запроса
    assert.isString(event.trackId);
    //  узел в котором было подожжено событие
    assert.strictEqual(event.path, 'demo');
    //  время, прошедшее с момента создания контекста узла до триггера
    assert.isNumber(event.time);

    assert.strictEqual(event.data, 42);
});

app.unit({
    path: 'demo',
    data: function (track, ctx) {
        ctx.trigger('my-event', 42);
    }
});

```

#Роутер
Роутер является неотъемлемой частью фреймворка. Синтаксис шаблонов роутера можно найти [тут](//github.com/golyshevd/finger)
