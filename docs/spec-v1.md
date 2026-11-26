# Спецификация банка: язык schema v1

Спецификация — это JSON-документ, который исполняет движок приложения. Кода в
ней нет: только данные, шаблоны и вызовы фиксированного набора функций. Хосты,
пути, заголовки, имена полей и ветвления описаны в спецификации, а не в
приложении.

Это публичный справочник формата. Движок и его исходники приватные. Если для
вашего банка нужен примитив, которого здесь нет, напишите в issue: примитив
добавят в движок обобщённо и с тестами. Обходной путь в спецификации не нужен.

Полный рабочий пример — [`examples/bank1.spec.json`](examples/bank1.spec.json)
(демо-банк с живым сервером, см. [`examples/README.md`](examples/README.md)).

## 1. Верхний уровень

| Поле | Обяз. | Смысл |
|---|---|---|
| `schema` | да | Версия формата. Сейчас `1`. |
| `id` | да | Стабильный id банка: латиница, цифры, дефис. Используется для секретов и каталога. |
| `version` | да | semver спецификации, независимый от приложения. Первая версия — `1.0.0`. |
| `minEngine` | нет | Минимальная версия движка. **Не указывайте сами** — её выставит сопровождающий при ревью. |
| `meta` | да | `name`, `shortName?`, `color?`, `sortOrder?`, `status?` (`stable` / `beta` / `blocked`), `statusReason?`, `lastVerifiedAt?`, `legal?`. |
| `baseUrl` | да | `https://…`; его хост обязан входить в `hosts`. |
| `hosts` | да | Allowlist хостов. Запрос на хост вне списка отклоняется движком. |
| `capabilities` | нет | `cards`, `transactions`, `cashback`, `cashbackSelection`; `transferBySbp` зарезервировано (переводы движок не исполняет). `accounts` подразумевается всегда. |
| `const` | нет | Константы, доступные в выражениях как `$const.*`. |
| `auth` | да | Граф авторизации: `start` + `steps`. |
| `read` | нет | Коллекции `accounts`, `cards`, `transactions`. |
| `check` | нет | Лёгкая проверка живости сессии. |
| `refresh` | нет | Тихий рефреш сессии. |
| `cashback` | нет | Программы кэшбэка и выбор категорий. |
| `errors` | нет | Правила «ответ банка → `BankError.kind`». |

Пример шапки:

```json
{
  "schema": 1,
  "id": "example",
  "version": "1.0.0",
  "meta": {
    "name": "Пример Банк",
    "shortName": "Пример",
    "color": "#0057FF",
    "sortOrder": 10,
    "status": "beta",
    "statusReason": "Проверено на одном аккаунте, кэшбэк не разобран.",
    "lastVerifiedAt": "2026-11-23"
  },
  "baseUrl": "https://api.example-bank.ru",
  "hosts": ["api.example-bank.ru", "sso.example-bank.ru"],
  "capabilities": ["cards", "transactions"]
}
```

Правила:

- `baseUrl` и все `request.url`/`request.baseUrl` должны быть `https://` и
  указывать на хост из `hosts`. Редирект на незадекларированный хост — ошибка.
- `id` не меняется никогда: он используется в именах секретов и в каталоге.
- `meta.status` на время разработки — `beta`; `stable` ставит сопровождающий
  после живого прогона.

## 2. Авторизация

```json
"auth": {
  "start": "phone",
  "steps": {
    "phone": {
      "emit": { "kind": "phone", "title": "Телефон", "field": "phone" },
      "next": "phone-submit"
    },
    "phone-submit": {
      "request": {
        "method": "POST",
        "path": "/auth/start",
        "json": {
          "phone": "${digits10($input.phone)}",
          "captcha": "${$input.captcha}"
        }
      },
      "extract": { "challengeId": "$.challengeId" },
      "on": [
        { "when": { "status": 200 }, "goto": "otp" },
        { "when": { "status": 401, "path": "$.type", "eq": "captcha_required" }, "goto": "captcha" },
        { "when": {}, "error": { "kind": "unavailable", "message": "auth/start failed" } }
      ]
    },
    "captcha": {
      "emit": { "kind": "captcha", "title": "Код с картинки", "field": "captcha" },
      "next": "phone-submit"
    },
    "otp": {
      "emit": { "kind": "otp", "title": "Код из СМС", "field": "code", "length": 4, "secret": true },
      "next": "verify"
    },
    "verify": {
      "request": {
        "method": "POST",
        "path": "/auth/verify",
        "json": { "challengeId": "${$state.challengeId}", "code": "${$input.code}" }
      },
      "store": {
        "session": {
          "accessToken": "${$.accessToken}",
          "refreshToken": "${$.refreshToken}",
          "expiresAt": "${isoAfter($.expiresIn)}"
        }
      },
      "on": [
        { "when": { "status": 200 }, "goto": "done" },
        { "when": { "status": 401 }, "error": { "kind": "invalid-credentials" } },
        { "when": {}, "error": { "kind": "unavailable", "message": "verify failed" } }
      ]
    },
    "done": { "session": true }
  },
  "restartOn": ["phone"]
}
```

