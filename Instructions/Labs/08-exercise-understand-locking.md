---
lab:
  title: Entender o bloqueio
  module: Understand concurrency in PostgreSQL
---

# Entender o bloqueio

Neste exercício, você examinará os parâmetros e metadados do sistema no PostgreSQL.

## Antes de começar

Você precisará usar sua assinatura do Azure para concluir os exercícios deste módulo. Caso ainda não tenha uma assinatura do Azure, inscreva-se em uma conta de avaliação gratuita em [Criar soluções na nuvem com uma conta gratuita do Azure](https://azure.microsoft.com/free/).

## Crie o ambiente de exercício

### Implantar recursos na assinatura do Azure

Esta etapa orienta você no uso de comandos da CLI do Azure do Azure Cloud Shell para criar um grupo de recursos e executar um script Bicep para implantar os serviços do Azure necessários para concluir este exercício em sua assinatura do Azure.

1. Abra um navegador da Web e acesse o [portal do Azure](https://portal.azure.com/).

2. SElecione o ícone do **Cloud Shell** na barra de ferramentas do portal do Azure para abrir um novo painel do [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) na parte inferior da janela do navegador.

    ![Captura de tela da barra de ferramentas do Azure com o ícone do Cloud Shell realçado por uma caixa vermelha.](media/08-portal-toolbar-cloud-shell.png)

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
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD databaseName=adventureworks
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

## Conectar o banco de dados usando o psql no Azure Cloud Shell

Nesta tarefa, você se conectará ao banco de dados `adventureworks` no servidor do Banco de Dados do Azure para PostgreSQL usando o [utilitário de linha de comando psql](https://www.postgresql.org/docs/current/app-psql.html) do [Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview).

1. No [portal do Azure](https://portal.azure.com/), navegue até o servidor flexível do Banco de Dados do Azure para PostgreSQL.

2. No menu de recursos, em **Configurações**, selecione **Bancos de dados** e selecione **Conectar** para o banco de dados `adventureworks`.

    ![Captura de tela da página de banco de dados do Banco de Dados do Azure para PostfreSQL. Os bancos de dados e o Connect para o banco de dados adventureworks estão destacados por caixas vermelhas.](media/08-postgresql-adventureworks-database-connect.png)

3. No prompt "Senha para o usuário pgAdmin" no Cloud Shell, insira a senha gerada aleatoriamente para o logon do **pgAdmin**.

    Uma vez conectado, o prompt `psql` do banco de dados `adventureworks` é exibido.

4. Durante o restante deste exercício, você continuará trabalhando no Cloud Shell, portanto, pode ser útil expandir o painel na janela do navegador selecionando o botão **Maximizar** no canto superior direito do painel.

    ![Captura de tela do painel do Azure Cloud Shell com o botão Maximizar realçado por uma caixa vermelha.](media/08-azure-cloud-shell-pane-maximize.png)

### Preencher o banco de dados com dados

1. Você precisa criar uma tabela no banco de dados e preenchê-la com dados de exemplo para ter informações com as quais trabalhar enquanto revisa o bloqueio neste exercício.
1. Execute o comando a seguir para criar a tabela `production.workorder` para adicionar os dados:

    ```sql
    DROP SCHEMA IF EXISTS production CASCADE;
    CREATE SCHEMA production;
    
    DROP TABLE IF EXISTS production.workorder;
    CREATE TABLE production.workorder
    (
        workorderid integer NOT NULL,
        productid integer NOT NULL,
        orderqty integer NOT NULL,
        scrappedqty smallint NOT NULL,
        startdate timestamp without time zone NOT NULL,
        enddate timestamp without time zone,
        duedate timestamp without time zone NOT NULL,
        scrapreasonid smallint,
        modifieddate timestamp without time zone NOT NULL DEFAULT now()
    )
    WITH (
        OIDS = FALSE
    )
    TABLESPACE pg_default;
    ```

1. Em seguida, use o comando `COPY` para carregar dados de arquivos CSV na tabela que você criou acima. Inicie ao execute o comando a seguir para preencher a tabela `production.workorder`:

    ```sql
    \COPY production.workorder FROM 'mslearn-postgresql/Allfiles/Labs/08/Lab8_workorder.csv' CSV HEADER
    ```

    A saída do comando deve ser `COPY 72591`, indicando que 72591 linhas foram gravadas na tabela a partir do arquivo CSV.

1. Feche o painel do Cloud Shell depois que os dados forem carregados

### Conectar-se ao banco de dados com o Azure Data Studio

1. Se você ainda não o instalou, [baixe e instale ***o Azure Data Studio***](https://go.microsoft.com/fwlink/?linkid=2282284).
1. Inicie o Azure Data Studio.
1. Se você não instalou a extensão **PostgreSQL** no Azure Data Studio, instale-a agora.
1. Selecione **Servidores** e **Nova conexão**.
1. Em **Tipo de conexão**, selecione **PostgreSQL**.
1. Em **nome do servidor**, digite o valor especificado ao implantar o servidor.
1. Em **Nome de usuário**, digite **pgAdmin**.
1. Em **Senha**, digite a senha gerada aleatoriamente para o login do **pgAdmin** que você gerou
1. Selecione **Lembrar senha**.
1. Clique em **Conectar**

## Tarefa 1: Investigar o comportamento de bloqueio padrão

1. Abra o Azure Data Studio.
1. Expanda **Bancos de dados**, clique com o botão direito do mouse em **adventureworks** e selecione **Nova Consulta**.
   
    ![Captura de tela do banco de dados adventureworks realçando o item de menu de contexto Nova Consulta.](media/08-new-query.png)

1. Vá para **Arquivo** e **Nova Consulta**. Agora você deve ter uma guia de consulta com um nome começando com **SQL_Query_1** e outra guia de consulta com um nome começando com **SQL_Query_2**.
1. Selecione a guia **SQLQuery_1**, digite a consulta a seguir e selecione **Executar**.

    ```sql
    SELECT * FROM production.workorder
    ORDER BY scrappedqty DESC;
    ```

1. Observe que o valor de **scrappedqty** para a primeira linha é **673**.
1. Selecione a guia **SQLQuery_2**, digite a consulta a seguir e selecione **Executar**.

    ```sql
    BEGIN TRANSACTION;
    UPDATE production.workorder
        SET scrappedqty=scrappedqty+1;
    ```

1. Observe que a segunda consulta inicia uma transação, mas não a confirma.
1. Volte para **SQLQuery_1** e execute a consulta novamente.
1. Observe que o valor de **stockedqty** para a primeira linha ainda é **673**. A consulta está usando um instantâneo dos dados e não está vendo as atualizações da outra transação.
1. Selecione a guia **SQLQuery_2**, exclua a consulta existente, digite a consulta a seguir e selecione **Executar**.

    ```sql
    ROLLBACK TRANSACTION;
    ```

## Tarefa 2: aplicar bloqueios de tabela a uma transação

1. Selecione a guia **SQLQuery_2**, digite a consulta a seguir e selecione **Executar**.

    ```sql
    BEGIN TRANSACTION;
    LOCK TABLE production.workorder IN ACCESS EXCLUSIVE MODE;
    UPDATE production.workorder
        SET scrappedqty=scrappedqty+1;
    ```

1. Observe que a segunda consulta inicia uma transação, mas não a confirma.
1. Volte para **SQLQuery_1** e execute a consulta novamente.
1. Observe que a transação está bloqueada e não será concluída, por mais tempo que você aguarde.
1. Selecione a guia **SQLQuery_2**, exclua a consulta existente, digite a consulta a seguir e selecione **Executar**.

    ```sql
    ROLLBACK TRANSACTION;
    ```

1. Volte para **SQLQuery_1**, aguarde alguns segundos e observe que a consulta foi concluída quando o bloqueio foi removido.

Neste exercício, vimos o comportamento de bloqueio padrão. Em seguida, aplicamos bloqueios explicitamente e vimos que, embora alguns bloqueios forneçam níveis muito altos de proteção, esses bloqueios também podem ter implicações de desempenho.

## Limpeza do exercício

O Banco de Dados do Azure para PostgreSQL que implantamos neste exercício incorrerá custos: você pode excluir o servidor após este exercício. Como alternativa, você pode excluir o **grupo de recursos rg-learn-work-with-postgresql-eastus** para remover todos os recursos que implantamos como parte deste exercício.
