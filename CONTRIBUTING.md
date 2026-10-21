# Как опубликовать коннектор в магазине gate4.ai

Магазин — это `index.json` в этом репозитории, подписанный ключом registry. Хост ставит коннектор, только если индекс подписан, пакет подписан ключом издателя из индекса, совпадают sha256 и дайджест, а возможности пакета — те, что прошли ревью. Попасть в индекс можно только через pull request и ревью; автоматически, в том числе для коннекторов gate4ai, ничего не публикуется.

## 1. Коннектор

- Пишется на [sdk-go](https://github.com/gate4ai/gate4ai/tree/main/sdk-go) (`require github.com/gate4ai/gate4ai`, импорт `github.com/gate4ai/gate4ai/sdk-go/...`) и собирается в WASM-модуль; заготовка — `gate4ai connectors new`, образец — [connectors/mail](https://github.com/gate4ai/gate4ai/tree/main/connectors/mail) (код, `manifest.yaml`, тесты на фейковом хосте `hosttest`).
- Нативные пакеты (`runtime: native`: отдельный процесс без песочницы, с правами пользователя) магазин пока принимает только от gate4.ai. Если вы хотите создать такой пакет, свяжитесь с нами — мы к этому благосклонны. WASM-коннектор может пользоваться нативными через `service_call` (например, браузером — [connectors/browser](https://github.com/gate4ai/gate4ai/tree/main/connectors/browser)).
- Исходники открыты, лицензия совместима с AGPL-3.0 (рекомендуем AGPL-3.0, как у коннекторов gate4ai, или пермиссивную).
- Имя репозитория — `gate4ai-<имя>` (коннектор в корне или в каталоге — тогда `--path`); `version` в `manifest.yaml` растёт с каждым выпуском, тег — `v<version>`.
- Возможности в манифесте — минимально нужные: каждую увидит владелец при установке и ревьюер в pull request.

## 2. Ключ издателя

```sh
gate4ai connectors keygen -o publisher.key   # печатает открытый ключ ed25519:…
```

Приватный ключ — в секрет репозитория коннектора (`GATE4AI_PUBLISHER_KEY`), в git он не попадает. Этим ключом подписаны все ваши пакеты; смена ключа — отдельный pull request с объяснением, ревьюер проверяет, что его делает тот же издатель.

## 3. Релиз

Workflow на тег `v*` собирает модуль, подписывает пакет и создаёт релиз с `<connector>-<version>.g4c` и `pack.json`:

```yaml
on: { push: { tags: ["v*"] } }
permissions: { contents: write }
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      # Сторонние actions — по полному SHA (тег — в комментарии), gate4ai — конкретной версии:
      # пакет пересобирается из тега той же версией, иначе байты могут не совпасть.
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - uses: actions/setup-go@b7ad1dad31e06c5925ef5d2fc7ad053ef454303e # v7.0.0
        with: { go-version-file: go.mod }
      - run: go install github.com/gate4ai/gate4ai/cmd/gate4ai@v0.7.2
      - env: { PUBLISHER_KEY: "${{ secrets.GATE4AI_PUBLISHER_KEY }}" }
        run: |
          umask 077; printf '%s\n' "$PUBLISHER_KEY" > "$RUNNER_TEMP/publisher.key"
          gate4ai connectors pack . --key "$RUNNER_TEMP/publisher.key" | tee pack.json
      - run: gh release create "$GITHUB_REF_NAME" *.g4c pack.json --generate-notes
        env: { GH_TOKEN: "${{ github.token }}" }
```

Сборка воспроизводима: `pack` собирает Go из `go.mod` (`toolchain`, иначе `go`) без сведений о git, поэтому любой может пересобрать пакет из тега и получить те же байты:

```sh
gate4ai connectors verify <каталог исходников> <connector>-<version>.g4c
```

## 4. Pull request

Форкните этот репозиторий и добавьте версию командой (не руками):

```sh
gh release download v1 -R acme/gate4ai-hello -p pack.json
gate4ai registry add index.json pack.json --publisher acme \
  --repo https://github.com/acme/gate4ai-hello --license AGPL-3.0-only --description "…"
```

`--repo`, `--path` (каталог коннектора, если не корень), `--license` и `--description` нужны только для нового коннектора; адрес пакета по умолчанию — релиз `v<version>` на GitHub (иначе `--url`). Новый издатель попадает в индекс с уровнем `community` и ключом, которым подписан пакет. `index.json.sig` не меняйте.

CI pull request'а скачивает каждый пакет и сверяет его с записью, а новые версии пересобирает из тега `v<version>` (в каталоге `path`) и сравнивает с опубликованным пакетом. CI работает той версией gate4ai, что записана в `.github/gate4ai-version` (её обновляет релиз gate4ai), а не `@latest`; соберите пакет той же или совместимой версией. Ещё CI проверяет, что в индексе есть официальные коннекторы gate4ai (`email`, `browser`) с версией этого релиза. У нативного пакета запись ссылается на список платформ `<connector>-<version>.platforms.json`, и CI пересобирает пакет под каждую платформу списка.

## 5. Ревью и уровни доверия

Ревьюер смотрит код, возможности (что изменилось с прошлой версии, нет ли лишних доменов и секретов), для действий класса `money` — независимую проверку результата. После слияния владелец registry подписывает индекс:

```sh
gate4ai registry sign index.json --key registry.key
```

| Уровень | Кто назначает | Что видит пользователь |
|---|---|---|
| `community` | по умолчанию для нового издателя | предупреждение; доступны только действия `read` |
| `publisher` | ревьюер, после проверки личности издателя | обычная установка с согласием на возможности |
| `verified` | ревьюер, после ревью кода gate4.ai | обычная установка; отметка «проверен gate4.ai» |

## Лицензия записи

Присылая запись в `index.json` (в том числе описание коннектора и сведения об издателе), вы передаёте её в общественное достояние по [CC0 1.0](LICENSE), как и весь индекс. На сам коннектор это не распространяется: его лицензия — та, что указана в `license` и в его репозитории.

## Без магазина

- Свой пакет у себя: `gate4ai connectors install hello-1.g4c --dev`.
- Доверять издателю напрямую: его ключ в `store.trusted_keys` конфигурации.
- Свой registry: форк этого репозитория со своим ключом, у пользователей — `store.registry` и `store.registry_keys`.
