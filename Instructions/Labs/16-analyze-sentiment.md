---
lab:
  title: Analisar sentimento
  module: Perform Sentiment Analysis and Opinion Mining using Azure Database for PostgreSQL
---

# Analisar sentimento

Como parte do aplicativo com inteligência artificial que você está criando para a Margie's Travel, você deve fornecer aos usuários informações sobre o sentimento de avaliações individuais e o sentimento geral de todas as avaliações de um determinado imóvel alugado. Para realizar isso, use a extensão `azure_ai` em um servidor flexível do Banco de Dados do Azure para PostgreSQL para integrar a funcionalidade de análise de sentimento ao seu banco de dados.

## Antes de começar

Você precisa de uma [assinatura do Azure](https://azure.microsoft.com/free) com direitos administrativos e deve ter permissão de acesso ao OpenAI do Azure nessa assinatura. Caso precise ter acesso ao OpenAI do Azure, solicite-o na página [Acesso limitado do OpenAI do Azure](https://learn.microsoft.com/legal/cognitive-services/openai/limited-access).

### Implantar recursos na assinatura do Azure

Esta etapa orienta você no uso de comandos da CLI do Azure do Azure Cloud Shell para criar um grupo de recursos e executar um script Bicep para implantar os serviços do Azure necessários para concluir este exercício em sua assinatura do Azure.

1. Abra um navegador da Web e acesse o [portal do Azure](https://portal.azure.com/).

2. SElecione o ícone do **Cloud Shell** na barra de ferramentas do portal do Azure para abrir um novo painel do [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) na parte inferior da janela do navegador.

    ![Captura de tela da barra de ferramentas do Azure com o ícone do Cloud Shell realçado por uma caixa vermelha.](media/11-portal-toolbar-cloud-shell.png)

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

## Conectar o banco de dados usando o psql no Azure Cloud Shell

Nesta tarefa, você se conecta ao banco de dados `rentals` no servidor do Banco de Dados do Azure para PostgreSQL usando o [utilitário de linha de comando psql](https://www.postgresql.org/docs/current/app-psql.html) do [Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview).

1. No [portal do Azure](https://portal.azure.com/), navegue até a instância recém-criada do servidor flexível do Banco de Dados do Azure para PostgreSQL.

2. No menu de recursos, em **Configurações**, selecione **Bancos de dados** e selecione **Conectar** para o banco de dados`rentals`.

    ![Captura de tela da página de banco de dados do Banco de Dados do Azure para PostfreSQL. Os bancos de dados e o Connect para o banco de dados de aluguéis são destacados por caixas vermelhas.](media/17-postgresql-rentals-database-connect.png)

3. No prompt "Senha para o usuário pgAdmin" no Cloud Shell, insira a senha gerada aleatoriamente para o logon do **pgAdmin**.

    Uma vez conectado, o prompt `psql` do banco de dados `rentals` é exibido.

## Preencher o banco de dados com alguns dados de exemplo

Antes de analisar o sentimento das avaliações de imóveis alugados usando a extensão `azure_ai`, você deve adicionar dados de amostra ao seu banco de dados. Adicione uma tabela ao banco de dados `rentals` e preencha-a com avaliações de clientes para que você tenha dados para executar a análise de sentimento.

1. Execute o seguinte comando para criar uma tabela nomeada `reviews` para armazenar avaliações de propriedades enviadas por clientes:

    ```sql
    DROP TABLE IF EXISTS reviews;

    CREATE TABLE reviews (
        id int,
        listing_id int, 
        date date,
        comments text
    );
    ```

2. Em seguida, use o comando `COPY` para preencher a tabela com dados de um arquivo CSV. Execute o comando abaixo para carregar as avaliações dos clientes na tabela `reviews`:

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

As integrações de serviços de IA do Azure incluídas no esquema `azure_cognitive` da extensão `azure_ai` fornecem um conjunto avançado de recursos da Linguagem de IA do Azure acessíveis diretamente do banco de dados. Os recursos de análise de sentimento são habilitados por meio do [serviço de linguagem de IA do Azure](https://learn.microsoft.com/azure/ai-services/language-service/overview).

1. Para fazer chamadas com êxito no serviço de linguagem de IA do Azure usando a extensão `azure_ai`, você deve fornecer o ponto de extremidade do serviço e uma chave da extensão. Usando a mesma guia do navegador em que o Cloud Shell está aberto, navegue até o recurso de serviço de linguagem no [portal do Azure](https://portal.azure.com/) e selecione o item **Chaves e ponto de extremidade** em **Gerenciamento de recursos** no menu de navegação à esquerda.

    ![Captura de tela da página Chaves e pontos de extremidade do serviço de linguagem do Azure, com os botões de cópia CHAVE 1 e Ponto de extremidade realçados por caixas vermelhas.](media/16-azure-language-service-keys-endpoints.png)

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

## Examinar os recursos de análise de sentimento da extensão

Nesta tarefa, a função `azure_cognitive.analyze_sentiment()` é usada para avaliar revisões de listagens de imóveis para locação.

1. Para o restante deste exercício, você trabalhará exclusivamente no Cloud Shell, portanto, pode ser útil expandir o painel na janela do navegador selecionando o botão **Maximizar** no canto superior direito do painel do Cloud Shell.

    ![Captura de tela do painel do Azure Cloud Shell com o botão Maximizar realçado por uma caixa vermelha.](media/16-azure-cloud-shell-pane-maximize.png)

2. Ao trabalhar com `psql` no Cloud Shell, habilitar a exibição estendida para resultados de consulta pode ser útil, pois melhora a legibilidade da saída para comandos subsequentes. Execute o seguinte comando para permitir que a exibição estendida seja aplicada automaticamente.

    ```sql
    \x auto
    ```

3. Os recursos de análise de sentimento da extensão `azure_ai` podem ser acessados no esquema `azure_cognitive`. Você usa a função `analyze_sentiment()`. Use o [metacomando `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) para examinar a função ao executar:

    ```sql
    \df azure_cognitive.analyze_sentiment
    ```

    A saída do metacomando mostra o esquema, o nome, o tipo de dados de resultado e os argumentos da função. Essas informações ajudam você a entender como interagir com a função de suas consultas.

    A saída mostra três sobrecargas da função `analyze_sentiment()`, permitindo que você verifique as diferenças. A propriedade `Argument data types` na saída revela a lista de argumentos que as sobrecargas das três funções esperam:

    | Argumento | Tipo | Padrão | Descrição |
    | -------- | ---- | ------- | ----------- |
    | text | `text` ou `text[]` || O(s) texto(s) para o(s) qual(is) sentimento deve ser analisado. |
    | language_text | `text` ou `text[]` || Código de linguagem (ou matriz de códigos de linguagem) que representa o idioma do texto a ser analizado em buca do sentimento. Revise a [lista de idiomas com suporte](https://learn.microsoft.com/azure/ai-services/language-service/sentiment-opinion-mining/language-support) para obter os códigos de idioma necessários. |
    | batch_size | `integer` | 10 | Somente para as duas sobrecargas que esperam uma entrada de `text[]`. Especifica o número de registros a serem processados por vez. |
    | disable_service_logs | `boolean` | false | Sinalizador que indica se os logs de serviço devem ser desativados. |
    | timeout_ms | `integer` | NULO | Tempo limite em milissegundos após o qual a operação é interrompida. |
    | throw_on_error | `boolean` | true | Sinalizador que indica se a função deve, em caso de erro, gerar uma exceção que resulte em uma reversão das transações de encapsulamento. |
    | max_attempts | `integer` | 1 | Número de novas tentativas de chamar os Serviços de IA do Azure em caso de falha. |
    | retry_delay_ms | `integer` | 1000 | Tempo, em milissegundos, para aguardar antes de tentar chamar novamente o ponto de extremidade dos Serviços de IA do Azure. |

4. Também é imperativo entender a estrutura do tipo de dados que uma função retorna para que você possa manipular corretamente a saída em suas consultas. Execute o seguinte comando para inspecionar o tipo `sentiment_analysis_result`:

    ```sql
    \dT+ azure_cognitive.sentiment_analysis_result
    ```

5. A saída do comando acima revela que o tipo `sentiment_analysis_result` é um `tuple`. Você pode se aprofundar na estrutura do `tuple` executando o seguinte comando para examinar as colunas contidas no tipo `sentiment_analysis_result`:

    ```sql
    \d+ azure_cognitive.sentiment_analysis_result
    ```

    A saída do comando deve ser semelhante ao seguinte:

    ```sql
                     Composite type "azure_cognitive.sentiment_analysis_result"
         Column     |     Type         | Collation | Nullable | Default | Storage  | Description 
    ----------------+------------------+-----------+----------+---------+----------+-------------
     sentiment      | text             |           |          |         | extended | 
     positive_score | double precision |           |          |         | plain    | 
     neutral_score  | double precision |           |          |         | plain    | 
     negative_score | double precision |           |          |         | plain    |
    ```

    O tipo `azure_cognitive.sentiment_analysis_result` é um composto contendo as previsões de sentimento do texto de entrada. Inclui o sentimento — que pode ser positivo, negativo, neutro ou misto —, e as pontuações para os aspectos positivos, neutros e negativos encontrados no texto. As pontuações são representadas por números reais entre 0 e 1. Por exemplo, em (neutro: 0,26, 0,64, 0,09), o sentimento é neutro, com uma pontuação positiva de 0,26, neutra de 0,64 e negativa de 0,09.

## Analisar o sentimento das avaliações

1. Agora que você revisou a função `analyze_sentiment()` e o `sentiment_analysis_result` retorna, vamos colocá-la em uso. Execute a seguinte consulta simples, que executa a análise de sentimento em alguns comentários na tabela `reviews`:

    ```sql
    SELECT
        id,
        azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment
    FROM reviews
    WHERE id <= 10
    ORDER BY id;
    ```

    Nos dois registros analisados, observe os valores `sentiment` na saída, `(mixed,0.71,0.09,0.2)` e `(positive,0.99,0.01,0)`. Eles representam o `sentiment_analysis_result` retornado pela função `analyze_sentiment()` na consulta acima. A análise foi realizada sobre o campo `comments` da tabela `reviews`.

    > [!Note]
    >
    > O uso da função `analyze_sentiment()` embutida permite analisar rapidamente o sentimento do texto em suas consultas. Embora isso funcione bem para um pequeno número de registros, pode não ser ideal para analisar o sentimento de um grande número de registros ou atualizar todos os registros em uma tabela que pode conter dezenas de milhares de revisões ou mais.

2. Outra abordagem que pode ser útil para revisões mais longas é analisar o sentimento de cada frase dentro dela. Para fazer isso, use a sobrecarga da função `analyze_sentiment()`, que aceita uma matriz de texto.

    ```sql
    SELECT
        azure_cognitive.analyze_sentiment(ARRAY_REMOVE(STRING_TO_ARRAY(comments, '.'), ''), 'en') AS sentence_sentiments
    FROM reviews
    WHERE id = 1;
    ```

    Na consulta acima, você usou a função `STRING_TO_ARRAY` do PostgreSQL. Além disso, a função `ARRAY_REMOVE` foi usada para remover quaisquer elementos de matriz que sejam cadeias de caracteres vazias, pois eles causarão erros com a função `analyze_sentiment()`.

    A saída da consulta permite que você obtenha uma melhor compreensão do sentimento `mixed` atribuído à revisão geral. As frases são uma mistura de sentimentos positivos, neutros e negativos.

3. As duas consultas anteriores retornaram o `sentiment_analysis_result` diretamente da consulta. No entanto, você provavelmente deverá recuperar os valores subjacentes dentro do `sentiment_analysis_result``tuple`. Execute a seguinte consulta que procura revisões extremamente positivas e extrai os componentes de sentimento em campos individuais:

    ```sql
    WITH cte AS (
        SELECT id, comments, azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment FROM reviews
    )
    SELECT
        id,
        (sentiment).sentiment,
        (sentiment).positive_score,
        (sentiment).neutral_score,
        (sentiment).negative_score,
        comments
    FROM cte
    WHERE (sentiment).positive_score > 0.98
    LIMIT 5;
    ```

    A consulta acima usa uma expressão de tabela comum ou CTE para obter as pontuações de sentimento para todos os registros na tabela `reviews`. Em seguida, ela seleciona as colunas `sentiment` do tipo composto `sentiment_analysis_result` retornadas pela CTE para extrair os valores individuais de `tuple.`

## Armazenar sentimento na tabela de avaliações

Para o sistema de recomendação de imóveis para aluguel que você está criando para a Margie's Travel, você deseja armazenar classificações de sentimento no banco de dados para não precisar fazer chamadas e incorrer em custos sempre que avaliações de sentimento forem solicitadas. Realizar análises de sentimento em tempo real pode ser ótimo para um pequeno número de registros ou analisar dados quase em tempo real. Ainda assim, adicionar os dados de sentimento ao banco de dados para uso em seu aplicativo faz sentido para suas revisões armazenadas. Para fazer isso, você deve alterar a tabela `reviews` para adicionar colunas para armazenar a avaliação de sentimento e as pontuações positivas, neutras e negativas.

1. Execute a seguinte consulta para atualizar a tabela `reviews` para que ela possa armazenar detalhes de sentimento:

    ```sql
    ALTER TABLE reviews
    ADD COLUMN sentiment varchar(10),
    ADD COLUMN positive_score numeric,
    ADD COLUMN neutral_score numeric,
    ADD COLUMN negative_score numeric;
    ```

2. Em seguida, você deve atualizar os registros existentes na tabela `reviews` com seu valor de sentimento e pontuações associadas.

    ```sql
    WITH cte AS (
        SELECT id, azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment FROM reviews
    )
    UPDATE reviews AS r
    SET
        sentiment = (cte.sentiment).sentiment,
        positive_score = (cte.sentiment).positive_score,
        neutral_score = (cte.sentiment).neutral_score,
        negative_score = (cte.sentiment).negative_score
    FROM cte
    WHERE r.id = cte.id;
    ```

    A execução dessa consulta leva muito tempo porque os comentários de cada revisão na tabela são enviados individualmente para o ponto de extremidade do serviço de linguagem para análise. O envio de registros em lotes é mais eficiente ao lidar com muitos registros.

3. Vamos executar a consulta abaixo para executar a mesma ação de atualização, mas desta vez enviar comentários da tabela `reviews` em lotes de 10 (esse é o tamanho máximo de lote permitido) e avaliar a diferença de desempenho.

    ```sql
    WITH cte AS (
        SELECT azure_cognitive.analyze_sentiment(ARRAY(SELECT comments FROM reviews ORDER BY id), 'en', batch_size => 10) as sentiments
    ),
    sentiment_cte AS (
        SELECT
            ROW_NUMBER() OVER () AS id,
            sentiments AS sentiment
        FROM cte
    )
    UPDATE reviews AS r
    SET
        sentiment = (sentiment_cte.sentiment).sentiment,
        positive_score = (sentiment_cte.sentiment).positive_score,
        neutral_score = (sentiment_cte.sentiment).neutral_score,
        negative_score = (sentiment_cte.sentiment).negative_score
    FROM sentiment_cte
    WHERE r.id = sentiment_cte.id;
    ```

    Embora essa consulta seja um pouco mais complexa, usando dois CTEs, o desempenho é muito melhor. Nessa consulta, a primeira CTE analisa o sentimento de lotes de comentários de revisão e a segunda extrai a tabela resultante de `sentiment_analysis_results` em uma nova tabela contendo um `id` com base na posição ordinal e ”sentiment_analysis_result” para cada linha. A segunda CTE pode ser usada na instrução de atualização para gravar os valores no banco de dados.

4. Em seguida, execute uma consulta para observar os resultados da atualização, procurando avaliações com um sentimento **negativo**, começando com a mais negativa primeiro.

    ```sql
    SELECT
        id,
        negative_score,
        comments
    FROM reviews
    WHERE sentiment = 'negative'
    ORDER BY negative_score DESC;
    ```

## Limpar

Depois de concluir este exercício, exclua os recursos do Azure que você criou. Você é cobrado pela capacidade configurada, não por quanto do banco de dados é utilizado. Siga estas instruções para excluir seu grupo de recursos e todos os recursos que você criou para este laboratório.

1. Abra um navegador da Web e navegue até o [portal do Azure](https://portal.azure.com/) e, na home page, selecione **Grupos de recursos** em Serviços do Azure.

    ![Captura de tela de grupos de recursos realçados por uma caixa vermelha em Serviços do Azure no portal do Azure.](media/16-azure-portal-home-azure-services-resource-groups.png)

2. Na caixa de pesquisa de filtro para qualquer campo, insira o nome do grupo de recursos que você criou para este laboratório e selecione seu grupo de recursos na lista.

3. Na página **Visão geral** do grupo de recursos, selecione **Excluir grupo de recursos**.

    ![Captura de tela da folha Visão geral do grupo de recursos com o botão Excluir grupo de recursos realçado por uma caixa vermelha.](media/16-resource-group-delete.png)

4. Na caixa de diálogo de confirmação, insira o nome do seu grupo de recursos que deseja excluir e selecione **Excluir**.
