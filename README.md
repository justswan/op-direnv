# op-direnv

Секреты из 1Password в shell без повторных запросов авторизации.

## Проблема

1Password CLI (`op`) при интеграции с десктопным приложением запрашивает Touch ID на **каждый** вызов `op read` / `op inject`. Если в `.envrc` несколько секретов — получаешь несколько промптов. При множестве открытых терминалов это превращается в постоянные запросы.

## Почему не 1Password Environments

У 1Password есть [Environments](https://developer.1password.com/docs/environments/) — встроенное решение для `.env` файлов. Но оно требует **копировать секреты** в отдельную сущность "Environment". Нельзя просто сослаться на существующий item в vault — нужно продублировать значение. Это неудобно: секрет обновляется в одном месте, но не обновляется в Environment, и наоборот.

`op-direnv` использует обычные `op://` ссылки на существующие items в vault'ах — один источник правды, без дублирования.

## Решение

Двухуровневая схема с Service Account (SA) token:

```
Touch ID (один раз)
  └─► op read — получает SA token из 1Password
        └─► кэш в $TMPDIR (живёт до ребута)
              └─► op inject использует SA token — без промптов
```

1. Первый `cd` в проект → direnv вызывает `op_inject` → нужен SA token
2. `op_auth` делает `op read` через личный аккаунт → **Touch ID один раз**
3. SA token сохраняется в `$TMPDIR/.op-sa-*`
4. Все последующие `op inject` (в любом терминале) используют кэшированный SA token
5. После ребута — снова один Touch ID

## Требования

- [1Password CLI](https://developer.1password.com/docs/cli/) (`op`)
- [1Password desktop app](https://1password.com/downloads) с включённым "Integrate with 1Password CLI"
- [direnv](https://direnv.net/)
- Service Account в 1Password с доступом к нужным vault'ам

## Как создать Service Account

1. 1Password → Settings → Developer → Service Accounts → New Service Account
2. Дать доступ к vault'ам, из которых будут читаться секреты
3. Сохранить SA token как item в 1Password (например `op://Private/1pass-service-accounts/src-envs-shell`)

## Установка

```bash
# Скопировать direnv helpers
cp direnvrc ~/.config/direnv/direnvrc

# Отредактировать _OP_DEFAULT_SA — путь к вашему SA token в 1Password
```

## Использование

В каждом проекте создать два файла:

### `.env.op` — ссылки на секреты

```bash
export GITHUB_TOKEN="op://VaultName/item-name/field"
export DB_PASSWORD="op://VaultName/database/password"
```

### `.envrc` — загрузка через direnv

```bash
export AWS_PROFILE=myproject

# Секреты из 1Password
op_inject .env.op

# Kubernetes config с секретами (опционально)
op_inject .kube/config KUBECONFIG "$TMPDIR/kubeconfig-myproject"
```

Не забудьте `direnv allow` и добавить `.env.op` в `.gitignore` если vault/item пути приватные.

### Без direnv (только `.zshrc`)

Добавьте в `~/.zshrc`:

```bash
# 1Password: SA token (биометрия один раз, кэш до ребута)
_OP_SA_REF="op://Private/1pass-service-accounts/src-envs-shell"
_OP_SA_CACHE="$TMPDIR/.op-sa-$(echo "$_OP_SA_REF" | md5 -q /dev/stdin)"

if [[ ! -s "$_OP_SA_CACHE" ]]; then
  op read "$_OP_SA_REF" > "$_OP_SA_CACHE" || rm -f "$_OP_SA_CACHE"
fi
[[ -s "$_OP_SA_CACHE" ]] && export OP_SERVICE_ACCOUNT_TOKEN="$(cat "$_OP_SA_CACHE")"

# Секреты (резолвятся при старте шелла)
eval "$(op inject --in-file ~/.env.op)"
```

Создайте `~/.env.op`:

```bash
export GITHUB_TOKEN="op://VaultName/item-name/field"
```

При открытии первого терминала — Touch ID один раз. Все последующие терминалы подхватят кэшированный SA token без промптов.

## Как это работает

### `op_auth [ref]`

Получает SA token через `op read` (требует авторизации в личном аккаунте), кэширует в `$TMPDIR`. При повторном вызове читает из кэша.

### `op_inject src`

Вызывает `op inject --in-file` с автоматической авторизацией через SA token. Два режима:
- `op_inject .env.op` — резолвит `op://` ссылки и экспортирует переменные
- `op_inject .kube/config KUBECONFIG /tmp/kube` — резолвит файл, записывает результат, экспортирует переменную

## Файлы

| Файл | Назначение |
|------|-----------|
| `direnvrc` | Функции `op_auth` и `op_inject` → `~/.config/direnv/direnvrc` |
| `example.envrc` | Пример `.envrc` для проекта |
| `example.env.op` | Пример `.env.op` со ссылками на секреты |
