# 1. Dimensional Modelling

&ensp;The dimensional modeling developed was focused on the tax payment process and was structured using the Star Schema approach. This is because the Star Schema focuses on simplicity, ease of navigation, and high performance in analytical queries, characteristics that fit scenarios aiming for BI use where the main objective is to allow quick analyses of payments made by companies over time. Furthermore, no need for complex hierarchies was identified, which might otherwise lead us to choose a Snowflake approach.

## 2. Implemented Star Schema for this sprint

&ensp;&ensp;Following the Star Schema standard, the model we built organizes the information about invoices in a simple and easy-to-analyze way. At the center is the fact table, which stores the invoice values, and around it are the dimension tables, which bring the details about time, company, and type of operation. The idea is that each invoice is linked to a single date, a specific operation, and the companies involved, while each date, each operation, and each company can appear in several different invoices. This view greatly simplifies day-to-day analysis: you just choose a period, a type of operation, or a set of companies and see how that affects the total amount transacted.

### Detailed Table Analysis

#### 1. Fact Table: `fato_tributos`

The fact table, `fato_tributos`, represents the most granular and valuable part of the system: **every single record corresponds to a distinct tax or tribute payment event recorded in the system**. This is the repository for the quantitative measures that are the focus of business intelligence: `valor_principal` (the base amount), `valor_juros` (interest accrued), `valor_multa` (fines), `valor_descontos` (any reductions), and the key metric, `valor_total` (the final paid amount).

Along with these values, we store crucial tracking information, such as the `numero_principal`, the fiscal `codigo_receita` (revenue code), the `competencia` (the reference period the tax applies to), and the `orgao_emissor` (the issuing body). This table achieves its analytical power by connecting to the context dimensions through codes—the foreign keys: `tipo_documento_key`, `cnpj_contribuinte_key`, `data_emissao_key`, `data_vencimento_key`, and `localidade_key`. These foreign keys are the indispensable bridges used to retrieve descriptive details for every recorded payment.


<div align="center">
  <sub>Figure 1 - Fact table Fato Triburtos </sub><br>
  <img src="https://res.cloudinary.com/dm5korpwy/image/upload/v1764775118/fato_tributos_zt1yb7.png">
  <sup>Source: Material produced by the authors (2025).</sup>
</div>

The analytical framework is established by the four shared dimension tables that surround `fato_tributos`, providing a comprehensive context for every payment.

