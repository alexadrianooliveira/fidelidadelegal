# Fidelidade PDVLEGAL — API v1 (padrão de integração)

Contrato **padrão** que as empresas de fidelidade/CRM devem expor para se integrar ao PDV Legal / TabletCloud.
Um parceiro que implemente os **8 endpoints** abaixo exatamente como descrito entra em produção **sem alteração no PDV**:
6 de operação (login, consulta e transações) e 2 de **provisionamento da loja** (`client_partner` e `client_partner_cancel`).

## Arquivos

| Arquivo | O que é |
|---|---|
| `Fidelidade-PDVLEGAL-v1.postman_collection.json` | Collection Postman (schema v2.1.0) com os 8 endpoints, descrições, exemplos de resposta e testes. |
| `Fidelidade-PDVLEGAL-Homologacao.postman_environment.json` | Environment com `baseUrl` e demais variáveis parametrizáveis. |

## Como importar

1. Postman → **Import** → arraste os dois `.json`.
2. Selecione o environment **Fidelidade PDVLEGAL (Homologação)** no canto superior direito.
3. Ajuste `baseUrl` para o domínio do parceiro (**tudo antes de `/v1/`**).

## Variáveis do environment

| Variável | Preenchimento | Descrição |
|---|---|---|
| `baseUrl` | manual | Domínio do parceiro, sem `/v1/`. Ex: `https://api-hml.parceiro.com.br` |
| `email` / `password` | manual | Credenciais da **loja** (as que o `client_partner` devolveu). |
| `partnerEmail` / `partnerPassword` | manual | Credenciais **master** do PDV Legal no parceiro, usadas só no provisionamento. |
| `token` | **automático** | JWT gravado pelo script do Login (prefixo `JWT `/`Bearer ` removido). |
| `id_user` | **automático** | `user._id` do Login — operador responsável pela operação. |
| `id_passbook` | **automático** | `user.id_passbooks` do Login — cartão/programa da loja. |
| `uniqueId` | manual | CPF do consumidor (somente números). |
| `source` | manual | Nome da loja. |
| `location` | manual | CNPJ da loja, **somente números**. |
| `externalId` | auto se vazio | Código da venda no PDV Legal. Se ficar vazio, o pre-request script gera um número. |
| `licenca` | manual | Código da filial no PDV Legal (provisionamento). |

## Endpoints

| # | Método | Path | Função |
|---|---|---|---|
| 1 | POST | `/v1/auth/login` | Autentica e devolve o JWT + `user._id` + `user.id_passbooks`. |
| 2 | POST | `/v1/pointsv2` | Consulta saldo (selos/pontos/cashback) e prêmios do CPF. |
| 3 | POST | `/v1/transaction/addPoints` | Credita a venda no programa. |
| 4 | POST | `/v1/transaction/removePoints` | Resgata prêmio (debita saldo). |
| 5 | POST | `/v1/transaction/cancelPoints` | Estorna uma venda já creditada. |
| 6 | POST | `/v1/auth/new-member` | *Opcional.* Adesão do consumidor pelo caixa. |
| 7 | POST | `/v1/client_partner` | Cadastra a loja no parceiro quando o módulo de fidelidade é contratado. |
| 8 | POST | `/v1/client_partner_cancel` | Cancela a loja no parceiro quando o módulo é retirado. |

## Regras obrigatórias do padrão

- **Versionamento:** todos os endpoints sob `/v1/`. O host é configurável; o path não.
- **Login:** `email` e `password` no **corpo**, em JSON (`{"email": "...", "password": "..."}`). Nunca na query string.
- **Autenticação:** `Authorization: Bearer <token>` em todos os endpoints exceto o Login.
- **Content-Type:** `multipart/form-data` nas transações, na consulta e no `client_partner`; JSON no Login e no `client_partner_cancel`.
- **Chave do consumidor:** `uniqueId` = CPF somente números (o parceiro pode ser multichave, mas o CPF é obrigatório).
- **Identificação da loja:** `source` = nome da loja, `location` = CNPJ somente números.
  **O CNPJ não é chave da loja:** é comum duas ou mais lojas do mesmo cliente terem o mesmo CNPJ. A loja é identificada
  pelo token do Login dela (cada filial tem o próprio login, criado no `client_partner`) e, no cadastro, pelo `licenca`
  (código da filial, único no PDV Legal). Use o `location` só como informação.
- **Valores:** decimal com 2 casas, separador **ponto** (`1234.56`).
- **Booleanos:** `true` / `false` minúsculos.
- **Envelope de erro:** HTTP 422 com `{ "status": "<codigo>", "error": "<mensagem>" }`, **inclusive para erro de validação de campo**.
  HTTP 401 só para token inválido ou vencido (o PDV gera um token novo e repete a chamada).

