---
lab:
  title: Executar a instrução EXPLAIN
  module: Understand PostgreSQL query processing
---

# Executar a instrução EXPLAIN

Neste exercício, você examinará a função EXPLAIN e como ela pode exibir o plano de execução que o planejador do PostgreSQL gera para uma instrução fornecida.

## Antes de começar

Você precisa ter uma assinatura própria do Azure para concluir este exercício. Se você não tiver uma assinatura do Azure, crie uma [conta de avaliação gratuita do Azure](https://azure.microsoft.com/free).

## Crie o ambiente de exercício

Neste exercício e em todos os exercícios posteriores, você usará o Bicep no Azure Cloud Shell para implantar seu servidor PostgreSQL.
Ignore a implantação de recursos e a instalação do Azure Data Studio se você já os tiver instalado.

### Implantar recursos na assinatura do Azure

Esta etapa orienta você no uso de comandos da CLI do Azure do Azure Cloud Shell para criar um grupo de recursos e executar um script Bicep para implantar os serviços do Azure necessários para concluir este exercício em sua assinatura do Azure.

> Observação
>
> Se você estiver fazendo vários módulos neste roteiro de aprendizagem, poderá compartilhar o ambiente do Azure entre eles. Nesse caso, você só precisa concluir essa etapa de implantação de recursos uma vez.

1. Abra um navegador da Web e acesse o [portal do Azure](https://portal.azure.com/).

2. SElecione o ícone do **Cloud Shell** na barra de ferramentas do portal do Azure para abrir um novo painel do [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) na parte inferior da janela do navegador.

    ![Captura de tela da barra de ferramentas do Azure com o ícone do Cloud Shell realçado por uma caixa vermelha.](media/03-portal-toolbar-cloud-shell.png)

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
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- Se o script não conseguir criar um recurso de IA devido ao requisito de aceitar o contrato de IA responsável, você poderá enfrentar o seguinte erro; nesse caso, use a interface do usuário do Portal do Azure para criar um recurso dos Serviços de IA do Azure e, em seguida, execute novamente o script de implantação.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## Antes de continuar

certifique-se de:

1. Ter instalado e iniciado o servidor flexível do Banco de Dados do Azure para PostgreSQL. Ele deve ter sido instalado pelo script Bicep anterior.
1. Ter clonado os scripts de laboratório do [PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql.git). Caso ainda não tenha feito isso, clone o repositório localmente:
    1. Abra uma linha de comando/terminal.
    1. Execute o comando:
       ```bash
       md .\DP3021Lab
       git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git .\DP3021Lab
       ```
       > OBSERVAÇÃO
       > 
       > Se o **git** não estiver instalado, [baixe e instale o aplicativo ***git***](https://git-scm.com/download) e tente executar os comandos anteriores novamente.
1. Você instalou o Azure Data Studio. Se você ainda não tiver feito isso, [baixe e instale o ***Azure Data Studio***](https://go.microsoft.com/fwlink/?linkid=2282284).
1. Instale a extensão do **PostgreSQL** no Azure Data Studio.
1. Abra o Azure Data Studio e conecte-se ao servidor flexível do Banco de Dados do Azure para PostgreSQL criado pelo script do Bicep. Digite o nome de usuário **pgAdmin** e a **senha de administrador** aleatória que você criou anteriormente.
1. Se você ainda não criou o banco de dados zoodb, selecione **Arquivo**, **Abrir arquivo** e navegue até a pasta em que salvou os scripts. Selecione **../Allfiles/Labs/02/Lab2_ZooDb.sql** e **Abrir**.
   1. Realce as instruções **DROP** e **CREATE** e execute-as.
   1. Na parte superior da tela, use a seta suspensa para exibir os bancos de dados no servidor, incluindo o zoodb e os bancos de dados do sistema. Selecione o banco de dados **zoodb**.
   1. Realce as seções **Criar tabelas**, **Criar chaves estrangeiras** e **Preencher tabelas** e execute-as.
   1. Realce as três instruções **SELECT** no final do script e execute-as para verificar se as tabelas foram criadas e preenchidas.

## Praticar o uso de EXPLAIN ANALYZE

1. No [portal do Azure](https://portal.azure.com), navegue até o servidor flexível do Banco de Dados do Azure para PostgreSQL. Verifique se o servidor foi iniciado ou reinicie-o se necessário.
1. Abra o Azure Data Studio e conecte-se ao servidor flexível do Banco de Dados do Azure para PostgreSQL.
1. Selecione **Arquivo**, **Abrir Arquivo** e navegue até a pasta em que você salvou os scripts. Abra **../Allfiles/Labs/03/Lab3_RepopulateZoo.sql**. Reconecte-se ao servidor, se necessário.
1. Selecione **Executar** para executar a consulta. Isso preenche novamente o banco de dados zoodb.
1. Selecione Arquivo, **Abrir arquivo** e  **../Allfiles/Labs/03/Lab3_explain.sql**.
1. No arquivo do laboratório, na seção **1. Investigar EXPLAIN ANALYZE**, realce e execute a Instrução A e a Instrução B separadamente.
    1. Qual instrução atualizou o banco de dados e por quê?
    1. Quantos milissegundos foram necessários para planejar a Instrução A?
    1. Qual foi o tempo de execução da Instrução B?

## Praticar o uso de EXPLAIN

1. No arquivo do laboratório, na seção **2. Investigar EXPLAIN**, realce e execute essa instrução.
    1. Qual chave de classificação foi usada e por quê?
1. No arquivo do laboratório, na seção **3. Investigar as opções de EXPLAIN**, realce e execute cada instrução separadamente. Compare as estatísticas do plano de consulta para cada opção.

## Limpar

1. Exclua o grupo de recursos criado neste exercício para evitar incorrer custos desnecessários do Azure.
1. Se necessário, exclua a pasta **.\DP3021Lab**.