`auth.start` — id первого шага. `steps` — словарь шагов. Шаг делает одно из
четырёх:

1. **Вопрос** — есть `emit`, нет `request`. Движок возвращает шаг в UI и ждёт
   ответ. Вопрос без запроса обязан объявить `next` — куда идти с ответом.
2. **Запрос** — есть `request`. Выполняется сразу.
3. **Конец** — `"session": true`. Накопленные `$secret` пишутся в хранилище
   секретов, вызывающему возвращается сессия.
4. **Служебный шаг** — нет ни `request`, ни `emit`. Выполняется сразу, корнем
   `$` служит накопленный `$input`. Нужен, когда данные приходят не из HTTP,
   а от приложения (результат WebView-входа).

Порядок внутри шага: `set` → `emit`/`request` → `extract`/`store` шага → первая
подходящая ветка `on` (со своими `extract`/`store`) → `next`, если ни одна не
совпала.

### 2.1. `emit` — что показать человеку

| Поле | Обяз. | Смысл |
|---|---|---|
| `kind` | да | `phone`, `otp`, `password`, `security-question`, `passcode`, `confirm`, `notice`, `stage-choice`, `browser`, `captcha`. |
| `title` | да | Заголовок экрана. |
| `field` | да | Ключ, под которым ответ попадёт в `$input`. |
| `hint` | нет | Подсказка (например, демо-код). |
| `secret` | нет | Скрывать ввод (коды, пароли). |
| `length` | нет | Длина кода. |
| `choices` | нет | Для `stage-choice`: `[{ "id", "label" }]`. |
| `payload` | нет | Данные для UI: URL входа, ключ капчи. Шаблон, не секрет. |
| `capture` | нет | Для `browser`: что забрать со страницы банка (см. §2.4). |

UI рисует экран по `kind`, а не по банку.

### 2.2. `extract` и `store`

- `extract` — словарь «имя → выражение». Значения попадают в `$state` и видны
  следующим шагам. Секреты сюда не кладём.
- `store` — словарь «имя → значение». Попадает в `$secret`. Материал сессии
  складывается в один ключ `session`, чтобы движок мог его сохранить и продлить.
- **Строки в `store` хранятся буквально.** Выражение оборачивается в `${…}`:
  `"deviceId": "${uuid()}"`. Если написать `"deviceId": "uuid()"`, в банк уйдёт
  текст `uuid()`. Валидатор такие значения отвергает.
- В `extract`, `map` и условиях выражения пишутся **без** `${…}`.

### 2.3. `on` и `next`

`on` — массив веток, проверяются по порядку, срабатывает первая подходящая.
Ветка без `when` — fallback.

```json
{ "when": { "status": 200 }, "goto": "otp" }
{ "when": {}, "error": { "kind": "unavailable", "message": "auth/start failed" } }
```

- `goto` — id следующего шага.
- `error` — прервать вход с `BankError`. `kind` — см. §6.3; `message` — для
  диагностики, пользователю показывается сообщение по `kind`.
- `extract`/`store` внутри ветки выполняются только при её срабатывании.

`when` понимает:

| Поле | Смысл |
|---|---|
| `status` | число или массив чисел. |
| `path` | JSON-путь в теле ответа, например `$.type`. |
| `eq` / `neq` | Равенство/неравенство значению. |
| `match` | Регулярное выражение (строка). |
| `exists` | `true`/`false` — есть ли значение по пути. |
| `headers` | Условия на заголовки ответа; имена без учёта регистра. |

Проверка по телу работает **независимо от HTTP-статуса**: банк может ответить
`200` с ошибкой в теле.

```json
"when": { "headers": { "location": { "match": "access_token=" } } }
```

