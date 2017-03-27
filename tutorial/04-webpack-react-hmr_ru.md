# 04 - Webpack, React и Hot Module Replacement

Код для этой главы доступен [здесь](https://github.com/verekia/js-stack-walkthrough/tree/master/04-webpack-react-hmr).

## Webpack

> 💡 **[Webpack](https://webpack.js.org/)** - *сборщик модулей*. Он берет все возможные исходные файлы, обрабатывает их и собирает в один (обычно) JavaScript файл, называемый сборкой, и это будет единственный файл исполняемый на клиенте.

Давайте создадим какой-нибудь простой *hello world* и соберем его с помощью Webpack.

- В `src/shared/config.js` добавьте следующие константы:

```js
export const WDS_PORT = 7000

export const APP_CONTAINER_CLASS = 'js-app'
export const APP_CONTAINER_SELECTOR = `.${APP_CONTAINER_CLASS}`
```

- Создайте файл `src/client/index.js`, содержащий:

```js
import 'babel-polyfill'

import { APP_CONTAINER_SELECTOR } from '../shared/config'

document.querySelector(APP_CONTAINER_SELECTOR).innerHTML = '<h1>Hello Webpack!</h1>'
```

Если вы хотите использовать новейшие возможности ES в клиентском коде, такие как `Promise`, то вам нужно включить [Babel Polyfill](https://babeljs.io/docs/usage/polyfill/) до какого-либо другого кода в сборке.

- Запустите `yarn add babel-polyfill`

Если вы запустите ESLint на этом файле, он будет жаловаться, что `document` undefined.

- Добавьте раздел `env` в `.eslintrc.json`, чтобы позволить использование `window` и `document`:

```json
"env": {
  "browser": true,
  "jest": true
}
```

Хорошо, теперь нам нужно собрать это клиентское ES6 приложение в ES5 сборку.

- Создайте файл `webpack.config.babel.js` содержащий:

```js
// @flow

import path from 'path'

import { WDS_PORT } from './src/shared/config'
import { isProd } from './src/shared/util'

export default {
  entry: [
    './src/client',
  ],
  output: {
    filename: 'js/bundle.js',
    path: path.resolve(__dirname, 'dist'),
    publicPath: isProd ? '/static/' : `http://localhost:${WDS_PORT}/dist/`,
  },
  module: {
    rules: [
      { test: /\.(js|jsx)$/, use: 'babel-loader', exclude: /node_modules/ },
    ],
  },
  devtool: isProd ? false : 'source-map',
  resolve: {
    extensions: ['.js', '.jsx'],
  },
  devServer: {
    port: WDS_PORT,
  },
}
```

Этот файл используется для описания того, как должна быть устроена наша сборка: `entry` - стартовая точка нашего приложения, `output.filename` - имя генерируемой сборки, `output.path` и `output.publicPath` описывают путь до папки со сборкой и URL. Мы поместим сборку в папку  `dist`, которая будет содержать автоматически генерируемые вещи (в отличие от обитающих в `public` декларативных CSS, которые мы создавали до этого). В `module.rules` мы сообщаем Webpack к каким типам файлов применять какие обработчики. Здесь мы говорим, что хотим пропускать все `.js` и `.jsx` (для реакта) файлы через нечто, называемое `babel-loader`, за исключением того, что находится в `node_modules`. Мы также хотим *разрешать* (`resolve`) эти два расширения при `import` модулей (т.е. эти расширения можно будет опускать при импорте - прим. пер.)

**Примечание**: Расширение `.babel.js` сообщает Webpack применять трансформации Babel к данному конфигурационному файлу. 

`babel-loader` - это плагин для Webpack, транспилирующий код, так же как мы это делали с начала этого руководства. Единственное на данный момент отличие, что этот код исполняется в браузере а не на сервере.

- Запустите `yarn add --dev webpack webpack-dev-server babel-core babel-loader`

`babel-core` is a peer-dependency of `babel-loader`, so we installed it as well.
Мы установили также `babel-core`, поскольку это peer-dependency (требуемая зависимость) для `babel-loader`.

- Добавьте `/dist/` в `.gitignore`

### Обновление задач

В режиме разработки мы будем использовать `webpack-dev-server` чтобы пользоваться преимуществами Hot Module Reloading (позже в этой главе), а в продакшене мы просто используем `webpack`, чтобы сгенерировать сборку. В обоих случаях, флаг `--progress` будет полезен для вывода дополнительной информации когда Webpack компилирует файлы. В продакшене мы также передаем в `webpack` флаг `-p` для минификации кода и переменную `NODE_ENV` установленную в `production`.

Давайте обновим наши `scripts` чтобы реализовать это, а также улучшим некоторые другие задачи:

```json
"scripts": {
  "start": "yarn dev:start",
  "dev:start": "nodemon -e js,jsx --ignore lib --ignore dist --exec babel-node src/server",
  "dev:wds": "webpack-dev-server --progress",
  "prod:build": "rimraf lib dist && babel src -d lib --ignore .test.js && cross-env NODE_ENV=production webpack -p --progress",
  "prod:start": "cross-env NODE_ENV=production pm2 start lib/server && pm2 logs",
  "prod:stop": "pm2 delete server",
  "lint": "eslint src webpack.config.babel.js --ext .js,.jsx",
  "test": "yarn lint && flow && jest --coverage",
  "precommit": "yarn test",
  "prepush": "yarn test && yarn prod:build"
},
```

В `dev:start` мы явно указываем расширения для наблюдения: `.js` и `.jsx`, и добавляем `dist` в игнорируемые директории.

Мы создали отдельную задачу `lint` и добавили `webpack.config.babel.js` в список проверяемых файлов.

- Затем давайте создадим контейнер для нашего приложения в `src/server/render-app.js` и включим его в генерируемую сборку:

```js
// @flow

import { APP_CONTAINER_CLASS, STATIC_PATH, WDS_PORT } from '../shared/config'
import { isProd } from '../shared/util'

const renderApp = (title: string) =>
`<!doctype html>
<html>
  <head>
    <title>${title}</title>
    <link rel="stylesheet" href="${STATIC_PATH}/css/style.css">
  </head>
  <body>
    <div class="${APP_CONTAINER_CLASS}"></div>
    <script src="${isProd ? STATIC_PATH : `http://localhost:${WDS_PORT}/dist`}/js/bundle.js"></script>
  </body>
</html>
`

export default renderApp
```

В зависимости от того, какое у нас окружение, мы включаем сборку Webpack Dev Server либо продакшен сборку. Обратите внимание на *виртуальный* путь к сборке Webpack Dev Server: `dist/js/bundle.js`, который на самом деле не читается с жесткого диска в режиме разработки. Также необходимо задать для Webpack Dev Server порт отличный от основного веб порта.

- И наконец, в `src/server/index.js`, настройте сообщение от `console.log` таким образом:

```js
console.log(`Server running on port ${WEB_PORT} ${isProd ? '(production)' :
  '(development).\nKeep "yarn dev:wds" running in an other terminal'}.`)
```

Это даст другим разработчикам подсказку, что делать, если они просто пытаются запустить `yarn start` без Webpack Dev Server.

Хорошо, мы произвели много изменений, давайте посмотрим, все ли работает как ожидалось:

🏁 Запустите `yarn start` в терминале. Откройте другую вкладку или окошко с терминалом и запустите в ней `yarn dev:wds`. Как только Webpack Dev Server завершит генерацию сборки и sourcemap карт (вместе должно быть ~600kB файлов), и оба процесса повиснут в терминале, откройте `http://localhost:8000/` и вы должны увидеть "Hello Webpack!". Откройте консоль Chrome и на вкладке Source проверьте какие файлы включены. Вы должны увидеть только `static/css/style.css` под `localhost:8000/`, а все ваши исходные ES6 файлы должны располагаться в `webpack://./src`. Это значит, что sourcemap работают. Попробуйте изменить `Hello Webpack!` в `src/client/index.js` на любую другую строку с помощью редактора. Как только вы сохраните файл, вы должны увидеть в терминале, что Webpack Dev Server сгенерировал новую сборку, и вкладка Chrome автоматически обновилась.

- Завершите предыдущие процессы в терминалах с помощью Ctrl+C, затем запустите `yarn prod:build` и затем `yarn prod:start`. Откройте `http://localhost:8000/`, и вы по прежнему должны видеть "Hello Webpack!". На этот раз, во вкладке Source консоли Chrome под `localhost:8000/` должно быть `static/js/bundle.js`, но без исходников в `webpack://`. Кликните на `bundle.js` чтобы убедиться, что он минифицирован. Запустите `yarn prod:stop`.

Отличная работа, знаю, это было довольно плотно. Вы заслужили перерыв! Следующий раздел будет легче.

**Примечание**: Я бы рекомендовал открывать как минимум 3 терминала: один для сервера Express, один для Webpack Dev Server и один для Git, тестов и основных команд, таких как установка пакетов с помощью `yarn`. В идеале, нужно разделить окно терминала на несколько панелей, чтобы видеть их все.

## React

> 💡 **[React](https://facebook.github.io/react/)** - библиотека для построения пользовательских интерфейсов от Facebook. Она использует синтаксис **[JSX](https://facebook.github.io/react/docs/jsx-in-depth.html)** для представления HTML элементов и компонентов, сочетая его с мощью JavaScript.

В этой части мы будем генерировать некоторый текст с помощью React и JSX.

Для начала, давайте установим React и ReactDOM:

- Запустите `yarn add react react-dom`

Переименуйте файл `src/client/index.js` в `src/client/index.jsx` и напишите в нем следующий React код:

```js
// @flow

import 'babel-polyfill'

import React from 'react'
import ReactDOM from 'react-dom'

import App from './app'
import { APP_CONTAINER_SELECTOR } from '../shared/config'

ReactDOM.render(<App />, document.querySelector(APP_CONTAINER_SELECTOR))
```

- Создайте фпйл `src/client/app.jsx` содержащий:

```js
// @flow

import React from 'react'

const App = () => <h1>Hello React!</h1>

export default App
```

Поскольку мы тут используем синтаксис JSX, нам нужно чтобы Babel трансформировал его с помощью пресета `babel-preset-react`. Заодно, мы добавим плагин для Babel `flow-react-proptypes`, который автоматически генерирует PropTypes из аннотаций Flow для React компонентов.


- Запустите `yarn add --dev babel-preset-react babel-plugin-flow-react-proptypes` и отредактируйте файл `.babelrc` так:

```json
{
  "presets": [
    "env",
    "flow",
    "react"
  ],
  "plugins": [
    "flow-react-proptypes"
  ]
}
```

🏁 Запустите `yarn start` и `yarn dev:wds`, откройте `http://localhost:8000`. Вы должны увидеть "Hello React!".

Теперь попробуйте изменить текст в `src/client/app.jsx` на какой-нибудь другой. Webpack Dev Server должен автоматически перезагрузить страницу, что довольно изящно, но мы собираемся сделать даже еще лучше.

## Hot Module Replacement

> 💡 **[Hot Module Replacement](https://webpack.js.org/concepts/hot-module-replacement/)** (*HMR*) - мощная способность Webpack заменять модули на лету без перезагрузки целой страницы.

Чтобы заставить HMR работать с React, нам потребуется немного поднастроить.

- Запустите `yarn add react-hot-loader@next`

- Отредактируйте `webpack.config.babel.js` так:

```js
import webpack from 'webpack'
// [...]
entry: [
  'react-hot-loader/patch',
  './src/client',
],
// [...]
devServer: {
  port: WDS_PORT,
  hot: true,
},
plugins: [
  new webpack.optimize.OccurrenceOrderPlugin(),
  new webpack.HotModuleReplacementPlugin(),
  new webpack.NamedModulesPlugin(),
  new webpack.NoEmitOnErrorsPlugin(),
],
```

- Отредактируйте файл `src/client/index.jsx`:

```js
// @flow

import 'babel-polyfill'

import React from 'react'
import ReactDOM from 'react-dom'
import { AppContainer } from 'react-hot-loader'

import App from './app'
import { APP_CONTAINER_SELECTOR } from '../shared/config'

const rootEl = document.querySelector(APP_CONTAINER_SELECTOR)

const wrapApp = AppComponent =>
  <AppContainer>
    <AppComponent />
  </AppContainer>

ReactDOM.render(wrapApp(App), rootEl)

if (module.hot) {
  // flow-disable-next-line
  module.hot.accept('./app', () => {
    // eslint-disable-next-line global-require
    const NextApp = require('./app').default
    ReactDOM.render(wrapApp(NextApp), rootEl)
  })
}
```

Нам нужно, чтобы `App` был дочерним по отношению к `AppContainer` из `react-hot-loader`, и также нам требуется добавить `require` для получения следующей версии `App` при hot-reloading (горячей перезагрузке). Чтобы сделать этот процесс ясным и следовать принципу DRY, мы создали небольшую функию `wrapApp`, которую используем в обоих местах, где требуется генерировать `App`. Вы можете перенести `eslint-disable global-require` в начало файла чтобы сделать его более читабельным.

🏁 Перезапустите процесс `yarn dev:wds`, если онивсе еще запущен. Откройте `localhost:8000`. В консоли, вы должы увидеть некоторые логи об HMR. Возьмите и измените что-нибудь в `src/client/app.jsx`, и ваши изменения будут отражены в браузере через несколько секунд без полной перезагрузки страницы.

Следующий раздел: [05 - Redux, Immutable, Fetch](05-redux-immutable-fetch.md#readme)

Назад в [предыдущий раздел](03-express-nodemon-pm2_ru.md#readme) или [содержание](https://github.com/verekia/js-stack-from-scratch#table-of-contents).