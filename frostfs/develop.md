Контейнер в frostfs = бакет в s3

# Запустить frostfs локально 

1. Форкаем и клонируем:

* https://github.com/TrueCloudLab/frostfs-node

* https://github.com/TrueCloudLab/frostfs-contract

* https://github.com/TrueCloudLab/frostfs-dev-env

2. В https://github.com/TrueCloudLab/frostfs-contract выполняем генерацию контрактов (см. README.md в репе)

3. В https://github.com/TrueCloudLab/frostfs-node выполняем сборку: `make all`

4. В https://github.com/TrueCloudLab/frostfs-dev-env:

    1. Меняем `.env` файл (раскомментить FROSTFS_*_PATH и указать пути):

    ````yaml
    # FrostFS CLI binary
    FROSTFS_CLI_URL=https://http.t5.fs.neo.org/AQgse8bPCZx4zScMuAKxowJdZPbKHp8NDcp15o6VUNmk/C6BNLpYg5gWLHp3DrXozSxxGLDahBuSBCyJoYSSR1M3Q
    FROSTFS_CLI_PATH=/home/dstepanov/src/frostfs-node/bin/frostfs-cli

    # FrostFS ADM tool binary
    FROSTFS_ADM_VERSION=e3554425
    FROSTFS_ADM_URL=https://http.t5.fs.neo.org/AQgse8bPCZx4zScMuAKxowJdZPbKHp8NDcp15o6VUNmk/sXZxy9vbFyJiLhN9qTSXozXK7SN9H8ZC6dpvAt59Zaj
    FROSTFS_ADM_PATH=/home/dstepanov/src/frostfs-node/bin/frostfs-adm

    # Compiled FrostFS Smart Contracts
    FROSTFS_CONTRACTS_VERSION=4f3c08f5
    FROSTFS_CONTRACTS_URL=https://http.t5.fs.neo.org/AQgse8bPCZx4zScMuAKxowJdZPbKHp8NDcp15o6VUNmk/c1nGtturFrSeygYP3AyNHDDLNbs7HhJiH2BQkgZxEmZ
    FROSTFS_CONTRACTS_PATH=/home/dstepanov/src/frostfs-contract
    ````

    2. Запускаем `make up`

# Подебажиться (vs code)

1. В форке https://github.com/TrueCloudLab/frostfs-node:

    1. Создаем отдельный dockerfile с именем, например, `.docker/Dockerfile.storage-debug`, в который собираем frostfs-node и dlv. Пример:

```yaml
FROM golang:1.20 as builder
ARG BUILD=now
ARG VERSION=dev
ARG REPO=repository
WORKDIR /src
COPY . /src

RUN go install github.com/go-delve/delve/cmd/dlv@latest
RUN CGO_ENABLED=0 go build -v -trimpath -ldflags "-X $(REPO)/misc.Version=$(VERSION)" -gcflags="all=-N -l" -o bin/frostfs-node ./cmd/frostfs-node

# Executable image
FROM debian:buster AS frostfs-node

EXPOSE 8000 2345

WORKDIR /

COPY --from=builder /src/bin/frostfs-node /bin/frostfs-node
COPY --from=builder /go/bin/dlv /bin/dlv

CMD ["frostfs-node"]
```



    2. Собираем докер образ: `make image-storage-debug`

2. В форке https://github.com/TrueCloudLab/frostfs-dev-env:

    1. Меняем `services/storage/docker-compose.yml`, в описание сервиса storage01 добавляем/заменяем:

```yaml
    expose:
      - "2345"
    ports:
      - "2345:2345"
    security_opt:
      - apparmor=unconfined
    cap_add:
      - SYS_PTRACE

    command: ["dlv", "--listen=:2345", "--headless=true", "--api-version=2", "--accept-multiclient", "exec", "/bin/frostfs-node", "--","--config", "/etc/frostfs/storage/config.yml"]
```



    2. Меняем `.env`:

```yaml
# FrostFS Storage nodes
NODE_VERSION=<версия образа из пункта 1>
NODE_IMAGE=truecloudlab/frostfs-storage-debug
```



    3. Разворачиваем окружение: `make up`

3. В форке https://github.com/TrueCloudLab/frostfs-node:

    3.1 Создаем/добавляем конфигурацию для удалённого подключения к дебаггеру (файл .vscode/launch.json):

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "dev env: frostfs-node",
            "type": "go",
            "request": "attach",
            "mode": "remote",
            "port": 2345,
            "host": "s01.frostfs.devenv",
            "showLog": true
        }
    ]
}
```



    3.2 Подключаемся: Run and Debug -> выбрать конфигурацию dev env: frostfs-node -> Start debugging