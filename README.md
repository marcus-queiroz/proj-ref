
# Portfólio - Projetos Referência

Este documento tem como objetivo apresentar os projetos de referência do meu portfólio, que marcam minha transição de Business Intelligence tradicional para o campo da engenharia de dados moderna. Ao longo da minha trajetória profissional, adquiri uma sólida base em BI, mas diante das crescentes demandas tecnológicas, tenho direcionado meu foco para o uso de tecnologias de processamento distribuído, como Apache Spark, Delta Lake e Databricks.

Minha escolha por com Spark em um container Docker é motivada pelo desejo de entender melhor as abstrações das plataformas de dados. Ao trabalhar dessa forma, tenho a oportunidade de compreender como o Spark funciona pra além das abstrações e falicidades proporcionadas por um ambiente gerenciado, algo que aplico também a outras tecnologias.

Essa abordagem me capacita a desenvolver uma visão técnica mais profunda, permitindo a criação de soluções robustas, flexíveis e com adaptabilidade necessária para responder às diversas exigências dos projetos de engenharia de dados contemporâneos. Seja em ambientes locais ou de larga escala.





## Projeto 1: **Delta Lakehouse com Spark em um container Docker comparado ao Databricks**
[Link para o projeto](https://github.com/marcus-queiroz/proj-ref-spark-delta)

**Objetivo:** Construir um Data Lakehouse local utilizando Apache Spark, Delta Lake e Docker, organizando os dados em camadas Bronze, Silver e Gold, e comparar com a implementação no Databricks para explorar as diferenças entre os ambientes.

**Tecnologias Envolvidas:**

1. **Apache Spark**:
    - Framework de processamento de dados distribuído utilizado para realizar transformações de dados em larga escala. Spark processa dados de diferentes fontes e gerencia a movimentação entre as camadas Bronze, Silver e Gold.
    - Oferece suporte a consultas SQL e transformação de dados, facilitando a criação de pipelines de dados eficientes.

2. **Delta Lake**:
    - Camada de armazenamento que adiciona suporte a transações ACID ao Spark. Delta Lake é usado para armazenar dados em um formato otimizado, permitindo operações de leitura/escrita eficientes e suporte a controle de versão.
    - A integração nativa com Spark permite operações como merges e time travel, facilitando a gestão de dados ao longo do tempo.

3. **Databricks**:
    - Plataforma unificada para processamento de dados e machine learning que integra Apache Spark e Delta Lake, oferecendo um ambiente otimizado para desenvolvimento, orquestração e escalabilidade de pipelines de dados.
    - No Databricks, os pipelines de ETL são gerenciados por meio de notebooks interativos e jobs automatizados, substituindo a necessidade de uma ferramenta externa de orquestração.
    - A interface simplificada permite monitoramento em tempo real, versionamento de código e fácil integração com outras ferramentas do ecossistema Azure.
    - A capacidade de escalar recursos conforme a demanda possibilita o processamento eficiente em grandes volumes de dados, sem a complexidade de gerenciar infraestrutura diretamente.

4. **Docker**:
    - Usado para configurar e executar um ambiente Spark/Delta Lake local, permitindo que o pipeline seja facilmente replicado e executado em diferentes máquinas.
	- Garante que todas as dependências sejam gerenciadas dentro de containers, simplificando a configuração do ambiente.

5. **Web Scraping:**
    - Responsável pela extração de dados de fontes externas, automatizando o processo de coleta. Essa técnica é essencial para obter dados que não estão diretamente disponíveis em bancos de dados ou APIs.
    - Os dados extraídos serão armazenados inicialmente na camada Bronze, em seu formato bruto, para posterior transformação.




















