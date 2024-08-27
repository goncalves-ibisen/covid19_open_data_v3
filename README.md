
# Desafio NT Consult covid19_open_data_v3

Este projeto se trata da realização de um desafio elaborado pela NT Consult, com a finalidade de avaliar o conhecimento técnico do participante em Engenharia de Dados e/ou Ciência de Dados.

Mais detalhes sobre o desafio proposto em: https://gitsrv.ntconsult.com.br/desafios-ntconsult/desafio-data-science

## 1. Proposta de arquitetura de dados para o pipeline de ETL:

Proposta de arquitetura, como foi exigida a utilização do Databricks, faremos uso do Databricks Community Edition para elucidação do problema. De antemão, é importante salientar que o Databricks Community Edition possui várias limitações por sua natureza gratuita, no entanto, tal fato não é impedimento para realizar uma proposta de Data Lake em sua totalidade.

Desta forma, trago como proposta para o ETL em questão, a utilização de um cluster Databricks para execução de Databricks Notebooks, os quais contém scripts em Apache Spark, PySpark, Spark SQL, Python, Pandas, Databricks SQL, entre outras linguagens, frameworks e bibliotecas. Também faremos uso de arquivos CSV e Parquet Delta Live Table para persistência dos dados, bem como, Dataframes para manipulação de dados.
![arquitetura-databricks-community-edition drawio](https://github.com/user-attachments/assets/23a0ad2c-fd7b-4fbd-ae66-7cea85d9e127)
_Figura 1: proposta de arquitetura Azure Databricks Communit Edition Delta Lake_

A Figura 1 representa uma arquitetura de dados utilizando o **Databricks** em conjunto com a **Arquitetura Medalhão** (Medallion Architecture).

### 1.1 **Fonte de Dados (Source Google Storage API)**
   - **Aggregated CSV COVID-19 Open Data V3**: Os dados brutos estão armazenados em um arquivo CSV no Google Storage. Esses dados são relacionados ao COVID-19, e são acessados via uma API.

### 1.2 **Pipeline de Transformação de Dados no Databricks**
   - **Notebook de Extração API**: Este notebook extrai os dados da fonte e os coloca na camada **Landing** no Databricks DBFS (Databricks File System).

### 1.3 **Arquitetura de Medalhão: Camadas Bronze, Silver e Gold**
   - **Bronze Layer**:
     - O notebook **Bronze** processa os dados extraídos, armazenando-os na camada **DBFS - BRONZE**. Aqui, os dados geralmente ainda estão em um formato bruto, com mínima transformação.
     - Um **Databricks SQL Warehouse BRONZE** está configurado para consultas SQL sobre os dados dessa camada.

   - **Silver Layer**:
     - O notebook **Silver** aplica mais transformações nos dados da camada Bronze, refinando e limpando os dados para armazenamento na **DBFS - SILVER**.
     - Um **Databricks SQL Warehouse SILVER** está disponível para realizar consultas SQL mais otimizadas com os dados refinados.

   - **Gold Layer**:
     - O notebook **Gold** realiza transformações adicionais, focando em criar datasets altamente refinados e prontos para análise, armazenando-os na **DBFS - GOLD**.
     - Um **Databricks SQL Warehouse GOLD** permite realizar consultas diretamente nos dados de alta qualidade dessa camada.

### 1.4 **Camada de Analytics e Dataviz**
   - Finalmente, os dados refinados na camada Gold são disponibilizados para a **Camada de Analytics e Dataviz** do Databricks. Essa camada é usada para criar visualizações, dashboards e análises avançadas.

### 1.5 **Infraestrutura**
   - Toda a arquitetura está configurada dentro de uma **VPC (Virtual Private Cloud)**, garantindo segurança e isolamento dos dados e serviços.

Em resumo, trata-se de um pipeline típico de dados em uma plataforma Databricks utilizando a Arquitetura Medalhão. Os dados brutos são extraídos de um CSV, transformados e refinados em três camadas (Bronze, Silver, Gold), com cada camada aumentando em refinamento e prontidão para análise, até serem utilizados para análises e visualizações finais. Cada camada é acompanhada por um Databricks SQL Warehouse correspondente para consultas.

Essa arquitetura é ideal para gerenciar grandes volumes de dados e melhorar a qualidade dos dados à medida que eles avançam pelo pipeline.

---

## 2. Armazenamento na Plataforma Databricks utilizando o DBFS - Databricks File System

### 2.1 **Camada Landing com CSV**
   - **Escolha de Formato (CSV)**: 
     - Para a camada **Landing**, foi escolhido o formato **CSV** para armazenar os dados no estado bruto. 
     - **Vantagem**: O CSV é um formato de arquivo simples e amplamente suportado, ideal para a ingestão inicial de dados antes de qualquer processamento.
     - **Consideração**: Embora fácil de manipular, o CSV não oferece vantagens em termos de compactação, indexação ou particionamento, tornando-o menos eficiente para consultas e manipulações em grandes volumes de dados.

### 2.2 **Camadas Bronze, Silver e Gold com Parquet Delta Live Tables**
   - **Escolha de Formato (Parquet)**:
     - As camadas **Bronze**, **Silver** e **Gold** utilizam o formato **Parquet** com suporte a **Delta Live Tables**.
     - **Compactação e Eficiência**: O Parquet é um formato colunar que oferece compactação superior e é altamente eficiente para dados tabulares. Seu layout colunar permite a leitura seletiva de colunas, reduzindo o volume de dados lido do disco e otimizando a performance.
     - **Particionamento**: Parquet permite particionamento por colunas, o que melhora significativamente a performance em operações de leitura/escrita ao trabalhar com grandes volumes de dados.
     - **Delta Live Tables**:
       - Delta Live Tables permite a criação de pipelines ETL eficientes e automatizados, com suporte à versionamento de dados, transações ACID, e tratamento de dados incrementais. 
       - **Time Travel e Controle de Versões**: A funcionalidade de Delta Lake permite "time travel", onde versões anteriores dos dados podem ser acessadas para auditoria ou rollback.
   
### 2.3 **Databricks SQL Warehouse e Tabelas Externas**
   - **Criação de Delta Tables**:
     - A partir dos dados armazenados em arquivos Parquet nas camadas Bronze, Silver, e Gold, são criadas **Delta Tables** no **Databricks SQL Warehouse** como **External Tables**.
     - **Vantagens de Delta Tables**:
       - **Consistência de Dados**: Delta Tables oferecem garantia de transações ACID, assegurando que as operações de leitura e escrita sejam consistentes e confiáveis.
       - **Performance de Consultas**: Delta Tables são otimizadas para consultas interativas e analíticas, especialmente em ambientes de alta concorrência e grande escala.
       - **Gerenciamento de Esquema**: Suporte a evolução de esquemas, permitindo adição de colunas ou mudanças de tipo sem necessidade de recriação das tabelas.
       - **Integração com Databricks SQL**: Como External Tables, essas Delta Tables permitem que as consultas SQL aproveitem a performance e as otimizações do Delta Lake diretamente no Databricks SQL Warehouse.

### 2.4 **Considerações Técnicas**
   - **Spark Tuning**:
     - A escolha do Parquet em conjunto com Delta Tables é ideal para **Spark Tuning**, pois permite a aplicação de técnicas como **partition pruning** e **predicate pushdown**, que são fundamentais para melhorar a performance de consultas em grandes datasets.
   - **Escalabilidade e Performance**:
     - O uso de Parquet aliado às otimizações de Delta Lake garante que o pipeline seja escalável, suportando grandes volumes de dados com alta eficiência e performance.
     - **Compactação e Armazenamento**: Parquet, com sua compactação eficiente, reduz significativamente o espaço em disco necessário, tornando a arquitetura não só performática, mas também economicamente viável.

Essa abordagem maximiza a eficiência do pipeline de dados, oferecendo uma solução robusta e escalável para grandes volumes de dados, com suporte a análises avançadas e consultas de alta performance.

---


## 3. Resumo das Análises de Dados

As análises dos dados relacionados ao COVID-19, assim como as métricas, KPIs, visualizações e insights gerados, estão documentados de forma detalhada no notebook `nt1_covid19_open_data_v3_analises`. Este notebook serve como um repositório central de análise, proporcionando uma visão abrangente e concisa dos dados processados.

#### Principais Características:
- **Métricas e KPIs**: O notebook contém cálculos e interpretações das principais métricas e KPIs derivados dos dados, oferecendo uma compreensão clara do impacto e das tendências associadas ao COVID-19.
- **Visualizações Interativas**: São apresentadas diversas visualizações gráficas que facilitam a interpretação dos dados, permitindo uma análise visual das principais tendências e padrões.
- **Insights Gerados**: A partir das análises, insights valiosos são extraídos e documentados, fornecendo uma base sólida para a tomada de decisões estratégicas.
- **Explicações Concisas**: Cada etapa do processo analítico é acompanhada de explicações sucintas, assegurando que o racional por trás de cada análise seja facilmente compreendido.

![image](https://github.com/user-attachments/assets/37985716-0085-43d1-8abd-5729dfaaa431)
_Figura 2: exemplo de análise gerada_


![image](https://github.com/user-attachments/assets/d8654bbf-b368-4e75-9120-580d412a0907)

_Figura 3: exemplo de visualização de gráfico gerada_

Este notebook não só centraliza o trabalho analítico realizado, mas também serve como um guia técnico para a interpretação dos resultados, facilitando a comunicação e o compartilhamento dos insights com outras partes interessadas.

---

## 4. Implementação de Medidas de Segurança
#### 4.1 Garantia da Segurança dos Dados Durante o Processo de ETL
A segurança dos dados durante o processo de ETL (Extração, Transformação e Carga) é de extrema importância. Para garantir a proteção dos dados ao longo deste pipeline, são implementadas as seguintes práticas:

- **Transmissão Segura de Dados**: Durante a transferência de dados, utilizamos protocolos de comunicação seguros, como HTTPS e SSL/TLS, para criptografar os dados em trânsito, assegurando que informações sensíveis não sejam interceptadas ou comprometidas.
  
- **Criptografia de Dados em Repouso**: Os dados são criptografados antes de serem armazenados, utilizando algoritmos de criptografia robustos para proteger os dados em repouso contra acessos não autorizados.

- **Controles de Acesso Baseados em Funções (RBAC)**: Implementamos controles de acesso rigorosos baseados em funções, garantindo que apenas usuários autorizados possam acessar, modificar ou processar os dados.

- **Auditoria e Monitoramento**: Mecanismos de auditoria são empregados para registrar e monitorar todas as atividades relacionadas aos dados, possibilitando a detecção de acessos ou ações não autorizadas e facilitando a resposta rápida a incidentes de segurança.

#### 4.2 Medidas de Segurança Implementadas para Proteger os Dados na Plataforma Databricks

Embora o Databricks Community Edition ofereça uma plataforma poderosa para aprendizado e prototipagem, ele possui limitações em termos de funcionalidades de segurança, devido à sua natureza gratuita. Isso significa que algumas das medidas de segurança avançadas disponíveis em ambientes empresariais não estão presentes. No entanto, se estivéssemos desenvolvendo o Data Lake em um ambiente mais robusto, como a **Microsoft Azure**, poderíamos implementar as seguintes medidas de segurança:

- **VPN (Virtual Private Network)**: Utilização de VPNs para garantir que o tráfego de dados entre os usuários e a infraestrutura do Databricks seja seguro, reduzindo a exposição a ameaças externas.

- **Azure Defender for Cloud**: Implementação do Azure Defender para monitorar e proteger a infraestrutura de dados contra ameaças potenciais, fornecendo proteção contínua para os recursos armazenados na nuvem.

- **Microsoft Entra ID**: Uso do Microsoft Entra ID para a criação, login e autenticação de usuários, garantindo um gerenciamento seguro e centralizado das identidades e acessos.

- **VNETs (Virtual Networks)**: Configuração de VNETs para isolar a rede de dados, controlando e limitando o tráfego de rede entre diferentes serviços e recursos, garantindo um ambiente seguro e segmentado.

- **Firewall**: Implementação de firewalls para proteger as fronteiras da rede, permitindo apenas tráfego autorizado e bloqueando acessos não autorizados aos recursos do Databricks.

- **Autenticação Multifator (MFA)**: Utilização de MFA para adicionar uma camada extra de segurança durante o login do usuário, exigindo múltiplas formas de verificação de identidade antes de conceder acesso.

Essas medidas de segurança seriam fundamentais para proteger os dados em um ambiente de produção, especialmente em um cenário de grandes volumes de dados e requisitos rigorosos de conformidade e segurança.

---

## 6. Estratégia de Monitoramento:
#### 6.1 Estratégia de Monitoramento para o Pipeline de ETL

**Observação:** Devido a severas limitações do Databricks Community Edition, faço aqui uma reflexão se estivéssemos utilizando um Data Lake na Azure, AWS e/ou GCP.

O monitoramento eficaz de um pipeline de ETL (Extração, Transformação e Carga) é crucial para garantir que o processo ocorra de forma eficiente, segura e sem interrupções. A estratégia de monitoramento para o pipeline de ETL deve incluir os seguintes elementos:

- **Monitoramento Contínuo e em Tempo Real**: Implementação de um sistema de monitoramento contínuo que capture e analise dados de performance em tempo real. Isso permite a detecção precoce de anomalias, falhas ou gargalos no processo de ETL, possibilitando ações corretivas imediatas.

- **Alertas e Notificações**: Configuração de alertas automáticos para notificar a equipe responsável em caso de falhas críticas, degradação de performance ou violações de SLAs (Acordos de Nível de Serviço). Isso assegura uma resposta rápida e minimiza o impacto de potenciais problemas.

- **Auditoria e Registro de Logs**: Manutenção de um log detalhado de todas as atividades e transações do pipeline. Esses registros são essenciais para auditoria, análise de incidentes e para entender o comportamento histórico do sistema.

- **Automatização de Tarefas de Manutenção**: Automatização de tarefas de manutenção rotineiras, como limpeza de logs antigos, verificação de integridade de dados e reprocessamento de tarefas falhadas. Isso reduz a carga de trabalho manual e aumenta a confiabilidade do pipeline.

- **Verificação de Qualidade de Dados**: Implementação de verificações de qualidade em cada etapa do pipeline, assegurando que os dados sejam consistentes, completos e livres de erros antes de avançarem para as próximas fases do processo.

#### 6.2 Métricas Chave a Serem Monitoradas e Ferramentas Utilizadas

Identificar as métricas chave e utilizar as ferramentas adequadas para monitoramento é fundamental para garantir a eficiência e eficácia do pipeline de ETL. As principais métricas e ferramentas incluem:

- **Métricas de Performance**:
  - **Latência**: Tempo total que o pipeline leva para processar um lote de dados, desde a extração até a carga final. É crucial para garantir que os dados estejam disponíveis dentro dos SLAs estabelecidos.
  - **Throughput**: Quantidade de dados processados em um determinado período de tempo. Monitorar essa métrica ajuda a avaliar a capacidade do pipeline e identificar potenciais gargalos.
  - **Taxa de Erros**: Frequência de falhas durante o processamento, como erros de leitura/escrita, transformações incorretas ou falhas na carga dos dados. Manter essa métrica baixa é essencial para a confiabilidade do pipeline.

- **Métricas de Qualidade de Dados**:
  - **Integridade dos Dados**: Percentual de dados válidos versus dados inválidos ou corrompidos. Essa métrica assegura que os dados carregados estão em conformidade com as expectativas.
  - **Completude dos Dados**: Percentual de registros que possuem todos os campos obrigatórios preenchidos. Isso ajuda a identificar lacunas ou inconsistências no dataset.

- **Ferramentas de Monitoramento**:
  - **Databricks Operational Metrics**: Ferramenta nativa do Databricks que permite o monitoramento detalhado de jobs, pipelines, e clusters em tempo real, incluindo alertas customizáveis e dashboards interativos.
  - **Azure Monitor** (em caso de uso da Azure): Ferramenta de monitoramento que pode ser integrada para acompanhar a saúde e performance dos recursos, oferecendo métricas e insights detalhados sobre o pipeline de ETL.
  - **Prometheus e Grafana**: Combinação poderosa para coleta e visualização de métricas, especialmente útil em ambientes de alta escala, onde é necessário monitorar múltiplos componentes do pipeline.
  - **Log Analytics**: Utilizado para armazenar, consultar e visualizar logs, permitindo uma análise profunda dos eventos e o rastreamento das atividades do pipeline ao longo do tempo.

A combinação dessas métricas e ferramentas assegura um monitoramento abrangente e eficaz do pipeline de ETL, permitindo uma gestão proativa e a manutenção da performance e integridade dos dados ao longo de todo o processo.

---

## 7. Execução e utilização da Arquitetura e Scripts elaborados neste Desafio

### 7.1 Dos equipamentos e softwares necessários para execução do trabalho realizado 
1. É necessário possuir um computador com acesso à internet;
2. É necessário ter uma conta no Databricks Community Edition;
3. Logar na plataforma do Databricks Community Edition;
4. Criar um Cluster Apache Spark 14.3 LTS ou Superior;
5. Enviar os notebooks no formato "dbc" para o workspace do Databricks Community Edition;
6. Executar os notebooks seguindo a ordem sequencial **nt1..., nt2..., nt3...**;
   6.1 Outra alternativa mais rápida é executar o seguinte notebook: **nt_executar_notebooks_em_cadeia**

### 7.2 Passo a passo de como executar o ETL proposto:

Acessar a página de Login do Databricks Community Edition disponível em: https://community.cloud.databricks.com/login.html
![image](https://github.com/user-attachments/assets/0a587b70-f26d-44b3-ad09-53c2caf786a4)
_Figura 4: tela de Login do Databricks Community Edition_

Digite o seu e-mail e sua senha para acessar a plataforma do Databricks Community Edition.

Se você não possuir uma conta, crie uma cliando em **Sign Up**.

![image](https://github.com/user-attachments/assets/c447d3c3-de27-4c72-a917-4599b4df9de3)
_Figura 5: tela inicial do painel administrativo do Databricks Community Edition_

Após login no painel administratitvo do Databricks Community Edition, clique em **Compute** no menu lateral esquerdo. Obs.: o mesmo pode estar colapsado, acaso necessário, clique em **Expand Menu**.

![image](https://github.com/user-attachments/assets/71ab75a0-382e-42c2-9c37-c91f709ac0c2)

_Figura 6: tela do Menu Compute_

Clique em **Create Compute**.

![image](https://github.com/user-attachments/assets/e729ca9a-bb13-45da-9932-9612e03c110c)
_Figura 7: tela Create Compute/Cluster_

Nesta seção: dê um nome para seu compute.
Databricks runtime version: escolha a versão **Runtime 14.3 LTS (Scala 2.12, Spark 3.5.0)** ou superior.
Ao final, clique em **CREATE COMPUTE** e aguarde o cluster inicializar e ficar operacional.


