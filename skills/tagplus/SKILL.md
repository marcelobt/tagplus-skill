---
name: tagplus
description: >
  Integração com a API REST do ERP TagPlus. Use este skill sempre que o usuário pedir para:
  conectar ao Tagplus, buscar ou criar clientes no Tagplus, listar ou criar pedidos de venda,
  consultar ou atualizar produtos e estoque, verificar contas a pagar ou receber, emitir notas
  fiscais (NF-e/NFS-e), configurar webhooks, renovar tokens OAuth, fazer qualquer operação via
  API do TagPlus, integrar um sistema com o TagPlus, ou depurar erros de integração com o TagPlus.
  Palavras-gatilho: tagplus, tag plus, api tagplus, erp tagplus, integrar tagplus.
---

# Integração com a API do TagPlus

## Visão Geral

A API do TagPlus é REST/JSON com autenticação OAuth 2.0.

- **Base URL**: `https://api.tagplus.com.br`
- **Versão obrigatória no header**: `x-api-version: 2.0`
- **Auth header**: `Authorization: Bearer <access_token>`
- **Documentação**: https://developers.tagplus.com.br/doc
- **Guia de início**: https://developers.tagplus.com.br/guia

---

## 1. Autenticação OAuth 2.0

### Registrar Aplicação
Acesse https://developers.tagplus.com.br → seu usuário → **Aplicações** → criar nova app.
Guarde o `client_id` e `client_secret`.

### Fluxo de Autorização (Authorization Code)

**Passo 1 — Redirecionar usuário para autorizar:**
```
GET https://apidoc.tagplus.com.br/authorize
  ?client_id=SEU_CLIENT_ID
  &redirect_uri=SUA_CALLBACK_URL
  &response_type=code
  &scope=read:clientes write:clientes read:pedidos write:pedidos read:produtos write:produtos read:financeiros write:financeiros
```

**Passo 2 — Trocar code por tokens:**
```
POST https://api.tagplus.com.br/oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=CODIGO_RECEBIDO
&client_id=SEU_CLIENT_ID
&client_secret=SEU_CLIENT_SECRET
&redirect_uri=SUA_CALLBACK_URL
```

Resposta:
```json
{
  "access_token": "...",
  "refresh_token": "...",
  "token_type": "Bearer",
  "expires_in": 86400
}
```

### Renovar Token (Refresh)
```
POST https://api.tagplus.com.br/oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=SEU_REFRESH_TOKEN
&client_id=SEU_CLIENT_ID
&client_secret=SEU_CLIENT_SECRET
```

> **CRÍTICO**: ao renovar, o par anterior é imediatamente invalidado. Persista os novos tokens
> antes de qualquer outra requisição. Access token expira em 24h; refresh token em 15 dias.

### Escopos Disponíveis
| Escopo | Acesso |
|---|---|
| `read:clientes` / `write:clientes` | Clientes e fornecedores |
| `read:pedidos` / `write:pedidos` | Pedidos de venda e compra |
| `read:produtos` / `write:produtos` | Produtos, estoque, serviços |
| `read:financeiros` / `write:financeiros` | Contas a pagar/receber |
| `read:nfe` / `write:nfe` | Emissão de NF-e |
| `read:contratos` / `write:contratos` | Contratos |

---

## 2. Padrões de Requisição

Todas as requisições precisam dos headers:
```
Authorization: Bearer <access_token>
x-api-version: 2.0
Content-Type: application/json   (para POST/PUT)
```

### Paginação
A API usa paginação via query params:
```
GET /clientes?page=1&limit=50
```
Resposta inclui metadados de paginação no body ou nos headers `X-Total-Count`, `X-Page`.

### Filtros e Busca
Use query params para filtrar:
```
GET /clientes?nome=João&cpf=123.456.789-00
GET /produtos?ativo=true&categoria_id=5
GET /pedidos?data_inicio=2024-01-01&data_fim=2024-12-31&status=confirmado
```

---

## 3. Endpoints por Recurso

Consulte `references/endpoints.md` para detalhes completos de cada endpoint.

### Clientes/Entidades (`/clientes`)
```
GET    /clientes              → listar clientes
GET    /clientes/{id}         → buscar cliente por ID
POST   /clientes              → criar cliente
PUT    /clientes/{id}         → atualizar cliente
DELETE /clientes/{id}         → remover cliente
```

