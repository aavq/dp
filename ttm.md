Коротко:

1. В Kiali **нет настройки**, которая позволяет переключить способ аутентификации клиента на token endpoint с `client_secret_basic` на `client_secret_post` – ни в 2.15, ни в более свежих версиях (по текущей документации).
2. Ваш CIDP, который *запрещает* `client_secret_basic`, формально несовместим с текущей реализацией `auth.strategy: openid` в Kiali.
3. Варианты:

   * Обойти проблему архитектурно (прокси/Envoy/oauth2-proxy и `auth.strategy: header`/`token`).
   * Или делать форк Kiali и добавить поддержку `client_secret_post`, затем предлагать PR в upstream.

Разберём по шагам.

---

## 1. Что умеет Kiali сейчас

### 1.1. Настройки OpenID в CR

В актуальной CR-спецификации Kiali (обновлена буквально несколько недель назад) для блока `spec.auth.openid` перечислены:

* `authorization_endpoint`
* `issuer_uri`
* `client_id`
* `api_token` (выбор `id_token` или `access_token`)
* `scopes`, `username_claim`, `allowed_domains`, прокси и т.п. ([kiali.io][1])

**Никаких полей вида** `client_auth_method`, `token_endpoint_auth_method`, `client_secret_post` там нет.

Документация по OpenID-стратегии Kiali тоже описывает только то, как задать `client_id`, `issuer_uri`, секрет (через `oidc-secret`), scopes и т.д., но не позволяет выбирать метод аутентификации клиента на token endpoint.([kiali.io][2])

Отсюда вывод:

* Kiali **сам решает**, как аутентифицироваться на `/token` (реализовано в коде, без конфиг-флага).
* По факту он ведёт себя как классический confidential client, использующий `client_secret_basic`.

### 1.2. Подтверждение со стороны сторонних интеграций

Авторы Authelia (IdP/проксирующее решение) отдельно отмечают, что у Kiali как клиента есть баг, из-за которого он **не соответствует полностью OpenID Connect 1.0** и им даже нужен “escape hatch” для обхода этого поведения.([Authelia][3])

Это косвенно подтверждает, что:

* поведение клиента (в частности, работа с токенами) зашито в коде и ограничено,
* и гибкой настройки client auth method там, увы, нет.

С учётом того, что даже в свежей CR-спецификации (осень 2025) нет нужной опции, можно честно сказать:

> **На сегодня Kiali не даёт штатной возможности переключиться на `client_secret_post`.**

---

## 2. Что значит отказ CIDP от `client_secret_basic`

CIDP вам прямо сказал:

> Мы **не поддерживаем** `client_secret_basic` для обмена кода на токен.

Вы опытным путём показали:

* Если руками отправить `POST` с `client_id` и `client_secret` **в теле** (`client_secret_post`) – всё ок, access token выдаётся.
* Если использовать Basic Auth (как делает Kiali) – IdP отказывает.

С точки зрения спецификации OpenID Connect по умолчанию, если у клиента не прописан явный метод, сервер должен принимать `client_secret_basic`.([GitHub][4])
Но ваш CIDP явно так не делает и требует `client_secret_post`.

В результате:

* Стандартный Kiali (который использует `client_secret_basic`) **не сможет обменять код на токен** с этим CIDP.
* И без изменения **либо** Kiali, **либо** CIDP вы этот клинч не разрулите.

---

## 3. Практические варианты решения

### Вариант A. Архитектурный обход: прокси + `auth.strategy: header/token`

Идея: раз Kiali не умеет `client_secret_post`, пусть за него это делает **отдельный компонент**, а Kiali видит уже готовый заголовок `Authorization: Bearer …` или headers impersonation.

1. Перед Kiali ставим:

   * либо `oauth2-proxy` (или аналоги: Pomerium, forward-auth в Envoy/NGINX),
   * либо Envoy/Istio с `ext_authz`/Lua/Wasmtime фильтром, который ходит в CIDP и делает код→токен с `client_secret_post`.