The **`dim_data`** dimension acts as the master time reference, playing two distinct roles through `data_emissao_key` (the document's creation date) and `data_vencimento_key` (the required payment date). This dual role is crucial for comparing timely payments versus overdue liabilities, or tracking latency between issuance and due dates, using ready-made calendar attributes like Year, Quarter, and Month Name.

The **`dim_empresa`** table provides the identity of the **contributing company** via the `cnpj_contribuinte_key`. By centralizing attributes like `razao_social`, `inscricao_estadual`, and `tipo_pessoa` here, analysis becomes contributor-centric, allowing easy pivoting to view total tax values by state registration status or entity type.

The **`dim_localidade`** dimension links the tax record to its geographical context through the `localidade_key`. This table, containing the Municipality, State (UF), and neighborhood, enables powerful regional analysis, allowing users to drill down or roll up tax revenue based on geography.

Finally, the **`dim_tipo_documento`** dimension, linked by `tipo_documento_key`, clarifies the nature of the tax obligation itself. Attributes like Subtype, Category, and Origin Label allow analysts to segment the fact data based on the source or classification of the underlying document.

Together, these dimensions transform raw financial figures into meaningful business insights, allowing analysts to quickly answer sophisticated questions regarding **who** paid **what** **when**, **where**, and under **which category** of fiscal obligation.

##### DBML Code 

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
```
#### 2. Fact Table: `fato_imoveis`

The fact table, `fato_imoveis`, centralizes data related to property records and associated fiscal details. **Each row represents a specific property record or transaction event**, storing key metrics associated with the property's value and fiscal obligations. The core measurable metrics include `valor_principal` (principal property value), `valor_descontos` (any granted discounts), and `valor_total` (total assessed value).

The table also holds key non-metric identifiers such as `inscricao_imobiliaria` (real estate registration number), `exercicio` (fiscal year), and `parcela` (installment number). This table links to dimensions that identify the owner, the location of the property, the document's date, and the type of fiscal document related to the property.


<div align="center">
  <sub>Figure 2 - Fact table Fato Imóveis </sub><br>
  <img src="https://res.cloudinary.com/dm5korpwy/image/upload/v1764776794/fato_imoveis_d2bxqk.png">
  <sup>Source: Material produced by the authors (2025).</sup>
</div>

The analytical framework is established by the four shared dimension tables that surround `fato_imoveis`, providing comprehensive context for every property record.

The **`dim_data`** dimension acts as the master time reference, playing a role through `data_emissao_key` (the document's creation date). This linkage allows for easy analysis of property valuation or assessment trends over specific periods using calendar attributes like Year, Quarter, and Month Name.

The **`dim_empresa`** table provides the identity of the **property owner** via the `cnpj_proprietario_key`. By centralizing attributes like `razao_social`, `inscricao_estadual`, and `tipo_pessoa` here, analysis becomes owner-centric, allowing easy pivoting to view total property values by owner type or registration status.

The **`dim_localidade`** dimension links the property record to its geographical context through the `localidade_key`. This table, containing the Municipality, State (UF), and neighborhood, enables powerful regional analysis, allowing users to drill down or roll up total property values based on geography, which is crucial for urban and fiscal planning analysis.

Finally, the **`dim_tipo_documento`** dimension, linked by `tipo_documento_key`, clarifies the nature of the fiscal document associated with the property (e.g., assessment notice, tax bill). Attributes like Subtype, Category, and Origin Label allow analysts to segment the fact data based on the document's source or classification.

Together, these dimensions transform raw property and fiscal figures into meaningful business insights, allowing analysts to quickly answer sophisticated questions regarding **who** owns **what** property, **where** it is located, **when** it was assessed, and under **which category** of fiscal document.

##### DBML Code 

```dbml
Table fato_imoveis {
  imovel_id varchar [pk]
  ingested_at timestamp
  tipo_documento_key int
  cnpj_proprietario_key varchar
  localidade_key int
  data_emissao_key int
  inscricao_imobiliaria varchar
  exercicio int
  parcela int
  valor_principal decimal
  valor_descontos decimal
  valor_total decimal
  data_validade date
  orgao_emissor varchar
  arquivo varchar
  caminho_completo varchar
  label varchar

  // Foreign Keys
  tipo_documento_key int [ref: > dim_tipo_documento.tipo_documento_key]
  cnpj_proprietario_key varchar [ref: > dim_empresa.cnpj]
  localidade_key int [ref: > dim_localidade.localidade_key]
  data_emissao_key int [ref: > dim_data.data_key]
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
```
#### 3. Fact Table: `fato_folha_pagamento`

The fact table, `fato_folha_pagamento` (Payroll), is the central point for analyzing monthly payroll expenses and related liabilities. **Each row represents a summary of the payroll for a specific company and competence period (month)**. This table captures the key financial metrics related to employment costs: `valor_principal` (total salaries/wages), `valor_descontos` (total deductions), `valor_encargos` (total charges/onus), `valor_fgts` (FGTS contributions), `valor_inss` (INSS contributions), and the crucial `valor_total` (total payroll expense).

It also stores important operational metrics, such as `quantidade_funcionarios` (the number of employees) and descriptive fields like `observacoes`. This table links to the identity of the employing company and the two essential date dimensions required for payroll analysis: the date the document was created and the specific competence period it covers.


<div align="center">
  <sub>Figure 3 - Fact table Fato Folha Pagamento </sub><br>
  <img src="https://res.cloudinary.com/dm5korpwy/image/upload/v1764777022/fato_folha_pagamento_cjmu2f.png">
  <sup>Source: Material produced by the authors (2025).</sup>
</div>

The analytical framework is established by the shared dimension tables that surround `fato_folha_pagamento`, providing comprehensive context for every payroll record.

The **`dim_data`** dimension is utilized in a classic **role-playing** scenario, linking to the fact table through two distinct foreign keys: `data_emissao_key` (the date the payroll document was generated) and `competencia_key` (the period, usually the month, to which the expenses apply). This dual linkage allows analysts to differentiate between when the expense was recorded versus the period it relates to, enabling accurate time-series analysis based on the calendar attributes like Year and Month.

The **`dim_empresa`** table provides the identity of the **employing company** via the `cnpj_empresa_key`. By centralizing attributes like `razao_social`, `inscricao_estadual`, and `tipo_pessoa` here, analysis becomes company-centric, allowing users to track payroll metrics by entity type, state registration, or simply the business name.

Together, these dimensions transform raw payroll figures into meaningful business insights, allowing analysts to quickly answer sophisticated questions regarding **how much** was paid in salaries, **how many** employees were involved, **when** the expense was incurred, and **which company** generated the liability, all segmented by time components.

##### DBML Code 

```dbml
Table fato_folha_pagamento {
  folha_id varchar [pk]
  ingested_at timestamp
  cnpj_empresa_key varchar
  data_emissao_key int
  competencia_key int
  quantidade_funcionarios int
  valor_principal decimal
  valor_descontos decimal
  valor_encargos decimal
  valor_fgts decimal
  valor_inss decimal
  valor_total decimal
  observacoes text
  arquivo varchar
  caminho_completo varchar
  label varchar

  // Foreign Keys
  cnpj_empresa_key varchar [ref: > dim_empresa.cnpj]
  data_emissao_key int [ref: > dim_data.data_key]
  competencia_key int [ref: > dim_data.data_key] 
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
```
#### 4. Fact Table: `fato_escrituracao`

The fact table, `fato_escrituracao`, deals with fiscal declaration records and general bookkeeping summaries. **Each row represents a specific filing or a summarized entry for a given entity and competence period**. The table stores the aggregated measure, `valor_total` (total value declared/recorded), and numerous non-metric attributes essential for tracking compliance and filing details.

Key descriptive fields include `ano_calendario` (calendar year), `competencia` (reference period), `protocolo_entrega` (delivery protocol number), `hash_receita` (revenue service hash), `tipo_escrituracao` (type of filing), and `situacao_especial` (special status). This table links to the entity responsible for the filing, the date of issuance, and the type of document/filing involved.


<div align="center">
  <sub>Figure 4 - Fact table Fato Escrituração </sub><br>
  <img src="https://res.cloudinary.com/dm5korpwy/image/upload/v1764777226/fato_escrituracao_lxpcsg.png">
  <sup>Source: Material produced by the authors (2025).</sup>
</div>

The analytical framework is established by the shared dimension tables that surround `fato_escrituracao`, providing essential context for every filing record.

The **`dim_data`** dimension, linked by `data_emissao_key`, provides the temporal context (when the filing document was created). This allows analysts to track the submission date and aggregate metrics based on calendar attributes like Year, Quarter, and Month Name.

The **`dim_empresa`** table provides the identity of the **filing entity** via the `cnpj_entidade_key`. Centralizing attributes like `razao_social` and `inscricao_estadual` here ensures that analyses are consistently tracked against the responsible corporate entity, enabling easy filtering and segmentation of filings by company attributes.

The **`dim_tipo_documento`** dimension, linked by `tipo_documento_key`, is crucial for classifying the nature of the fiscal filing (e.g., SPED, ECF, ECD). Attributes like Subtype, Category, and Origin Label allow analysts to segment the total declared values (`valor_total`) based on the type of compliance document used.

Together, these dimensions enable complex analysis focused on compliance and reporting integrity, allowing analysts to quickly answer sophisticated questions regarding **who** filed **what** type of document, **when** it was submitted, and **what was the total value** reported in the declaration, all segmented by time and entity attributes.

##### DBML Code 

```dbml
Table fato_escrituracao {
  escrituracao_id varchar [pk]
  ingested_at timestamp
  tipo_documento_key int
  cnpj_entidade_key varchar
  data_emissao_key int
  ano_calendario int
  competencia varchar
  protocolo_entrega varchar
  hash_receita varchar
  tipo_escrituracao varchar
  situacao_especial varchar
  valor_total decimal
  orgao_emissor varchar
  arquivo varchar
  caminho_completo varchar
  label varchar

  // Foreign Keys
  tipo_documento_key int [ref: > dim_tipo_documento.tipo_documento_key]
  cnpj_entidade_key varchar [ref: > dim_empresa.cnpj]
  data_emissao_key int [ref: > dim_data.data_key]
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

Table dim_tipo_documento {
  tipo_documento_key int [pk]
  tipo_documento varchar
  subtipo_documento varchar
  categoria varchar
  origem_label varchar
}
```
#### 5. Fact Table: `fato_escritura`

The fact table, `fato_escritura`, represents records of notarized acts and legal instruments. **Each row is a single notarized event**, containing descriptive attributes crucial for legal and fiscal analysis, rather than relying solely on additive numerical measures. As this is often a **factless fact table** or a **fact table containing only descriptive attributes**, it captures the context of the legal event itself.

Key descriptive fields include `tipo_documento`, `subtipo_documento`, `categoria`, and `origem_label`. These attributes are denormalized from the type dimension to provide quick access to the legal nature of the act. This table links extensively to several dimensions to fully contextualize **who**, **when**, **where**, and **what** happened in the legal transaction.



<div align="center">
  <sub>Figure 5 - Fact table Fato Escritura </sub><br>
  <img src="https://res.cloudinary.com/dm5korpwy/image/upload/v1764777345/fato_escritura_qwuss4.png">
  <sup>Source: Material produced by the authors (2025).</sup>
</div>

The analytical framework is established by the five shared dimension tables that surround `fato_escritura`, providing comprehensive context for every legal record.

The **`dim_data`** dimension, linked by `data_escritura_key`, establishes the precise temporal context (when the act was notarized), allowing analysis of the legal activity volume over specific periods using calendar attributes like Year, Quarter, and Month Name.

The **`dim_empresa`** table provides the identity of the **main entity** involved via the `cnpj_key`.

The **`dim_prestator_tornador`** dimension links via `cnpj_prestator_tornador_key` to identify the role of the other party involved in the legal instrument (e.g., service provider or taker in the context of the act).

The **`dim_localidade`** dimension, linked by `localidade_key`, provides the necessary geographical context (Municipality, State, CEP) related to where the notarization took place.

Finally, the **`dim_tipo_documento`** dimension, although denormalized into the fact table, is linked by `tipo_documento_key` to ensure the integrity of the document's classification (Category, Subtype).

Together, these dimensions enable complex analysis focused on transactional context, allowing analysts to quickly answer sophisticated questions regarding **who** was involved in **which type** of legal act, **when** and **where** it was executed, all segmented by entity and location attributes.

##### DBML Code 

```dbml
Table fato_escritura {
  escritura_id varchar [pk]
  tipo_documento_key int
  cnpj_key varchar
  data_escritura_key int
  localidade_key int
  cnpj_prestator_tornador_key varchar // Added for the complete model
  
  // Descriptive attributes (often denormalized in factless facts)
  tipo_documento varchar
  subtipo_documento varchar
  categoria varchar
  origem_label varchar
  
  // Example of potential measures (if any)
  // valor_ato decimal
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

Table dim_prestator_tornador {
  cnpj varchar [pk]
  razao_social varchar
  tipo varchar
}
```
#### 6. Fact Table: `fato_documentos_fiscais`

The fact table, `fato_documentos_fiscais`, is the transactional core of the fiscal documentation domain, representing **every individual fiscal document** (such as an invoice or a note) processed by the system. This table holds a large variety of measurable metrics that are critical for analysis, including `valor_servicos` (service value), `base_calculo` (calculation base), `valor_iss` (ISS tax value), `valor_deducoes` (deductions), `valor_descontos` (discounts), and the overall `valor_total` (total value).

Beyond the metrics, it stores unique identifiers and tracking information such as `numero_documento`, `numero_serie`, `access_key`, and details about the document's authorization (`authorization_protocol`, `authorization_datetime`). This table is complex due to its multiple links to identity dimensions, requiring **role-playing dimensions** to distinguish between the various parties involved (Principal, Provider, Taker).


<div align="center">
  <sub>Figure 6 - Fact table Fato Documentos Fiscais </sub><br>
  <img src="https://res.cloudinary.com/dm5korpwy/image/upload/v1764777394/fato_documentos_fiscais_yeifbh.png">
  <sup>Source: Material produced by the authors (2025).</sup>
</div>

The analytical framework is established by the shared dimension tables that surround `fato_documentos_fiscais`, providing comprehensive context for every fiscal document.

The **`dim_data`** dimension, linked by `data_emissao_key`, establishes the precise temporal context (when the document was created) and allows analysts to aggregate metrics by Year, Quarter, or Month Name.

The company dimensions play a crucial and multi-faceted role:
* The **`dim_empresa`** dimension is linked via `cnpj_principal_key` to identify the **main entity** responsible for the document, providing consistent access to attributes like the business's `razao_social` and `inscricao_estadual`.
* The **`dim_prestator_tornador`** dimension is utilized twice: once via `cnpj_prestador_key` to identify the **service provider**, and a second time via `cnpj_tomador_key` to identify the **service taker** (recipient). This double linking is vital for performing comparisons and flow analysis (e.g., "Total services provided to CNPJ X versus total services received from CNPJ Y").

The **`dim_localidade`** dimension, linked by `localidade_key`, provides the necessary geographical context (Municipality, State, CEP) related to the document's origin.

Finally, the **`dim_tipo_documento`** dimension, linked by `tipo_documento_key`, clarifies the category and subtype of the fiscal document, enabling segmentation of financial metrics based on the document's legal nature.

Together, these connections allow for advanced analysis such as tracking the flow of service values between different geographical regions, distinguishing between services provided and services received, all within a specific document category and time period.

##### DBML Code 

```dbml
Table fato_documentos_fiscais {
  documento_id varchar [pk]
  ingested_at timestamp
  numero_documento varchar
  numero_serie varchar
  codigo_autenticidade varchar
  aliquota_iss decimal
  base_calculo decimal
  valor_servicos decimal
  valor_total decimal
  valor_iss decimal
  valor_deducoes decimal
  valor_descontos decimal
  descricao_servicos text
  competencia varchar
  arquivo varchar
  caminho_completo varchar
  access_key varchar
  authorization_protocol varchar
  authorization_datetime timestamp
  label varchar

  // Foreign Keys (Dimensions are shared across the data model)
  cnpj_principal_key varchar [ref: > dim_empresa.cnpj]
  cnpj_prestador_key varchar [ref: > dim_prestator_tornador.cnpj]
  cnpj_tomador_key varchar [ref: > dim_prestator_tornador.cnpj]
  tipo_documento_key int [ref: > dim_tipo_documento.tipo_documento_key]
  data_emissao_key int [ref: > dim_data.data_key]
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

Table dim_prestator_tornador {
  cnpj varchar [pk]
  razao_social varchar
  tipo varchar
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
```

## 3. Conclusion: The Integrated Dimensional Bus

The resulting data model represents a robust **Dimensional Bus Architecture**, integrating six distinct business processes (`fato_tributos`, `fato_imoveis`, `fato_folha_pagamento`, `fato_escrituracao`, `fato_escritura`, and `fato_documentos_fiscais`) through a set of shared and standardized dimensions (`dim_data`, `dim_empresa`, `dim_localidade`, `dim_tipo_documento`, `dim_prestator_tornador`).

This centralized design ensures maximum data consistency and analytical flexibility. By establishing **common dimensions**—such as `dim_data` for all time-based events and `dim_empresa` for all corporate entities—the model supports **cross-process analysis**. Analysts can now compare tax payments (`fato_tributos`) against declared values in fiscal filings (`fato_escrituracao`) or track operational costs in payroll (`fato_folha_pagamento`) against service revenues (`fato_documentos_fiscais`), all using the same set of trusted business attributes (CNPJ, date, location).

The use of **role-playing keys** (e.g., `data_emissao_key` vs. `data_vencimento_key`) and **multi-role dimensions** (e.g., `dim_prestator_tornador` acting as both the Provider and Taker) addresses the complexity of fiscal data while preserving the simplicity and high performance inherent to the Star Schema approach. This architecture is scalable, ready to integrate future fact tables without redefining existing context, making it a sustainable foundation for advanced business intelligence and regulatory compliance analysis.
