# Docker Compose

[Русская версия](#docker-compose-на-русском) · [Deployment options](README.md)

Use [docker-compose.community.yml](docker-compose.community.yml) for a single keyless Community server with SQLite. The [Community Server quick start](../README.md#start-community-server-with-docker) shows how to start and stop it while keeping your data.

For PostgreSQL, use [docker-compose.minimal.yml](docker-compose.minimal.yml). For distributed mock serving and tests, see [docker-compose.distributed.yml](docker-compose.distributed.yml). Check the variables in [.env.example](.env.example) and use the images and tags listed with the selected [Mockarty release](https://github.com/mockarty/releases/releases).

Start the admin node and create the required integration tokens before starting resolver or runner services. See the [deployment guide](https://mockarty.ru/docs/deployment-guide) for shared installations.

## Docker Compose на русском

[Варианты развёртывания](README.md#развёртывание-mockarty)

Для одной ноды Community без ключа с SQLite используйте [docker-compose.community.yml](docker-compose.community.yml). В [быстром старте Community Server](../README.md#запуск-community-server-в-docker) показано, как запустить и остановить сервер с сохранением данных.

Для PostgreSQL используйте [docker-compose.minimal.yml](docker-compose.minimal.yml). Для распределённых моков и тестов см. [docker-compose.distributed.yml](docker-compose.distributed.yml). Проверьте переменные в [.env.example](.env.example) и используйте образы и теги из выбранного [релиза Mockarty](https://github.com/mockarty/releases/releases).

Запустите админ-ноду и выпустите необходимые токены интеграции перед запуском резолвера или раннера. Общие установки описаны в [руководстве по развёртыванию](https://mockarty.ru/docs/deployment-guide?lang=ru).