Если ни одна ветка не совпала, выполняется `next`. Если нет и его — ошибка
спецификации.

### 2.4. Вход через страницу банка (WebView)

Банк сам проводит телефон/СМС/пароль на своей странице; приложение открывает
WebView и забирает результат. Спецификация **не содержит JavaScript** — она описывает,
что искать:

```json
{
  "emit": {
    "kind": "browser",
    "title": "Вход в банк",
    "field": "callback",
    "payload": { "url": "https://online.example-bank.ru/login" },
    "capture": {
      "redirect": "^https://online\\.example-bank\\.ru/",
      "params": ["access_token", "code"],
      "storage": { "kinds": ["localStorage", "sessionStorage"], "claims": ["sub", "aud"] },
      "cookies": ["https://online.example-bank.ru"]
    }
  },
  "next": "after-browser"
}
```

`capture`:

| Поле | Смысл |
|---|---|
| `redirect` | Регэксп финального URL. |
| `params` | Какие query/fragment-параметры вернуть в `$input`. |
| `storage.kinds` | Где искать токен: `localStorage`, `sessionStorage`, `indexedDB`. |
| `storage.token` | Регэксп значения, похожего на токен (JWT и т. п.). |
| `storage.key` / `refreshKey` | Регэксп **имени** ключа: значение берётся как есть. Приоритетнее эвристики по форме. |
| `storage.keys` | Словарь «имя → регэксп ключа» для дополнительных значений. |
| `storage.claims` | Какие JWT-claims должны присутствовать. |
| `globals` | Имена `window.<name>` (только строки). |
| `fresh` | Начать с чистых cookie и web storage (банк может молча отдать старую сессию). |
| `settleMs` | Пауза перед захватом: банк может заменить токен сразу после входа. |
| `userAgent` | Свой User-Agent для WebView, если антибот банка не принимает стандартный. |
| `cookies` | Какие origin’ы собрать. |
| `requireCookies` | Какие cookie должны появиться, чтобы захват сработал. |
| `requireAnyCookies` | ИЛИ между наборами требований. |

Приложение собирает данные по этим правилам и возвращает их в `$input`:
`callback` (параметры), `session` (найденные значения), `cookies` (собранные
cookie). Дальше обычный шаг-запрос при необходимости обменивает код на токены.

Cookie-сессия (банк не выдаёт токенов вообще) описывается без `params`:
достаточно `cookies` + `requireCookies`.

`requireCookies` — массив строк или объектов:

```json
"requireCookies": [
  "SESSION",
  { "name": "user-id", "notValue": ["", "0"] },
  { "name": "preauth", "claim": "uid", "notValue": ["0"] }
]
```

Строка значит «cookie с таким именем есть». `notValue` требует, чтобы значение
не совпало с перечисленными. `claim` проверяет claim внутри JWT-значения.

Если банк отдаёт одноразовый `code`, а токены приходят в редиректе, их читают из
`$headers.location`, а если транспорт прошёл редирект сам — из `$url`:

```json
"extract": {
  "access": "coalesce(match($headers.location,'[?&]access_token=([^&]+)'), match($url,'[?&]access_token=([^&]+)'))"
}
```

### 2.5. Повторный вход и `restartOn`

`restartOn` — список ключей `$input`, которые начинают вход заново (обычно
`["phone"]`): старый `challengeId`, токены и куки не переиспользуются.

Если первый вызов `authorize` уже содержит значения (человек ввёл телефон в
форме приложения), движок пропускает стартовый вопрос и сразу выполняет шаг,
указанный в его `next`.

## 3. Запросы

```json
{
  "method": "POST",
  "path": "/v1/login",
  "query": { "session": "${$secret.session.id}" },
  "headers": { "x-device-id": "${$input.deviceId}" },
  "json": { "code": "${$input.code}" },
  "form": { "grant_type": "password" }
}
```

| Поле | Смысл |
|---|---|
| `method` | `GET`, `POST`, `PUT`, `PATCH`, `DELETE`. |
| `path` | Путь относительно `baseUrl`. Шаблоны разрешены. |
| `baseUrl` | Переопределить базовый хост для этого запроса (SSO и бизнес-API на разных хостах). |
| `url` | Абсолютный адрес вместо `baseUrl` + `path`. Нужен, когда банк выдаёт одноразовую ссылку, которую надо повторить байт в байт. Свой query сохраняется, `query` не добавляется. |
| `query` | Query-параметры. Шаблоны. |
| `headers` | Заголовки. Шаблоны. |
| `json` | JSON-тело. Шаблоны. |
| `form` | `application/x-www-form-urlencoded`. |
| `sign` | Подпись запроса (см. ниже). |
| `jsonAfter` | Маркер, после которого в тексте ответа лежит JSON: движок вырежет и разберёт его. Нужен, когда банк отдаёт данные страницей, а не отдельным API. |

