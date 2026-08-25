# Integração Zoho CRM + Stokki + Conta Azul

Documentação e código-fonte (Deluge) das automações que integram o **Zoho CRM**
(módulos Accounts, Deals e Products) com o **Stokki** (OMS - gestão de pedidos
e estoque) e o **Conta Azul** (ERP financeiro/fiscal).

## Visão geral do fluxo

```
                         ┌─────────────────────┐
                         │   Zoho CRM - Deal    │
                         │  (Stage = Won)       │
                         └──────────┬───────────┘
                                    │ Workflow Rule dispara
                                    ▼
                    ┌───────────────────────────────┐
                    │   enviarPedidoStokki(dealId)   │  ◄── orquestrador
                    └───────────────┬───────────────┘
                                    │
              ┌─────────────────────┼──────────────────────┐
              │ 1. Valida campos obrigatórios do Deal        │
              │    e CNPJ da Account                         │
              │ 2. Consulta estoque disponível no Stokki      │
              │    (por SKU, só leitura) e bloqueia se        │
              │    insuficiente                               │
              └─────────────────────┬──────────────────────┘
                                    │ OK
                                    ▼
                    ┌───────────────────────────────┐
                    │  criarVendaContaAzul(dealId)    │  ◄── financeiro
                    └───────────────┬───────────────┘        primeiro
                                    │
              ┌─────────────────────┼──────────────────────┐
              │ 1. renovarTokenContaAzul() — OAuth2           │
              │ 2. Busca/cria cliente no Conta Azul por CNPJ  │
              │ 3. Localiza produtos por SKU (Product_Code)   │
              │ 4. Monta condição de pagamento (parcelas)     │
              └─────────────────────┬──────────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │   POST /v1/venda (Conta Azul)   │
                    │   grava Conta_Azul_Venda_Id      │
                    └───────────────┬───────────────┘
                            venda criada?
                     ┌───────────────┴───────────────┐
                   NÃO                               SIM
                     │                                 │
                     ▼                                 ▼
       Para aqui. Stokki NUNCA         ┌───────────────────────────────┐
       é acionado — nenhum pedido      │  POST /oms/outbound/store       │
       físico é criado.                │  (Stokki) — cria pedido de saída │
       Stage volta p/ revisão.         └───────────────┬───────────────┘
                                                        │ grava Stokki_Order_Id
                                                        ▼
                                                    (fim do fluxo)


      ── (fluxo separado, após faturamento da NF-e no Conta Azul) ──

                    ┌───────────────────────────────┐
                    │ enviarNFeParaStokki(chave, ...) │
                    └───────────────┬───────────────┘
                                    │
              ┌─────────────────────┼──────────────────────┐
              │ 1. Baixa XML da NF-e (Conta Azul)             │
              │ 2. Upload do XML no Supabase Storage          │
              │    (gera URL pública)                         │
              │ 3. Envia URL + chave de acesso ao Stokki       │
              │    (POST /oms/outbound/invoice)                │
              └───────────────────────────────────────────────┘
```

Além disso, **de forma independente** desse fluxo principal, existe:

```
┌─────────────────────┐        ┌──────────────────────────────┐
│  Account criada/     │        │  preencherDadosCNPJ(accountId) │
│  CNPJ preenchido     │──────► │  Consulta BrasilAPI e completa │
│                      │        │  Razão Social, endereço, etc.  │
└─────────────────────┘        └──────────────────────────────┘
```

## Funções (Deluge)

| Arquivo | Função | Descrição |
|---|---|---|
| [`deluge/functions/preencherDadosCNPJ.dg`](deluge/functions/preencherDadosCNPJ.dg) | `automation.preencherDadosCNPJ(accountId)` | Enriquece a Account com dados da Receita Federal (BrasilAPI) a partir do CNPJ. |
| [`deluge/functions/renovarTokenContaAzul.dg`](deluge/functions/renovarTokenContaAzul.dg) | `automation.renovarTokenContaAzul()` | Renova o `access_token` OAuth2 do Conta Azul, rotacionando o `refresh_token` salvo na Account de configuração. |
| [`deluge/functions/enviarPedidoStokki.dg`](deluge/functions/enviarPedidoStokki.dg) | `automation.enviarPedidoStokki(dealId)` | **Orquestrador.** Valida o Deal, checa estoque no Stokki (leitura), cria a venda no Conta Azul e — só se der certo — envia o pedido (outbound) ao Stokki. |
| [`deluge/functions/criarVendaContaAzul.dg`](deluge/functions/criarVendaContaAzul.dg) | `automation.criarVendaContaAzul(dealId)` | Cria/atualiza o cliente no Conta Azul por CNPJ, resolve produtos por SKU, monta parcelas e envia a venda. |
| [`deluge/functions/enviarNFeParaStokki.dg`](deluge/functions/enviarNFeParaStokki.dg) | `automation.enviarNFeParaStokki(chaveAcesso, numberErp)` | Baixa o XML da NF-e do Conta Azul, hospeda no Supabase Storage e envia a URL ao Stokki para vincular a nota ao pedido. |

