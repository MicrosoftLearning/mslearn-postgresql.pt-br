---
lab:
  title: Migração de anco de dados PostgreSQL online
  module: Migrate to Azure Database for PostgreSQL Flexible Server
---

## Migração de anco de dados PostgreSQL online

Neste exercício, você configurará a replicação lógica entre um servidor PostgreSQL de origem e o servidor flexível do Banco de Dados do Azure para PostgreSQL para permitir que uma atividade de migração online ocorra.

## Antes de começar

Você precisa ter uma assinatura própria do Azure para concluir este exercício. Se você não tiver uma assinatura do Azure, crie uma [conta de avaliação gratuita do Azure](https://azure.microsoft.com/free).

> **Observação:** este exercício exigirá que o servidor que você usa como origem para a migração seja acessível pelo servidor flexível do Banco de Dados do Azure para PostgreSQL para que ele possa se conectar e migrar bancos de dados. Isso exigirá que o servidor de origem seja acessível por meio de um endereço IP e porta públicos. > Uma lista de endereços IP da região do Azure pode ser baixada de [Intervalos de IP e marcas de serviço do Azure – Nuvem pública](https://www.microsoft.com/en-gb/download/details.aspx?id=56519) para ajudar a minimizar os intervalos permitidos de endereços IP em suas regras de firewall com base na região do Azure usada.

Abra o firewall do servidor para permitir que o recurso de migração de servidor flexível do Banco de Dados do Azure para PostgreSQL acesse a origem do servidor PostgreSQL, que por padrão é a porta TCP 5432.
>
Ao usar um dispositivo de firewall na frente de seus bancos de dados de origem, talvez seja necessário adicionar regras de firewall para permitir que o serviço de Migração de servidor flexível do Banco de Dados do Azure para PostgreSQL acesse os bancos de dados de origem para migração.
>
> A versão máxima suportada do PostgreSQL para migração é a versão 16.

### Pré-requisitos

> **Observação**: antes de iniciar este exercício, você precisará ter concluído o exercício anterior para ter os bancos de dados de origem e de destino prontos para configurar a replicação lógica, pois este exercício se baseia na atividade do exercício.

## Criar publicação – Servidor de origem

1. Abra o PGAdmin e conecte-se ao servidor de origem que contém o banco de dados que atuará como a origem da sincronização de dados com o servidor flexível do Banco de Dados do Azure para PostgreSQL.
1. Abra uma nova janela de consulta conectada ao banco de dados de origem com os dados que queremos sincronizar.
1. Configure o servidor de origem wal_level como **lógico** para permitir a publicação de dados.
    1. Localize e abra o arquivo **postgresql.conf** no diretório bin dentro do diretório de instalação do PostgreSQL.
    1. Encontre a linha com a definição de configuração **wal_level**.
    1. Certifique-se de que a linha não seja comentada e defina o valor como **lógico**.
    1. Salve e feche o arquivo.
    1. Reinicie o serviço PostgreSQL.
1. Agora configure uma publicação que conterá todas as tabelas do banco de dados.

    ```SQL
    CREATE PUBLICATION migration1 FOR ALL TABLES;
    ```

## Criar assinatura - Servidor de destino

1. Abra o PGAdmin e conecte-se ao servidor flexível do Banco de Dados do Azure para PostgreSQL, que contém o banco de dados que atuará como o destino para a sincronização de dados do servidor de origem.
1. Abra uma nova janela de consulta conectada ao banco de dados de origem com os dados que queremos sincronizar.
1. Criar a assinatura para o servidor de origem.

    ```sql
    CREATE SUBSCRIPTION migration1
    CONNECTION 'host=<source server name> port=<server port> dbname=adventureworks application_name=migration1 user=<username> password=<password>'
    PUBLICATION migration1
    WITH(copy_data = false)
    ;    
    ```

1. Verificar o status de replicação de tabela.

    ```SQL
    SELECT PT.schemaname, PT.tablename,
        CASE PS.srsubstate
            WHEN 'i' THEN 'initialize'
            WHEN 'd' THEN 'data is being copied'
            WHEN 'f' THEN 'finished table copy'
            WHEN 's' THEN 'synchronized'
            WHEN 'r' THEN ' ready (normal replication)'
            ELSE 'unknown'
        END AS replicationState
    FROM pg_publication_tables PT,
            pg_subscription_rel PS
            JOIN pg_class C ON (C.oid = PS.srrelid)
            JOIN pg_namespace N ON (N.oid = C.relnamespace)
    ;
    ```

## Testar replicação de dados

1. No servidor de origem, verifique a contagem de linhas da tabela de ordem de serviço.

    ```SQL
    SELECT COUNT(*) FROM production.workorder;
    ```

1. No servidor de destino, verifique a contagem de linhas da tabela da ordem de serviço.

    ```SQL
    SELECT COUNT(*) FROM production.workorder;
    ```

1. Verifique se os valores de contagem de linhas correspondem.
1. Agora baixe o arquivo Lab11_workorder.csv do repositório [aqui](https://github.com/MicrosoftLearning/mslearn-postgresql/tree/main/Allfiles/Labs/11) para C:\
1. Carregue novos dados na tabela de ordem de serviço no servidor de origem do CSV usando o comando a seguir.

    ```Bash
    psql --host=localhost --port=5432 --username=postgres --dbname=adventureworks --command="\COPY production.workorder FROM 'C:\Lab11_workorder.csv' CSV HEADER"
    ```

A saída do comando deve ser `COPY 490`, indicando que 490 linhas adicionais foram gravadas na tabela a partir do arquivo CSV.

1. Verifique as contagens de linhas para a tabela de ordem de serviço na correspondência de origem (72591 linhas) e destino para verificar se a replicação de dados está funcionando.

## Limpeza do exercício

O Banco de Dados do Azure para PostgreSQL que implantamos neste exercício incorrerá custos: você pode excluir o servidor após este exercício. Como alternativa, você pode excluir o **grupo de recursos rg-learn-work-with-postgresql-eastus** para remover todos os recursos que implantamos como parte deste exercício.
