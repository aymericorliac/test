# 1. Dimensional Modelling

&ensp;The dimensional modeling developed was focused on the tax payment process and was structured using the Star Schema approach. This is because the Star Schema focuses on simplicity, ease of navigation, and high performance in analytical queries, characteristics that fit scenarios aiming for BI use where the main objective is to allow quick analyses of payments made by companies over time. Furthermore, no need for complex hierarchies was identified, which might otherwise lead us to choose a Snowflake approach.

## 2. Implemented Star Schema for this sprint

&ensp;&ensp;Following the Star Schema standard, the model we built organizes the information about invoices in a simple and easy-to-analyze way. At the center is the fact table, which stores the invoice values, and around it are the dimension tables, which bring the details about time, company, and type of operation. The idea is that each invoice is linked to a single date, a specific operation, and the companies involved, while each date, each operation, and each company can appear in several different invoices. This view greatly simplifies day-to-day analysis: you just choose a period, a type of operation, or a set of companies and see how that affects the total amount transacted.

### Detailed Table Analysis

#### 1. Fact Table: `fato_tributos`

The fact table, `fato_tributos`, represents the most granular and valuable part of the system: **every single record corresponds to a distinct tax or tribute payment event recorded in the system**. This is the repository for the quantitative measures that are the focus of business intelligence: `valor_principal` (the base amount), `valor_juros` (interest accrued), `valor_multa` (fines), `valor_descontos` (any reductions), and the key metric, `valor_total` (the final paid amount).

Along with these values, we store crucial tracking information, such as the `numero_principal`, the fiscal `codigo_receita` (revenue code), the `competencia` (the reference period the tax applies to), and the `orgao_emissor` (the issuing body). This table achieves its analytical power by connecting to the context dimensions through codes—the foreign keys: `tipo_documento_key`, `cnpj_contribuinte_key`, `data_emissao_key`, `data_vencimento_key`, and `localidade_key`. These foreign keys are the indispensable bridges used to retrieve descriptive details for every recorded payment.



##### 1.1 The Dimensional Roles of `fato_tributos`

The analytical framework is established by the four shared dimension tables that surround `fato_tributos`, providing a comprehensive context for every payment.

The **`dim_data`** dimension acts as the master time reference, playing two distinct roles through `data_emissao_key` (the document's creation date) and `data_vencimento_key` (the required payment date). This dual role is crucial for comparing timely payments versus overdue liabilities, or tracking latency between issuance and due dates, using ready-made calendar attributes like Year, Quarter, and Month Name.

The **`dim_empresa`** table provides the identity of the **contributing company** via the `cnpj_contribuinte_key`. By centralizing attributes like `razao_social`, `inscricao_estadual`, and `tipo_pessoa` here, analysis becomes contributor-centric, allowing easy pivoting to view total tax values by state registration status or entity type.

The **`dim_localidade`** dimension links the tax record to its geographical context through the `localidade_key`. This table, containing the Municipality, State (UF), and neighborhood, enables powerful regional analysis, allowing users to drill down or roll up tax revenue based on geography.

Finally, the **`dim_tipo_documento`** dimension, linked by `tipo_documento_key`, clarifies the nature of the tax obligation itself. Attributes like Subtype, Category, and Origin Label allow analysts to segment the fact data based on the source or classification of the underlying document.

Together, these dimensions transform raw financial figures into meaningful business insights, allowing analysts to quickly answer sophisticated questions regarding **who** paid **what** **when**, **where**, and under **which category** of fiscal obligation.

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