Правила:

- Значения, которые вычислились в `undefined`, из `query`/`headers`/`json`/`form`
  выбрасываются, а не отправляются как `null`.
- В `json`/`form`/`query`/`headers` строки-литералы должны быть литералами.
  Строка, похожая на выражение (`$foo`, `fn(...)`), — ошибка валидации: её
  оборачивают в `${…}`. Иначе банк получит текст выражения.
- Cookie jar живёт в транспорте приложения, не в спецификации. Cookie-сессии
  выражаются через `check`/`read` без токена и при необходимости через
  `mergeCookies` в `refresh`.

Подпись:

```json
"sign": {
  "alg": "hmac-sha256",
  "key": "${$secret.session.hmacKey}",
  "input": "${method}\n${path}\n${query}\n${body}",
  "encoding": "base64",
  "header": "x-api-signature",
  "prefix": ""
}
```

- `alg` — сейчас только `hmac-sha256`.
- `key` — выражение, дающее ключ (строка или байты).
- `input` — шаблон. В нём доступны встроенные `${method}`, `${path}`,
  `${query}`, `${body}` — ровно те строки, что уйдут в сеть, — и обычные
  выражения.
- `encoding` — `base64`, `base64url`, `hex`.
- `header` — куда положить подпись; `prefix` — префикс значения (`Bearer `,
  `hmac `).

## 4. Выражения

Шаблон — строка с `${выражение}`. Если строка состоит ровно из одного
выражения, возвращается исходный тип (объект, число, массив); иначе — строка.

Пути:

| Путь | Значение |
|---|---|
| `$.field`, `$.items[0].id` | Текущий корень: тело ответа или элемент коллекции. |
| `$input.phone` | Ответы пользователя. |
| `$state.token` | Значения, извлечённые `extract`. |
| `$secret.session.token` | Секреты (сессия, устройство). |
| `$const.scope` | Константы спецификации. |
| `$parent.id` | Родительский элемент при `expand`. |
| `$index` | Индекс текущего элемента. |
| `$root.accounts` | Полный ответ, когда `$` — один элемент. |
| `$results` | Накопленные элементы в `attempts`. |
| `$joins.accounts` | Строки из `read.joins`. |
| `$headers.location` | Заголовки ответа (ключи в нижнем регистре; дефис — `$headers['set-cookie']`). |
| `$url` | URL ответа. |

Функции — фиксированный набор. Неизвестная функция — ошибка валидации; циклов,
побочных эффектов и `eval` нет.

**Строки:** `digits10` (последние 10 цифр), `digits`, `lower`, `upper`, `trim`,
`mask` (оставить последние N, по умолчанию 4), `replace`, `replaceRe`, `match`
(первая группа регэкспа), `matchAll` (все совпадения: группа 1, иначе всё
совпадение), `split`, `join`, `slice`, `contains`, `startsWith`,
`mergeCookies` (слияние `Set-Cookie` в строку `Cookie` по имени).

**Значения:** `str`, `int`, `num`, `bool`, `json`, `coalesce`, `concat`,
`default`, `if`, `not`, `len`, `empty`, `notEmpty`, `obj` (объект из пар
ключ-значение), `compact`, `distinct`, `strings`.

**Логика и массивы:** `eq`, `ne`, `and`, `or`, `gt`, `lt`, `abs`, `sub`,
`add`, `mul`, `positiveInt`, `positiveInts`, `where(list, field, value)` (все
строки с полем = значением), `has(list, field, value)` (есть ли такая строка),
`lookup(rows, keyField, value, resultField)` (join по одному полю),
`filterIn(rows, field, values)`, `sum(rows, path?)` (сумма числового поля, путь
может быть вложенным).

**Деньги:** `moneyMajor` (число в основных единицах → `{ minor, currency }`),
`moneyMinor` (уже минорные), `moneyText` (строка `"1 234,56"` или число),
`wireMoney` (`{ currency, value, minorUnits }`), `absMoney`, `currency`
(нормализация `RUR`/`810` → `RUB`), `last4` (`•• 1234`).