### `externalId` (idempotência)

O `externalId` é o código da venda no PDV Legal. Reenvio do mesmo `externalId` na mesma ação retorna `status 105` sem duplicar.

| Ação | `externalId` enviado pelo PDV |
|---|---|
| `addPoints` | `<codigoVenda>` |
| `removePoints` | `removePoints-<codigoVenda>` |
| `cancelPoints` | `<codigoVenda>` — o mesmo do `addPoints`, que é a transação a estornar |

### Códigos de negócio (`status`)

| status | Significado |
|---|---|
| `ok` | Operação concluída. |
| `101` | Saldo insuficiente **ou** registro não encontrado para cancelamento. |
| `103` | Cliente não encontrado (transação). |
| `104` | Consumidor sem fidelidade (consulta). |
| `105` | `externalId` já processado — duplicidade bloqueada. |

## Contratos que o PDV lê

### Login (`/v1/auth/login`)

```json
{
  "token": "<JWT sem prefixo>",
  "user": {
    "_id": "<id do operador da loja>",
    "id_passbooks": "<id do programa da loja>"
  }
}
```

`token`, `user._id` e `user.id_passbooks` são obrigatórios. O PDV guarda o token por 24 h e gera outro ao receber 401.

### Transações (`addPoints`, `removePoints`, `cancelPoints`)

Campos enviados (multipart): `id_passbook`, `id_user`, `uniqueId`, `value`, `source`, `location`, `externalId`.
`addPoints` e `removePoints` enviam também `is_delivery` e `details` (JSON com os itens da venda:
`[{"sku": "<código do produto>", "name": "<descrição>", "count": <quantidade>}]`). O PDV também manda `id_usuariopdv`
(operador do caixa); o parceiro pode ignorar.

No `removePoints`, `value` é o desconto concedido na venda e `details` são os **itens vendidos**, não o código do prêmio.

### Provisionamento da loja (`client_partner`)

Chamado pelo PDV Legal com o token **master** (login com `partnerEmail`/`partnerPassword`) quando o módulo de
fidelidade é gravado na licença. Campos (multipart): `email`, `password` (senha gerada pelo PDV para a loja), `name`,
`cnpj`, `licenca` (código da filial), `tipoPrograma` (`Cashback`), `planoBasico`, `unico`, `status`.

Resposta (HTTP 200 ou 201):

```json
{
  "status": "ok",
  "userinfo": { "login": "<email da loja no parceiro>", "senha": "<senha da loja; opcional>" }
}
```

`userinfo.login` vira o `email` do Login da loja. Se `userinfo.senha` não vier, o PDV usa a senha que enviou.

### Cancelamento da loja (`client_partner_cancel`)

Token master. Corpo em JSON: `{"operator": "<login da loja>", "name": "<nome>", "cnpj": "<somente números>", "licenca": <filial>}`.
Qualquer HTTP 200/201 é sucesso.

### Parceiro sem provisionamento

Enquanto o parceiro não tiver os endpoints 7 e 8, a loja é cadastrada **manualmente** dos dois lados: o parceiro cria a
loja e envia o login/senha, e o PDV Legal registra essas credenciais à mão. Funciona, mas cada loja vendida depende de
alguém fazer o cadastro; por isso o provisionamento é parte do padrão.

## Fluxo no PDV

```
Módulo de fidelidade gravado na licença
  -> POST /v1/auth/login (master) -> POST /v1/client_partner   (cadastra a loja)

Venda finalizada
  -> POST /v1/auth/login (loja)         (só quando o token venceu)
  -> POST /v1/pointsv2                  (consulta saldo e prêmios do CPF)
  -> POST /v1/transaction/addPoints     (credita a venda)
  -> POST /v1/transaction/removePoints  (quando o cliente resgata um prêmio)
  -> POST /v1/transaction/cancelPoints  (quando a venda é estornada)

Módulo retirado da licença
  -> POST /v1/auth/login (master) -> POST /v1/client_partner_cancel
```

## Ponto de atenção do padrão

O `token` do Login pode vir prefixado com `JWT ` (comportamento observado no parceiro de referência), mas o header exigido é `Bearer <token>`. O padrão PDVLEGAL normaliza isso: o script do Login remove o prefixo e as demais chamadas sempre enviam `Bearer`. Parceiros novos devem devolver o token **sem prefixo**.

## Histórico

| Data | Mudança |
|---|---|
| 20/08/2026 | Versão inicial, a partir do modelo do parceiro de referência. |
| 30/10/2026 | Login com credencial no corpo (antes a collection mostrava query string); `client_partner` e `client_partner_cancel` documentados; retorno do Login; `externalId` real de cada ação; `location` só números; erro de validação também em 422. |

---
*Alex Adriano 20/08/2026 — revisado em 30/10/2026*
