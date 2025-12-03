## Data Model Documentation: `fato_tributos`

Following the Star Schema standard, the model built organizes information about tax and tribute payments (`tributos`) in a simple and easy-to-analyze way. The central fact table stores the financial values, and the surrounding dimension tables provide the necessary context: time, company, location, and document type.

The core idea is that each tribute record is linked to a specific contributor, location, document type, and two distinct date roles (issuance and due date).


### Detailed Table Analysis

#### 1. Fact Table: `fato_tributos`

The fact table, `fato_tributos`, represents the most granular level: **each row is an individual tax or tribute payment**. This is where we store the key measurable metrics: `valor_principal` (Principal value), `valor_juros` (Interest), `valor_multa` (Fine), `valor_descontos` (Discounts), and `valor_total` (Total value).

Key non-metric attributes include `numero_principal`, `codigo_receita` (revenue code), `competencia` (reference period), and `orgao_emissor` (issuing body). The table uses five foreign keys to connect to its dimensions: `tipo_documento_key`, `cnpj_contribuinte_key`, `data_emissao_key`, `data_vencimento_key`, and `localidade_key`.

<div align="center">
  <sub>Figure 1 - Fact Fiscal Notes Modelling </sub><br>
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
