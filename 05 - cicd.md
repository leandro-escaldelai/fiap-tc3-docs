# CI/CD

A solução adotará práticas de **CI/CD** com uso de **GitHub Actions**, **Pull Requests** e separação entre repositórios de código-fonte e repositório de infraestrutura.

Cada microsserviço possuirá seu próprio repositório no GitHub, permitindo evolução, versionamento e validação independentes. Além disso, haverá um repositório específico para os arquivos de configuração da infraestrutura local e de publicação, incluindo:

- `docker-compose.yml`
- configurações do Kong
- variáveis de ambiente de exemplo
- scripts auxiliares de inicialização

---

## Repositórios

A estrutura será organizada da seguinte forma:

- **UserApi**
  - código-fonte do microsserviço de usuários e autenticação
  - testes unitários
  - arquivos de inicialização do banco MongoDB

- **VehicleApi**
  - código-fonte do microsserviço de veículos
  - testes unitários
  - arquivos de inicialização do banco MongoDB

- **SalesApi**
  - código-fonte do microsserviço de vendas
  - testes unitários
  - arquivos de inicialização do banco PostgreSQL

- **Infrastructure**
  - arquivos do Docker Compose
  - configuração declarativa do Kong
  - configuração do RabbitMQ
  - orquestração local dos serviços

---

## Branches

Serão utilizadas duas branches principais:

- **develop**
  - branch de integração e validação contínua
  - recebe Pull Requests de desenvolvimento

- **main**
  - branch estável
  - representa a versão pronta para publicação

---

## Pipeline de validação

Toda publicação direta ou abertura/atualização de Pull Request nas branches `develop` ou `main` deverá disparar automaticamente uma pipeline via **GitHub Actions**.

A pipeline deverá executar:

1. restauração das dependências
2. build da aplicação
3. execução dos testes unitários
4. validação de falha ou sucesso da alteração

Esse processo deverá ser aplicado individualmente em cada repositório de microsserviço.

---

## Testes automatizados

Cada microsserviço deverá conter testes unitários para:

- **UseCases**
- **Services**
- **Domain Models**

Os testes devem validar as regras de negócio de forma isolada, sem dependência direta de banco de dados, mensageria, rede ou infraestrutura externa.

---

## Inicialização dos bancos de dados

Cada microsserviço será responsável por manter os arquivos necessários para inicialização da estrutura de seu próprio banco de dados.

Isso inclui, quando aplicável:

- criação de collections
- criação de tabelas
- índices
- dados mínimos de inicialização
- scripts de migração ou bootstrap

Essa decisão mantém a responsabilidade da estrutura de dados próxima ao serviço que a utiliza.

---

## Publicação da solução

A publicação da solução será feita a partir do repositório de infraestrutura.

O processo deverá:

1. baixar todos os repositórios dos microsserviços
2. baixar o repositório de infraestrutura
3. executar o Docker Compose
4. gerar as imagens Docker de todos os microsserviços
5. subir os componentes na ordem correta

---

## Ordem de inicialização

A ordem de inicialização da infraestrutura deverá ser:

1. **Bancos de dados**
   - MongoDB do UserApi
   - MongoDB do VehicleApi
   - PostgreSQL do SalesApi
   - Redis

2. **RabbitMQ**

3. **Microsserviços**
   - UserApi
   - VehicleApi
   - SalesApi

4. **Kong API Gateway**

Essa ordem reduz falhas de inicialização causadas por dependências ainda indisponíveis.

---

## Trade-offs

### Vantagens

- separação clara entre código de aplicação e infraestrutura
- validação automática antes da integração
- redução de regressões
- autonomia dos microsserviços
- ambiente reprodutível com Docker Compose
- facilidade para demonstrar a solução ponta a ponta

### Desvantagens

- maior quantidade de repositórios para gerenciar
- necessidade de sincronizar versões entre microsserviços
- publicação depende da disponibilidade de todos os repositórios
- Docker Compose exige cuidado com ordem de inicialização e health checks

---

## Observações

O Docker Compose deve utilizar mecanismos de verificação de saúde dos serviços, como `healthcheck`, para garantir que os microsserviços sejam iniciados somente após a disponibilidade mínima de suas dependências.

Mesmo com `depends_on`, a simples ordem de subida dos containers não garante que o serviço esteja pronto para uso. Por isso, bancos de dados, RabbitMQ e demais componentes críticos devem possuir validação explícita de disponibilidade.