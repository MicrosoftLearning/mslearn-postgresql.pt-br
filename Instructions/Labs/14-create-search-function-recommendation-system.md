---
lab:
  title: Cria uma função de pesquisa para um sistema de recomendação
  module: Enable Semantic Search with Azure Database for PostgreSQL
---

# Cria uma função de pesquisa para um sistema de recomendação

Vamos criar um sistema de recomendação usando a pesquisa semântica. O sistema recomendará várias listagens com base em uma listagem de amostra fornecida. O exemplo pode ser da listagem que o usuário está visualizando ou de suas preferências. Implementaremos o sistema como uma função do PostgreSQL aproveitando a extensão `azure_openai`.

Ao final deste exercício, você terá definido uma função `recommend_listing` que fornece, na maioria das listagens `numResults`, listagens mais semelhantes `sampleListingId` que foi fornecido. Você pode usar esses dados para gerar novas oportunidades, como unir listagens recomendadas a listagens com desconto.

## Antes de começar

Você precisa de uma [assinatura do Azure](https://azure.microsoft.com/free) com direitos administrativos e deve ter permissão de acesso ao OpenAI do Azure nessa assinatura. Caso precise ter acesso ao OpenAI do Azure, solicite-o na página [Acesso limitado do OpenAI do Azure](https://learn.microsoft.com/legal/cognitive-services/openai/limited-access).

### Implantar recursos na assinatura do Azure

Esta etapa orienta você no uso de comandos da CLI do Azure do Azure Cloud Shell para criar um grupo de recursos e executar um script Bicep para implantar os serviços do Azure necessários para concluir este exercício em sua assinatura do Azure.

1. Abra um navegador da Web e acesse o [portal do Azure](https://portal.azure.com/).

2. SElecione o ícone do **Cloud Shell** na barra de ferramentas do portal do Azure para abrir um novo painel do [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) na parte inferior da janela do navegador.

    ![Captura de tela da barra de ferramentas do Azure com o ícone do Cloud Shell realçado por uma caixa vermelha.](media/14-portal-toolbar-cloud-shell.png)

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

    ![Captura de tela da página de banco de dados do Banco de Dados do Azure para PostfreSQL. Os bancos de dados e o Connect para o banco de dados de aluguéis são destacados por caixas vermelhas.](media/14-postgresql-rentals-database-connect.png)

3. No prompt "Senha para o usuário pgAdmin" no Cloud Shell, insira a senha gerada aleatoriamente para o logon do **pgAdmin**.

    Uma vez conectado, o prompt `psql` do banco de dados `rentals` é exibido.

4. Durante o restante deste exercício, você continuará trabalhando no Cloud Shell, portanto, pode ser útil expandir o painel na janela do navegador selecionando o botão **Maximizar** no canto superior direito do painel.

    ![Captura de tela do painel do Azure Cloud Shell com o botão Maximizar realçado por uma caixa vermelha.](media/14-azure-cloud-shell-pane-maximize.png)

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

2. Em seguida, use o comando `COPY` para carregar dados de arquivos CSV em cada tabela criada acima. Execute o comando a seguir para preencher a tabela `listings`:

    ```sql
    \COPY listings FROM 'mslearn-postgresql/Allfiles/Labs/Shared/listings.csv' CSV HEADER
    ```

    A saída do comando deve ser `COPY 50`, indicando que 50 linhas foram gravadas na tabela a partir do arquivo CSV.

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

## Criar a função de recomendação

A função de recomendação recebe um `sampleListingId` e retorna as `numResults` listagens que mais se assemelham. Para fazer isso, ele cria uma incorporação do nome e da descrição da listagem de exemplo e executa uma pesquisa semântica desse vetor de consulta nas incorporações de listagem.

```sql
CREATE FUNCTION
    recommend_listing(sampleListingId int, numResults int) 
RETURNS TABLE(
            out_listingName text,
            out_listingDescription text,
            out_score real)
AS $$ 
DECLARE
    queryEmbedding vector(1536); 
    sampleListingText text; 
BEGIN 
    sampleListingText := (
     SELECT
        name || ' ' || description
     FROM
        listings WHERE id = sampleListingId
    ); 

    queryEmbedding := (
     azure_openai.create_embeddings('embedding', sampleListingText, max_attempts => 5, retry_delay_ms => 500)
    );

    RETURN QUERY 
    SELECT
        name::text,
        description,
        -- cosine distance:
        (listings.listing_vector <=> queryEmbedding)::real AS score
    FROM
        listings 
    ORDER BY score ASC LIMIT numResults;
END $$
LANGUAGE plpgsql; 
```

Consulte o exemplo do [Sistema de recomendação](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/generative-ai-recommendation-system) para obter mais maneiras de personalizar essa função, por exemplo, combinando várias colunas de texto em um vetor de incorporação.

## Consultar a função de recomendação

Para consultar a função de recomendação, passe a ela uma ID de listagem e o número de recomendações que ela deve fazer.

```sql
select out_listingName, out_score from recommend_listing( (SELECT id from listings limit 1), 20); -- search for 20 listing recommendations closest to a listing
```

O resultado será semelhante a isso:

```sql
            out_listingname          | out_score 
-------------------------------------+-------------
 Sweet Seattle Urban Homestead 2 Bdr | 0.012512862
 Lovely Cap Hill Studio has it all!  | 0.09572035
 Metrobilly Retreat                  | 0.0982959
 Cozy Seattle Apartment Near UW      | 0.10320047
 Sweet home in the heart of Fremont  | 0.10442386
 Urban Chic, West Seattle Apartment  | 0.10654513
 Private studio apartment with deck  | 0.107096426
 Light and airy, steps to the Lake.  | 0.11008232
 Backyard Studio Apartment near UW   | 0.111279964
 2bed Rm Inner City Suite Near Dwtn  | 0.111340374
 West Seattle Vacation Junction      | 0.111758955
 Green Lake Private Ground Floor BR  | 0.112196356
 Stylish Queen Anne Apartment        | 0.11250153
 Family Friendly Modern Seattle Home | 0.11257711
 Bright Cheery Room in Seattle House | 0.11290849
 Big sunny central house with view!  | 0.11382967
 Modern, Light-Filled Fremont Flat   | 0.114443965
 Chill Central District 2BR          | 0.1153879
 Sunny Bedroom w/View: Wallingford   | 0.11549795
 Seattle Turret House (Apt 4)        | 0.11590502
```

Para ver o runtime da função, verifique se `track_functions` está habilitado na seção Parâmetros do servidor no Portal do Azure (você pode usar `PL` ou `ALL`):

![Uma captura de tela da seção de configuração Parâmetros do servidor mostrando o track_functions](media/14-track-functions.png)

Em seguida, você pode consultar a tabela de estatísticas da função:

```sql
SELECT * FROM pg_stat_user_functions WHERE funcname = 'recommend_listing';
```

O resultado deve ser algo parecido com isso:

```sql
 funcid | schemaname |    funcname       | calls | total_time | self_time 
--------+------------+-------------------+-------+------------+-----------
  28753 | public     | recommend_listing |     1 |    268.357 | 268.357
(1 row)
```

Neste parâmetro de comparação, obtivemos a incorporação da listagem de amostra e realizamos a pesquisa semântica em cerca de 4 mil documentos em ~270 ms.

## Verifique seu trabalho

1. Verifique se a função existe com a assinatura correta:

    ```sql
    \df recommend_listing
    ```

    Você deve ver o seguinte:

    ```sql
    public | recommend_listing | TABLE(out_listingname text, out_listingdescription text, out_score real) | samplelistingid integer, numre
    sults integer | func
    ```

2. Verifique se você pode consultá-la usando a seguinte consulta:

    ```sql
    select out_listingName, out_score from recommend_listing( (SELECT id from listings limit 1), 20); -- search for 20 listing recommendations closest to a listing
    ```

## Limpar

Depois de concluir este exercício, exclua os recursos do Azure que você criou. Você é cobrado pela capacidade configurada, não por quanto do banco de dados é utilizado. Siga estas instruções para excluir seu grupo de recursos e todos os recursos que você criou para este laboratório.

> [!Note]
>
> Se você planeja concluir módulos adicionais neste roteiro de aprendizagem, pode ignorar esta tarefa até concluir todos os módulos que pretende concluir.

1. Abra um navegador da Web e navegue até o [portal do Azure](https://portal.azure.com/) e, na home page, selecione **Grupos de recursos** em Serviços do Azure.

    ![Captura de tela de grupos de recursos realçados por uma caixa vermelha em Serviços do Azure no portal do Azure.](media/14-azure-portal-home-azure-services-resource-groups.png)

2. Na caixa de pesquisa de filtro para qualquer campo, insira o nome do grupo de recursos que você criou para este laboratório e selecione seu grupo de recursos na lista.

3. Na página **Visão geral** do grupo de recursos, selecione **Excluir grupo de recursos**.

    ![Captura de tela da folha Visão geral do grupo de recursos com o botão Excluir grupo de recursos realçado por uma caixa vermelha.](media/14-resource-group-delete.png)

4. Na caixa de diálogo de confirmação, insira o nome do seu grupo de recursos que deseja excluir e selecione **Excluir**.
