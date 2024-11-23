---
lab:
  title: Realizar as sumarizações extrativa e abstrativa
  module: Summarize data using Azure AI Services and Azure Database for PostgreSQL
---

# Realizar as sumarizações extrativa e abstrativa

O aplicativo de aluguel de imóveis mantido pela Margie's Travel fornece uma maneira para os gerentes de propriedades descreverem as listagens de aluguel. Muitas das descrições no sistema são longas, fornecendo muitos detalhes sobre o imóvel alugado, sua vizinhança e atrações, lojas e outras comodidades locais. Um recurso que foi solicitado à medida que você implementa novos recursos baseados em IA para o aplicativo é usar IA generativa para criar resumos concisos dessas descrições, tornando mais fácil para os usuários revisar as propriedades rapidamente. Neste exercício, você usará a extensão `azure_ai` em um servidor flexível do Banco de Dados do Azure para PostgreSQL para executar resumos abstratos e extrativos em descrições de propriedades de aluguel e comparar os resumos resultantes.

## Antes de começar

Você precisa de uma [assinatura do Azure](https://azure.microsoft.com/free) com direitos administrativos e deve ter permissão de acesso ao OpenAI do Azure nessa assinatura. Caso precise ter acesso ao OpenAI do Azure, solicite-o na página [Acesso limitado do OpenAI do Azure](https://learn.microsoft.com/legal/cognitive-services/openai/limited-access).

### Implantar recursos na assinatura do Azure

Esta etapa orienta você no uso de comandos da CLI do Azure do Azure Cloud Shell para criar um grupo de recursos e executar um script Bicep para implantar os serviços do Azure necessários para concluir este exercício em sua assinatura do Azure.

1. Abra um navegador da Web e acesse o [portal do Azure](https://portal.azure.com/).

2. SElecione o ícone do **Cloud Shell** na barra de ferramentas do portal do Azure para abrir um novo painel do [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) na parte inferior da janela do navegador.

    ![Captura de tela da barra de ferramentas do Azure com o ícone do Cloud Shell realçado por uma caixa vermelha.](media/15-portal-toolbar-cloud-shell.png)

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

    O script de implantação do Bicep provisiona os serviços do Azure necessários para concluir este exercício em seu grupo de recursos. Os recursos implantados incluem um servidor flexível do Banco de Dados do Azure para PostgreSQL, o OpenAI do Azure e um serviço de Linguagem de IA do Azure. O script Bicep também executa algumas etapas de configuração, como adicionar as extensões `azure_ai` e `vector` à _lista de permitidos_ do servidor PostgreSQL (por meio do parâmetro azure.extensions server), criar um banco de dados nomeado `rentals` no servidor e adicionar uma implantação nomeada `embedding` usando o modelo `text-embedding-ada-002` ao serviço OpenAI do Azure. Observe que o arquivo Bicep é compartilhado por todos os módulos neste roteiro de aprendizagem, portanto, você só pode usar alguns dos recursos implantados em alguns exercícios.

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

Nesta tarefa, você se conectará ao banco de dados `rentals` no servidor flexível do Banco de Dados do Azure para PostgreSQL usando o [utilitário de linha de comando psql](https://www.postgresql.org/docs/current/app-psql.html) do [Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview).

1. No [portal do Azure](https://portal.azure.com/), navegue até a instância recém-criada do servidor flexível do Banco de Dados do Azure para PostgreSQL.

2. No menu de recursos, em **Configurações**, selecione **Bancos de dados** e selecione **Conectar** para o banco de dados`rentals`.

    ![Captura de tela da página de banco de dados do Banco de Dados do Azure para PostfreSQL. Os bancos de dados e o Connect para o banco de dados de aluguéis são destacados por caixas vermelhas.](media/15-postgresql-rentals-database-connect.png)

3. No prompt "Senha para o usuário pgAdmin" no Cloud Shell, insira a senha gerada aleatoriamente para o logon do **pgAdmin**.

    Uma vez conectado, o prompt `psql` do banco de dados `rentals` é exibido.

4. Durante o restante deste exercício, você continuará trabalhando no Cloud Shell, portanto, pode ser útil expandir o painel na janela do navegador selecionando o botão **Maximizar** no canto superior direito do painel.

    ![Captura de tela do painel do Azure Cloud Shell com o botão Maximizar realçado por uma caixa vermelha.](media/15-azure-cloud-shell-pane-maximize.png)

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

    Antes que uma extensão possa ser instalada e usada em um banco de dados de servidor flexível do Banco de Dados do Azure para PostgreSQL, ela deve ser adicionada à _lista de permitidos_ do servidor, conforme descrito em [como usar extensões do PostgreSQL](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions).

2. Agora, você está pronto para instalar a extensão `azure_ai` usando o comando [CREATE EXTENSION](https://www.postgresql.org/docs/current/sql-createextension.html).

    ```sql
    CREATE EXTENSION IF NOT EXISTS azure_ai;
    ```

    `CREATE EXTENSION` carrega uma nova extensão no banco de dados executando seu arquivo de script. Esse script normalmente cria novos objetos SQL, como funções, tipos de dados e esquemas. Um erro será gerado se já existir uma extensão com o mesmo nome. A adição de `IF NOT EXISTS` permite que o comando seja executado sem gerar um erro se ele já estiver instalado.

## Conecte a sua conta dos serviços de IA do Azure

As integrações de serviços de IA do Azure incluídas no esquema `azure_cognitive` da extensão `azure_ai` fornecem um conjunto avançado de recursos da Linguagem de IA do Azure acessíveis diretamente do banco de dados. Os recursos de resumo de texto são habilitados por meio do [serviço de Linguagem de IA do Azure](https://learn.microsoft.com/azure/ai-services/language-service/overview).

1. Para fazer chamadas com êxito no serviço de linguagem de IA do Azure usando a extensão `azure_ai`, você deve fornecer o ponto de extremidade do serviço e uma chave da extensão. Usando a mesma guia do navegador em que o Cloud Shell está aberto, navegue até o recurso de serviço de linguagem no [portal do Azure](https://portal.azure.com/) e selecione o item **Chaves e ponto de extremidade** em **Gerenciamento de recursos** no menu de navegação à esquerda.

    ![Captura de tela da página Chaves e pontos de extremidade do serviço de linguagem do Azure, com os botões de cópia CHAVE 1 e Ponto de extremidade realçados por caixas vermelhas.](media/15-azure-language-service-keys-endpoints.png)

    > [!Note]
    >
    > Se você recebeu a mensagem `NOTICE: extension "azure_ai" already exists, skipping CREATE EXTENSION` ao instalar a extensão `azure_ai` acima e configurou anteriormente a extensão com o ponto de extremidade e a chave do serviço de linguagem, poderá usar a função `azure_ai.get_setting()` para confirmar se essas configurações estão corretas e, em seguida, ignorar a etapa 2 se estiverem.

2. Copie os valores do ponto de extremidade e da chave de acesso e, nos comandos abaixo, substitua os tokens `{endpoint}` e `{api-key}` pelos valores copiados do portal do Azure. Execute os comandos no prompt de comando `psql` no Cloud Shell para adicionar seus valores à tabela `azure_ai.settings`.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '{api-key}');
    ```

## Revise os recursos de resumo da extensão

Nesta tarefa, você revisará as duas funções de resumo no esquema `azure_cognitive`.

1. Para o restante deste exercício, você trabalhará exclusivamente no Cloud Shell, portanto, pode ser útil expandir o painel na janela do navegador selecionando o botão **Maximizar** no canto superior direito do painel do Cloud Shell.

    ![Captura de tela do painel do Azure Cloud Shell com o botão Maximizar realçado por uma caixa vermelha.](media/15-azure-cloud-shell-pane-maximize.png)

2. Ao trabalhar com `psql` no Cloud Shell, habilitar a exibição estendida para resultados de consulta pode ser útil, pois melhora a legibilidade da saída para comandos subsequentes. Execute o seguinte comando para permitir que a exibição estendida seja aplicada automaticamente.

    ```sql
    \x auto
    ```

3. As funções de resumo de texto da extensão `azure_ai` são encontradas no esquema `azure_cognitive`. Para sumarização extrativa, use a função `summarize_extractive()`. Use o [metacomando `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) para examinar a função ao executar:

    ```sql
    \df azure_cognitive.summarize_extractive
    ```

    A saída do metacomando mostra o esquema, o nome, o tipo de dados de resultado e os argumentos da função. Essas informações ajudam você a entender como interagir com a função de suas consultas.

    A saída mostra três sobrecargas da função `summarize_extractive()`, permitindo que você revise suas diferenças. A propriedade `Argument data types` na saída revela a lista de argumentos que as sobrecargas das três funções esperam:

    | Argumento | Tipo | Padrão | Descrição |
    | -------- | ---- | ------- | ----------- |
    | text | `text` ou `text[]` || Os textos para os quais os resumos devem ser gerados. |
    | language_text | `text` ou `text[]` || Código de idioma (ou matriz de códigos de idioma) que representa o idioma do texto a ser resumido. Revise a [lista de idiomas com suporte](https://learn.microsoft.com/azure/ai-services/language-service/summarization/language-support) para obter os códigos de idioma necessários. |
    | sentence_count | `integer` | 3 | O número de frases resumidas a serem geradas. |
    | sort_by | `text` | “deslocamento” | A ordem de classificação para as frases resumidas geradas. Os valores aceitáveis são “deslocamento” e “classificação”, com deslocamento representando a posição inicial de cada frase extraída dentro do conteúdo original e classificação sendo um indicador gerado por IA do grau de relevância de uma frase é para a ideia principal do conteúdo. |
    | batch_size | `integer` | 25 | Somente para as duas sobrecargas que esperam uma entrada de `text[]`. Especifica o número de registros a serem processados por vez. |
    | disable_service_logs | `boolean` | false | Sinalizador que indica se os logs de serviço devem ser desativados. |
    | timeout_ms | `integer` | NULO | Tempo limite em milissegundos após o qual a operação é interrompida. |
    | throw_on_error | `boolean` | true | Sinalizador que indica se a função deve, em caso de erro, gerar uma exceção que resulte em uma reversão das transações de encapsulamento. |
    | max_attempts | `integer` | 1 | Número de novas tentativas de chamar os Serviços de IA do Azure em caso de falha. |
    | retry_delay_ms | `integer` | 1000 | Tempo, em milissegundos, para aguardar antes de tentar chamar novamente o ponto de extremidade dos Serviços de IA do Azure. |

4. Repita a etapa acima, mas desta vez execute o [metacomando `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) para a função `azure_cognitive.summarize_abstractive()` e revise a saída.

    As duas funções têm assinaturas semelhantes, embora `summarize_abstractive()` não tem o parâmetro `sort_by`, e retornam uma matriz de `text` versus a matriz de tipos compostos `azure_cognitive.sentence` retornados pela função `summarize_extractive()`. Essa disparidade tem a ver com a maneira como os dois métodos diferentes geram resumos. A sumarização extrativa identifica as frases mais críticas dentro do texto que está resumindo, classifica-as e as retorna como resumo. Por outro lado, a sumarização abstrato usa IA generativa para criar frases novas e originais que resumem os pontos-chave do texto.

5. Também é imperativo entender a estrutura do tipo de dados que uma função retorna para que você possa manipular corretamente a saída em suas consultas. Para inspecionar o tipo `azure_cognitive.sentence` retornado pela função `summarize_extractive()`, execute:

    ```sql
    \dT+ azure_cognitive.sentence
    ```

6. A saída do comando acima revela que o tipo `sentence` é um `tuple`. Para examinar a estrutura do `tuple` e examinar as colunas contidas no tipo composto `sentence`, execute:

    ```sql
    \d+ azure_cognitive.sentence
    ```

    A saída do comando deve ser semelhante ao seguinte:

    ```sql
                            Composite type "azure_cognitive.sentence"
        Column  |     Type         | Collation | Nullable | Default | Storage  | Description 
    ------------+------------------+-----------+----------+---------+----------+-------------
     text       | text             |           |           |        | extended | 
     rank_score | double precision |           |           |        | plain    |
    ```

    O `azure_cognitive.sentence` é um tipo composto que contém o texto de uma frase extraída e uma pontuação de classificação para cada frase, indicando quão relevante a frase é para o tópico principal do texto. A sumarização de documentos classifica as frases extraídas, e você pode determinar se elas são retornadas na ordem em que aparecem ou de acordo com a classificação delas.

## Criar resumos para descrições de propriedades

Nesta tarefa, você usa as funções `summarize_extractive()` e `summarize_abstractive()` para criar sumarizações concisas de duas frases para descrições de propriedade.

1. Agora que você revisou a função `summarize_extractive()` e o `sentiment_analysis_result` retornado, vamos colocá-la em uso. Execute a seguinte consulta simples, que executa a análise de sentimento em alguns comentários na tabela `reviews`:

    ```sql
    SELECT
        id,
        name,
        description,
        azure_cognitive.summarize_extractive(description, 'en', 2) AS extractive_summary
    FROM listings
    WHERE id IN (1, 2);
    ```

    Compare as duas frases no campo `extractive_summary` na saída com o original `description`, observando que as frases não são originais, mas extraídas do `description`. Os valores numéricos listados após cada frase são a pontuação de classificação atribuída pelo serviço de Linguagem.

2. Em seguida, execute a sumarização abstrata nos registros idênticos:

    ```sql
    SELECT
        id,
        name,
        description,
        azure_cognitive.summarize_abstractive(description, 'en', 2) AS abstractive_summary
    FROM listings
    WHERE id IN (1, 2);
    ```

    Os recursos de sumarização abstrata da extensão fornecem um resumo exclusivo em linguagem natural que encapsula a intenção geral do texto original.

    Se você receber um erro semelhante ao seguinte, você escolheu uma região que não dá suporte à sumarização abstrata ao criar seu ambiente do Azure:

    ```bash
    ERROR: azure_cognitive.summarize_abstractive: InvalidRequest: Invalid Request.

    InvalidParameterValue: Job task: 'AbstractiveSummarization-task' failed with validation errors: ['Invalid Request.']

    InvalidRequest: Job task: 'AbstractiveSummarization-task' failed with validation error: Document abstractive summarization is not supported in the region Central US. The supported regions are North Europe, East US, West US, UK South, Southeast Asia.
    ```

    Para poder executar essa etapa e concluir as tarefas restantes usando a sumarização abstrata, você deve criar um novo serviço de Linguagem de IA do Azure em uma das regiões com suporte especificadas na mensagem de erro. Esse serviço pode ser provisionado no mesmo grupo de recursos usado para outros recursos de laboratório. Como alternativa, você pode substituir as tarefas restantes pela sumarização extrativa, mas não obterá o benefício de poder comparar a saída das duas técnicas de sumarização diferentes.

3. Execute uma consulta final para fazer uma comparação lado a lado das duas técnicas de sumarização:

    ```sql
    SELECT
        id,
        azure_cognitive.summarize_extractive(description, 'en', 2) AS extractive_summary,
        azure_cognitive.summarize_abstractive(description, 'en', 2) AS abstractive_summary
    FROM listings
    WHERE id IN (1, 2);
    ```

    Ao colocar os resumos gerados lado a lado, é fácil comparar a qualidade dos resumos gerados por cada método. Para o aplicativo Margie's Travel, a sumarização abstrata é a melhor opção, fornecendo resumos concisos que fornecem informações de alta qualidade de maneira natural e legível. Embora forneçam alguns detalhes, as sumarizações extrativas são mais desconexas e oferecem menos valor do que o conteúdo original criado pela sumarização abstrata.

## Resumo da descrição da loja no banco de dados

1. Execute a seguinte consulta para alterar a tabela `listings`, adicionando uma nova coluna `summary`:

    ```sql
    ALTER TABLE listings
    ADD COLUMN summary text;
    ```

2. Para usar a IA generativa para criar resumos para todas as propriedades existentes no banco de dados, é mais eficiente enviar as descrições em lotes, permitindo que o serviço de linguagem processe vários registros simultaneamente.

    ```sql
    WITH batch_cte AS (
        SELECT azure_cognitive.summarize_abstractive(ARRAY(SELECT description FROM listings ORDER BY id), 'en', batch_size => 25) AS summary
    ),
    summary_cte AS (
        SELECT
            ROW_NUMBER() OVER () AS id,
            ARRAY_TO_STRING(summary, ',') AS summary
        FROM batch_cte
    )
    UPDATE listings AS l
    SET summary = s.summary
    FROM summary_cte AS s
    WHERE l.id = s.id;
    ```

    A instrução de atualização usa duas CTEs (expressões de tabela comuns) para trabalhar nos dados antes de atualizar a tabela `listings` com resumos. A primeira CTE (`batch_cte`) envia todos os valores `description` da tabela `listings` para o serviço de linguagem para gerar sumarizações abstratas. Ela faz isso em lotes de 25 registros por vez. A segunda CTE (`summary_cte`) usa a posição ordinal dos resumos retornados pela função `summarize_abstractive()` para atribuir a cada sumarização um `id` correspondente ao registro de onde `description` veio na tabela `listings`. Ela também usa a função `ARRAY_TO_STRING` para extrair os resumos gerados do valor de retorno da matriz de texto (`text[]`) e convertê-lo em uma cadeia de caracteres simples. Por fim, a instrução `UPDATE` grava a sumarização na tabela `listings` da listagem associada.

3. Como última etapa, execute uma consulta para exibir as sumarizações gravadas na tabela `listings`:

    ```sql
    SELECT
        id,
        name,
        description,
        summary
    FROM listings
    LIMIT 5;
    ```

## Gerar um resumo de IA das avaliações de uma listagem

Para o aplicativo da Margie's Travel, exibir um resumo de todas as avaliações de uma propriedade ajuda os usuários a avaliar rapidamente a essência geral das avaliações.

1. Execute a seguinte consulta, que combina todas as revisões de uma listagem em uma única cadeia de caracteres e, em seguida, gera um resumo abstrato sobre essa cadeia de caracteres:

    ```sql
    SELECT unnest(azure_cognitive.summarize_abstractive(reviews_combined, 'en')) AS review_summary
    FROM (
        -- Combine all reviews for a listing
        SELECT string_agg(comments, ' ') AS reviews_combined
        FROM reviews
        WHERE listing_id = 1
    );
    ```

## Limpar

Depois de concluir este exercício, exclua os recursos do Azure que você criou. Você é cobrado pela capacidade configurada, não por quanto do banco de dados é utilizado. Siga estas instruções para excluir seu grupo de recursos e todos os recursos que você criou para este laboratório.

1. Abra um navegador da Web e navegue até o [portal do Azure](https://portal.azure.com/) e, na home page, selecione **Grupos de recursos** em Serviços do Azure.

    ![Captura de tela de grupos de recursos realçados por uma caixa vermelha em Serviços do Azure no portal do Azure.](media/15-azure-portal-home-azure-services-resource-groups.png)

2. Na caixa de pesquisa de filtro para qualquer campo, insira o nome do grupo de recursos que você criou para este laboratório e selecione seu grupo de recursos na lista.

3. Na página **Visão geral** do grupo de recursos, selecione **Excluir grupo de recursos**.

    ![Captura de tela da folha Visão geral do grupo de recursos com o botão Excluir grupo de recursos realçado por uma caixa vermelha.](media/15-resource-group-delete.png)

4. Na caixa de diálogo de confirmação, insira o nome do seu grupo de recursos que deseja excluir e selecione **Excluir**.
