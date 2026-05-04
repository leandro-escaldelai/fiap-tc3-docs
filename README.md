# Tech Challenge — Plataforma de Revenda de Veículos

Plataforma de revenda de veículos automotores construída como conjunto de microsserviços em **ASP.NET Core (C#, .NET 8)**, seguindo Clean Architecture e Arquitetura Hexagonal, comunicação híbrida (REST + RabbitMQ) e Kong como API Gateway.

## Repositórios neste workspace

| Pasta | Conteúdo |
|---|---|
| `src/UserApi/` | Microsserviço de Usuários e autenticação (MongoDB + Redis + JWT RS256) |
| `src/VehicleApi/` | Microsserviço de Veículos (MongoDB + RabbitMQ consumer) |
| `src/SalesApi/` | Microsserviço de Vendas (PostgreSQL + RabbitMQ publisher + REST → VehicleApi) |
| `infrastructure/` | Docker Compose, Kong, scripts de chaves JWT, deploy CI |
| `docs/` | Documentação de arquitetura, modelagem, event storming e CI/CD |

Cada microsserviço é tratado como um repositório autônomo (possui sua própria solução, Dockerfile, scripts de inicialização do seu banco e workflow de CI em `.github/workflows/ci.yml`). A pasta `infrastructure/` corresponde ao repositório de infraestrutura.

## Arquitetura resumida

```
                    ┌─────────────┐
   Apps / Web ──▶  │   Kong API  │
                    │   Gateway   │
                    └──────┬──────┘
            ┌──────────────┼─────────────┐
            ▼              ▼             ▼
       UserApi        VehicleApi      SalesApi
       (Mongo+Redis)  (Mongo)         (Postgres)
            │              ▲             │
            │              │ VehicleSold │
            │              └──── RabbitMQ ◀──┘
            │
            └─ Publica JWKS ─▶ usado por Vehicle/Sales para validar JWT
```

- **Comunicação síncrona**: SalesApi consulta VehicleApi (REST) ao iniciar uma venda.
- **Comunicação assíncrona**: SalesApi publica `VehicleSold` em RabbitMQ; VehicleApi consome e marca o veículo como vendido.
- **Autenticação distribuída**: UserApi emite JWT assinado com RS256 (chave privada). VehicleApi e SalesApi validam usando a chave pública. Endpoint `/.well-known/jwks.json` exposto pelo UserApi.

Mais detalhes em `docs/04 - arquitetura.md`.

## Pré-requisitos

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) com Compose v2
- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0) (apenas se for rodar/testar localmente sem Docker)
- OpenSSL no PATH (apenas para gerar as chaves JWT na primeira vez)

## Subindo a solução completa (ponta a ponta)

```bash
# 1) Gere o par de chaves JWT (somente uma vez)
cd infrastructure/certs
./generate-keys.sh           # Linux/macOS
# ou
./generate-keys.ps1          # Windows PowerShell

# 2) (opcional) Copie e ajuste o .env
cd ..
cp .env.example .env

# 3) Suba o ambiente
docker compose up -d --build
```

Isso provisiona, na ordem correta de saúde:

1. MongoDB (Users e Vehicles), PostgreSQL (Sales) e Redis
2. RabbitMQ
3. UserApi, VehicleApi, SalesApi
4. Kong API Gateway

Endpoints expostos no host:

| Componente | Porta direta | Via Kong |
|---|---|---|
| UserApi | http://localhost:5001 | http://localhost:8000 |
| VehicleApi | http://localhost:5002 | http://localhost:8000 |
| SalesApi | http://localhost:5003 | http://localhost:8000 |
| RabbitMQ Management | http://localhost:15672 (guest/guest) | — |
| Kong Admin | http://localhost:8001 | — |
| Swagger UI | `/swagger` em cada microsserviço | — |

## Fluxo de demonstração ponta a ponta

```bash
GW=http://localhost:8000

# 1) Cadastro do comprador
curl -s -X POST $GW/users -H "Content-Type: application/json" -d '{
  "nome":"Maria Compradora","cpf":"39053344705","sexo":"F",
  "dataNascimento":"1990-01-01","email":"maria@example.com","senha":"senha123"
}'

# 2) Cadastro do proprietário
curl -s -X POST $GW/users -H "Content-Type: application/json" -d '{
  "nome":"João Proprietário","cpf":"45317828791","sexo":"M",
  "dataNascimento":"1985-05-12","email":"joao@example.com","senha":"senha123"
}'

# 3) Login do proprietário
TOKEN_OWNER=$(curl -s -X POST $GW/auth/login -H "Content-Type: application/json" \
  -d '{"email":"joao@example.com","senha":"senha123"}' | jq -r .accessToken)

# 4) Cadastro de veículo (proprietário autenticado)
VEHICLE_ID=$(curl -s -X POST $GW/vehicles -H "Authorization: Bearer $TOKEN_OWNER" \
  -H "Content-Type: application/json" -d '{
    "marca":"Toyota","modelo":"Corolla","ano":2022,"cor":"Prata","preco":95000
  }' | jq -r .id)

# 5) Listagem de veículos disponíveis (público, ordenado por preço asc)
curl -s $GW/vehicles/available

# 6) Login do comprador
TOKEN_BUYER=$(curl -s -X POST $GW/auth/login -H "Content-Type: application/json" \
  -d '{"email":"maria@example.com","senha":"senha123"}' | jq -r .accessToken)

# 7) Iniciar venda (comprador autenticado)
SALE_ID=$(curl -s -X POST $GW/sales -H "Authorization: Bearer $TOKEN_BUYER" \
  -H "Content-Type: application/json" -d "{\"idVeiculo\":\"$VEHICLE_ID\",\"proposta\":92000}" | jq -r .id)

# 8) Concluir venda (proprietário autenticado) → publica VehicleSold
curl -s -X POST $GW/sales/$SALE_ID/complete -H "Authorization: Bearer $TOKEN_OWNER"

# 9) Listagem de veículos vendidos (após o consumer atualizar)
sleep 2 && curl -s $GW/vehicles/sold
```

## Rodando os testes localmente

Cada microsserviço possui sua própria solution. A partir da raiz:

```bash
dotnet test src/UserApi/UserApi.sln
dotnet test src/VehicleApi/VehicleApi.sln
dotnet test src/SalesApi/SalesApi.sln
```

Os testes cobrem **Domain Models** e **UseCases** sem dependência de infraestrutura externa, conforme exigido em `docs/04 - arquitetura.md`.

## CI/CD

- Cada microsserviço possui workflow `.github/workflows/ci.yml` que dispara em PR e push para `develop`/`main`, executando restore → build → test.
- O repositório de infraestrutura possui workflow `deploy.yml` que clona os repos dos microsserviços, gera as chaves e constrói as imagens.
- Convenção de branches: `develop` para integração e `main` para release. Pull Requests obrigatórios.

Mais detalhes em `docs/05 - cicd.md`.

## Observações de implementação

- **JWT assimétrico (RS256)**: chaves PEM montadas via volume Docker; ausência das chaves impede a inicialização. UserApi também publica JWKS para integração externa.
- **Cache Redis**: stale-tolerant. Falhas de conexão não derrubam o UserApi — apenas degradam o cache.
- **Idempotência**: o consumer `VehicleSold` ignora veículos já marcados como vendidos.
- **Healthchecks**: todos os componentes possuem `healthcheck` no compose, garantindo ordem de inicialização correta mesmo com dependências assíncronas.
