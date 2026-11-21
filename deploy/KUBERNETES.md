# Kubernetes

[Русская версия](#kubernetes-на-русском) · [Deployment options](README.md)

If your Mockarty distribution includes a Helm chart, use the chart and image versions listed together with that distribution. Check the settings before installation:

```bash
helm show values "$MOCKARTY_HELM_CHART"
```

Set `MOCKARTY_HELM_CHART` to the path of the chart you received. Use the values from the chart to configure the admin node, PostgreSQL, your ingress and any optional resolver or runner pools. Review the [deployment guide](https://mockarty.ru/docs/deployment-guide) and [scaling guide](https://mockarty.ru/docs/scaling-architecture) before sharing access.

## Kubernetes на русском

[Варианты развёртывания](README.md#развёртывание-mockarty)

Если в комплект поставки входит Helm-чарт, используйте версии чарта и образов из этого комплекта. Перед установкой проверьте настройки командой выше. Переменная `MOCKARTY_HELM_CHART` должна указывать путь к полученному чарту.

Используйте параметры чарта для настройки админ-ноды, PostgreSQL, ingress и дополнительных пулов резолверов и раннеров. Перед предоставлением доступа см. [руководство по развёртыванию](https://mockarty.ru/docs/deployment-guide?lang=ru) и [руководство по масштабированию](https://mockarty.ru/docs/scaling-architecture?lang=ru).
