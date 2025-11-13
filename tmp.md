Ты всё сделал правильно — лог с
`"error_description":"Invalid authentication method for accessing this endpoint","error":"invalid_client"`
даёт точный диагноз: **токен-эндпойнт CIDP отклоняет способ аутентификации клиента, который использует Kiali**.

В OAuth2 для token endpoint есть минимум два варианта аутентификации клиента:

* **client_secret_basic** — `Authorization: Basic base64(client_id:client_secret)`
* **client_secret_post** — `client_id` и `client_secret` в теле формы

У тебя руками обмен кода на токен уже работает (мы это доказали), а Kiali получает 401 с текстом *Invalid authentication method…*. Это классическая ситуация «метод, разрешённый для клиента в CIDP, не совпадает с тем, который шлёт Kiali».

Ниже — короткий план, как за 2 шага подтвердить гипотезу и починить.

---

# 1) Быстро выясняем, какой метод разрешён в CIDP

Оставь **Kiali на время в `strategy: anonymous`**, чтобы он не «съедал» код.

В поде `istio-proxy` получи новый `code` (как ты уже делал), а затем **попробуй оба обмена**:

**A. client_secret_post (в теле) — то, что ты, скорее всего, уже делал и оно работало:**

```bash
curl -sk -w "\nHTTP_STATUS=%{http_code}\n" \
  -X POST "$OIDC_TOKEN_ENDPOINT" \
  -d "grant_type=authorization_code" \
  -d "client_id=$OIDC_CLIENT_ID" \
  -d "client_secret=$OIDC_CLIENT_SECRET" \
  -d "redirect_uri=$OIDC_REDIRECT_URI" \
  -d "code=$CODE"
```

**B. client_secret_basic (в заголовке) — альтернативный метод:**

```bash
curl -sk -w "\nHTTP_STATUS=%{http_code}\n" \
  -u "$OIDC_CLIENT_ID:$OIDC_CLIENT_SECRET" \
  -X POST "$OIDC_TOKEN_ENDPOINT" \
  -d "grant_type=authorization_code" \
  -d "redirect_uri=$OIDC_REDIRECT_URI" \
  -d "code=$CODE"
```

* Если **A=200 OK**, **B=401 invalid_client** → в CIDP клиент разрешён **только POST**, а Kiali шлёт **Basic**.
* Если **B=200 OK**, **A=401 invalid_client** → в CIDP клиент разрешён **только Basic**, а Kiali шлёт **POST**.

(Текст `"Invalid authentication method for accessing this endpoint"` как раз про это.)

---

# 2) Доводим Kiali до того же метода

Kiali **не умеет** переключать метод через конфиг (нет поля типа `token_endpoint_auth_method`). Он шлёт *фиксированный* способ (в конкретной версии Kiali это обычно **client_secret_basic**). Значит, чинится на стороне CIDP:

**Попроси команду CIDP для клиента `Cloud-Global`:**

* включить **оба** метода client authentication на token endpoint:
  `client_secret_basic` **и** `client_secret_post`
  (или как минимум тот, который мы только что подтвердили тестом как необходимый для Kiali);
* убедиться, что для клиента разрешён **authorization_code**;
* в списке **Redirect URI** есть ровно
  `https://kiali.traefik-uk-uk-site.uat.hc.intranet.linux.org/kiali`.

> Готовая формулировка:
>
> > “Our service (Kiali) sends the token request with **<basic|post>** client authentication. The token endpoint currently returns `{"error":"invalid_client","error_description":"Invalid authentication method for accessing this endpoint"}`.
> > Please enable **client_secret_basic** (and preferably **client_secret_post** as well) for client `Cloud-Global`. Redirect URI registered: `https://kiali.traefik-uk-uk-site.uat.hc.intranet.linux.org/kiali`. Grant type: `authorization_code`.”

После изменения на CIDP:

1. Верни в Kiali `auth.strategy: openid`.
2. Проверь, что в `server` указан внешний порт:

   ```yaml
   server:
     web_fqdn: kiali.traefik-uk-uk-site.uat.hc.intranet.linux.org
     web_root: /kiali
     web_schema: https
     web_port: 443          # важно!
     port: 20001
   ```
3. На время отладки можно оставить `client_secret` прямо в `config.yaml`:

   ```yaml
   auth:
     strategy: openid
     openid:
       client_id: "Cloud-Global"
       client_secret: "<секрет>"
       issuer_uri: "https://uat.cidp.linux.org/am/oauth2/global"
       authorization_endpoint: "https://uat.cidp.linux.org/am/oauth2/global/authorize"
       scopes: ["openid","email"]     # см. примечание про кастомные scope ниже
       username_claim: "email"
   ```
4. `kubectl -n kiali rollout restart deploy kiali`, включи `LOG_LEVEL=debug` и зайди в Kiali.

Если обмен прошёл, в логах исчезнет 401, а в ответе на callback Kiali выставит cookie `kiali-token-...` — ошибка “Session not found” пропадёт.

---

## Про предупреждение «some of the configured scopes are unsupported»

Это отдельная мелочь:

* CIDP в `/.well-known` отдаёт список `scopes_supported`, и, скорее всего, **не включает туда** ваш кастомный `az1prod`.
* Kiali честно предупреждает, что запрошен scope, которого провайдер «не заявлял». Это **не фатально**, но засоряет логи.
* Для входа Kiali **достаточно** `openid` (иногда `email`). Я бы **в Kiali оставил только `openid` и `email`**, а кастомные `az1*` использовал там, где они реально нужны (downstream-сервисы). Это уберёт предупреждение.

---

## Быстрый чек-лист на один экран

1. **Определи метод**: получи fresh `code`, протестируй из пода оба варианта (POST vs Basic).
2. **Попроси CIDP** включить метод, который использует Kiali (или оба).
3. **Kiali config**:

   * `server.web_port: 443`, `web_schema: https`, `web_root: /kiali`, `web_fqdn` как у тебя.
   * `auth.openid.client_id`, `client_secret`, `issuer_uri`, `authorization_endpoint`.
   * `auth.openid.scopes: ["openid","email"]`.
4. **Перезапусти**, включи `LOG_LEVEL=debug`, зайди в Kiali и проверь, что cookie `kiali-token-*` появляется.
5. Убери `client_secret` из ConfigMap и верни секрет в `Secret kiali` (`oidc-secret` смонтирован в `/kiali-secret`).

Если после того как CIDP включит правильный метод client auth, 401 останется — пришли 2–3 строки debug-лога вокруг “OpenId token request failed …”, я сверю параметры (`redirect_uri`, `grant_type`, способ client auth) и скажу, куда копать дальше.
