---
lab:
  title: Configurar parâmetros do sistema e explorar metadados com catálogos e exibições do sistema
  module: Configure and manage Azure Database for PostgreSQL
---

# Configurar parâmetros do sistema e explorar metadados com catálogos e exibições do sistema

Neste exercício, você examinará os parâmetros e metadados do sistema no PostgreSQL.

## Antes de começar

> [!IMPORTANT]
> Você precisará usar sua assinatura do Azure para concluir os exercícios deste módulo. Caso ainda não tenha uma assinatura do Azure, inscreva-se em uma conta de avaliação gratuita em [Criar soluções na nuvem com uma conta gratuita do Azure](https://azure.microsoft.com/free/).

## Crie o ambiente de exercício

### Implantar recursos na assinatura do Azure

Esta etapa orienta você no uso de comandos da CLI do Azure do Azure Cloud Shell para criar um grupo de recursos e executar um script Bicep para implantar os serviços do Azure necessários para concluir este exercício em sua assinatura do Azure.

> Observação
>
> Se você estiver fazendo vários módulos neste roteiro de aprendizagem, poderá compartilhar o ambiente do Azure entre eles. Nesse caso, você só precisa concluir essa etapa de implantação de recursos uma vez.

1. Abra um navegador da Web e acesse o [portal do Azure](https://portal.azure.com/).

2. SElecione o ícone do **Cloud Shell** na barra de ferramentas do portal do Azure para abrir um novo painel do [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) na parte inferior da janela do navegador.

    ![Captura de tela da barra de ferramentas do Azure com o ícone do Cloud Shell realçado por uma caixa vermelha.](media/07-portal-toolbar-cloud-shell.png)

    Se solicitado, selecione as opções necessárias para abrir um shell do *Bash*. Se você já usou um console do *PowerShell*, alterne-o para um shell do *Bash*.

3. No prompt do Cloud Shell, insira o seguinte para clonar o repositório GitHub que contém os recursos do exercício:

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

4. Em seguida, você executará três comandos para definir variáveis para reduzir a digitação redundante ao usar comandos da CLI do Azure para criar recursos do Azure. As variáveis representam o nome a ser atribuído ao seu grupo de recursos (`RG_NAME`), a região do Azure (`REGION`) na qual os recursos serão implantados e uma senha gerada aleatoriamente para o logon de administrador do PostgreSQL (`ADMIN_PASSWORD`).

    No primeiro comando, a região atribuída à variável correspondente é `eastus`, mas você também pode substituí-la por um local de sua preferência.

    ```bash
    REGION=eastus
    ```

    O comando a seguir atribui o nome a ser usado para o grupo de recursos que abrigará todos os recursos usados neste exercício. O nome do grupo de recursos atribuído à variável correspondente é `rg-learn-work-with-postgresql-$REGION`, onde `$REGION` é o local especificado acima. No entanto, você pode alterá-lo para qualquer outro nome de grupo de recursos que atenda às suas preferências.

    ```bash
    RG_NAME=rg-learn-work-with-postgresql-$REGION
    ```

    O comando final gera aleatoriamente uma senha para o login de administrador do PostgreSQL. Certifique-se de copiá-lq para um local seguro para que você possa usá-la mais tarde para se conectar ao seu servidor flexível PostgreSQL.

    ```bash
    a=()
    for i in {a..z} {A..Z} {0..9}; 
       do
       a[$RANDOM]=$i
    done
    ADMIN_PASSWORD=$(IFS=; echo "${a[*]::18}")
    echo "Your randomly generated PostgreSQL admin user's password is:"
    echo $ADMIN_PASSWORD
    ```

5. Se você tiver acesso a mais de uma assinatura do Azure e sua assinatura padrão não for aquela na qual você deseja criar o grupo de recursos e outros recursos para este exercício, execute este comando para definir a assinatura apropriada, substituindo o token `<subscriptionName|subscriptionId>` pelo nome ou ID da assinatura que você deseja usar:

    ```azurecli
    az account set --subscription <subscriptionName|subscriptionId>
    ```

6. Execute o seguinte comando da CLI do Azure para criar um grupo de recursos:

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

7. Por fim, use a CLI do Azure para executar um script de implantação do Bicep para provisionar recursos do Azure em seu grupo de recursos:

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    O script de implantação do Bicep provisiona os serviços do Azure necessários para concluir este exercício em seu grupo de recursos. Os recursos implantados são um Banco de Dados do Azure para PostgreSQL – servidor flexível O script bicep também cria um banco de dados que pode ser configurado na linha de comando como um parâmetro.

    A implantação tipicamente leva vários minutos para ser concluída. Você pode monitorá-lo no Cloud Shell ou navegar até a página **Implantações** do grupo de recursos criado acima e observar o progresso da implantação lá.

8. Feche o painel do Cloud Shell quando a implantação do recurso for concluída.

### Solucionar erros de implantação

Você pode encontrar alguns erros ao executar o script de implantação do Bicep. As mensagens mais comuns e as etapas para resolvê-las são:

- Se você executou anteriormente o script de implantação do Bicep para este roteiro de aprendizagem e, posteriormente, excluiu os recursos, poderá receber uma mensagem de erro como a seguinte se estiver tentando executar novamente o script dentro de 48 horas após a exclusão dos recursos:

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is '4e87a33d-a0ac-4aec-88d8-177b04c1d752'. See inner errors for details."}
    
    Inner Errors:
    {"code": "FlagMustBeSetForRestore", "message": "An existing resource with ID '/subscriptions/{subscriptionId}/resourceGroups/rg-learn-postgresql-ai-eastus/providers/Microsoft.CognitiveServices/accounts/{accountName}' has been soft-deleted. To restore the resource, you must specify 'restore' to be 'true' in the property. If you don't want to restore existing resource, please purge it first."}
    ```

    Se você receber essa mensagem, modifique o comando `azure deployment group create` acima para definir o parâmetro `restore` igual a `true` e execute-o novamente.

- Se a região selecionada estiver impedida de provisionar recursos específicos, você deverá definir a variável `REGION` como um local diferente e executar novamente os comandos para criar o grupo de recursos e executar o script de implantação do Bicep.

    ```bash
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- Se o script não conseguir criar um recurso de IA devido ao requisito de aceitar o contrato de IA responsável, você poderá enfrentar o seguinte erro; nesse caso, use a interface do usuário do Portal do Azure para criar um recurso dos Serviços de IA do Azure e, em seguida, execute novamente o script de implantação.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

### Conectar-se ao banco de dados com o Azure Data Studio

1. Se você ainda não tiver feito isso, clone os scripts de laboratório do repositório GitHub [PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql.git) localmente:
    1. Abra uma linha de comando/terminal.
    1. Execute o comando:
       ```bash
       md .\DP3021Lab
       git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git .\DP3021Lab
       ```
       > OBSERVAÇÃO
       > 
       > Se o **git** não estiver instalado, [baixe e instale o aplicativo ***git***](https://git-scm.com/download) e tente executar os comandos anteriores novamente.
1. Se você ainda não o instalou, [baixe e instale ***o Azure Data Studio***](https://go.microsoft.com/fwlink/?linkid=2282284).
1. Se você não instalou a extensão **PostgreSQL** no Azure Data Studio, instale-a agora.
1. Abra o Azure Data Studio.
1. Selecione **Conexões**.
1. Selecione **Servidores** e **Nova conexão**.
1. Em **Tipo de conexão**, selecione **PostgreSQL**.
1. Em **nome do servidor**, digite o valor especificado ao implantar o servidor.
1. Em **Nome de usuário**, digite **pgAdmin**.
1. Em **Senha**, digite a senha gerada aleatoriamente para o login do **pgAdmin** que você gerou
1. Selecione **Lembrar senha**.
1. Clique em **Conectar**
1. Se você ainda não criou o banco de dados zoodb, selecione **Arquivo**, **Abrir arquivo** e navegue até a pasta em que salvou os scripts. Selecione **../Allfiles/Labs/02/Lab2_ZooDb.sql** e **Abrir**.
   1. Realce as instruções **DROP** e **CREATE** e execute-as.
   1. Na parte superior da tela, use a seta suspensa para exibir os bancos de dados no servidor, incluindo o zoodb e os bancos de dados do sistema. Selecione o banco de dados **zoodb**.
   1. Realce as seções **Criar tabelas**, **Criar chaves estrangeiras** e **Preencher tabelas** e execute-as.
   1. Realce as três instruções **SELECT** no final do script e execute-as para verificar se as tabelas foram criadas e preenchidas.

## Tarefa 1: explorar o processo de limpeza no PostgreSQL

1. Se ainda não estiver aberto, abra o Azure Data Studio.
1. No Azure Data Studio, selecione **Arquivo**, **Abrir Arquivo** e navegue até os scripts de laboratório. Selecione **../Allfiles/Labs/07/Lab7_vacuum.sql** e selecione **Abrir**. Se necessário, reconecte-se ao servidor.
1. Selecione o banco de dados **zoodb** na lista suspensa de banco de dados.
1. Realce e execute a seção **Verificar se o banco de dados zoodb está selecionado**. Se necessário, faça do zoodb o banco de dados atual usando a lista suspensa.
1. Destaque e execute a seção **Exibir tuplas mortas**. Essa consulta exibe o número de tuplas mortas e ativas no banco de dados. Anote o número de tuplas mortas.
1. Realce e execute a seção **Alterar peso** 10 vezes em uma linha. Essa consulta atualiza a coluna de peso para todos os animais.
1. Execute a seção em **Exibir tuplas mortas** novamente. Anote o número de tuplas mortas após a conclusão das atualizações.
1. Execute a seção em **Executar VACUUM manualmente** para executar o processo de aspiração.
1. Execute a seção em **Exibir tuplas mortas** novamente. Anote o número de tuplas mortas após a execução do processo de aspiração.

## Tarefa 2: configurar parâmetros de limpeza de servidor automática

1. No portal do Azure, navegue até o servidor flexível do Banco de Dados do Azure para PostgreSQL.
1. Em **Configurações**, selecione **Parâmetros do servidor**.
1. Na barra de pesquisa, digite **`vacuum`**. Localize os seguintes parâmetros e altere os valores da seguinte maneira:
    1. autovacuum = ON (deve ser ON por padrão)
    1. autovacuum_vacuum_scale_factor = 0.1
    1. autovacuum_vacuum_threshold = 50

    Isso é como executar o processo de aspiração automática quando 10% de uma tabela tem linhas marcadas para exclusão ou 50 linhas atualizadas ou excluídas em qualquer tabela.

1. Selecione **Salvar**. O servidor é reiniciado.

## Tarefa 3: exibir metadados do PostgreSQL no portal do Azure

1. Acesse o [portal do Azure](https://portal.azure.com) e entre.
1. Pesquise **Banco de Dados do Azure para PostgreSQL** e selecione-o.
1. Selecione o servidor flexível do Banco de Dados do Azure para PostgreSQL criado para este exercício.
1. Em **Monitoramento**, selecione **Métricas**.
1. Selecione **Métrica** e selecione **Percentual de CPU**.
1. Observe que você pode exibir várias métricas sobre seus bancos de dados.

## Tarefa 4: exibir dados em tabelas de catálogo do sistema

1. Alternar para o Azure Data Studio.
1. Em **SERVERS**, selecione o servidor PostgreSQL e aguarde até que uma conexão seja feita e um círculo verde seja exibido no servidor.
1. Clique com o botão direito do mouse no servidor e selecione **Nova Consulta**.
1. Digite o seguinte SQL e selecione **Executar**:

    ```sql
    SELECT datname, xact_commit, xact_rollback FROM pg_stat_database;
    ```

1. Observe que você pode exibir confirmações e reversões para cada banco de dados.

## Exibir uma consulta de metadados complexa usando uma exibição do sistema

1. Clique com o botão direito do mouse no servidor e selecione **Nova Consulta**.
1. Digite o seguinte SQL e selecione **Executar**:

    ```sql
    SELECT *
    FROM pg_catalog.pg_stats;
    ```

1. Observe que você pode exibir uma grande quantidade de informações de estatísticas.
1. Usando exibições do sistema, você pode reduzir a complexidade do SQL que precisa gravar. A consulta anterior precisaria do seguinte código se você não estivesse usando a exibição **pg_stats**:

    ```sql
    SELECT n.nspname AS schemaname,
    c.relname AS tablename,
    a.attname,
    s.stainherit AS inherited,
    s.stanullfrac AS null_frac,
    s.stawidth AS avg_width,
    s.stadistinct AS n_distinct,
        CASE
            WHEN s.stakind1 = 1 THEN s.stavalues1
            WHEN s.stakind2 = 1 THEN s.stavalues2
            WHEN s.stakind3 = 1 THEN s.stavalues3
            WHEN s.stakind4 = 1 THEN s.stavalues4
            WHEN s.stakind5 = 1 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS most_common_vals,
        CASE
            WHEN s.stakind1 = 1 THEN s.stanumbers1
            WHEN s.stakind2 = 1 THEN s.stanumbers2
            WHEN s.stakind3 = 1 THEN s.stanumbers3
            WHEN s.stakind4 = 1 THEN s.stanumbers4
            WHEN s.stakind5 = 1 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS most_common_freqs,
        CASE
            WHEN s.stakind1 = 2 THEN s.stavalues1
            WHEN s.stakind2 = 2 THEN s.stavalues2
            WHEN s.stakind3 = 2 THEN s.stavalues3
            WHEN s.stakind4 = 2 THEN s.stavalues4
            WHEN s.stakind5 = 2 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS histogram_bounds,
        CASE
            WHEN s.stakind1 = 3 THEN s.stanumbers1[1]
            WHEN s.stakind2 = 3 THEN s.stanumbers2[1]
            WHEN s.stakind3 = 3 THEN s.stanumbers3[1]
            WHEN s.stakind4 = 3 THEN s.stanumbers4[1]
            WHEN s.stakind5 = 3 THEN s.stanumbers5[1]
            ELSE NULL::real
        END AS correlation,
        CASE
            WHEN s.stakind1 = 4 THEN s.stavalues1
            WHEN s.stakind2 = 4 THEN s.stavalues2
            WHEN s.stakind3 = 4 THEN s.stavalues3
            WHEN s.stakind4 = 4 THEN s.stavalues4
            WHEN s.stakind5 = 4 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS most_common_elems,
        CASE
            WHEN s.stakind1 = 4 THEN s.stanumbers1
            WHEN s.stakind2 = 4 THEN s.stanumbers2
            WHEN s.stakind3 = 4 THEN s.stanumbers3
            WHEN s.stakind4 = 4 THEN s.stanumbers4
            WHEN s.stakind5 = 4 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS most_common_elem_freqs,
        CASE
            WHEN s.stakind1 = 5 THEN s.stanumbers1
            WHEN s.stakind2 = 5 THEN s.stanumbers2
            WHEN s.stakind3 = 5 THEN s.stanumbers3
            WHEN s.stakind4 = 5 THEN s.stanumbers4
            WHEN s.stakind5 = 5 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS elem_count_histogram
    FROM pg_statistic s
     JOIN pg_class c ON c.oid = s.starelid
     JOIN pg_attribute a ON c.oid = a.attrelid AND a.attnum = s.staattnum
     LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
    WHERE NOT a.attisdropped AND has_column_privilege(c.oid, a.attnum, 'select'::text) AND (c.relrowsecurity = false OR NOT row_security_active(c.oid));
    ```

## Limpeza do exercício

1. O Banco de Dados do Azure para PostgreSQL que implantamos neste exercício incorrerá custos: você pode excluir o servidor após este exercício. Como alternativa, você pode excluir o **grupo de recursos rg-learn-work-with-postgresql-eastus** para remover todos os recursos que implantamos como parte deste exercício.
1. Se necessário, exclua a pasta .\DP3021Lab.
