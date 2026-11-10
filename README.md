# b3-price-alert

Serviço que monitora preços de ações da B3 e dispara alertas quando um ativo cruza
o preço definido pelo usuário.

> Status: bootstrap. Ainda não há regra de preço, scheduler nem integração com a brapi.

## Stack

| Item | Versão |
|---|---|
| Java | 21 |
| Spring Boot | 4.1.2 |
| Build | Maven Wrapper (`./mvnw`) |
| Banco | PostgreSQL 16 (container) |
| Container | Docker + Docker Compose v2 |

## Pré-requisitos

- JDK 21 (`java -version` deve mostrar 21)
- Docker com o plugin `compose` (`docker compose version`)
- Seu usuário no grupo `docker` — sem isso todo comando precisa de `sudo`:
  ```bash
  sudo usermod -aG docker $USER   # depois faça logout e login
  ```

## Configuração

O projeto usa variáveis de ambiente. Crie o arquivo local a partir do exemplo:

```bash
cp .env.example .env
```

E preencha com valores seus. O `.env` está no `.gitignore` e **não** deve ser commitado.

> Sem aspas no valor da senha. O Docker Compose até aceita `'senha'`, mas o
> `spring.config.import` de `.env` (se usado) e o Maven não removem as aspas.

## Rodar a stack completa (app + banco)

```bash
docker compose up -d --build
```

O `--build` recompila o código dentro da imagem. Sem ele, o Compose reaproveita a
imagem já existente e você roda um jar desatualizado.

Verificando:

```bash
docker compose ps                                 # os dois serviços Up
docker compose logs --tail=30 b3-price-alert      # até "Started B3PriceAlertApplication"
curl localhost:8080/actuator/health               # {"status":"UP"}
```

## Rodar a aplicação na IDE (desenvolvimento)

O ciclo na IDE é mais rápido; use o container apenas para o banco:

```bash
docker compose up -d postgres
```

Depois execute `B3PriceAlertApplication` na IDE com estas variáveis de ambiente
(Run Configuration → Environment variables):

```
POSTGRES_DB=b3-price-alert
POSTGRES_USER=b3
POSTGRES_PASSWORD=<a mesma do .env>
```

A aplicação usa `localhost:5430`, que é a porta do Postgres publicada no host.

> Não rode a IDE e o container do app ao mesmo tempo: os dois usam a porta 8080.

## Portas

| Porta no host | Para onde vai | Uso |
|---|---|---|
| 8080 | app:8080 | API e actuator |
| 5430 | postgres:5432 | acesso ao banco de fora do Docker (IDE, DBeaver, psql) |

Dentro da rede do Compose, o app fala com o banco por `postgres:5432` — o nome do
serviço, não `localhost`. O `5430` existe só do lado do host.

## Variáveis de ambiente

| Variável | Usada por | Descrição |
|---|---|---|
| `POSTGRES_DB` | Postgres + Spring | nome do banco criado no primeiro start |
| `POSTGRES_USER` | Postgres + Spring | usuário criado no primeiro start |
| `POSTGRES_PASSWORD` | Postgres + Spring | senha do usuário |
| `SPRING_DATASOURCE_URL` | Spring | definida no `docker-compose.yaml`, aponta para `postgres:5432` |

O `POSTGRES_USER` e o `POSTGRES_DB` só têm efeito quando o volume do banco está
**vazio**. Se você mudá-los e o banco já foi inicializado, é preciso recriar o
volume:

```bash
docker compose down -v
```

## Testes

Os testes sobem o contexto do Spring com JPA, então precisam de um banco
acessível. Com o Postgres do Compose no ar:

```bash
bash -c 'set -a; source .env; set +a; ./mvnw test'
```

O `source` exporta o `.env` para o processo do Maven. Sem isso, as variáveis
`${POSTGRES_*}` do `application.yaml` ficam literais e o teste falha com
`Unable to determine Dialect without JDBC metadata` — que é consequência, não a
causa.

## Estrutura

```
src/main/java/io/github/cauapincellicacchero/b3pricealert/
src/main/resources/application.yaml
Dockerfile              # multi-stage: build com Maven, runtime só com JRE
docker-compose.yaml
.env.example
```

## Solução de problemas

**`permission denied ... /var/run/docker.sock`** — seu usuário não está no grupo
`docker`. Veja Pré-requisitos.

**Container aparece `Created` e não `Up`** — o start falhou. `docker compose ps -a`
mostra o estado, `docker compose logs <serviço>` mostra o motivo. Causa comum é
porta já ocupada.

**`bind: address already in use` na 5430** — existe outro Postgres rodando.
Descubra com `docker ps --format '{{.Names}}\t{{.Ports}}'`.

**App sobe e cai com `Unable to determine Dialect`** — falha de conexão com o
banco, não configuração de JPA. Procure o primeiro `Caused by` com `PSQLException`
no log.

**Log de um container que nunca subiu vem vazio** — `Created` não gera log. Não é
falta de configuração de logging, é container que não rodou.

## Licença

MIT.
