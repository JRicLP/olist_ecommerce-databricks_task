# E-Commerce Olist — Pipeline de Dados com Arquitetura Medallion

Pipeline de Engenharia de Dados desenvolvido no Databricks para estruturar e analisar dados de e-commerce utilizando a arquitetura Medallion (Bronze → Silver → Gold), com ingestão de dados externos via API do Banco Central do Brasil e orquestração automatizada via Databricks Workflows.

---

## Arquitetura

```
Fontes de Dados
├── CSVs Olist (9 arquivos)    →  Volume (Unity Catalog)
└── API PTAX (Banco Central)   →  Requisição HTTP

            ↓  Atividade_land_to_bronze

┌──────────────────────────────────────────────┐
│  BRONZE — Dados brutos sem transformação     │
│  10 tabelas | timestamp_ingestion em todas   │
└──────────────────────────────────────────────┘

            ↓  Atividade_bronze_to_silver

┌──────────────────────────────────────────────┐
│  SILVER — Limpeza, tipagem e regras          │
│  10 tabelas | colunas em português           │
│  Deduplicação | Forward fill | ZORDER        │
└──────────────────────────────────────────────┘

            ↓  Atividade_silver_to_gold

┌──────────────────────────────────────────────┐
│  GOLD — Data Marts analíticos                │
│  2 tabelas | Rankings via display()          │
└──────────────────────────────────────────────┘
```

---

## Estrutura do Repositório

```
olist-medallion-databricks/
├── notebooks/
│   ├── Atividade_land_to_bronze.ipynb
│   ├── Atividade_bronze_to_silver.ipynb
│   └── Atividade_silver_to_gold.ipynb
├── workflow/
│   └── workflow.yaml
├── assets/
│   └── job_execution_success.png
└── README.md
```

---

## Camadas

### Bronze — `Atividade_land_to_bronze.ipynb`

Ingestão fiel das fontes sem nenhuma transformação. A única adição é `timestamp_ingestion`, que registra o momento exato de cada carga e serve de base para a deduplicação na Silver.

| Tabela | Origem |
|---|---|
| `bronze.tb_customers` | olist_customers_dataset.csv |
| `bronze.tb_geolocalizacao` | olist_geolocation_dataset.csv |
| `bronze.tb_order_items` | olist_order_items_dataset.csv |
| `bronze.tb_order_payments` | olist_order_payments_dataset.csv |
| `bronze.tb_order_reviews` | olist_order_reviews_dataset.csv |
| `bronze.tb_orders` | olist_orders_dataset.csv |
| `bronze.tb_products` | olist_products_dataset.csv |
| `bronze.tb_sellers` | olist_sellers_dataset.csv |
| `bronze.tb_product_category_name_translation` | product_category_name_translation.csv |
| `bronze.tb_cotacao_dolar` | API PTAX — Banco Central do Brasil |

---

### Silver — `Atividade_bronze_to_silver.ipynb`

Limpeza, padronização e aplicação das regras de negócio. Todas as colunas estão em português com tipagem explícita. A Bronze nunca é modificada.

| Tabela | Principais transformações |
|---|---|
| `silver.dim_consumidores` | Deduplicação Sênior (Window Function) + Upper Case |
| `silver.fat_pedidos` | Tradução de 8 status + 4 colunas derivadas de tempo |
| `silver.fat_itens_pedidos` | Renomeação e tipagem |
| `silver.fat_pagamentos_pedidos` | Tradução de 5 tipos de pagamento |
| `silver.fat_avaliacoes_pedidos` | `try_to_timestamp` + filtros de qualidade + tratamento de nulos |
| `silver.dim_produtos` | Deduplicação Sênior (Window Function) |
| `silver.dim_vendedores` | Deduplicação Sênior + Upper Case |
| `silver.dim_categoria_produtos_traducao` | Renomeação PT/EN |
| `silver.dim_cotacao_dolar` | Calendário contínuo + forward fill (`last(ignorenulls=True)`) |
| `silver.fat_pedido_total` | Join triplo (pedidos + pagamentos + cotação) + BRL/USD arredondados |

Ao final do notebook, `OPTIMIZE + ZORDER BY (id_pedido, data_pedido)` é aplicado nas 3 tabelas fato para garantir alta performance analítica na Gold.

---

### Gold — `Atividade_silver_to_gold.ipynb`

Data Marts consolidados para análise de negócio. Todas as gravações usam `mode("overwrite")`.

**Projeto 1 — Visão Comercial**
- `gold.fat_vendas_comercial`: agrupamento por ano, mês e categoria com 7 métricas (`ano_venda`, `mes_venda`, `categoria_produto`, `total_pedidos`, `qtd_itens_vendidos`, `receita_total_brl`, `receita_total_usd`, `ticket_medio_brl`)
- Top 5 produtos mais vendidos via `display()`
- Top 5 produtos menos vendidos via `display()`

**Projeto 2 — Satisfação de Clientes**
- `gold.fat_avaliacoes_clientes`: agrupamento por categoria, vendedor e estado com 5 métricas (`total_avaliacoes`, `avaliacao_media`, `total_avaliacoes_positivas`, `total_avaliacoes_negativas`, `percentual_satisfacao`)
- 4 rankings de qualidade via `display()` com critério composto: nota (primeiro) + volume de avaliações como desempate

---

## Orquestração

O pipeline é orquestrado via **Databricks Workflows** com 3 tasks sequenciais:

```
bronze  →  silver  →  gold
```

- **Agendamento:** diário às 13:00 (America/Sao_Paulo)
- **Duração média:** ~3 minutos
- **Compute:** Serverless
- **Configuração completa:** `workflow/workflow.yaml`

---

## Como Executar

### Pré-requisitos
- Conta no Databricks (Free Edition ou superior)
- Dataset Olist disponível no [Kaggle](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)

### 1. Upload dos dados

No Databricks, acesse **Catalog → seu schema → Volumes**, crie um Volume chamado `raw_data` e faça upload dos 9 arquivos CSV do dataset Olist.

### 2. Ajuste do caminho do Volume

No notebook `Atividade_land_to_bronze`, o widget `volume_path` está configurado com o caminho padrão:

```
/Volumes/workspace/olist_ecommerce/raw_data
```

Ajuste para o caminho do seu Volume antes de executar.

### 3. Parâmetros da API PTAX

Os widgets `data_inicio` e `data_fim` controlam o período de cotação buscado na API do Banco Central. O formato obrigatório é `MM-DD-AAAA`. Os valores padrão cobrem o período do dataset Olist:

```
data_inicio: 01-01-2016
data_fim:    12-31-2018
```

### 4. Execução

Execute os notebooks na ordem ou configure o Workflow:

```
Atividade_land_to_bronze → Atividade_bronze_to_silver → Atividade_silver_to_gold
```

---

## Tecnologias

| Tecnologia | Uso |
|---|---|
| Databricks Free Edition | Plataforma de processamento |
| Apache Spark / PySpark | Engine de transformação distribuída |
| Delta Lake | Armazenamento com ACID e time travel |
| Unity Catalog | Governança e metastore |
| Databricks Workflows | Orquestração e agendamento |
| API PTAX — Banco Central | Cotação histórica do dólar |

---

*Visagio | Rocket Lab 2026 — Atividade de Engenharia de Dados*