### Pedidos de Venda (`/pedidos`)
```
GET    /pedidos               → listar pedidos
GET    /pedidos/{id}          → buscar pedido por ID
POST   /pedidos               → criar pedido
PUT    /pedidos/{id}          → atualizar pedido
POST   /pedidos/{id}/confirmar → confirmar pedido
POST   /pedidos/{id}/cancelar  → cancelar pedido
```

### Produtos (`/produtos`)
```
GET    /produtos              → listar produtos
GET    /produtos/{id}         → buscar produto por ID
POST   /produtos              → criar produto
PUT    /produtos/{id}         → atualizar produto
GET    /produtos/{id}/estoque → consultar estoque
```

### Financeiro (`/financeiros` ou `/contas`)
```
GET    /financeiros           → listar lançamentos
GET    /financeiros/{id}      → buscar lançamento
POST   /financeiros           → criar lançamento (conta a pagar/receber)
PUT    /financeiros/{id}      → atualizar lançamento
POST   /financeiros/{id}/baixar → dar baixa no lançamento
```

### NF-e / NFS-e
```
POST   /nfe                   → emitir NF-e
GET    /nfe/{id}              → consultar NF-e
POST   /nfe/{id}/cancelar     → cancelar NF-e
GET    /nfe/{id}/xml          → download do XML
POST   /nfse                  → emitir NFS-e
```

### Usuário Autenticado
```
GET /me → dados do usuário/empresa autenticada
```

---

## 4. Tratamento de Erros

| HTTP Status | Significado |
|---|---|
| `400` | Dados inválidos — verifique o body da requisição |
| `401` | Token expirado ou inválido — renovar access_token |
| `403` | Sem permissão — verificar escopo da aplicação |
| `404` | Recurso não encontrado |
| `422` | Erro de validação de negócio (ex: CPF duplicado) |
| `429` | Rate limit atingido — aguardar antes de tentar novamente |
| `500` | Erro interno do TagPlus |

Erros retornam JSON com detalhes:
```json
{
  "message": "Descrição do erro",
  "errors": {
    "campo": ["mensagem de validação"]
  }
}
```

---

## 5. Webhooks

Configure em https://developers.tagplus.com.br/webhook para receber eventos em tempo real.

Eventos disponíveis: criação/atualização de pedidos, emissão de NF-e, alterações em clientes/produtos.

O payload inclui o tipo do evento e os dados do recurso afetado.

---

## 6. Exemplos de Código

### Python (requests)
```python
import requests

BASE_URL = "https://api.tagplus.com.br"
HEADERS = {
    "Authorization": f"Bearer {access_token}",
    "x-api-version": "2.0",
    "Content-Type": "application/json"
}

# Listar clientes
resp = requests.get(f"{BASE_URL}/clientes", headers=HEADERS, params={"page": 1, "limit": 50})
clientes = resp.json()

# Criar cliente
novo_cliente = {
    "nome": "João Silva",
    "cpf_cnpj": "123.456.789-00",
    "email": "joao@email.com",
    "telefone": "(11) 99999-9999"
}
resp = requests.post(f"{BASE_URL}/clientes", headers=HEADERS, json=novo_cliente)
```

### Renovação Automática de Token (Python)
```python
def get_valid_token(tokens: dict, client_id: str, client_secret: str) -> dict:
    """Renova o access_token se necessário e retorna tokens atualizados."""
    resp = requests.post(
        f"{BASE_URL}/oauth2/token",
        data={
            "grant_type": "refresh_token",
            "refresh_token": tokens["refresh_token"],
            "client_id": client_id,
            "client_secret": client_secret,
        }
    )
    new_tokens = resp.json()
    # PERSISTA IMEDIATAMENTE — o token anterior é invalidado agora
    save_tokens(new_tokens)
    return new_tokens
```

---

## 7. Dicas Importantes

- **Sempre verifique a expiração** do access_token antes de requisições críticas.
- **Nunca reuse refresh_token** após receber novos tokens — o anterior é invalidado.
- **Rate limiting**: espaçe requisições em lote com um pequeno delay (ex: 100ms entre chamadas).
- **Ambientes**: a API tem apenas produção; use dados de teste com cautela.
- Se os endpoints retornarem 404, confirme na documentação oficial se o path está correto —
  pode haver variações como `/entidades` em vez de `/clientes`.

Para detalhes completos de campos de request/response de cada endpoint, consulte:
- Documentação oficial: https://developers.tagplus.com.br/doc
- Artigos de ajuda: https://ajuda.tagplus.com.br/pt-BR/collections/1747294-api
