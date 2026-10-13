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
      "repo": "https://github.com/gate4ai/gate4ai-mail",
      "license": "AGPL-3.0-only WITH gate4ai host exception",
      "description": "Email over IMAP/SMTP",
      "versions": [
        {
          "version": "2",
          "url": "https://github.com/gate4ai/gate4ai-mail/releases/download/v2/email-2.g4c",
          "sha256": "<sha256 файла пакета>",
          "digest": "<дайджест пакета, подписанный издателем>",
          "capabilities": ["network tls://${setting:imap.host}:${setting:imap.port}", "…"]
        }
      ]
    }
  },
  "revoked": [
    { "connector": "email", "version": "1", "reason": "…" }
  ]
}
```

| Поле | Смысл |
|---|---|
| `publishers.*.trust` | `verified` — проверен gate4.ai (ревью кода и возможностей); `publisher` — проверенный издатель; `community` — сообщество (предупреждение, доступны только действия `read`) |
| `connectors.*.repo` | исходник обязателен: пакет должен воспроизводимо собираться из него |
| `versions[].capabilities` | возможности, которые прошли ревью; пакет с другими возможностями не установится |
| `versions[].sha256` / `digest` | хэш файла пакета и подписанный издателем дайджест (`gate4ai connectors pack` печатает оба) |
| `revoked` | отозванные версии (`"version": "*"` — все); хост отключает их у всех, кто обновил индекс |

## Публикация версии коннектора

1. Издатель собирает и подписывает пакет в CI репозитория коннектора: `gate4ai connectors pack . --key "$PUBLISHER_KEY" -o <name>-<version>.g4c` и прикладывает его к релизу.
2. Pull request в этот репозиторий добавляет версию в `index.json` (`url`, `sha256`, `digest`, `capabilities` — из вывода `pack`).
3. Ревью: код, возможности (изменились ли по сравнению с прошлой версией, нет ли подозрительных доменов, у `money` — независимая проверка другим коннектором).
4. После слияния владелец registry подписывает индекс: `gate4ai registry sign index.json --key registry.key`; CI проверяет: `gate4ai registry check index.json --key ed25519:<ключ registry>`.

## Лицензия

Данные индекса — [CC0 1.0](LICENSE).