2. Kiali настраиваем на:

   * `auth.strategy: header` – когда upstream уже аутентифицировал пользователя и проставляет ему `Authorization` и, при необходимости, импёрсонационные заголовки.([kiali.io][5])
   * либо `auth.strategy: token` – если вы сами генерируете kube-токен и отдаёте его пользователю/прокси.

Плюсы:

* Не нужно форкать Kiali.
* Логику OIDC можно унифицировать: этим же прокси потом прикрывать и другие веб-морды (Kubernetes Dashboard, Prometheus UI и т.д.).
* Легче согласовать с CIDP (они видят одного доверенного confidential-клиента – ваш прокси).

Минусы/нюансы:

* Если вы используете **multi-cluster** режим Kiali, официально он сейчас поддержан только для стратегий `anonymous` и `openid`.([Kiali][6])
  То есть при переходе на `header` вы теоретически можете потерять часть мультикластер-фич (надо смотреть, как именно вы сейчас организовали мультикластер).
* Появляется ещё один компонент в цепочке (прокси).

Чисто с инженерной точки зрения это **самый быстрый и наименее рискованный** путь.

---

### Вариант B. Форк Kiali с поддержкой `client_secret_post` + PR в upstream

Раз Kiali – open source, его действительно можно доработать.

#### 3.2.1. Что именно нужно добавить

На концептуальном уровне:

1. **Новая настройка в конфиге** (и в CR-спеке Kiali):

   ```yaml
   spec:
     auth:
       strategy: openid
       openid:
         client_id: "kiali"
         issuer_uri: "https://cidp.example.com"
         # новое поле:
         client_auth_method: "client_secret_post"  # или "client_secret_basic" (по умолчанию)
   ```

2. В Go-структуру конфига (`config.AuthConfig` / `config.OpenId` – точное имя можно посмотреть в пакете `config` репозитория Kiali([pkg.go.dev][7])) нужно добавить поле:

   ```go
   type OpenIdAuth struct {
       // ...
       ClientAuthMethod string `yaml:"client_auth_method,omitempty"`
   }
   ```

   И задать default: если пусто → `"client_secret_basic"` (для обратной совместимости).

3. В коде, который делает запрос к token endpoint, нужно:

   * Если `ClientAuthMethod == "" || "client_secret_basic"`:
     формировать заголовок `Authorization: Basic base64(urlencode(client_id):urlencode(client_secret))`.
   * Если `ClientAuthMethod == "client_secret_post"`:
     убрать Basic-заголовок и положить `client_id`/`client_secret` в тело `application/x-www-form-urlencoded` (что вы уже руками делали).

   В псевдокоде это выглядит так:

   ```go
   form := url.Values{}
   form.Set("grant_type", "authorization_code")
   form.Set("code", code)
   form.Set("redirect_uri", redirectURI)

   req, _ := http.NewRequest("POST", tokenEndpoint, strings.NewReader(form.Encode()))
   req.Header.Set("Content-Type", "application/x-www-form-urlencoded")

   switch config.Auth.OpenId.ClientAuthMethod {
   case "", "client_secret_basic":
       req.SetBasicAuth(clientID, clientSecret)
   case "client_secret_post":
       form.Set("client_id", clientID)
       form.Set("client_secret", clientSecret)
       // Пересобрать тело и Content-Length после добавления параметров
       req.Body = io.NopCloser(strings.NewReader(form.Encode()))
   default:
       // лог + ошибка конфигурации
   }
   ```

4. Обновить документацию:

   * Страницу `OpenID Connect strategy` – описать новое поле `client_auth_method` и поддерживаемые значения.
   * CR reference – добавить это поле в блок `spec.auth.openid`.([kiali.io][1])

5. Добавить unit-тесты/интеграционные тесты, которые проверяют:

   * Что при `client_auth_method: client_secret_basic` запрос идёт с Basic.
   * Что при `client_auth_method: client_secret_post` – без Basic, но с параметрами в body.

#### 3.2.2. Как практически это выкатить у себя

