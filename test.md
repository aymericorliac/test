# 1. Dimensional Modelling

&ensp;The dimensional modeling developed was focused on the tax payment process and was structured using the Star Schema approach. This is because the Star Schema focuses on simplicity, ease of navigation, and high performance in analytical queries, characteristics that fit scenarios aiming for BI use where the main objective is to allow quick analyses of payments made by companies over time. Furthermore, no need for complex hierarchies was identified, which might otherwise lead us to choose a Snowflake approach.

## 2. Implemented Star Schema for this sprint

&ensp;&ensp;Following the Star Schema standard, the model we built organizes the information about invoices in a simple and easy-to-analyze way. At the center is the fact table, which stores the invoice values, and around it are the dimension tables, which bring the details about time, company, and type of operation. The idea is that each invoice is linked to a single date, a specific operation, and the companies involved, while each date, each operation, and each company can appear in several different invoices. This view greatly simplifies day-to-day analysis: you just choose a period, a type of operation, or a set of companies and see how that affects the total amount transacted.

### Detailed Table Analysis

#### 1. Fact Table: `fato_tributos`

The fact table, `fato_tributos`, represents the most granular level: **each row is an individual tax or tribute payment**. This is where we store the key measurable metrics: `valor_principal` (Principal value), `valor_juros` (Interest), `valor_multa` (Fine), `valor_descontos` (Discounts), and `valor_total` (Total value).

Key non-metric attributes include `numero_principal`, `codigo_receita` (revenue code), `competencia` (reference period), and `orgao_emissor` (issuing body). The table uses five foreign keys to connect to its dimensions: `tipo_documento_key`, `cnpj_contribuinte_key`, `data_emissao_key`, `data_vencimento_key`, and `localidade_key`.


<div align="center">
  <sub>Figure 1 - Fact table Fato Triburtos </sub><br>
  <img src="https://res.cloudinary.com/dm5korpwy/image/upload/v1764775118/fato_tributos_zt1yb7.png">
  <sup>Source: Material produced by the authors (2025).</sup>
</div>




#### 2. Dimensions and Their Roles

This fact table is linked to four shared dimensions:

| Dimension | Primary Key | Role(s) in Fact Table | Key Attributes |
| :--- | :--- | :--- | :--- |
| **dim_data** | `data_key` | Date of Issuance (`data_emissao_key`), Due Date (`data_vencimento_key`) | Year, Quarter, Month Name |
| **dim_empresa** | `cnpj` | Contributing Company CNPJ (`cnpj_contribuinte_key`) | Business Name, State (UF), Type |
| **dim_localidade** | `localidade_key` | Document Location (`localidade_key`) | Municipality, State (UF), CEP |
| **dim_tipo_documento** | `tipo_documento_key` | Document Context (`tipo_documento_key`) | Subtype, Category, Origin Label |

This structure facilitates analyses such as: "What is the total `valor_multa` accrued on taxes whose `data_vencimento_key` falls in the last quarter?" or "Compare the `valor_total` of taxes for different `tipo_pessoa` from `dim_empresa`."

### DBML Code (Diagram)

```dbml
Table fato_tributos {
  tributo_id varchar [pk]
  ingested_at timestamp
  numero_principal varchar
  codigo_receita varchar
  competencia varchar
  periodo_apuracao varchar
  valor_principal decimal
  valor_juros decimal
  valor_multa decimal
  valor_descontos decimal
  valor_total decimal
  orgao_emissor varchar
  arquivo varchar
  caminho_completo varchar
  label varchar

  // Foreign Keys
  tipo_documento_key int [ref: > dim_tipo_documento.tipo_documento_key]
  cnpj_contribuinte_key varchar [ref: > dim_empresa.cnpj]
  data_emissao_key int [ref: > dim_data.data_key]
  data_vencimento_key int [ref: > dim_data.data_key]
  localidade_key int [ref: > dim_localidade.localidade_key]
}

Table dim_data {
  data_key int [pk]
  data_completa date
  ano int
  mes int
  trimestre int
  dia_mes int
  dia_semana int
  nome_mes varchar
  ano_mes varchar
}

Table dim_empresa {
  cnpj varchar [pk]
  razao_social varchar
  inscricao_estadual varchar
  inscricao_municipal varchar
  tipo_pessoa varchar
}

Table dim_localidade {
  localidade_key int [pk]
  municipio varchar
  uf varchar
  bairro varchar
  cep varchar
  logradouro varchar
  numero varchar
  complemento varchar
}

Table dim_tipo_documento {
  tipo_documento_key int [pk]
  tipo_documento varchar
  subtipo_documento varchar
  categoria varchar
  origem_label varchar
}
