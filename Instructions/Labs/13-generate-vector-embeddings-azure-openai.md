---
lab:
  title: Gerar inserir de vetor com o OpenAI do Azure
  module: Enable Semantic Search with Azure Database for PostgreSQL
---

# Gerar inserir de vetor com o OpenAI do Azure

Para executar pesquisas semânticas, você deve primeiro gerar vetores de inserção de um modelo, armazená-los em um banco de dados de vetor e, em seguida, consultar as inserções. Você criará um banco de dados, o preencherá com dados de exemplo e executará pesquisas semânticas nessas listagens.

Ao final deste exercício, você terá uma instância de servidor flexível do Banco de Dados do Azure para PostgreSQL com as extensões `vector` e `azure_ai` habilitadas. Você gerará incorporações para a tabela `listings` do conjunto de dados de [dados abertos do Airbnb de Seattle](https://www.kaggle.com/datasets/airbnb/seattle?select=listings.csv). Você também executará pesquisas semânticas nessas listagens gerando o vetor de incorporação de uma consulta e executando uma pesquisa de distância de cosseno vetorial.

## Antes de começar

Você precisa de uma [assinatura do Azure](https://azure.microsoft.com/free) com direitos administrativos e deve ter permissão de acesso ao OpenAI do Azure nessa assinatura. Caso precise ter acesso ao OpenAI do Azure, solicite-o na página [Acesso limitado do OpenAI do Azure](https://learn.microsoft.com/legal/cognitive-services/openai/limited-access).

### Implantar recursos na assinatura do Azure

Esta etapa orienta você no uso de comandos da CLI do Azure do Azure Cloud Shell para criar um grupo de recursos e executar um script Bicep para implantar os serviços do Azure necessários para concluir este exercício em sua assinatura do Azure.

> **Observação**: se você estiver fazendo vários módulos neste roteiro de aprendizagem, poderá compartilhar o ambiente do Azure entre eles. Nesse caso, você só precisa concluir essa etapa de implantação de recursos uma vez.

1. Abra um navegador da Web e acesse o [portal do Azure](https://portal.azure.com/).

2. SElecione o ícone do **Cloud Shell** na barra de ferramentas do portal do Azure para abrir um novo painel do [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) na parte inferior da janela do navegador.

    ![Captura de tela da barra de ferramentas do Azure com o ícone do Cloud Shell realçado por uma caixa vermelha.](media/13-portal-toolbar-cloud-shell.png)

    Se solicitado, selecione as opções necessárias para abrir um shell do *Bash*. Se você já usou um console do *PowerShell*, alterne-o para um shell do *Bash*.

3. No prompt do Cloud Shell, insira o seguinte para clonar o repositório GitHub que contém os recursos do exercício:

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

4. Em seguida, você executará três comandos para definir variáveis para reduzir a digitação redundante ao usar comandos da CLI do Azure para criar recursos do Azure. As variáveis representam o nome a ser atribuído ao seu grupo de recursos (`RG_NAME`), a região do Azure (`REGION`) na qual os recursos serão implantados e uma senha gerada aleatoriamente para o logon de administrador do PostgreSQL (`ADMIN_PASSWORD`).

    No primeiro comando, a região atribuída à variável correspondente é `eastus`, mas você também pode substituí-la por um local de sua preferência. No entanto, se estiver substituindo o padrão, você deverá selecionar outra [região do Azure que dê suporte ao resumo abstrato](https://learn.microsoft.com/azure/ai-services/language-service/summarization/region-support) para garantir que você possa concluir todas as tarefas nos módulos deste roteiro de aprendizagem.

    ```bash
    REGION=eastus
    ```

    O comando a seguir atribui o nome a ser usado para o grupo de recursos que abrigará todos os recursos usados neste exercício. O nome do grupo de recursos atribuído à variável correspondente é `rg-learn-postgresql-ai-$REGION`, onde `$REGION` é o local especificado acima. No entanto, você pode alterá-lo para qualquer outro nome de grupo de recursos que atenda às suas preferências.

    ```bash
    RG_NAME=rg-learn-postgresql-ai-$REGION
    ```

    O comando final gera aleatoriamente uma senha para o login de administrador do PostgreSQL. **Certifique-se de copiá-la** para um local seguro para usar mais tarde para se conectar ao seu servidor flexível do PostgreSQL.

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
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy.bicep" --parameters restore=false adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    O script de implantação do Bicep provisiona os serviços do Azure necessários para concluir este exercício em seu grupo de recursos. Os recursos implantados incluem um Banco de Dados do Azure para PostgreSQL – servidor flexível, OpenAI do Azure e um serviço de Linguagem de IA do Azure. O script Bicep também executa algumas etapas de configuração, como adicionar as extensões `azure_ai` e `vector` à _lista de permitidos_ do servidor PostgreSQL (por meio do parâmetro server `azure.extensions`), criar um banco de dados nomeado `rentals` no servidor e adicionar uma implantação nomeada `embedding` usando o modelo `text-embedding-ada-002` ao serviço OpenAI do Azure. Observe que o arquivo Bicep é compartilhado por todos os módulos neste roteiro de aprendizagem, portanto, você só pode usar alguns dos recursos implantados em alguns exercícios.

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

1. No [portal do Azure](https://portal.azure.com/), navegue até o servidor flexível do Banco de Dados do Azure para PostgreSQL recém-criado.

2. No menu de recursos, em **Configurações**, selecione **Bancos de dados** e selecione **Conectar** para o banco de dados`rentals`.

    ![Captura de tela da página de banco de dados do Banco de Dados do Azure para PostfreSQL. Os bancos de dados e o Connect para o banco de dados de aluguéis são destacados por caixas vermelhas.](media/13-postgresql-rentals-database-connect.png)

3. No prompt "Senha para o usuário pgAdmin" no Cloud Shell, insira a senha gerada aleatoriamente para o logon do **pgAdmin**.

    Uma vez conectado, o prompt `psql` do banco de dados `rentals` é exibido.

4. Durante o restante deste exercício, você continuará trabalhando no Cloud Shell, portanto, pode ser útil expandir o painel na janela do navegador selecionando o botão **Maximizar** no canto superior direito do painel.

    ![Captura de tela do painel do Azure Cloud Shell com o botão Maximizar realçado por uma caixa vermelha.](media/13-azure-cloud-shell-pane-maximize.png)

## Configuração: configurar extensões

Para armazenar e consultar vetores e gerar inserções, você precisa colocar na lista de permitidos e habilitar duas extensões para o servidor flexível do Banco de Dados do Azure para PostgreSQL: `vector` e `azure_ai`.

1. Para permitir ambas as extensões, adicione `vector` e `azure_ai` ao parâmetro do servidor `azure.extensions`, de acordo com as instruções fornecidas em [Como usar extensões do PostgreSQL](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions).

2. Para habilitar a extensão `vector`, execute o seguinte comando SQL. Para instruções detalhadas, consulte [Como habilitar e usar `pgvector` no Banco de Dados do Azure para PostgreSQL - servidor flexível](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-use-pgvector#enable-extension).

    ```sql
    CREATE EXTENSION vector;
    ```

3. Para habilitar a extensão `azure_ai`, execute o seguinte comando SQL. Você precisará do ponto de extremidade e da chave de API do recurso OpenAI do Azure. Para obter instruções detalhadas, leia [Habilitar a extensão `azure_ai`](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/generative-ai-azure-overview#enable-the-azure_ai-extension).

    ```sql
    CREATE EXTENSION azure_ai;
    SELECT azure_ai.set_setting('azure_openai.endpoint', 'https://<endpoint>.openai.azure.com');
    SELECT azure_ai.set_setting('azure_openai.subscription_key', '<API Key>');
    ```

## Preencher o banco de dados com alguns dados de exemplo

Antes de explorar a extensão `azure_ai`, adicione algumas tabelas ao banco de dados `rentals` e preencha-as com dados de exemplo para que você tenha informações para trabalhar enquanto examina a funcionalidade da extensão.

1. Execute os seguintes comandos para criar as tabelas `listings` e `reviews` para armazenar dados de listagem de imóveis alugados e avaliação do cliente:

    ```sql
    DROP TABLE IF EXISTS listings;
    
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

    ```sql
    DROP TABLE IF EXISTS reviews;
    
    CREATE TABLE reviews (
        id int,
        listing_id int, 
        date date,
        comments text
    );
    ```

2. Em seguida, use o comando `COPY` para carregar dados de arquivos CSV em cada tabela criada acima. Execute o comando a seguir para preencher a tabela `listings`:

    ```sql
    \COPY listings FROM 'mslearn-postgresql/Allfiles/Labs/Shared/listings.csv' CSV HEADER
    ```

    A saída do comando deve ser `COPY 50`, indicando que 50 linhas foram gravadas na tabela a partir do arquivo CSV.

3. Por fim, execute o comando abaixo para carregar as avaliações dos clientes na tabela `reviews`:

    ```sql
    \COPY reviews FROM 'mslearn-postgresql/Allfiles/Labs/Shared/reviews.csv' CSV HEADER
    ```

    A saída do comando deve ser `COPY 354`, indicando que 354 linhas foram gravadas na tabela a partir do arquivo CSV.

Para redefinir os dados de amostra, você pode executar `DROP TABLE listings` e repetir essas etapas.

## Criar e armazenar vetores de incorporação

Agora que temos alguns dados de amostra, é hora de gerar e armazenar os vetores de incorporação. A extensão `azure_ai` facilita a chamada da API de inserção do OpenAI do Azure.

1. Adicione a coluna de vetor de incorporação.

    O modelo `text-embedding-ada-002` está configurado para retornar 1.536 dimensões, portanto, use-o para o tamanho da coluna vetorial.

    ```sql
    ALTER TABLE listings ADD COLUMN listing_vector vector(1536);
    ```

1. Gere um vetor de inserção para a descrição de cada listagem chamando o OpenAI do Azure por meio da create_embeddings função definida pelo usuário, que é implementada pela extensão azure_ai:

    ```sql
    UPDATE listings
    SET listing_vector = azure_openai.create_embeddings('embedding', description, max_attempts => 5, retry_delay_ms => 500)
    WHERE listing_vector IS NULL;
    ```

    Observe que isso pode levar vários minutos, dependendo da cota disponível.

1. Veja um vetor de exemplo executando esta consulta:

    ```sql
    SELECT listing_vector FROM listings LIMIT 1;
    ```

    Você obterá um resultado semelhante a este, mas com 1536 colunas vetoriais:

    ```sql
    postgres=> SELECT listing_vector FROM listings LIMIT 1;
    -[ RECORD 1 ]--+------ ...
    listing_vector | [-0.0018742813,-0.04530062,0.055145424, ... ]
    ```

## Executar uma consulta de pesquisa semântica

Agora que você tem dados de listagem aumentados com vetores de inserção, é hora de executar uma consulta de pesquisa semântica. Para fazer isso, obtenha o vetor de incorporação da cadeia de caracteres de consulta e, em seguida, execute uma pesquisa de cosseno para localizar as listagens cujas descrições são semanticamente mais semelhantes à consulta.

1. Gere a inserção para a cadeia de caracteres de consulta.

    ```sql
    SELECT azure_openai.create_embeddings('embedding', 'bright natural light');
    ```

    Você poderá receber uma resposta como esta:

    ```sql
    -[ RECORD 1 ]-----+-- ...
    create_embeddings | {-0.0020871465,-0.002830255,0.030923981, ...}
    ```

1. Use a incorporação em uma pesquisa de cosseno (`<=>` representa a operação de distância de cosseno), buscando as 10 principais listagens mais semelhantes à consulta.

    ```sql
    SELECT id, name FROM listings ORDER BY listing_vector <=> azure_openai.create_embeddings('embedding', 'bright natural light')::vector LIMIT 10;
    ```

    Você receberá um resultado semelhante a este. Os resultados podem variar, pois não há garantia de que os vetores de incorporação sejam determinísticos:

    ```sql
        id    |                name                
    ----------+-------------------------------------
     6796336  | A duplex near U district!
     7635966  | Modern Capitol Hill Apartment
     7011200  | Bright 1 bd w deck. Great location
     8099917  | The Ravenna Apartment
     10211928 | Charming Ravenna Bungalow
     692671   | Sun Drenched Ballard Apartment
     7574864  | Modern Greenlake Getaway
     7807658  | Top Floor Corner Apt-Downtown View
     10265391 | Art filled, quiet, walkable Seattle
     5578943  | Madrona Studio w/Private Entrance
    ```

1. Você também pode projetar a coluna `description` para poder ler o texto das linhas correspondentes cujas descrições eram semanticamente semelhantes. Por exemplo, esta consulta retorna a melhor correspondência:

    ```sql
    SELECT id, description FROM listings ORDER BY listing_vector <=> azure_openai.create_embeddings('embedding', 'bright natural light')::vector LIMIT 1;
    ```

    O que imprime algo como: 

    ```sql
       id    | description
    ---------+------------
     6796336 | This is a great place to live for summer because you get a lot of sunlight at the living room. A huge living room space with comfy couch and one ceiling window and glass windows around the living room.
    ```

Para entender intuitivamente a pesquisa semântica, observe que a descrição não contém os termos "brilhante" ou "natural". Mas destaca "verão" e "luz do sol", "janelas" e uma "janela do teto".

## Verifique seu trabalho

Depois de executar as etapas acima, a tabela `listings` contém dados de amostra do [Seattle Airbnb Open Data](https://www.kaggle.com/datasets/airbnb/seattle/data?select=listings.csv) no Kaggle. As listagens foram aumentadas com vetores de incorporação para executar pesquisas semânticas.

1. Confirme se a tabela de listagens tem quatro colunas: `id`, `name`, `description`, e `listing_vector`.

    ```sql
    \d listings
    ```

    Será algo como:

    ```sql
                            Table "public.listings"
          Column    |         Type           | Collation | Nullable | Default 
    ----------------+------------------------+-----------+----------+---------
      id            | integer                |           | not null | 
      name          | character varying(255) |           | not null | 
      description   | text                   |           | not null | 
     listing_vector | vector(1536)           |           |          | 
     Indexes:
        "listings_pkey" PRIMARY KEY, btree (id)
    ```

1. Confirme se pelo menos uma linha tem uma coluna listing_vector preenchida.

    ```sql
    SELECT COUNT(*) > 0 FROM listings WHERE listing_vector IS NOT NULL;
    ```

    O resultado deve mostrar um `t`, que significa verdadeiro. Uma indicação de que há pelo menos uma linha com incorporações de sua coluna de descrição correspondente:

    ```sql
    ?column? 
    ----------
    t
    (1 row)
    ```

    Confirme se o vetor de incorporação tem 1536 dimensões:

    ```sql
    SELECT vector_dims(listing_vector) FROM listings WHERE listing_vector IS NOT NULL LIMIT 1;
    ```

    Resultando em:

    ```sql
    vector_dims 
    -------------
            1536
    (1 row)
    ```

1. Confirme se as pesquisas semânticas retornam resultados.

    Use a incorporação em uma pesquisa de cosseno, buscando as 10 listagens mais semelhantes à consulta.

    ```sql
    SELECT id, name FROM listings ORDER BY listing_vector <=> azure_openai.create_embeddings('embedding', 'bright natural light')::vector LIMIT 10;
    ```

    Você obterá um resultado como este, dependendo de quais linhas foram atribuídas a vetores de incorporação:

    ```sql
     id |                name                
    --------+-------------------------------------
     315120 | Large, comfy, light, garden studio
     429453 | Sunny Bedroom #2 w/View: Wallingfrd
     17951  | West Seattle, The Starlight Studio
     48848  | green suite seattle - dog friendly
     116221 | Modern, Light-Filled Fremont Flat
     206781 | Bright & Spacious Studio
     356566 | Sunny Bedroom w/View: Wallingford
     9419   | Golden Sun vintage warm/sunny
     136480 | Bright Cheery Room in Seattle House
     180939 | Central District Green GardenStudio
    ```

## Limpar

Depois de concluir este exercício, exclua os recursos do Azure que você criou. Você é cobrado pela capacidade configurada, não por quanto do banco de dados é utilizado. Siga estas instruções para excluir seu grupo de recursos e todos os recursos que você criou para este laboratório.

1. Abra um navegador da Web e navegue até o [portal do Azure](https://portal.azure.com/) e, na home page, selecione **Grupos de recursos** em Serviços do Azure.

    ![Captura de tela de grupos de recursos realçados por uma caixa vermelha em Serviços do Azure no portal do Azure.](media/13-azure-portal-home-azure-services-resource-groups.png)

2. Na caixa de pesquisa de filtro para qualquer campo, insira o nome do grupo de recursos que você criou para este laboratório e selecione seu grupo de recursos na lista.

3. Na página **Visão geral** do grupo de recursos, selecione **Excluir grupo de recursos**.

    ![Captura de tela da folha Visão geral do grupo de recursos com o botão Excluir grupo de recursos realçado por uma caixa vermelha.](media/13-resource-group-delete.png)

4. Na caixa de diálogo de confirmação, insira o nome do seu grupo de recursos que deseja excluir e selecione **Excluir**.
