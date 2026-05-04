# Modelagem de Dados

# Usuário

## Tipo de armazenamento

Document DB (MongoDB)

## Estrutura

```json
{
    "_id": "guid",
    "nome": "string",
    "cpf": "string",
    "sexo": "string",
    "dataNascimento": "date",
    "email": "string",
    "senhaHash": "string"
}
```

> **Nota:** O campo `senhaHash` deve armazenar apenas o hash da senha, nunca o valor em texto plano.

# Veículo

## Tipo de armazenamento

Document DB (MongoDB)

## Estrutura

```json
{
    "_id": "guid",
    "idProprietario": "guid",
    "marca": "string",
    "modelo": "string",
    "ano": "numeric (int)",
    "cor": "string",
    "preco": "numeric (float)",
    "vendido": "boolean"
}
```

# Venda

## Tipo de armazenamento

Banco de dados Relacional (PostgreSQL)

## Estrutura

```sql
Venda (
    id       guid         primary key,
    idComprador guid      not null,
    idVeiculo   guid      not null,
    data     datetime     not null,
    proposta float        not null,
    status   varchar(50)  not null
)
```