**Даты:** `date` (ISO из строки, `Date` или числа-мс), `dateAny` (ISO из
строки, epoch-секунд/мс или `{ milliseconds }`), `dateOpt` (как `dateAny`, но
`undefined` для пустого), `dateStartOfDay`, `dateAddDays`, `dateFromUnix`,
`epochMs`, `isoAfter(seconds)` (ISO-время через N секунд).

**Кодирование и крипто:** `base64`, `base64url`, `b64decode`, `sha256`, `hmac`,
`randomToken`, `randomHex`, `uuid`, `pkceChallenge` (S256), `jwtPayload`,
`jwtClaim`, `jsonAfter(text, marker)`.

**SRP-6a:** `srp6a({ n, g, hash, identity, password, salt, b, a, size,
xPrehash, xMod, encoding })` → `{ a, A, passwordData, extPasswordData, m1, m2 }`.
Приватные `x` и `S` наружу не выходят. Без `salt`/`b` возвращает только
публичную половину `{ a, A }`. Подробности — в исходниках движка; если банк
использует SRP, укажите это в issue.

**Прочее:** `firstUrl`, `image` (обложка: строка URL или `{ cover }`), `text`
(строка или `{ name }`), `percentOf` (первое `NN%` в тексте), `fill(template,
vars)` (подстановка `${name}` из объекта), `get(table, key, field)` (таблица из
`$const`), `now`, `timestamp`.

`uuid` и `random*` не криптостойкие: для ключей используйте `pkceChallenge`,
`srp6a` и подобные примитивы.

## 5. Чтение коллекций

```json
"read": {
  "accounts": {
    "request": { "method": "GET", "path": "/accounts" },
    "items": "$.accounts[*]",
    "map": {
      "id": "$.id",
      "name": "$.name",
      "kind": "$.kind",
      "currency": "$.currency",
      "balance": "moneyMajor($.amount,$.currency)"
    }
  }
}
```

- `items` — JSON-путь, оканчивающийся `[*]`, либо выражение, возвращающее
  массив. Поддерживаются массивы и объекты-словари (для объекта элемент
  получает синтетический ключ `__key`).
- `map` — словарь «поле домена → выражение». Значения можно писать без `${…}`
  или шаблоном. Вложенный объект — тоже маппинг; вложенная коллекция
  описывается как `{ "items": "$.essences[*]", "map": {…}, "filter": … }`.
- `filter` — выражение; ложное значение отбрасывает элемент до маппинга.
- `sources` — несколько коллекций в одном ответе (`accounts` + `deposits`);
  результат склеивается по порядку.
- `joins` — дополнительные запросы, строки которых доступны как
  `$joins.<as>` (например, карты → счета по `bindingId`). `optional: true` —
  ошибка такого запроса не роняет основной результат.
- `postFilter` — выражение по уже смаппленному элементу.
- `cursor` — выражение по ответу, дающее `Page.nextCursor` (пагинация).
- `attempts` — условные цепочки запросов: строгий список → шире → плоский
  резерв. `when` видит `$results`; `uniqueBy` задаёт ключ слияния;
  `onDuplicate` — `skip` (по умолчанию) или `replace`.
- `expand` — один элемент превращается в несколько вложенных (контейнер
  мультидепозита → счета). Внутри доступны `$parent` и `$index`. `fallback`
  используется, когда основная коллекция пуста (старая форма ответа).
- `single: true` — весь ответ считается одним элементом.
- `fanout: { request, extract }` — запрос на каждый элемент, результат в
  `$.__fanout`.

Пример `attempts`:

```json
"attempts": [
  {
    "request": { "method": "GET", "path": "/accounts", "query": { "strict": "false" } },
    "when": "not(has($results,'kind','credit'))",
    "items": "$.accounts[*]",
    "map": { "id": "$.id", "kind": "$.kind" },
    "uniqueBy": "$.id",
    "onDuplicate": "replace"
  }
]
```

Для `transactions` доступны `$input.accountId`, `$input.from`, `$input.to`,
`$input.cursor`, `$input.limit`. `cursor` в ответе даёт `nextCursor`; если банк
пагинацию не поддерживает, поле не указывают.

Движок добавляет `bankId`; остальные поля — из `map`. Обязательные поля домена —
в [`capabilities.md`](capabilities.md).

## 6. Проверка сессии, рефреш, ошибки

