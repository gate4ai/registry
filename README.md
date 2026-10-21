# gate4ai registry

Индекс магазина коннекторов [gate4.ai](https://github.com/gate4ai/gate4ai): какие коннекторы есть, кто их издатель, какие версии проверены и какие отозваны. Хост (`gate4ai connectors install|update|refresh`) скачивает `index.json`, проверяет его подпись `index.json.sig` ключом registry из конфигурации (`store.registry_keys`) и только потом верит его содержимому.

## Формат `index.json`

```json
{
  "version": 1,
  "publishers": {
    "gate4ai": { "keys": ["ed25519:<base64>"], "trust": "verified" }
  },
  "connectors": {
    "email": {
      "publisher": "gate4ai",
      "repo": "https://github.com/gate4ai/gate4ai",
      "path": "connectors/mail",
      "license": "AGPL-3.0-only",
      "description": "Email over IMAP/SMTP",
      "versions": [
        {
          "version": "0.7.0",
          "url": "https://github.com/gate4ai/gate4ai/releases/download/v0.7.0/email-0.7.0.g4c",
          "sha256": "<sha256 файла пакета>",
          "digest": "<дайджест пакета, подписанный издателем>",
          "capabilities": ["network tls://${setting:imap.host}:${setting:imap.port}", "…"]
        }
      ]
    }
  },
  "revoked": [
    { "connector": "email", "version": "0.6.9", "reason": "…" }
  ]
}
```

| Поле | Смысл |
|---|---|
| `publishers.*.trust` | `verified` — проверен gate4.ai (ревью кода и возможностей); `publisher` — проверенный издатель; `community` — сообщество (предупреждение, доступны только действия `read`) |
| `connectors.*.repo` | исходник обязателен: пакет должен воспроизводимо собираться из него по тегу `v<version>` |
| `connectors.*.path` | каталог коннектора в `repo`, если он не в корне (официальные коннекторы — в `connectors/<имя>` репозитория `gate4ai/gate4ai`) |
| `versions[].capabilities` | возможности, которые прошли ревью; пакет с другими возможностями не установится |
| `versions[].sha256` / `digest` | хэш файла пакета и подписанный издателем дайджест (`gate4ai connectors pack` печатает оба) |
| `revoked` | отозванные версии (`"version": "*"` — все); хост отключает их у всех, кто обновил индекс |

## Публикация версии коннектора

Официальные коннекторы выпускаются вместе с gate4ai: один тег — один релиз, и все их версии попадают сюда одним pull request. Порядок для сторонних издателей (ключ, релиз, `gate4ai registry add`, pull request, ревью, уровни доверия) — в [CONTRIBUTING.md](CONTRIBUTING.md). CI pull request'а скачивает каждый пакет и сверяет с записью, а новые версии пересобирает из исходников; после слияния владелец registry подписывает индекс.

## Лицензия

Индекс — [CC0 1.0](LICENSE): его можно копировать, зеркалировать и использовать без условий. Подпись `index.json.sig` удостоверяет, что индекс выпущен gate4.ai: хост принимает только индекс, подписанный ключом registry. У коннекторов из индекса — свои лицензии, указанные в поле `license`; CC0 на них не распространяется.
