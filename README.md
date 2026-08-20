# Fidelidade PDVLEGAL — API v1 (padrão de integração)

Contrato **padrão** que as empresas de fidelidade/CRM devem expor para se integrar ao PDV Legal / TabletCloud.
Um parceiro que implemente estes 6 endpoints exatamente como descrito entra em produção **sem alteração no PDV**.

## Arquivos

| Arquivo | O que é |
|---|---|
| `Fidelidade-PDVLEGAL-v1.postman_collection.json` | Collection Postman (schema v2.1.0) com os 6 endpoints, descrições, exemplos de resposta e testes. |
| `Fidelidade-PDVLEGAL-Homologacao.postman_environment.json` | Environment com `baseUrl` e demais variáveis parametrizáveis. |

## Como importar

1. Postman → **Import** → arraste os dois `.json`.
2. Selecione o environment **Fidelidade PDVLEGAL (Homologação)** no canto superior direito.
3. Ajuste `baseUrl` para o domínio do parceiro (**tudo antes de `/v1/`**).

## Variáveis do environment

| Variável | Preenchimento | Descrição |
|---|---|---|
| `baseUrl` | manual | Domínio do parceiro, sem `/v1/`. Ex: `https://api-hml.parceiro.com.br` |
| `email` / `password` | manual | Credenciais do operador fornecidas pelo parceiro. |
| `token` | **automático** | JWT gravado pelo script do Login (prefixo `JWT `/`Bearer ` removido). |
| `id_user` | **automático** | `user._id` do Login — operador responsável pela operação. |
| `id_passbook` | **automático** | `user.id_passbooks` do Login — cartão/programa da rede. |
| `uniqueId` | manual | CPF do consumidor (somente números). |
| `source` | manual | Nome fantasia da loja. |
| `location` | manual | CNPJ da loja (`XX.XXX.XXX/XXXX-XX`). |
| `externalId` | auto se vazio | ID único da venda. Se ficar vazio, o pre-request script gera `pdvlegal-<timestamp>`. |

## Endpoints

| # | Método | Path | Função |
|---|---|---|---|
| 1 | POST | `/v1/auth/login` | Autentica e devolve o JWT + `id_user` + `id_passbook`. |
| 2 | POST | `/v1/pointsv2` | Consulta saldo (selos/pontos/cashback) e prêmios do CPF. |
| 3 | POST | `/v1/transaction/addPoints` | Credita a venda no programa. |
| 4 | POST | `/v1/transaction/removePoints` | Resgata prêmio (debita saldo). |
| 5 | POST | `/v1/transaction/cancelPoints` | Estorna uma transação já registrada. |
| 6 | POST | `/v1/auth/new-member` | *Opcional.* Adesão do consumidor pelo caixa. |

## Regras obrigatórias do padrão

- **Versionamento:** todos os endpoints sob `/v1/`. O host é configurável; o path não.
- **Autenticação:** `Authorization: Bearer <token>` em todos os endpoints exceto o Login.
- **Content-Type:** `multipart/form-data` nas transações.
- **Chave do consumidor:** `uniqueId` = CPF somente números (o parceiro pode ser multichave, mas o CPF é obrigatório).
- **Idempotência:** `externalId` único **por ação**, prefixado pelo nome da ação (`addPoints-<idVenda>`, `removePoints-<idVenda>`). Reenvio retorna `status 105` sem duplicar.
- **Identificação da loja:** `source` = nome fantasia, `location` = CNPJ.
- **Valores:** decimal com 2 casas, separador **ponto** (`1234.56`).
- **Envelope de erro:** HTTP 422 com `{ "status": "<codigo>", "error": "<mensagem>" }`.

### Códigos de negócio (`status`)

| status | Significado |
|---|---|
| `ok` | Operação concluída. |
| `101` | Saldo insuficiente **ou** registro não encontrado para cancelamento. |
| `103` | Cliente não encontrado (transação). |
| `104` | Consumidor sem fidelidade (consulta). |
| `105` | `externalId` já processado — duplicidade bloqueada. |

## Fluxo no PDV

```
Venda finalizada
  -> POST /v1/pointsv2                  (consulta saldo e prêmios do CPF)
  -> POST /v1/transaction/addPoints     (credita a venda)
  -> POST /v1/transaction/removePoints  (quando o cliente resgata um prêmio)
  -> POST /v1/transaction/cancelPoints  (quando a venda é estornada)
```

## Ponto de atenção do padrão

O `token` do Login pode vir prefixado com `JWT ` (comportamento observado no parceiro de referência), mas o header exigido é `Bearer <token>`. O padrão PDVLEGAL normaliza isso: o script do Login remove o prefixo e as demais chamadas sempre enviam `Bearer`. Parceiros novos devem devolver o token **sem prefixo**.

---
*Documentação gerada a partir do modelo de referência do parceiro. Alex Adriano 20/08/2026*
