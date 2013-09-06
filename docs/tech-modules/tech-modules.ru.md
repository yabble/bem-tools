# Модули технологий

## API

Смотрите документацию в исходных файлах [lib/tech/v1.js](https://github.com/bem/bem-tools/blob/master/lib/tech/v1.js), [lib/tech/v2.js](https://github.com/bem/bem-tools/blob/master/lib/tech/v2.js).

## Создание модуля технологии

Существует несколько способов написания модулей технологии.

Во всех описанных ниже способах из методов можно обратиться к объекту технологии через `this`,
а через `this.__base(...)` можно вызвать метод одного из базовых классов. К классу технологии
можно обратиться через `this.__class`. Всё это является следствием использования модуля
[inherit](https://github.com/dfilatov/node-inherit) для органиазации наследования.

### Очень простой способ

Способ заключается в том, что вы создаёте обычный CommonJS модуль, из
которого экспортируете несколько функций, которые перекроют методы базового
класса `Tech` из модуля [lib/tech.js](https://github.com/bem/bem-tools/blob/master/lib/tech.js).

```js
exports.getCreateResult = function(...) {
    // ваш код
};
```

Вы так же можете сгруппироать все методы в объекте `techMixin`. Это рекомендованный способ.

```js
exports.techMixin = {

    getCreateResult: function(...) {
        // ваш код
    }

};
```

### Простой способ

В простом способе к экспортируемым функциям добавляется переменная `baseTechPath`, в которой
содержится абсолютный путь до расширяемого модуля технологии.

```js
var BEM = require('bem');

exports.baseTechPath = BEM.require.resolve('./techs/css');
```

Так же вы можете организовать контекстное наследование, используя переменную `baseTechName`.
В этом случае базовый класс будет выбран в зависимости от уровня переопределения, на котором
будет использован модуль технологии.

```js
exports.baseTechName = 'css';
```

В этом примере новая технология будет расширять технологию `css`, заданную на уровне переопределения
в файле `.bem/level.js`.

### Для продвинутых

Если вам нужен полный контроль, вы можете создать модуль, экспортирующий готовый класс технологии `Tech`.

```js
var INHERIT = require('inherit'),
    BaseTech = require('bem/lib/tech').Tech;

exports.Tech = INHERIT(BaseTech, {

    create: function(prefix, vars, force) {
        // do some creation work
    },

    build: function(prefixes, outputDir, outputName) {
        // organize own build process
    }

});
```

Если в качестве базовой технологии вы хотите использовать одну из существующих технологий,
написанных в простом стиле, воспользуйтесь функцией `getTechClass()` для получения класса
этой технологии. Мы рекомендуем всегда использовать `getTechClass()`, чтобы не зависеть
от реализации базовой технологии.

```js
var INHERIT = require('inherit'),
    BEM = require('bem'),
    BaseTech = BEM.getTechClass(require.resolve('path/to/tech/module'));

exports.Tech = INHERIT(BaseTech, {

    // ваш код

});
```

## API технологий v2

В версии bem-tools 0.6.4 появилось новое API для написания технологий, которое позволяет сделать сборку
bem make/server быстрее. Прирост скорости зависит от проекта и может быть от нескольких процентов до
десятка раз. Чтобы ускорить сборку вашего проекта, необходимо использовать модули технологий новой версии.

Пример project-stub, использующий v2 на [github](https://github.com/bem/project-stub/tree/v2).

Новые технологии немного отличаются от старых по API. При наследовании технологий все модули в цепочке
должны быть одной версии (старые не должны перемешиваться с новыми).

Для перехода на новые технологии нужно в файле `.bem/level.js` бандлов задекларировать их, например:
```js
exports.getTechs = function() {

    return {
        'bemjson.js'     : '',
        'js'             : 'v2/js-i',
        'bemdecl.js'     : 'v2/bemdecl.js',
        'deps.js'        : 'v2/deps.js',
        'i18n'           : '../bem-bl/blocks-common/i-bem/bem/techs/v2/i18n.js'),
        'i18n.js'        : '../bem-bl/blocks-common/i-bem/bem/techs/v2/i18n.js.js'),
        'css'            : 'v2/css'
    };
};
```
Старые модули с новой версией bem-tools будут работать без изменений скорости.
Отличие от того, как было со старыми технологиями, только в префиксе `v2/`. Это касается технологий,
которые идут в составе ``bem-tools``, и тех, которые входят в `bem-bl`.

Версия bem-bl, в которой уже есть все нужное для использования v2 находится [здесь](https://github.com/bem/bem-bl/tree/0.3).

Встречаются проекты, на которых технологии по умолчанию не прописаны в уровнях. В этом случае будут
использоваться старые версии. Чтобы использовать новые, их нужно прописать явно, как показано в примере выше.

При использовании новых технологий дополнительное ускорение можно получить за счет кеширования уровней
переопределения с блоками.

Если вы работаете над проектом, в котором подключается `bem-bl` (или другая библиотека блоков), блоки которой
вы не меняете, а правите блоки на других уровнях, то сборку можно настроить таким образом, чтобы `bem-bl`
просканировалась единожды, и при последующих сборках использовался кеш с диска.

Это можно сделать с помощью следующего кода в `.bem/make.js`
```js
MAKE.decl('Arch', {
    getLevelCachePolicy: function() {
        return {
                cache: false,
                except: ['bem-bl']
        }
    }

});
```

Здесь `cache:false` говорит, что по умолчанию кеш уровней выключен.
А `except` — массив путей уровней (или директорий, содержащих уровни), для которых будет действовать исключение,
т.е. в данном случае кеш будет включен.

Кеш регенерируется, если сборка запущена с опцией `--force`.

### Изменения в API

#### Настройка своей технологии для использования нового API.
Чтобы ваша технология использвала новый API, нужно экспоритровать свойство API_VER:
```js
exports.API_VER = 2;

exports.techMixin = {

...

};
```

#### Настройка расширений файлов (суффиксов), с которыми работает технология.

В старых технологиях для указания суффиксов, которые и из которых технология будет строить результат,
использовались методы: `getSuffixes()`, `getBuildSuffixes()`. В новых можно использовать те же методы,
но для большей гибкости и простоты понимания лучше использовать `getBuildSuffixesMap()`.

```js
{
    getBuildSuffixesMap: function() {
        return {
            'ie.css': ['ie.css', 'ie.hover.css'];
        }
    }
}
```

В этом примере мы говорим, что будем собирать файл `ie.css` из файлов `ie.css` и `ie.hover.css`.
Ключей в возвращаемом объекте может быть больше одного, если технология собирает несколько файлов.

#### Изменения сигнатур методов в базовом классе технологии.

| v1        | v2           |
| ------------- |-------------|
|buildByDecl(decl, levels, output)|buildByDecl(decl, levels, output, opts)|
|getBuildResult(prefixes, suffix, outputDir, outputName)|getBuildResult(files, suffix, output, opts)|
|getBuildResults(prefixes, outputDir, outputName)|getBuildResults(decl, levels, output, opts)|
|getBuildPrefixes(decl, levels)|:x:|
|build(prefixes, outputDir, outputName)|:x:|
|filterPrefixes(prefixes, suffixes)|:x:|
|:x:|getBuildPaths(decl, levels)|
|:x:|saveLastUsedData(file, data)|
|:x:|getLastUsedData(file)|

 * Во всех методах, где встречается аргумент `opts` — это хэш параметров, которые были переданы в `bem build`.
 В него же можно добавлять свои вспомогательные параметры.
 * Вместо пары `outputDir`, `outputName` передается один аргумент `output`, содержащий путь к файлу (без суффикса),
 который (-ые) будет (-ут) создаваться.
 * Вместо аргумента `prefixes`, который содержит пути к потенциально существующим на уровнях файлам,
 из которых будет происходить билд, передается аргумент `files`. Это массив файлов, которые существуют на уровнях
 и имеют суффиксы, подходящие для собираемой технологии. Элемент массива - это объект со свойствами:
   * file — имя файла.
   * absPath — абсолютный путь до файла.
   * lastUpdated — дата модификации файла.
   * suffix — суффикс файла.
 * `getBuildPaths()` по переданной декларации (`decl`) и списку уровней (`levels`) возвращает список существующих
 на них файлов, попадающих под декларацию. Список представлен в виде хэша, в котором файлы сгруппированы по суффиксу
 технологии. Например:

```js
{
    css: [{...}, {...}, {...}],
    js: [{...}, {...}],
    bemhtml: [{...}, {...}, {...}, {...}]
}
```
  * `saveLastUsedData(file, data)`/`getLastUsedData(file)` сохраняет/загружает список файлов, из которых технология
  строила файл (`file`) в последний раз. Используется для валидации: нужно строить файл, или же он уже существует,
  и был построен из тех же файлов, из которых может быть построен сейчас.

#### Стандартный ход выполнения методов v2 технологии

![схема](http://img-fotki.yandex.ru/get/9259/127846884.247/0_b0604_843e6646_XXL.png)

Точкой входа является метод `buildByDecl`. Он вызывает `getBuildResults`, результат работы которого - это хэш,
где ключ - суффикс файла, а значение - массив строк контента. Например, для технологии `i18n.js` он может выглядеть так:
```js
{
 'en.js': ['...', '...', ...],
 'ru.js': ['...', '...', '...', ...],
 'tr.js': ['...', '...', ...]
}
```

Этот хэш передается в `storeBuildResults`, который сохраняет контент в файлы.

Чтобы построить хэш, `getBuildResults` получает список файлов (через вызов `getBuildPaths`), которые есть на
используемых уровнях переопределения, и которые подходят для собираемой технологии. То есть суффиксы которых
соответствуют тому, что прописано в `getBuildSuffixesMap` технологии. Дальше для каждого суффикса (в случае
с `i18n.js` это `en.js`, `ru.js` и т.д) вызывается `getBuildResult`. Ему передается список файлов, отфильтрованный
по конкретному суффиксу.

Каждый путь файла обрабатывается `getBuildResultChunk`, где на выходе получается строка-контент.

В зависимости от технологии это может быть просто полученный путь файла, обернутый в директиву подключения,
или прочитанный с диска контент этого файла.

#### Валидация файлов при последующих сборках.
При повторных сборках часто возникает ситуация, когда собираемые файлы уже присутствуют в результирующем 
каталоге сборки. В таких ситуациях нужна валидация файлов -- определение, нужно ли пересобрать файл, 
или тот что уже есть на диске актуален.

В технологиях версии 1 проверкой валидности собираемого файла занималась команда bem make, точнее - код в BemBuildNode. 
В API v2 эта задача переложена на модули технологий. Модуль технологии может самостоятельно проверить валидность собираемых 
ей файлов, опираясь на знания об исходных и результирующих суффиксах файлов технологии. В случае если модуль не осуществляет 
валидацию результирующий файл всегда будет пересобираться.

В большинстве случаев для валидации достаточно логики, заложенной в базовую технологию v2. Если ваш модуль технологии 
производит сборку файлов на основе суффиксов из `getBuildSuffixesMap()` и не подмешивает в результирующий файл сторонний контент, 
то, вероятнее всего, писать специальную логику для валидации нет необходимости.

В базовой технологии v2 логика валидации реализована в методе `getBuildResults(decl, levels, output, opts)`. Чтобы определить, 
нужно ли вызывать `getBuildResult(files, suffix, output, opts)` (т.е. непосредственно собрать файл с данным build-суффиксом), 
метод получает список файлов, которые должны попасть в сборку. Затем метод проверяет возвращаемое значение метода 
`validate(file, filteredFiles, opts)`. При значении true файл считается валидным и не пересобирается.

Метод `validate(file, filteredFiles, opts)` принимает следующие параметры:  
{String} `file` - абсолютный путь собираемого файла.  
{Array} `filteredFiles` - список файлов, которые должны попасть в сборку.  
{Object} `opts` - хеш опциональных параметров, пробрасывается аргумент `opts`, приходящий в `getBuildResults(decl, levels, output, opts)`.  
Если в opts содержится ключ `force` с не `false` значением, метод `validate(file, filteredFiles, opts)` вернет `false`.  
Метод `validate(file, filteredFiles, opts)` загружает из кеша (.bem/cache) список файлов, из которых в последний раз собирался 
результирующий файл и сравнивает его со значением параметра `filteredFiles`. Если отличий нет, значит результирующий файл валиден.

Для получения списка файлов попавших в предыдущую сборку используется метод `getLastUsedData(file)`, для сохранения в кеш - 
`saveLastUsedData(file, data)`. Параметр `file` - это абсолютный путь до файла, кеш которого нужно загрузить или сохранить, 
`data` - объект со списком файлов, который запишется в кеш в виде JSON.

### Примеры модулей технологий

 * [bem-tools/lib/techs/](https://github.com/bem/bem-tools/tree/master/lib/techs)
 * [bem-bl/blocks-common/i-bem/bem/techs/](https://github.com/bem/bem-bl/tree/master/blocks-common/i-bem/bem/techs)