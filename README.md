# gate4ai registry

Индекс магазина коннекторов [gate4.ai](https://github.com/gate4ai/gate4ai): какие коннекторы есть, кто их издатель, какие версии проверены и какие отозваны. Хост (`gate4ai connectors install|update|refresh`) скачивает `index.json`, проверяет его подпись `index.json.sig` ключом registry из конфигурации (`store.registry_keys`) и только потом верит его содержимому.

## Формат `index.json`

Браузер и Файлы в индексе не лежат: они встроены в gate4ai (ставить и удалять нечего). В магазине — только WASM-коннекторы к сайтам и сервисам: у них нет своей сети, на сайты они ходят только через встроенный Браузер (`service_call`), к файлам — только через встроенные Файлы, в границах `sites`/`folders` своего манифеста и списков доступа владельца. Пример записи (вымышленный коннектор стороннего издателя):

```json
{
  "version": 1,
  "updated": "2026-10-24",
  "publishers": {
    "gate4ai": { "keys": ["ed25519:<base64>"], "trust": "verified" },
    "acme":    { "keys": ["ed25519:<base64>"], "trust": "community" }
  },
  "connectors": {
    "utilities": {
      "publisher": "acme",
      "repo": "https://github.com/acme/gate4ai-utilities",
      "license": "AGPL-3.0-only",
      "description": "Pays utility bills in the personal account on account.example.org",
      "versions": [
        {
          "version": "1",
          "url": "https://github.com/acme/gate4ai-utilities/releases/download/v1/utilities-1.g4c",
          "sha256": "<sha256 файла пакета>",
          "digest": "<дайджест пакета, подписанный издателем>",
          "capabilities": ["call browser (contract 1): act, observe, open", "site account.example.org: allow", "…"]
        }
      ]
    }
  },
  "revoked": [
    { "connector": "utilities", "version": "0", "reason": "…" }
  ]
}
```

| Поле | Смысл |
|---|---|
| `publishers.*.trust` | `verified` — проверен gate4.ai (ревью кода и возможностей); `publisher` — проверенный издатель; `community` — сообщество (предупреждение, доступны только действия `read`) |
| `connectors.*.repo` | исходник обязателен: пакет должен воспроизводимо собираться из него по тегу `v<version>` |
| `connectors.*.path` | каталог коннектора в `repo`, если он не в корне |
| `versions[].capabilities` | возможности, которые прошли ревью (сайты и папки манифеста, учётки, вызовы Браузера и Файлов, подвызовы действий); пакет с другими возможностями не установится |
| `versions[].sha256` / `digest` | хэш файла пакета и подписанный издателем дайджест (`gate4ai connectors pack` печатает оба) |
| `revoked` | отозванные версии (`"version": "*"` — все); хост отключает их у всех, кто обновил индекс |

## Публикация версии коннектора

Порядок для издателей (ключ, релиз, `gate4ai registry add`, pull request, ревью, уровни доверия) — в [CONTRIBUTING.md](CONTRIBUTING.md). CI pull request'а скачивает каждый пакет и сверяет с записью, а новые версии пересобирает из исходников; после слияния владелец registry подписывает индекс. Версия gate4ai, которой CI проверяет индекс, записана в `.github/gate4ai-version`; её меняет релиз gate4ai (`dev/scripts/deploy.sh registry`). Сам gate4ai пакетов в магазин не выпускает: Браузер и Файлы встроены в него, пустой список `connectors` — нормальное состояние индекса.

## Лицензия

Индекс — [CC0 1.0](LICENSE): его можно копировать, зеркалировать и использовать без условий. Подпись `index.json.sig` удостоверяет, что индекс выпущен gate4.ai: хост принимает только индекс, подписанный ключом registry. У коннекторов из индекса — свои лицензии, указанные в поле `license`; CC0 на них не распространяется.
