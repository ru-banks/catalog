# Пример спецификации: демо-банк Bank 1

[`bank1.spec.json`](bank1.spec.json) — полная валидная спецификация вымышленного
банка. В ней показаны основные примитивы языка: вход по телефону и коду,
продление сессии, счета, карты, история с курсором, кэшбэк и правила ошибок.

Это **пример формата**, а не эталон протокола. У реального банка будут другие
пути, заголовки, единицы и ветвления. Копируйте структуру, но не эндпоинты.

## Демо-сервер

Сервер живой, данные вымышленные, регистрация не нужна. Удобно потрогать
ответы руками и посмотреть, как они превращаются в поля домена.

База: `https://demo-banks.unibank-demo.workers.dev/bank1`

Учётные данные:

| Поле | Значение |
|---|---|
| Телефон | `+7 916 123-45-67` (или `79161234567`) |
| Код из «СМС» | `0000` |

```bash
BASE=https://demo-banks.unibank-demo.workers.dev/bank1

# 1. Начать вход
curl -s -X POST "$BASE/auth/start" \
  -H 'Content-Type: application/json' \
  -d '{"phone":"79161234567"}'
# → { "challengeId": "…", "maskedPhone": "+7 ••• 45-67", "expiresIn": 300 }

# 2. Подтвердить код
curl -s -X POST "$BASE/auth/verify" \
  -H 'Content-Type: application/json' \
  -d '{"challengeId":"…","code":"0000"}'
# → { "accessToken": "…", "refreshToken": "…", "expiresIn": 3600 }

# 3. Читать данные (подставьте accessToken)
curl -s "$BASE/accounts" -H "Authorization: Bearer …"
curl -s "$BASE/cards" -H "Authorization: Bearer …"
curl -s "$BASE/transactions?limit=5" -H "Authorization: Bearer …"
```

Контракт демо-банка целиком виден в [`bank1.spec.json`](bank1.spec.json): пути,
поля, маппинг и коды ошибок. Этого достаточно, чтобы понять формат.

## Что смотреть в спецификации

| Место | Чему учит |
|---|---|
| `auth.steps` | Вопрос (`emit`) отдельно от запроса; `next` и ветки `on`. |
| `store.session` | Материал сессии одним ключом, `${…}` вокруг выражений. |
| `refresh` | Продление по `401` и заранее по `expiresAt`. |
| `read.accounts.map` | `moneyMajor` и явный `kind`. |
| `read.transactions.map` | `dateAny`, `direction`, необязательный `cashback`. |
| `read.transactions.cursor` | Пагинация: `nextCursor` из ответа. |
| `cashback` | Программы, вложенные категории, выбор. |
| `errors` | Ответ банка → `BankError.kind`. |
