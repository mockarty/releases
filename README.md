<p align="center"><img src="logo.svg" width="160" alt="Mockarty"></p>

# Mockarty Community

[Русская версия](#mockarty-community-на-русском) · [Downloads](https://mockarty.ru/downloads) · [Documentation](https://mockarty.ru/docs) · [GitHub releases](https://github.com/mockarty/releases/releases)

> The first Community release is in preparation. The Community instructions below apply when a release lists Community Server support. Older preview downloads are available in Releases with their original release notes.

Use Mockarty to mock APIs, run tests, and share results with your team. **Community Server runs on your own machine or server without a licence key or a Mockarty Cloud account.** It includes ten workspaces and five seats in each core module: mocks, testing, reliability, basic fuzz testing, test case management, work management, and contract testing.

Desktop's interactive Harness and Community Server runner registration are available without an Enterprise licence. The autonomous mission engine requires Enterprise. Access to a team workspace follows the roles granted on that server.

## Choose where to start

| What you want to do | Where to start |
|---|---|
| Work on your computer | Install **Mockarty Desktop** from [Downloads](https://mockarty.ru/downloads) and choose local mode. You can connect it later to a team server or to [Mockarty Cloud](https://cloud.mockarty.ru/). |
| Run Community Server on your machine | Start [one container](#start-community-server-with-docker) or run a [native binary](#run-a-native-server-binary). |
| Set up a server for your team | Follow the [server deployment guide](https://mockarty.ru/docs/deployment-guide) and give your team the HTTPS address. |
| Connect a coding runner | Choose a published `mockarty-coder-runner` for your platform from [Downloads](https://mockarty.ru/downloads#coding-runner), verify its checksum and [configure trust with your server](https://mockarty.ru/docs/ai-components#trust-setup). |
| Use Android | Look for the signed **Mockarty Mobile** package on the [Android downloads page](https://mockarty.ru/downloads#android-mobile). Connect to the server or Cloud account where your team works. **Companion** is for a device you want to pair for testing. |

## Start Community Server with Docker

Clone this release repository. In the following commands, use the image tag listed in the [release notes](https://github.com/mockarty/releases/releases), for a release with a published server image. Run the commands from the cloned directory in Bash or Zsh.

```bash
printf 'Published Docker image tag (without v): '
read -r MOCKARTY_VERSION
export MOCKARTY_VERSION
printf 'Set a strong password for admin: '
read -rs FIRST_ADMIN_PASSWORD
printf '\n'
export FIRST_ADMIN_PASSWORD
docker compose -f deploy/docker-compose.community.yml up -d
```

Open <http://127.0.0.1:5770/ui/> and sign in as `admin` with the password you entered. Review and accept the Community terms on first sign-in.

The example binds the HTTP port to loopback and persists SQLite data in the `community_data` volume. To stop it, run `docker compose -f deploy/docker-compose.community.yml stop` in the same terminal. Keep the volume if you want to retain your data. For a shared server, follow the HTTPS setup in the deployment guide.

## Run a native server binary

Choose a [GitHub release](https://github.com/mockarty/releases/releases) that lists Community Server support and the `mockarty` binary for your operating system and CPU. Compare its SHA‑256 with the matching entry in `checksums.txt` from that release. On macOS and Linux, name the file `mockarty`, open its directory, and run:

```bash
chmod +x ./mockarty
printf 'Set a strong password for admin: '
read -rs FIRST_ADMIN_PASSWORD
printf '\n'
export FIRST_ADMIN_PASSWORD
DB_USE=sqlite DB_DSN="$PWD/mockarty.db" CACHE_TYPE=inmemory \
  HTTP_BIND_ADDR=127.0.0.1 HTTP_PORT=5770 AUTH_BASE_URL=http://127.0.0.1:5770 \
  COOKIE_SECURE=false FIRST_ADMIN_LOGIN=admin ./mockarty
```

Open <http://127.0.0.1:5770/ui/>, sign in as `admin` with the password you entered and accept the Community terms.

## Create your first mock

1. In **Constructor**, select the `sandbox` workspace. Set the mock ID to `my-first-mock`, protocol to HTTP, method to `GET` and route to `/hello`.
2. Set status `200`, content type `application/json` and body `{"message":"Hello from Mockarty"}`. Save the mock.
3. Call the URL below. The response should contain the message you saved.

```bash
curl --fail --silent --show-error http://127.0.0.1:5770/stubs/sandbox/hello
```

Next, follow the [API testing guide](https://mockarty.ru/docs/quick-start#api-testing) to save a request in a collection and share the run with a teammate.

For gRPC, WebSocket, broker and socket mocks, use the [protocol listeners built into Mockarty](https://mockarty.ru/docs/native-protocol-mocks). Use the [CLI releases](https://github.com/mockarty/mockarty-cli/releases) or the [SDK directory](docs/SDKS.md) to add Mockarty to your tests and CI jobs.

## Guides and licence

- [Admin node](docs/ADMIN_GUIDE.md) · [Mock Resolver](docs/RESOLVER_GUIDE.md) · [Test runner](docs/RUNNER_GUIDE.md) · [Native protocol mocks](https://mockarty.ru/docs/native-protocol-mocks)
- [Deployment options](deploy/README.md) · [Report a reproducible issue](https://github.com/mockarty/releases/issues)
- [Software licence](LICENSE) · [Community terms](https://mockarty.ru/legal/community) · [Mobile policies and account deletion](https://mockarty.ru/legal)

---

## Mockarty Community на русском

[English](#mockarty-community) · [Скачать](https://mockarty.ru/downloads) · [Документация](https://mockarty.ru/docs) · [Релизы GitHub](https://github.com/mockarty/releases/releases)

> Первый релиз Community готовится. Инструкции ниже подходят для версий, в описании которых указана поддержка Community Server. Предыдущие предварительные сборки и их описания доступны в Releases.

В Mockarty можно создавать моки API, запускать тесты и делиться результатами с командой. **Community Server работает на вашем компьютере или сервере без лицензионного ключа и учётной записи Mockarty Cloud.** Доступны десять пространств и пять мест в каждом из семи модулей: моки, тестирование, надёжность, базовый фаззинг, управление тестированием, управление работой и тестирование контрактов.

Ручной харнесс в Desktop и подключение раннеров к Community Server доступны без лицензии Enterprise. Движок автономных миссий требует Enterprise. Доступ к командному пространству определяется ролями на выбранном сервере.

### С чего начать

| Задача | Куда идти |
|---|---|
| Работать на своём компьютере | Установите **Mockarty Desktop** со [страницы загрузок](https://mockarty.ru/downloads) и выберите локальный режим. Позже можно подключить командный сервер или [Mockarty Cloud](https://cloud.mockarty.ru/). |
| Запустить Community Server на своём компьютере | Запустите [один контейнер](#запуск-community-server-в-docker) или [серверный бинарник](#запуск-серверного-бинарника). |
| Развернуть сервер для команды | Используйте [руководство по развёртыванию](https://mockarty.ru/docs/deployment-guide) и передайте команде HTTPS-адрес. |
| Подключить кодер-раннер | Выберите опубликованный `mockarty-coder-runner` для своей платформы на [странице загрузок](https://mockarty.ru/downloads?lang=ru#coding-runner), сверьте контрольную сумму и [настройте доверие к серверу](https://mockarty.ru/docs/ai-components?lang=ru#настройка-доверия). |
| Работать с Android | Выберите подписанный **Mockarty Mobile** на [странице загрузок Android](https://mockarty.ru/downloads?lang=ru#android-mobile) и подключитесь к серверу или аккаунту Cloud, где работает команда. **Companion** позволяет подключить устройство для тестирования. |

### Запуск Community Server в Docker

Клонируйте этот репозиторий релизов. Используйте тег образа из [описания релиза](https://github.com/mockarty/releases/releases), для которого опубликован образ сервера. Выполните команды из каталога репозитория в Bash или Zsh.

```bash
printf 'Тег опубликованного Docker-образа (без v): '
read -r MOCKARTY_VERSION
export MOCKARTY_VERSION
printf 'Задайте надёжный пароль для admin: '
read -rs FIRST_ADMIN_PASSWORD
printf '\n'
export FIRST_ADMIN_PASSWORD
docker compose -f deploy/docker-compose.community.yml up -d
```

Откройте <http://127.0.0.1:5770/ui/> и войдите как `admin` с паролем, который вы ввели. При первом входе прочитайте и примите условия Community.

Пример привязывает HTTP-порт к loopback и сохраняет SQLite в томе `community_data`. Чтобы остановить сервер, выполните `docker compose -f deploy/docker-compose.community.yml stop` в том же терминале. Сохраните том, чтобы сохранить данные. Для общего сервера настройте HTTPS по руководству по развёртыванию.

### Запуск серверного бинарника

Выберите [релиз GitHub](https://github.com/mockarty/releases/releases), в котором указана поддержка Community Server и есть `mockarty` для вашей ОС и процессора. Сверьте SHA‑256 файла с его строкой в `checksums.txt` того же релиза. На macOS и Linux назовите файл `mockarty`, откройте его каталог и выполните:

```bash
chmod +x ./mockarty
printf 'Задайте надёжный пароль для admin: '
read -rs FIRST_ADMIN_PASSWORD
printf '\n'
export FIRST_ADMIN_PASSWORD
DB_USE=sqlite DB_DSN="$PWD/mockarty.db" CACHE_TYPE=inmemory \
  HTTP_BIND_ADDR=127.0.0.1 HTTP_PORT=5770 AUTH_BASE_URL=http://127.0.0.1:5770 \
  COOKIE_SECURE=false FIRST_ADMIN_LOGIN=admin ./mockarty
```

Откройте <http://127.0.0.1:5770/ui/>, войдите как `admin` с паролем, который вы ввели, и примите условия Community.

### Первый мок

1. В **Конструкторе** выберите пространство `sandbox`. Укажите ID `my-first-mock`, протокол HTTP, метод `GET` и маршрут `/hello`.
2. Задайте статус `200`, тип `application/json` и тело `{"message":"Hello from Mockarty"}`. Сохраните мок.
3. Вызовите адрес ниже. В ответе появится сохранённое сообщение.

```bash
curl --fail --silent --show-error http://127.0.0.1:5770/stubs/sandbox/hello
```

Затем откройте [руководство по API-тестированию](https://mockarty.ru/docs/quick-start?lang=ru#api-testing), чтобы сохранить запрос в коллекцию и поделиться результатом прогона с коллегой.

Для моков gRPC, WebSocket, брокеров и сокетов используйте [встроенные обработчики Mockarty](https://mockarty.ru/docs/native-protocol-mocks?lang=ru). Для тестов и CI доступны [релизы CLI](https://github.com/mockarty/mockarty-cli/releases) и [SDK](docs/SDKS.md#sdk-на-русском).

### Руководства и лицензия

- [Админ-нода](docs/ADMIN_GUIDE.md) · [Mock Resolver](docs/RESOLVER_GUIDE.md) · [Тестовый раннер](docs/RUNNER_GUIDE.md) · [Нативные протокольные моки](https://mockarty.ru/docs/native-protocol-mocks)
- [Варианты развёртывания](deploy/README.md) · [Сообщить о воспроизводимой проблеме](https://github.com/mockarty/releases/issues)
- [Лицензия](LICENSE) · [Условия Community](https://mockarty.ru/legal/community) · [Мобильные политики и удаление аккаунта](https://mockarty.ru/legal)