Очень краткий план:

1. Клонировать `kiali` + `kiali-operator` (как советуют сами разработчики).([GitHub][8])

2. Внести изменения в конфиг и серверную часть (auth/openid).

3. Собрать собственный образ Kiali:

   ```bash
   make build        # backend
   make build-ui     # frontend
   # затем сборка docker-образа по инструкции из README
   ```

4. В `Kiali` CR в вашем кластере указать:

   ```yaml
   spec:
     deployment:
       image_name: "your-registry/kiali"
       image_version: "your-tag"
   ```

5. Протестировать на тестовом кластерe с вашим CIDP, где `client_secret_post` – единственный допустимый метод.

6. После успешного теста – оформить **issue + PR в upstream**:

   * В issue описать кейс: существующие IdP, которые запрещают `client_secret_basic` и требуют `client_secret_post`, ссылку на спецификацию и на ваш CIDP.
   * В PR – изменения в коде, тесты, доку.

С точки зрения проекта, это аккуратное расширение, не ломающее существующие инсталляции, если дефолт оставить `client_secret_basic`. Шансы, что такое изменение примут, выглядят разумными.

---

### Вариант C. Попросить CIDP включить базовую совместимость

Теоретически можно:

* убедить команду CIDP включить поддержку `client_secret_basic` **хотя бы для конкретного клиента Kiali** – это строго по спецификации (многие IdP умеют иметь разные `token_endpoint_auth_method` по клиентам).([docs.pingidentity.com][9])

Но вы уже получили от CIDP явный ответ "не поддерживаем" – поэтому, честно, я бы на это сильно не рассчитывал.

---

## 4. Что я бы сделал на твоём месте

С учётом всего:

1. **Краткосрочно / для боевого контура:**

   * Поставить перед Kiali прокси, который умеет OIDC с `client_secret_post` (oauth2-proxy / Envoy / Istio ext_authz).
   * Перевести Kiali на `auth.strategy: header` (если мультикластер не критичен) или аккуратно проверить влияние на мультикластер.

2. **Среднесрочно:**

   * Открыть issue в kiali/kiali на тему:
     “Support configurable token endpoint client authentication method (`client_secret_basic` / `client_secret_post`).”
   * Сделать небольшой PoC-форк с поддержкой `client_secret_post` (как выше), погонять на тестовом кластере, потом оформить PR.

Если хочешь, дальше можем:

* Разобрать конкретный стек (Istio Gateway + EnvoyFilter/ext_authz или oauth2-proxy) и собрать схему аутентификации Kiali через `header`.
* Или пойти по пути форка и я помогу пошагово наметить структуру патча (какие конкретно файлы в `kiali` смотреть, как проще всего тестировать обмен кода на токен).

[1]: https://kiali.io/docs/configuration/kialis.kiali.io/ "Kiali CR Reference | Kiali"
[2]: https://kiali.io/docs/configuration/authentication/openid/?utm_source=chatgpt.com "OpenID Connect strategy"
[3]: https://www.authelia.com/integration/openid-connect/clients/kiali/ "Kiali | OpenID Connect 1.0 | Integration"
[4]: https://github.com/oidc-wp/openid-connect-generic/issues/292?utm_source=chatgpt.com "Support Authentication Method client_secret_basic #292"
[5]: https://kiali.io/docs/configuration/kialis.kiali.io/?utm_source=chatgpt.com "Kiali CR Reference"
[6]: https://v1-73.kiali.io/docs/configuration/kialis.kiali.io/?utm_source=chatgpt.com "Kiali CR Reference"
[7]: https://pkg.go.dev/github.com/kiali/kiali/config?utm_source=chatgpt.com "config package - github.com/kiali/kiali/config"
[8]: https://github.com/kiali/kiali "GitHub - kiali/kiali: Kiali project, observability for the Istio service mesh"
[9]: https://docs.pingidentity.com/pingam/8/am-oidc1/oidc-client-auth.html?utm_source=chatgpt.com "OIDC client authentication | PingAM"
