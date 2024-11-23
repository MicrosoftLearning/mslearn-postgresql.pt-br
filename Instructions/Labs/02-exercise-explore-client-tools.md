---
lab:
  title: Explorar o PostgreSQL com ferramentas de cliente
  module: Understand client-server communication in PostgreSQL
---

# Explorar o PostgreSQL com ferramentas de cliente

Neste exercício, você baixa e instala o psql e o Azure Data Studio. Se você já tiver o Azure Data Studio instalado em seu computador, poderá ir para Conectar-se ao servidor flexível do Banco de Dados do Azure para PostrgreSQL.

## Antes de começar

Você precisa ter uma assinatura própria do Azure para concluir este exercício. Se você não tiver uma assinatura do Azure, crie uma [conta de avaliação gratuita do Azure](https://azure.microsoft.com/free).

## Crie o ambiente de exercício

Neste exercício e em todos os exercícios posteriores, você usará o Bicep no Azure Cloud Shell para implantar seu servidor PostgreSQL.

### Implantar recursos na assinatura do Azure

Esta etapa orienta você no uso de comandos da CLI do Azure do Azure Cloud Shell para criar um grupo de recursos e executar um script Bicep para implantar os serviços do Azure necessários para concluir este exercício em sua assinatura do Azure.

> Observação
>
> Se você estiver fazendo vários módulos neste roteiro de aprendizagem, poderá compartilhar o ambiente do Azure entre eles. Nesse caso, você só precisa concluir essa etapa de implantação de recursos uma vez.

1. Abra um navegador da Web e acesse o [portal do Azure](https://portal.azure.com/).

2. SElecione o ícone do **Cloud Shell** na barra de ferramentas do portal do Azure para abrir um novo painel do [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) na parte inferior da janela do navegador.

    ![Captura de tela da barra de ferramentas do Azure com o ícone do Cloud Shell realçado por uma caixa vermelha.](media/02-portal-toolbar-cloud-shell.png)

3. Se solicitado, selecione as opções necessárias para abrir um shell do *Bash*. Se você já usou um console do *PowerShell*, alterne-o para um shell do *Bash*.

4. No prompt do Cloud Shell, insira o seguinte para clonar o repositório GitHub que contém os recursos do exercício:

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

5. Em seguida, você executará três comandos para definir variáveis para reduzir a digitação redundante ao usar comandos da CLI do Azure para criar recursos do Azure. As variáveis representam o nome a ser atribuído ao seu grupo de recursos (`RG_NAME`), a região do Azure (`REGION`) na qual os recursos serão implantados e uma senha gerada aleatoriamente para o logon de administrador do PostgreSQL (`ADMIN_PASSWORD`).

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

6. Se você tiver acesso a mais de uma assinatura do Azure e sua assinatura padrão não for aquela na qual você deseja criar o grupo de recursos e outros recursos para este exercício, execute este comando para definir a assinatura apropriada, substituindo o token `<subscriptionName|subscriptionId>` pelo nome ou ID da assinatura que você deseja usar:

    ```azurecli
    az account set --subscription <subscriptionName|subscriptionId>
    ```

7. Execute o seguinte comando da CLI do Azure para criar um grupo de recursos:

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

8. Por fim, use a CLI do Azure para executar um script de implantação do Bicep para provisionar recursos do Azure em seu grupo de recursos:

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

## Ferramentas de cliente para se conectar ao PostgreSQL

### Conectar-se ao Banco de Dados do Azure para PostgreSQL com psql

Você pode instalar o psql localmente ou conectar-se no portal do Azure, que abrirá o Cloud Shell e solicitará a senha da conta de administrador.

#### Conectando-se localmente

1. Instale o psql [aqui](https://sbp.enterprisedb.com/getfile.jsp?fileid=1258893).
    1. No assistente de instalação, quando você chegar à caixa de diálogo **Selecionar Componentes**, selecione **Ferramentas de Linha de Comando**.
    > Observação
    >
    > Para verificar se o **psql** já está instalado em seu ambiente, abra a linha de comando/terminal e execute o comando ***psql***. Se ele retornar uma mensagem como "*psql: erro: conexão com o servidor no soquete...*", isso significa que a **psql** já está instalada em seu ambiente e não há necessidade de reinstalá-la.



1. Abra uma linha de comando.
1. A sintaxe para se conectar ao servidor é:

    ```sql
    psql --h <servername> --p <port> -U <username> <dbname>
    ```

1. No prompt de comando, insira **`--host=<servername>.postgres.database.azure.com`** onde `<servername>` é o nome do Banco de Dados do Azure para PostgreSQL criado acima.
    1. Você pode encontrar o nome do servidor em **Visão geral** no portal do Azure ou como uma saída do script Bicep.

    ```sql
   psql -h <servername>.postgres.database.azure.com -p 5432 -U pgAdmin postgres
    ```

    1. Você precisará definir uma senha para a conta de administrador que você copiou acima.

1. Para criar um banco de dados em branco no prompt, digite:

    ```sql
    CREATE DATABASE mypgsqldb;
    ```

1. No prompt, execute o seguinte comando para mudar a conexão para o banco de dados **mypgsqldb** recém-criado:

    ```sql
    \c mypgsqldb
    ```

1. Agora que você se conectou ao servidor e criou um banco de dados, você pode executar consultas SQL familiares, como criar tabelas no banco de dados:

    ```sql
    CREATE TABLE inventory (
        id serial PRIMARY KEY,
        name VARCHAR(50),
        quantity INTEGER
        );
    ```

1. Carregar dados nas tabelas

    ```sql
    INSERT INTO inventory (id, name, quantity) VALUES (1, 'banana', 150);
    INSERT INTO inventory (id, name, quantity) VALUES (2, 'orange', 154);
    ```

1. Consultar e atualizar os dados nas tabelas

    ```sql
    SELECT * FROM inventory;
    ```

1. Atualize os dados nas tabelas.

    ```sql
    UPDATE inventory SET quantity = 200 WHERE name = 'banana';
    ```

## Instalar o Azure Data Studio

> Observação
>
> Se o Azure Data Studio já estiver instalado, vá para a etapa *Instalar a extensão do PostgreSQL*.

Para instalar o Azure Data Studio para uso com o Banco de Dados do Azure para PostgreSQL:

1. Em um navegador, navegue para [Baixar e instalar o Azure Data Studio](https://go.microsoft.com/fwlink/?linkid=2282284) e, na plataforma Windows, selecione **Instalador de usuário (recomendado)**. O arquivo executável é baixado para a pasta Downloads.
1. Selecione **Abrir arquivo**.
1. O Contrato de licença é exibido. Revise e **aceite o contrato**, depois selecione **Avançar**.
1. Em **Selecionar Tarefas Adicionais**, selecione **Adicionar ao PATH** e as outras adições necessárias. Selecione **Avançar**.
1. A caixa de diálogo **Pronto para Instalar** é exibida. Examine suas configurações. Selecione **Voltar** para fazer alterações ou selecione **Instalar**.
1. A caixa de diálogo **Concluir o Assistente de Instalação do Azure Data Studio** é exibida. Selecione **Concluir**. O Azure Data Studio é iniciado.

## Instalar a extensão do PostgreSQL

1. Abra o Azure Data Studio se ainda não estiver aberto.
2. No menu esquerdo, selecione **Extensões** para exibir o painel Extensões.
3. Na barra de pesquisa, insira **PostgreSQL**. O ícone da extensão PostgreSQL para Azure Data Studio é exibido.
   
![Captura de tela da extensão do PostgreSQL para Azure Data Studio](media/02-postgresql-extension.png)
   
4. Selecione **Instalar**. A extensão é instalada.

## Conectar-se ao servidor flexível do Banco de Dados do Azure para PostgreSQL

1. Abra o Azure Data Studio se ainda não estiver aberto.
2. No menu à esquerda, selecione **Coleções**.
   
![Captura de tela de Conexões no Azure Data Studio](media/02-connections.png)

3. Selecione **Nova Conexão**.
   
![Captura de tela de como criar uma nova conexão no Azure Data Studio](media/02-create-connection.png)

4. Em **Detalhes da Conexão**, em **Tipo de conexão**, selecione **PostgreSQL** na lista suspensa.
5. No **Nome do servidor**, insira o nome completo do servidor como ele aparece no portal do Azure.
6. Mantenha o **Tipo de autenticação** como Senha.
7. Em Nome de usuário e senha, digite o nome de usuário **pgAdmin** e a **senha** de administrador aleatória que você criou acima
8. Selecione [ x ] Lembrar senha.
9. Os campos restantes são opcionais.
10. Selecione **Conectar**. Você está conectado ao servidor de Banco de Dados do Azure para PostgreSQL.
11. Uma lista dos bancos de dados do servidor é exibida. Isso inclui bancos de dados de sistema e de usuário.

## Criar o banco de dados do zoológico

1. Navegue até a pasta com seus arquivos de script de exercício ou baixe o **Lab2_ZooDb.sql** do [MSLearn PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql/tree/main/Allfiles/Labs/02).
1. Abra o Azure Data Studio se ainda não estiver aberto.
1. Selecione **Arquivo**, **Abrir arquivo** e navegue até a pasta em que você salvou os scripts. Selecione **../Allfiles/Labs/02/Lab2_ZooDb.sql** e **Abrir**.
   1. Realce as instruções **DROP** e **CREATE** e execute-as.
   1. Na parte superior da tela, use a seta suspensa para exibir os bancos de dados no servidor, incluindo o zoodb e os bancos de dados do sistema. Selecione o banco de dados **zoodb**.
   1. Realce as seções **Criar tabelas**, **Criar chaves estrangeiras** e **Preencher tabelas** e execute-as.
   1. Realce as três instruções **SELECT** no final do script e execute-as para verificar se as tabelas foram criadas e preenchidas.