## Módulos e campos customizados do Zoho CRM envolvidos

**Accounts**
- `CNPJ` — chave de busca/criação em Conta Azul e Stokki
- `Account_Name`, `Phone`, `Email`
- `Billing_Street`, `Billing_City`, `Billing_Code`, `Bairro`, `Estado` — endereço
- `ContaAzul_Refresh_Token` — usado apenas na Account de configuração "CONFIG - INTEGRACOES"

**Deals**
- `Account_Name` (lookup), `Deal_Name`, `Amount`, `Stage`
- `Produtos_API` (subform) — itens com `SKU` (lookup para Products), `Qtd_Decimal`, `Valor_Unit_Decimal`
- `Transportadora`, `Frete`, `Cond_Pagamento_Lista`, `Ordem_de_Compra`, `Unidade_de_Faturamento`
- `Stokki_Order_Id`, `Status_Integra_o_Stokki`
- `Conta_Azul_Venda_Id`, `Status_ContaAzul_Detalhe`

**Products**
- `Product_Code` — SKU usado tanto no Stokki quanto no Conta Azul
- `Product_Name`

## Integrações externas

| Sistema | Uso | Autenticação |
|---|---|---|
| **BrasilAPI** | Consulta pública de CNPJ (Receita Federal) | Nenhuma (API pública) |
| **Conta Azul** | Cadastro de clientes, produtos e criação de vendas/NF-e | OAuth2 (`refresh_token` rotativo, `client_id`/`client_secret` do app) |
| **Stokki** | Consulta de estoque e criação de pedidos de saída (outbound) | Token estático por requisição |
| **Supabase Storage** | Hospedagem temporária do XML da NF-e para gerar URL pública consumível pelo Stokki | Service Role Key |

## ⚠️ Segurança e credenciais

O código original enviado continha as seguintes credenciais **hardcoded** no
corpo das funções Deluge. Elas foram **redigidas** (substituídas por
placeholders `{{...}}`) nesta versão publicada:

- `{{SUPABASE_SERVICE_ROLE_KEY}}` (em `enviarNFeParaStokki.dg`)
- `{{STOKKI_TOKEN}}` (em `enviarNFeParaStokki.dg` e `enviarPedidoStokki.dg`)
- `{{CONTAAZUL_CLIENT_ID}}` / `{{CONTAAZUL_CLIENT_SECRET}}` (em `renovarTokenContaAzul.dg`)

**Ação recomendada:** como essas chaves passaram por texto puro fora do
Zoho, considere-as potencialmente expostas e **rotacione-as** (gere novas)
nos painéis do Supabase, Stokki e Conta Azul.

**Boas práticas para as próximas versões:**
- Mover essas credenciais para **Connections OAuth2** do Zoho (quando
  suportado, como no caso do Conta Azul) ou para **Variáveis do Zoho**
  (`zoho.variables`), nunca hardcoded no corpo da função.
- Restringir a Service Role Key do Supabase a um bucket/policy específico,
  ou trocar por uma signed URL de curta duração em vez de expor o arquivo
  publicamente.

## ⚠️ Ordem Conta Azul → Stokki

`enviarPedidoStokki` cria a venda no **Conta Azul antes** de enviar o pedido
físico ao **Stokki**, de propósito. A criação da venda pode falhar por
motivos financeiros que não têm relação com estoque (ex: saldo/crédito
insuficiente do cliente no Conta Azul). Se o Stokki fosse acionado primeiro,
um pedido de separação real chegaria ao depósito mesmo que a venda não
pudesse ser faturada — foi exatamente esse cenário que apareceu em teste
(pedido criado no Stokki com o Conta Azul retornando erro de saldo
insuficiente). Com a ordem invertida, nenhum pedido físico é criado quando o
Conta Azul rejeita a venda.

## Tratamento de erros

Todas as funções de escrita (`enviarPedidoStokki`, `criarVendaContaAzul`,
`enviarNFeParaStokki`) seguem o mesmo padrão:
- Validam pré-condições (CNPJ, produtos cadastrados, estoque, campos
  obrigatórios) **antes** de chamar qualquer API externa.
- Em caso de bloqueio ou erro, gravam o motivo em campos de status do Deal
  (`Status_Integra_o_Stokki`, `Status_ContaAzul_Detalhe`), devolvem o `Stage`
  para `"Negotiation/Review"` e enviam e-mail de alerta, permitindo que o
  time comercial corrija o dado e reprocesse o negócio.
