---
lab:
  title: Criar um procedimento armazenado no Banco de Dados do Azure para PostgreSQL
  module: Procedures and functions in PostgreSQL
---

# Criar um procedimento armazenado no Banco de Dados do Azure para PostgreSQL

Neste exercício, você criará alguns procedimentos armazenados e os executará.

## Antes de começar

Você precisa ter uma assinatura própria do Azure para concluir este exercício. Se você não tiver uma assinatura do Azure, crie uma [conta de avaliação gratuita do Azure](https://azure.microsoft.com/free).

Além disso, você deve ter os seguintes itens instalados no seu computador:

- Visual Studio Code.
- Extensão Postgres do Visual Studio Code da Microsoft.
- CLI do Azure.
- Git.

## Crie o ambiente de exercício

Neste e em exercícios posteriores, você usará um script Bicep para implantar o Banco de Dados do Azure para PostgreSQL – Servidor Flexível e outros recursos em sua assinatura do Azure. Os scripts Bicep estão localizados na pasta `/Allfiles/Labs/Shared` do repositório GitHub que você clonou anteriormente.

### Baixe e instale a extensão do Visual Studio Code e do PostgreSQL.

Se o Visual Studio Code não estiver instalado:

1. Em um navegador, navegue até [Baixar o Visual Studio Code](https://code.visualstudio.com/download) e selecione a versão apropriada para seu sistema operacional.

1. Siga as instruções de instalação para seu sistema operacional.

1. Abra o Visual Studio Code.

1. No menu esquerdo, selecione **Extensões** para exibir o painel Extensões.

1. Na barra de pesquisa, insira **PostgreSQL**. O ícone da extensão do PostgreSQL para Visual Studio Code será exibido. Certifique-se de selecionar o da Microsoft.

1. Selecione **Instalar**. A extensão é instalada.

### Baixar e instalar a CLI do Azure e o Git

Se a CLI do Azure ou o Git não estiverem instalados:

1. Em um navegador, navegue [Instalar a CLI do Azure](https://learn.microsoft.com/cli/azure/install-azure-cli) e siga as instruções para o seu sistema operacional.

1. Em um navegador, navegue até [Baixar e instalar o Git](https://git-scm.com/downloads) e siga as instruções para o seu sistema operacional.

### Baixar os arquivos do exercício

Se você já clonou o repositório GitHub que contém os arquivos de exercício, *Pule o download dos arquivos do exercício*.

Para baixar os arquivos do exercício, clone o repositório GitHub que contém os arquivos de exercício em seu computador local. O repositório contém todos os scripts e recursos necessários para concluir este exercício.

1. Abra o Visual Studio Code se ele ainda não estiver aberto.

1. Selecione **Mostrar todos os comandos** (Ctrl+Shift+P) para abrir a paleta de comandos.

1. Na paleta de comandos, pesquise e selecione **Git: Clonar**.

1. Na paleta de comandos, insira o seguinte para clonar o repositório GitHub que contém recursos de exercício e pressione **Enter**:

    ```bash
    https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

1. Siga os prompts para escolher a pasta onde clonará o repositório. O repositório será clonado em uma pasta nomeada `mslearn-postgresql` no local selecionado.

1. Na solicitação para abrir o repositório clonado, selecione **Abrir**. O repositório abrirá no Visual Studio Code.

### Implantar recursos na assinatura do Azure

Se os recursos do Azure já estiverem instalados, *Pule a implantação de recursos*.

Esta etapa orienta você no uso de comandos da CLI do Azure no Visual Studio Code para criar um grupo de recursos e executar um script Bicep para implantar os serviços do Azure necessários para concluir este exercício em sua assinatura do Azure.

> &#128221; Se você estiver fazendo vários módulos neste roteiro de aprendizagem, poderá compartilhar o ambiente do Azure entre eles. Nesse caso, você só precisa concluir essa etapa de implantação de recursos uma vez.

1. Abra o Visual Studio Code, se ainda não estiver aberto, e a pasta do repositório clonado.

1. Expanda a pasta **mslearn-postgresql** no painel Explorer.

1. Expanda a pasta **Allfiles/Labs/Shared**.

1. Clique com o botão direito do mouse na pasta **/Allfiles/Labs/Shared** e selecione **Abrir no Terminal Integrado**. Essa seleção abrirá uma janela de terminal na janela do Visual Studio Code.

1. O terminal pode abrir uma janela do **PowerShell** por padrão. Para esta seção do laboratório, você deseja usar o **Shell Bash**. Além do ícone **+**, há uma seta suspensa. Clique nela e selecione **Git Bash** ou **Bash** na lista de perfis disponíveis. Essa seleção abrirá uma nova janela de terminal com o **Shell Bash**.

    > &#128221; Você pode fechar a janela do terminal do **PowerShell** se quiser, mas não é necessário. Várias janelas de terminal podem estar abertas ao mesmo tempo.

1. Na janela do terminal, execute o seguinte comando para entrar em sua conta do Azure:

    ```bash
    az login
    ```

    Esse comando abrirá uma janela do navegador, que solicitará que você entre na sua conta do Azure. Depois de entrar, retorne à janela do terminal.

1. Em seguida, você executará três comandos para definir variáveis para reduzir a digitação redundante ao usar comandos da CLI do Azure para criar recursos do Azure. As variáveis representam o nome a ser atribuído ao seu grupo de recursos (`RG_NAME`), a região do Azure (`REGION`) na qual os recursos serão implantados e uma senha gerada aleatoriamente para entrar no administrador do PostgreSQL (`ADMIN_PASSWORD`).

    No primeiro comando, a região atribuída à variável correspondente é `eastus`, mas você também pode substituí-la por um local de sua preferência.

    ```bash
    REGION=eastus
    ```

    O comando a seguir atribuirá o nome a ser usado para o grupo de recursos que abriga todos os recursos usados neste exercício. O nome do grupo de recursos atribuído à variável correspondente é `rg-learn-work-with-postgresql-$REGION`, em que `$REGION` é o local especificado acima. *No entanto, você pode alterá-lo para qualquer outro nome de grupo de recursos que atenda às suas preferências ou que você já tenha*.

    ```bash
    RG_NAME=rg-learn-work-with-postgresql-$REGION
    ```

    O comando final gerará aleatoriamente uma senha para entrar no administrador do PostgreSQL. Certifique-se de copiá-lq para um local seguro para que você possa usá-la mais tarde para se conectar ao seu servidor flexível PostgreSQL.

    ```bash
    #!/bin/bash
    
    # Define array of allowed characters explicitly
    chars=( {a..z} {A..Z} {0..9} '!' '@' '#' '$' '%' '^' '&' '*' '(' ')' '_' '+' )
    
    a=()
    for ((i = 0; i < 100; i++)); do
        rand_char=${chars[$RANDOM % ${#chars[@]}]}
        a+=("$rand_char")
    done
    
    # Join first 18 characters without delimiter
    ADMIN_PASSWORD=$(IFS=; echo "${a[*]:0:18}")
    
    echo "Your randomly generated PostgreSQL admin user's password is:"
    echo "$ADMIN_PASSWORD"
    echo "Please copy it to a safe place, as you will need it later to connect to your PostgreSQL flexible server."
    ```

1. (Pule se você estiver usando uma assinatura padrão.) Se você tiver acesso a mais de uma assinatura do Azure e sua assinatura padrão *não for* aquela na qual você deseja criar o grupo de recursos e outros recursos para este exercício, execute este comando para definir a assinatura apropriada, substituindo o token `<subscriptionName|subscriptionId>` pelo nome ou ID da assinatura que você deseja usar:

    ```azurecli
    az account set --subscription 16b3c013-d300-468d-ac64-7eda0820b6d3
    ```

1. (Pular se você estiver usando um grupo de recursos existente.) Execute o seguinte comando da CLI do Azure para criar o grupo de recursos:

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

1. Por fim, use a CLI do Azure para executar um script de implantação do Bicep para provisionar recursos do Azure em seu grupo de recursos:

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    O script de implantação do Bicep provisiona os serviços do Azure necessários para concluir este exercício em seu grupo de recursos. Os recursos implantados são um Banco de Dados do Azure para PostgreSQL – servidor flexível O script bicep também cria um banco de dados que pode ser configurado na linha de comando como um parâmetro.

    A implantação tipicamente leva vários minutos para ser concluída. Você pode monitorá-la no terminal do Bash ou navegar até a página **Implantações** do grupo de recursos criado anteriormente e observar o progresso da implantação lá.

1. Como o script cria um nome aleatório para o servidor PostgreSQL, você pode encontrar o nome do servidor executando o seguinte comando:

    ```azurecli
    az postgres flexible-server list --query "[].{Name:name, ResourceGroup:resourceGroup, Location:location}" --output table
    ```

    Anote o nome do servidor, pois você precisará que ele se conecte ao servidor posteriormente neste exercício.

    > &#128221; Você também pode encontrar o nome do servidor no portal do Azure. No portal do Azure, navegue até **Grupos de recursos** e selecione o grupo de recursos que você criou anteriormente. O servidor PostgreSQL está listado no grupo de recursos.

### Solucionar erros de implantação

Você pode encontrar alguns erros ao executar o script de implantação do Bicep. As mensagens mais comuns e as etapas para resolvê-las são:

- Se você executou anteriormente o script de implantação do Bicep para este roteiro de aprendizagem e excluiu os recursos na sequência, poderá receber uma mensagem de erro como a seguinte se tentar executar novamente o script dentro de 48 horas após a exclusão dos recursos:

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

- Se o laboratório exigir recursos de IA, você poderá receber o seguinte erro. Esse erro ocorre quando o script não consegue criar um recurso de IA devido à necessidade de aceitar o contrato de IA responsável. Se esse for o caso, use a interface do usuário do portal do Azure para criar um recurso dos Serviços de IA do Azure e execute novamente o script de implantação.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## Conectar a extensão do PostgreSQL no Visual Studio Code

Nesta seção, você conectará o servidor do PostgreSQL usando a extensão PostgreSQL no Visual Studio Code. Use a extensão PostgreSQL para executar scripts SQL no servidor PostgreSQL.

1. Abra o Visual Studio Code, se ainda não estiver aberto, e a pasta do repositório clonado.

1. Selecione o ícone do **PostgreSQL** no menu à esquerda.

    > &#128221; Se você não vir o ícone do PostgreSQL, clique no ícone **Extensões** e procure por **PostgreSQL**. Selecione a extensão do **PostgreSQL** da Microsoft e escolha **Instalar**.

1. Se você já criou uma conexão com o servidor PostgreSQL, pule para a próxima etapa. Para criar uma nova conexão:

    1. Na extensão **PostgreSQL**, selecione **+ Adicionar conexão** para adicionar uma nova conexão.

    1. Na caixa de diálogo **NOVA CONEXÃO**, insira as informações a seguir:

        - **Nome do servidor**: `<your-server-name>`.postgres.database.azure.com
        - **Tipo de autenticação**: senha
        - **Nome de usuário**: pgAdmin
        - **Senha**: a senha aleatória que você gerou anteriormente.
        - Marque a caixa de seleção **Salvar senha**.
        - **Nome da conexão**: `<your-server-name>`

    1. Selecione **Testar Conexão** para testar a conexão. Se a conexão for bem-sucedida, selecione **Salvar e conectar** para salvar a conexão; caso contrário, revise as informações de conexão e tente novamente.

1. Se ainda não estiver conectado, selecione **Conectar** para o servidor PostgreSQL. Você se conectou ao servidor de Banco de Dados do Azure para PostgreSQL.

1. Expanda o nó Servidor e seus bancos de dados. Os bancos de dados existentes serão listados.

1. Se você ainda não criou o banco de dados zoodb, selecione **Arquivo**, **Abrir arquivo** e navegue até a pasta em que salvou os scripts. Selecione **../Allfiles/Labs/02/Lab2_ZooDb.sql** e **Abrir**.

1. No canto inferior direito do Visual Studio Code, verifique se a conexão está verde. Se não estiver, estará escrito **PGSQL Desconectado**. Clique no texto **PGSQL Desconectado** e, em seguida, selecione sua conexão com o servidor PostgreSQL na lista na paleta de comandos. Se ele solicitar uma senha, digite a senha que você gerou anteriormente.

1. Hora de criar o banco de dados.

    1. Realce as instruções **DROP** e **CREATE** e execute-as.

    1. Se você realçar apenas a instrução **SELECT current_database()** e executá-la, observará que o banco de dados está definido como `postgres`. Você precisa alterá-lo para `zoodb`.

    1. Clique nas reticências na barra de menus com o ícone de *execução* e selecione **Alterar banco de dados do PostgreSQL**. Selecione `zoodb` na lista de banco de dados.

    > &#128221; Você também pode alterar o banco de dados no painel de consulta. Você pode anotar o nome do servidor e o nome do banco de dados na própria guia de consulta. Uma lista de bancos de dados será exibida ao escolher o nome do banco de dados. Selecione o banco de dados `zoodb` na lista.

    1. Execute a instrução **SELECT current_database()** novamente para confirmar se o banco de dados agora está definido como `zoodb`.

    1. Realce e execute as seções **Criar tabelas**, **Criar chaves estrangeiras** e **Preencher tabelas**.

    1. Realce as três instruções **SELECT** no final do script e execute-as para verificar se as tabelas foram criadas e preenchidas.

## Criar o procedimento armazenado repopulate_zoo()

Nesta seção, você criará um procedimento armazenado `repopulate_zoo()`. Este procedimento é usado para preencher novamente o banco de dados do zoológico com dados. O procedimento trunca e exclui todos os dados nas tabelas e, em seguida, preenche-os com novos dados.

1. Na janela do Visual Studio Code, selecione **Arquivo**, **Abrir Arquivo** e navegue até os scripts de laboratório. Selecione **../Allfiles/Labs/03/Lab3_RepopulateZoo.sql** e selecione **Abrir**. Se necessário, reconecte-se ao servidor clicando no texto **PGSQL Desconectado** e, em seguida, selecionando a conexão do servidor PostgreSQL na lista da paleta de comandos. Se ele solicitar uma senha, digite a senha que você gerou anteriormente.

1. Execute a instrução **SELECT current_database()** para verificar seu banco de dados atual. Novamente, o banco de dados provavelmente está definido como `postgres`. Nesse caso, você precisa alterá-lo para `zoodb`. Clique nas reticências na barra de menus com o ícone de *execução* e selecione **Alterar banco de dados do PostgreSQL**. Selecione `zoodb` na lista de banco de dados. Teste a conexão novamente executando a instrução **SELECT current_database()**.

1. Realce a seção em **Criar procedimento armazenado** de **DROP PROCEDURE** para **END $$.** Execute o texto realçado.

1. Mantenha o Visual Studio Code aberto para continuar com a próxima seção.

## Criar o procedimento armazenado new_exhibit()

Neste exercício, você criará o procedimento armazenado `new_exhibit()`. Este procedimento é usado para adicionar uma nova exposição ao banco de dados do zoológico. O procedimento insere uma nova linha na tabela do recinto e, em seguida, insere linhas na tabela de animais para cada animal na exposição.

1. No Visual Studio Code, selecione **Arquivo**, **Abrir Arquivo** e navegue até os scripts de laboratório. Selecione **../Allfiles/Labs/05/Lab5_StoredProcedure.sql** e então, selecione **Abrir**. Se necessário, reconecte-se ao servidor clicando no texto **PGSQL Desconectado** e, em seguida, selecionando a conexão do servidor PostgreSQL na lista da paleta de comandos. Se ele solicitar uma senha, digite a senha que você gerou anteriormente.

1. Execute a instrução **SELECT current_database()** para verificar seu banco de dados atual. Novamente, o banco de dados provavelmente está definido como `postgres`. Nesse caso, você precisa alterá-lo para `zoodb`. Clique nas reticências na barra de menus com o ícone de *execução* e selecione **Alterar banco de dados do PostgreSQL**. Selecione `zoodb` na lista de banco de dados. Teste a conexão novamente executando a instrução **SELECT current_database()**.

1. Realce e execute a instrução **CALL repopulate_zoo()** para começar com dados limpos.

1. Realce a seção em **Criar procedimento armazenado** de **DROP PROCEDURE** para **END $$.** Execute o texto realçado. Leia o procedimento. Você verá que ele declara alguns parâmetros de entrada e os usa para inserir linhas na tabela de compartimentos e na tabela de animais.

1. Mantenha o Visual Studio Code aberto para continuar com a próxima seção.

## Chame o procedimento armazenado

Agora que você criou o procedimento armazenado `new_exhibit()`, pode chamá-lo para adicionar uma nova exibição ao banco de dados do zoológico. O procedimento usa vários parâmetros de entrada, incluindo o nome da exposição, o tipo de compartimento e o número de animais na exposição.

1. Realce a seção após o comentário **Chamar o procedimento armazenado**. Execute o texto realçado. Esse script chamará o procedimento armazenado passando valores para os parâmetros de entrada.

1. Realce e execute as duas instruções **SELECT**. Execute o texto realçado. Você verá que uma nova linha foi inserida em compartimento e cinco novas linhas foram inseridas em animal.

## Criar e chamar uma função com valor de tabela

Hora de criar uma função com valor de tabela. Uma função com valor de tabela é uma função definida pelo usuário que retorna uma tabela. Você pode usar uma função com valor de tabela em uma instrução `SELECT`, assim como uma tabela normal.

1. No Visual Studio Code, selecione **Arquivo**, **Abrir Arquivo** e navegue até os scripts de laboratório. Selecione **.,/Allfiles/Labs/05/Lab5_Table_Function.sql** e selecione **Abrir**. Se necessário, reconecte-se ao servidor clicando no texto **PGSQL Desconectado** e, em seguida, selecionando a conexão do servidor PostgreSQL na lista da paleta de comandos. Se ele solicitar uma senha, digite a senha que você gerou anteriormente.

1. Execute a instrução **SELECT current_database()** para verificar seu banco de dados atual. Novamente, o banco de dados provavelmente está definido como `postgres`. Nesse caso, você precisa alterá-lo para `zoodb`. Clique nas reticências na barra de menus com o ícone de *execução* e selecione **Alterar banco de dados do PostgreSQL**. Selecione `zoodb` na lista de banco de dados. Teste a conexão novamente executando a instrução **SELECT current_database()**.

1. Realce e execute o procedimento armazenado **CALL repopulate_zoo()** para começar com dados limpos.

1. Realce e execute a seção após o comentário **Criar uma função com valor de tabela**. Essa função retorna uma tabela chamada **enclosure_summary**. Leia o código da função para entender como a tabela é preenchida.

1. Realce e execute as duas instruções SELECT, passando uma ID de compartimento diferente de cada vez.

1. Realce e execute a seção após o comentário **Como usar uma função com valor de tabela com uma junção LATERAL**. O script mostrará a função com valor de tabela sendo usada no lugar de um nome de tabela em uma junção.

## Funções internas

Nesta seção, você explorará algumas das funções internas disponíveis no PostgreSQL. O PostgreSQL tem um rico conjunto de funções integradas que você pode usar para executar várias operações nos dados. Essas funções podem ser usadas em consultas SQL para manipular e analisar dados.

1. No Visual Studio Code, selecione **Arquivo**, **Abrir Arquivo** e navegue até os scripts de laboratório. Selecione **../Allfiles/Labs/05/Lab5_SimpleFunctions.sql** e selecione **Abrir**. Se necessário, reconecte-se ao servidor clicando no texto **PGSQL Desconectado** e, em seguida, selecionando a conexão do servidor PostgreSQL na lista da paleta de comandos. Se ele solicitar uma senha, digite a senha que você gerou anteriormente.

> &#128221; As funções neste script não são específicas do banco de dados do zoológico. São funções gerais do PostgreSQL que podem ser usadas em qualquer banco de dados. Você pode executá-las em qualquer banco de dados, incluindo o banco de dados `postgres`.

1. Realce e execute cada função para ver como ela funciona. Para mais informações, consulte o artigo da [documentação online](https://www.postgresql.org/docs/current/functions.html) para obter informações sobre cada função.

1. Se você quiser manter o servidor PostgreSQL em execução, poderá deixá-lo em execução. Caso contrário, você pode parar o servidor para evitar incorrer em custos desnecessários no terminal do Bash. Execute o comando a seguir para interromper o servidor:

    ```azurecli
    az postgres flexible-server stop --name <your-server-name> --resource-group $RG_NAME
    ```

    Substitua `<your-server-name>` pelo nome do SQL Server.

    > &#128221; Você também pode parar a VM no portal do Azure. No portal do Azure, navegue até **Grupos de recursos** e selecione o grupo de recursos que você criou anteriormente. Selecione o servidor PostgreSQL e, em seguida, selecione **Parar** no menu.

1. Feche o Visual Studio Code.

## Limpar

1. Se você não precisar mais desse servidor PostgreSQL para outros exercícios, para evitar incorrer em custos desnecessários do Azure, exclua o grupo de recursos criado neste exercício.

1. Se necessário, exclua o repositório Git que você clonou anteriormente.
