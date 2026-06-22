# Automotive Inventory Data Mart (SSIS ETL Project)

Este repositório contém o planeamento, a arquitetura e a implementação do pipeline de **ETL (Extract, Transform, Load)** para o sistema de Business Intelligence da *MotorData Car Dealership S.A.*, com foco no processo de negócio de **Gestão de Stock de Peças**. Os dados são fictícios, assim como a empresa. Este projeto foi desenvolvido numa unidade curricular do Mestrado em Sistemas Integrados de Apoio à Decisão.

---

## 📂 Estrutura do Repositório

* `/data_sources`: Tabelas de referência estáticas (ficheiros Excel).


* `/database_scripts`: Script SQL inicial para a criação dos schemas e povoamento dos dados operacionais.

  
* `/visual_studio_project_ssis`: Solução do Visual Studio (`.slnx`), projeto SSIS (`.dtproj`) e pacotes de fluxos de dados de ETL (`.dtsx`).


* `/documentation`: Artefactos de suporte, incluindo o documento de *Data Profiling* e o documento de *Source-to-Target Mapping* (ficheiros Excel).
---

## Arquitetura e Modelação Dimensional

### 1. Modelo Dimensional Elementar

O núcleo do Data Mart assenta numa tabela de factos transacional:

* **Tabela de Factos:** `FactMovimentoStock`.


* **Grão:** Uma linha por linha de documento de movimentação (entrada ou saída).


* **Métricas:** Quantidade Movimentada e Custo Total.


* **Dimensões:** `DimData`, `DimHora`, `DimProduto`, `DimTrabalhador`, `DimTipoMovimento` e `DimEntidade`.


### 2. Conceitos Relevantes de Modelação

* **Role-Playing Dimension (`DimEntidade`):** Uma única dimensão unificada que agrupa Concessionários e Fornecedores. Na tabela de factos, assume dois papéis distintos através de chaves estrangeiras separadas (`SK_Entidade_Concessionario` e `SK_Entidade_Fornecedor`).


* **Esquema Agregado (`FactMovimentoStockMensalAgg`):** Criado para otimizar o desempenho analítico em dashboards. Reduz a granularidade ao nível do **Mês / Produto / Entidade / Tipo de Movimento**.


---

## Processo e Engenharia de ETL

O pipeline foi totalmente desenvolvido em **SQL Server Integration Services (SSIS)** e encontra-se documentado na matriz **Source-to-Target Mapping** (`Grupo_3_SourceToTarget_Final.xlsm`).

### 1. Data Profiling (Perfil de Dados)

Antes da construção do ETL, foi conduzida uma análise sistemática de qualidade dos dados através de scripts SQL personalizados.

* **Tratamento de Anomalias:** Foi detetada e tratada uma anomalia controlada de valores nulos (`NULL`) na coluna de género da tabela dos trabalhadores, aplicando-se uma regra de substituição pelo membro desconhecido (`$n/a$`) no pipeline de transformação para mitigar falhas de integridade.


### 2. Carga Inicial vs. Carga Incremental

* **Carga Inicial (Full Load):** Responsável por limpar as tabelas de destino (respeitando a integridade referencial: factos primeiro, dimensões depois), instanciar as linhas especiais de membros desconhecidos e povoar as dimensões estáticas e dinâmicas a partir do zero.


* **Carga Incremental:** Desenvolvida para captar apenas os dados modificados desde a última execução bem-sucedida, recorrendo a uma tabela de controlo dedicada (`dsa.Control_ETL_Table`) que regista as marcas temporais por tabela fonte, evitando processamento redundante.


### 3. Pipeline das Dimensões (CDC, SKG e SCD)

As dimensões não estáticas seguem um padrão rígido de três sub-processos sequenciais executados em containers paralelos:

* **CDC (Change Data Capture):** Identificação de registos inseridos, modificados ou eliminados nas origens, cujo estado é persistido em `dsa.cdc_states`.


* **SKG (Surrogate Key Generation):** Mapeamento persistente na staging area (`DSA`) que associa cada *Business Key* da fonte a uma *Surrogate Key* (SK) gerada por auto-incremento, garantindo a estabilidade dos caminhos históricos.


* **SCD (Slowly Changing Dimensions):**
* **SCD Tipo 1:** Aplicado a atributos cujas alterações não justificam a preservação de histórico, sofrendo atualizações diretas (overwrite).


* **SCD Tipo 2:** Aplicado a atributos críticos de historização (ex: cargos de trabalhadores), gerindo novas versões de registos através de marcas temporais de validade (`Start_Date` e `End_Date`).


### 4. Carregamento da Tabela de Factos

* Consolida de forma incremental os fluxos de tabelas de staging independentes (`dsa.Entrada` e `dsa.Saida`) através de um operador `UNION ALL`.


* Realiza os *lookups* necessários para traduzir as chaves de negócio em *Surrogate Keys* antes da inserção final e calcula dinamicamente a métrica derivada de **Custo Total**.
