---
lab:
  title: Traduzir texto com o Tradutor de IA do Azure
  module: Translate Text using the Azure AI Translator and Azure Database for PostgreSQL
---

# Traduzir texto com o Tradutor de IA do Azure

Como desenvolvedor líder da Margie's Travel, você foi solicitado a ajudar em um esforço de internacionalização. Hoje, todas as listagens de aluguel para o serviço de aluguel de curto prazo da empresa estão em inglês. Você deseja traduzir essas listagens para uma variedade de idiomas sem grande esforço de desenvolvimento. Todos os seus dados estão hospedados em um servidor flexível do Banco de Dados do Azure para PostgreSQL e você deseja usar os Serviços de IA Azure para executar a tradução.

Neste exercício, você traduzirá texto em inglês para vários idiomas usando o serviço Tradutor de IA do Azure por meio de um banco de dados de servidor flexível de Banco de Dados do Azure para PostgreSQL.

## Antes de começar

Você precisa de uma [assinatura do Azure](https://azure.microsoft.com/free) para a qual você tem direitos administrativos.

### Implantar recursos na assinatura do Azure

Esta etapa orientará você no uso de comandos da CLI do Azure do Azure Cloud Shell para criar um grupo de recursos e executar um script Bicep para implantar os serviços do Azure necessários para concluir este exercício em sua assinatura do Azure.

1. Abra um navegador da Web e acesse o [portal do Azure](https://portal.azure.com/).

2. SElecione o ícone do **Cloud Shell** na barra de ferramentas do portal do Azure para abrir um novo painel do [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) na parte inferior da janela do navegador.

    ![Captura de tela da barra de ferramentas do Azure com o ícone do Cloud Shell realçado por uma caixa vermelha.](media/11-portal-toolbar-cloud-shell.png)

    Se solicitado, selecione as opções necessárias para abrir um shell do *Bash*. Se você já usou um console do *PowerShell*, alterne-o para um shell do *Bash*.

3. No prompt do Cloud Shell, insira o seguinte para clonar o repositório GitHub que contém os recursos do exercício:

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

    Se você já clonou esse repositório GitHub em um módulo anterior, ele ainda estará disponível e você poderá receber a seguinte mensagem de erro:

    ```bash
    fatal: destination path 'mslearn-postgresql' already exists and is not an empty directory.
    ```

    Se você receber essa mensagem, poderá continuar com segurança para a próxima etapa.

4. Em seguida, você executará três comandos para definir variáveis para reduzir a digitação redundante ao usar comandos da CLI do Azure para criar recursos do Azure. As variáveis representam o nome a ser atribuído ao seu grupo de recursos (`RG_NAME`), a região do Azure (`REGION`) na qual os recursos serão implantados e uma senha gerada aleatoriamente para o logon de administrador do PostgreSQL (`ADMIN_PASSWORD`).

    No primeiro comando, a região atribuída à variável correspondente é `eastus`, mas você também pode substituí-la por um local de sua preferência. No entanto, se estiver substituindo o padrão, você deverá selecionar outra [região do Azure que dê suporte ao resumo abstrato](https://learn.microsoft.com/azure/ai-services/language-service/summarization/region-support) para garantir que você possa concluir todas as tarefas nos módulos deste roteiro de aprendizagem.

    ```bash
    REGION=eastus
    ```

    O comando a seguir atribui o nome a ser usado para o grupo de recursos que abrigará todos os recursos usados neste exercício. O nome do grupo de recursos atribuído à variável correspondente é `rg-learn-postgresql-ai-$REGION`, onde `$REGION` é o local especificado acima. No entanto, você pode alterá-lo para qualquer outro nome de grupo de recursos que atenda às suas preferências.

    ```bash
    RG_NAME=rg-learn-postgresql-ai-$REGION
    ```

    O comando final gera aleatoriamente uma senha para o login de administrador do PostgreSQL. Certifique-se de copiá-la para um local seguro para usar mais tarde para se conectar ao seu servidor flexível do PostgreSQL.

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
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-translate.bicep" --parameters restore=false adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    O script de implantação do Bicep provisiona os serviços do Azure necessários para concluir este exercício em seu grupo de recursos. Os recursos implantados incluem um servidor flexível do Banco de Dados do Azure para PostgreSQL e o serviço Tradutor de IA do Azure. O script Bicep também executa algumas etapas de configuração, como adicionar as extensões `azure_ai` e `vector` à _lista de permitidos_ do servidor PostgreSQL (por meio do parâmetro de servidor azure.extensions) e criar um banco de dados nomeado `rentals` no servidor. **Observe que o arquivo Bicep difere dos outros módulos neste roteiro de aprendizagem.**

    A implantação tipicamente leva vários minutos para ser concluída. Você pode monitorá-lo no Cloud Shell ou navegar até a página **Implantações** do grupo de recursos criado acima e observar o progresso da implantação lá.

8. Feche o painel do Cloud Shell quando a implantação do recurso for concluída.

### Solucionar erros de implantação

Você pode encontrar alguns erros ao executar o script de implantação do Bicep.

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

## Conectar o banco de dados usando o psql no Azure Cloud Shell

Nesta tarefa, você se conecta ao banco de dados `rentals` no servidor do Banco de Dados do Azure para PostgreSQL usando o [utilitário de linha de comando psql](https://www.postgresql.org/docs/current/app-psql.html) do [Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview).

1. No [portal do Azure](https://portal.azure.com/), navegue até o servidor flexível do Banco de Dados do Azure para PostgreSQL.

2. No menu de recursos, em **Configurações**, selecione **Bancos de dados** e selecione **Conectar** para o banco de dados`rentals`.

    ![Captura de tela da página de banco de dados do Banco de Dados do Azure para PostfreSQL. Os bancos de dados e o Connect para o banco de dados de aluguéis são destacados por caixas vermelhas.](media/17-postgresql-rentals-database-connect.png)

3. No prompt "Senha para o usuário pgAdmin" no Cloud Shell, insira a senha gerada aleatoriamente para o logon do **pgAdmin**.

    Uma vez conectado, o prompt `psql` do banco de dados `rentals` é exibido.

4. Durante o restante deste exercício, você continuará trabalhando no Cloud Shell, portanto, pode ser útil expandir o painel na janela do navegador selecionando o botão **Maximizar** no canto superior direito do painel.

    ![Captura de tela do painel do Azure Cloud Shell com o botão Maximizar realçado por uma caixa vermelha.](media/17-azure-cloud-shell-pane-maximize.png)

## Preencher o banco de dados com dados de listagens

Você precisa ter dados de listagens em inglês disponíveis para traduzi-los. Se você não tiver criado a tabela `listings` no banco de dados `rentals` em um módulo anterior, siga estas instruções para criá-la.

1. Execute os seguintes comandos para criar a tabela `listings` para armazenar dados de listagem de imóveis alugados:

    ```sql
    CREATE TABLE listings (
        id int,
        name varchar(100),
        description text,
        property_type varchar(25),
        room_type varchar(30),
        price numeric,
        weekly_price numeric
    );
    ```

2. Em seguida, use o comando `COPY` para carregar dados de arquivos CSV em cada tabela criada acima. Execute o comando a seguir para preencher a tabela `listings`:

    ```sql
    \COPY listings FROM 'mslearn-postgresql/Allfiles/Labs/Shared/listings.csv' CSV HEADER
    ```

    A saída do comando deve ser `COPY 50`, indicando que 50 linhas foram gravadas na tabela a partir do arquivo CSV.

## Criar tabelas adicionais para tradução

Você tem os dados `listings` preparados, mas precisa de duas tabelas adicionais para realizar a tradução.

1. Execute o comando a seguir para criar as tabelas `languages` e `listing_translations`.

    ```sql
    CREATE TABLE languages (
        code VARCHAR(7) NOT NULL PRIMARY KEY
    );
    ```

    ```sql
    CREATE TABLE listing_translations(
        id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
        listing_id INT,
        language_code VARCHAR(7),
        description TEXT
    );
    ```

2. Em seguida, insira uma linha por idioma para tradução. Nesse caso, você criará linhas para cinco idiomas: alemão, chinês simplificado, hindi, húngaro e suaíli.

    ```sql
    INSERT INTO languages(code)
    VALUES
        ('de'),
        ('zh-Hans'),
        ('hi'),
        ('hu'),
        ('sw');
    ```

    A saída do comando deve ser `INSERT 0 5`, indicando que você inseriu cinco novas linhas na tabela.

## Instalar e configurar a extensão `azure_ai`

Antes de usar a extensão `azure_ai`, você deve instalá-la em seu banco de dados e configurá-la para se conectar aos recursos dos Serviços de IA do Azure. A extensão `azure_ai` permite integrar os serviços do OpenAI e Linguagem de IA do Azure ao banco de dados. Para habilitar a extensão em seu banco de dados, siga as etapas abaixo:

1. Execute o seguinte comando no prompt `psql` para verificar se as extensões `azure_ai` e `vector` foram adicionadas com êxito à _ lista de permitidos_ do servidor pelo script de implantação do Bicep que você executou ao configurar o ambiente:

    ```sql
    SHOW azure.extensions;
    ```

    O comando exibe a lista de extensões na _lista de permitidos_ do servidor. Se tudo foi instalado corretamente, sua saída deve incluir `azure_ai` e `vector`, desta maneira:

    ```sql
     azure.extensions 
    ------------------
     azure_ai,vector
    ```

    Antes que uma extensão possa ser instalada e usada no servidor flexível do Banco de Dados do Azure para PostgreSQL, ela deve ser adicionada à _lista de permitidos_ do servidor, conforme descrito em [como usar extensões do PostgreSQL](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions).

2. Agora, você está pronto para instalar a extensão `azure_ai` usando o comando [CREATE EXTENSION](https://www.postgresql.org/docs/current/sql-createextension.html).

    ```sql
    CREATE EXTENSION IF NOT EXISTS azure_ai;
    ```

    `CREATE EXTENSION` carrega uma nova extensão no banco de dados executando seu arquivo de script. Esse script normalmente cria novos objetos SQL, como funções, tipos de dados e esquemas. Um erro será gerado se já existir uma extensão com o mesmo nome. A adição de `IF NOT EXISTS` permite que o comando seja executado sem gerar um erro se ele já estiver instalado.

3. Em seguida, você deve usar a função `azure_ai.set_setting()` para configurar a conexão com o serviço Tradutor de Ia do Azure. Usando a mesma guia do navegador em que o Cloud Shell está aberto, minimize ou restaure o painel do Cloud Shell e navegue até o recurso do Tradutor de IA do Azure no [portal do Azure](https://portal.azure.com/). Quando estiver na página de recursos do Tradutor de IA do Azure, no menu de recursos, na seção **Gerenciamento de Recursos**, selecione **Chaves e Ponto de Extremidade** e copie uma das chaves disponíveis, sua região e seu ponto de extremidade de tradução de documentos.

    ![Captura de tela da página Chaves e Pontos de Extremidade do serviço Tradutor de IA do Azure sendo exibida, com os botões de cópia CHAVE 1, Região e Tradução de Documento realçados por caixas vermelhas.](media/18-azure-ai-translator-keys-and-endpoint.png)

    Você pode usar `KEY 1` ou `KEY 2`. Ter sempre duas chaves permite girar e regenerar chaves com segurança, sem causar interrupção de serviço.

4. Defina as configurações de `azure_cognitive` para apontar para o ponto de extremidade, a chave de assinatura e a região do Tradutor de IA. O valor para `azure_cognitive.endpoint` será o URL de tradução de documentos do seu serviço. O valor para `azure_cognitive.subscription_key` será Chave 1 ou Chave 2. O valor para `azure_cognitive.region` será a região da instância do Tradutor de IA do Azure.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint','https://<YOUR_ENDPOINT>.cognitiveservices.azure.com/');
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '<YOUR_KEY>');
    SELECT azure_ai.set_setting('azure_cognitive.region', '<YOUR_REGION>');
    ```

## Criar um procedimento armazenado para traduzir dados de listagens

Para preencher a tabela de tradução de idioma, você criará um procedimento armazenado para carregar dados em lotes.

1. Execute o comando a seguir no prompt `psql` para criar um novo procedimento armazenado chamado `translate_listing_descriptions`.

    ```sql
    CREATE OR REPLACE PROCEDURE translate_listing_descriptions(max_num_listings INT DEFAULT 10)
    LANGUAGE plpgsql
    AS $$
    BEGIN
        WITH batch_to_load(id, description) AS
        (
            SELECT id, description
            FROM listings l
            WHERE NOT EXISTS (SELECT * FROM listing_translations ll WHERE ll.listing_id = l.id)
            LIMIT max_num_listings
        )
        INSERT INTO listing_translations(listing_id, language_code, description)
        SELECT b.id, l.code, (unnest(tr.translations)).TEXT
        FROM batch_to_load b
            CROSS JOIN languages l
            CROSS JOIN LATERAL azure_cognitive.translate(b.description, l.code) tr;
    END;
    $$;
    ```

    Esse procedimento armazenado carregará um lote de 5 registros, traduzirá a descrição em cada idioma selecionado e inserirá as descrições traduzidas na tabela `listing_translations`.

2. Execute o procedimento armazenado usando o seguinte comando SQL:

    ```sql
    CALL translate_listing_descriptions(10);
    ```

    Essa chamada levará aproximadamente um segundo por listagem de aluguel para ser traduzida para cinco idiomas, portanto, cada execução deve levar aproximadamente 10 segundos. A saída do comando deve ser `CALL`, indicando que a chamada de procedimento armazenado foi bem-sucedida.

3. Chame o procedimento armazenado mais quatro vezes, para cada cinco vezes que você chamou esse procedimento. Isso gerará traduções para cada listagem na tabela.

4. Execute o script a seguir para obter a contagem de traduções de listagem.

    ```sql
    SELECT COUNT(*) FROM listing_translations;
    ```

    A chamada deve retornar um valor de 250, indicando que cada listagem foi traduzida para cinco idiomas. Você pode analisar ainda mais os dados consultando a tabela `listing_translations`.

## Criar um procedimento para adicionar uma nova listagem com traduções

Você tem um procedimento armazenado para traduzir listagens existentes, mas seus planos de internacionalização também exigem a conversão de novas listagens à medida que elas são criadas. Para fazer isso, você criará outro procedimento armazenado.

1. Execute o comando a seguir no prompt `psql` para criar um novo procedimento armazenado chamado `add_listing`.

    ```sql
    CREATE OR REPLACE PROCEDURE add_listing(id INT, name VARCHAR(255), description TEXT)
    LANGUAGE plpgsql
    AS $$
    DECLARE
    listing_id INT;
    BEGIN
        INSERT INTO listings(id, name, description)
        VALUES(id, name, description);

        INSERT INTO listing_translations(listing_id, language_code, description)
        SELECT id, l.code, (unnest(tr.translations)).TEXT
        FROM languages l
            CROSS JOIN LATERAL azure_cognitive.translate(description, l.code) tr;
    END;
    $$;
    ```

    Esse procedimento armazenado inserirá uma linha na tabela `listings`. Em seguida, ele traduzirá a descrição de cada idioma na tabela `languages` e as inserirá na tabela `listing_translations`.

2. Execute o procedimento armazenado usando o seguinte comando SQL:

    ```sql
    CALL add_listing(51, 'A Beautiful Home', 'This is a beautiful home in a great location.');
    ```

    A saída do comando deve ser `CALL`, indicando que a chamada de procedimento armazenado foi bem-sucedida.

3. Execute o script a seguir para obter as traduções para sua nova listagem.

    ```sql
    SELECT l.id, l.name, l.description, lt.language_code, lt.description AS translated_description
    FROM listing_translations lt
        INNER JOIN listings l ON lt.listing_id = l.id
    WHERE l.name = 'A Beautiful Home';
    ```

    A chamada deve retornar cinco linhas, com valores semelhantes à tabela a seguir.

    ```sql
     id  | listing_id | language_code |                    description                     
    -----+------------+---------------+------------------------------------------------------
     126 |          2 | de            | Dies ist ein schönes Haus in einer großartigen Lage.
     127 |          2 | zh-Hans       | 这是一个美丽的家，地理位置优越。
     128 |          2 | hi            | यह एक महान स्थान में एक सुंदर घर है।
     129 |          2 | hu            | Ez egy gyönyörű otthon egy nagyszerű helyen.
     130 |          2 | sw            | Hii ni nyumba nzuri katika eneo kubwa.
    ```

## Limpar

Depois de concluir este exercício, exclua os recursos do Azure que você criou. Você é cobrado pela capacidade configurada, não por quanto do banco de dados é utilizado. Siga estas instruções para excluir seu grupo de recursos e todos os recursos que você criou para este laboratório.

1. Abra um navegador da Web e navegue até o [portal do Azure](https://portal.azure.com/) e, na home page, selecione **Grupos de recursos** em Serviços do Azure.

    ![Captura de tela de grupos de recursos realçados por uma caixa vermelha em Serviços do Azure no portal do Azure.](media/17-azure-portal-home-azure-services-resource-groups.png)

2. Na caixa de pesquisa de filtro para qualquer campo, insira o nome do grupo de recursos que você criou para este laboratório e selecione o grupo de recursos na lista.

3. Na página **Visão geral** do grupo de recursos, selecione **Excluir grupo de recursos**.

    ![Captura de tela da folha Visão geral do grupo de recursos com o botão Excluir grupo de recursos realçado por uma caixa vermelha.](media/17-resource-group-delete.png)

4. Na caixa de diálogo de confirmação, insira o nome do seu grupo de recursos que deseja excluir e selecione **Excluir**.