### 6.1. `check`

Лёгкий запрос «жива ли сессия»:

```json
"check": {
  "request": { "method": "GET", "path": "/profile" },
  "ok": [200],
  "when": { "path": "$.dataModel", "exists": false }
}
```

- `ok` — статусы, означающие «сессия активна» (по умолчанию `[200]`).
- `when` — признак «сессии нет» по телу при живом статусе: банк может ответить
  `200` страницей входа вместо данных.
- `keepAliveSeconds` — подсказка приложению, как часто проверять сессию, пока
  банк не обновляли. Нужна банкам с коротким refresh-токеном (Ozon, ~40 минут)
  или cookie-сессией без refresh (Цифра). Движок поле не использует: приложение
  вызывает `check` по расписанию, в том числе из фоновой задачи. Без поля
  банк проверяется только при обычном обновлении данных.

Без `check` движок считает сессию активной, если секрет `session` существует.

### 6.2. `refresh`

Цепочка запросов, которая выполняется один раз, когда чтение или `check`
получает `onStatus` (по умолчанию `[401, 403]`) **или** ответ совпал с `when`,
после чего исходный запрос повторяется с обновлённой сессией. `when` нужен
банкам, которые отвечают `200` с ошибкой в теле.

```json
"refresh": {
  "beforeExpirySeconds": 120,
  "when": { "path": "$.resultCode", "match": "PRIVILEGES|SESSION|EXPIRED" },
  "steps": [
    {
      "request": {
        "method": "POST",
        "path": "/auth/token",
        "form": {
          "grant_type": "refresh_token",
          "refresh_token": "${$secret.session.refreshToken}"
        }
      },
      "store": {
        "accessToken": "${$.access_token}",
        "expiresAt": "${isoAfter($.expires_in)}"
      }
    }
  ]
}
```

- `store` каждого шага сливается в сессию; следующие шаги видят обновлённые
  значения через `$secret.session.*`. Движок сохраняет сессию после каждого
  успешного шага: ротируемый токен одноразовый и не должен теряться, если
  следующий шаг упадёт.
- `requires` — поля сессии, без которых продление не запускается. Если поле
  отсутствует или пусто, движок сразу отдаёт `auth-required` и не тратит
  попытку банка (код-пароль Цифры/Сбера, refresh-токен Т-Банка).
- `beforeExpirySeconds` — продлить заранее, если спецификация хранит `expiresAt`
  (по умолчанию 60 секунд).
- `set` в шаге рефреша вычисляет значения до запроса и кладёт их в `$state`
  цепочки (двухшаговые обмены, SRP).
- `attempts: [{ steps: [...] }]` — альтернативные цепочки: первая, которая
  прошла целиком, выигрывает. Отказ шага (`HTTP >= 400`) — повод перейти к
  следующей; сетевые ошибки не глотаются.
- У цепочки может быть `requires: ["поле"]`: без него она пропускается, а не
  отправляет пустой запрос. Шаг может объявить `failWhen` — ответ, который
  считается отказом при успешном статусе (ошибка в теле `200`).
- Отказ рефреша — `auth-required`: нужен повторный вход.

Шаг рефреша видит `$headers` и `$url`, поэтому cookie-сессию поворачивают так:

```json
"store": { "cookies": "${mergeCookies($secret.session.cookies,$headers['set-cookie'])}" }
```

Транспорт приложения не следует редиректам. Если банк ставит сессионные cookie
на `302`, их берут из `$headers['set-cookie']` этого же шага, а адрес кабинета —
из `$headers.location`.

### 6.3. `errors`

```json
"errors": [
  { "when": { "status": 429 }, "error": { "kind": "rate-limited", "retryAfter": "$.retryAfter" } },
  { "when": { "status": 403 }, "error": { "kind": "blocked", "message": "…" } },
  { "when": { "status": 401, "path": "$.type", "eq": "invalid_code" },
    "error": { "kind": "invalid-credentials" } }
]
```

Правила проверяются до ветвлений, в том числе в `check`. Поля `error`:
`kind` (обязательно), `message` (диагностика), `userMessageKey` (свой текст для
UI), `code` (выражение с кодом банка для логов), `retryAfter` (путь к
`Retry-After`-подобному значению).

`kind`:

| `kind` | Когда |
|---|---|
| `network` | Сеть недоступна, таймаут. |
| `auth-required` | Нужен повторный вход. |
| `invalid-credentials` | Неверный телефон/код/пароль. |
| `challenge-required` | Банк требует действие (PIN, капча, подтверждение). |
| `rate-limited` | Слишком много запросов. |
| `blocked` | Аккаунт/устройство заблокированы. |
| `unavailable` | Банк недоступен или технические работы. |
| `unsupported` | Банк не поддерживает операцию. |
| `unknown` | Всё остальное. |

Требование `requires-action` (PIN, капча) выражается шагом `emit`, а не
ошибкой.

## 7. Кэшбэк (необязательно)

```json
"cashback": {
  "programs": {
    "request": { "method": "GET", "path": "/cashback/programs" },
    "items": "$.programs[*]",
    "map": {
      "id": "$.id",
      "title": "$.title",
      "unit": "$.unit",
      "unitLabel": "$.unitLabel",
      "options": {
        "items": "$.categories[*]",
        "map": { "id": "$.id", "name": "$.name", "percent": "$.percent", "selected": "$.selected" }
      }
    }
  },
  "select": {
    "request": {
      "method": "POST",
      "path": "/cashback/select",
      "query": { "option": "${$input.optionId}" }
    }
  }
}
```

`programs` — обычное чтение (§5). Поля программы и категории — в
[`capabilities.md`](capabilities.md). `select` выбирает по одной категории и
пропускает уже выбранные; снятие выбора не поддерживается.

Расширения:

- `select.mode: "batch"` — отправить весь список одним запросом.
- `select.prepare` — подготовительное чтение строк перед мутацией (id категории
  → код банка); `select.require` — обязательные поля подготовленных строк.
- `select.attempts` — разные запросы под разные программы (`when` по
  `programId`).
- `fanout: { request, extract }` — запрос на каждый элемент (карусель).
- `single: true` — весь ответ считается одной программой.
- `items` может быть выражением.
- Во вложенной коллекции — `filter` и `uniqueBy`.

## 8. Чего в спецификации быть не может

- JavaScript, `eval`, инъекции в WebView.
- Прямой доступ к файлам, хранилищу, навигации, другим банкам.
- Хосты вне `hosts`, `http://`, редиректы на незадекларированный хост.
- Эвристики по тексту интерфейса («4 карты», названия кнопок): они ломаются от
  смены локали и редизайна.
- Разбор дерева SDUI-экрана: сначала ищем плоский или BFF-эндпоинт; если его
  нет — пишите в issue, нужен отдельный примитив.
- Запись, кроме выбора категорий кэшбэка. Переводы, платежи, блокировка карт и
  прочие операции с деньгами текущим языком спецификаций не поддерживаются.
- Токены, cookie, номера счетов, PAN, ФИО и телефоны в примерах и
  доказательствах.

## 9. Версии и совместимость

- `schema` — формат. Несовпадение = спецификация не загружается.
- `minEngine` — минимальная версия движка. Его выставляет сопровождающий.
- Неизвестные поля движок игнорирует. `capabilities` должны соответствовать
  секциям спецификации: объявленная возможность без данных даёт пустой экран или
  ошибку, а не тихое скрытие.
- Версия спецификации двигается независимо от приложения. Правила semver:
  - **мажор** — сломанный контракт или переименование полей;
  - **минор** — новая возможность (новая коллекция, кэшбэк, пагинация);
  - **патч** — исправление маппинга без изменения поведения.
- Откат — публикация предыдущей версии артефакта.

## 10. Частые ошибки

Валидатор движка запускает сопровождающий, но большую часть ошибок можно
поймать самому:

- `schema` не `1`; `id` пустой; `version` не semver.
- `baseUrl` не `https` или его хост не входит в `hosts`.
- В `store` выражение без `${…}` — в банк уйдёт текст функции.
- В `json`/`form`/`query` строка похожа на выражение (`$foo`, `fn(...)`) —
  нужна обёртка `${…}`.
- В `extract`/`map`/`filter`, наоборот, лишние `${…}` не нужны: там выражения
  пишутся как есть.
- `items` не оканчивается `[*]` и при этом не является выражением, возвращающим
  массив.
- В `map` нет полей, которые обязательны для домена (`id`, `balance`,
  `postedAt`, …).
- Неизвестная функция — проверьте имя по списку в §4.
- Шаг-вопрос без `next`.
- Ветка `on` с совпавшим условием, но без `goto`.
- `goto`/`next` ссылается на несуществующий шаг.
- `minEngine` новее движка.

