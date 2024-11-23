---
lab:
  title: 'Explorar a extensão de IA do Azure '
  module: Explore Generative AI with Azure Database for PostgreSQL
---

# Explorar a extensão de IA do Azure 

Como desenvolvedor líder da Margie's Travel, você foi encarregado de criar um aplicativo com inteligência artificial para fornecer aos seus clientes recomendações inteligentes sobre propriedades para aluguel. Você deseja saber mais sobre a extensão `azure_ai` do Banco de Dados do Azure para PostgreSQL e como ela pode ajudá-lo a integrar o poder da GenAI (IA generativa) em seu aplicativo. Neste exercício, você explorará a extensão `azure_ai` e sua funcionalidade ao instala-la em um banco de dados de servidor flexível do Banco de Dados do Azure para PostgreSQL e explorar seus recursos para integrar os serviços IA do Azure e ML.

## Antes de começar

Você precisa de uma [assinatura do Azure](https://azure.microsoft.com/free) com direitos administrativos e deve ter permissão de acesso ao OpenAI do Azure nessa assinatura. Caso precise ter acesso ao OpenAI do Azure, solicite-o na página [Acesso limitado do OpenAI do Azure](https://learn.microsoft.com/legal/cognitive-services/openai/limited-access).

### Implantar recursos na assinatura do Azure

Esta etapa orienta você no uso de comandos da CLI do Azure do Azure Cloud Shell para criar um grupo de recursos e executar um script Bicep para implantar os serviços do Azure necessários para concluir este exercício em sua assinatura do Azure.

1. Abra um navegador da Web e acesse o [portal do Azure](https://portal.azure.com/).

2. SElecione o ícone do **Cloud Shell** na barra de ferramentas do portal do Azure para abrir um novo painel do [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) na parte inferior da janela do navegador.

    ![Captura de tela da barra de ferramentas do Azure com o ícone do Cloud Shell realçado por uma caixa vermelha.](media/12-portal-toolbar-cloud-shell.png)

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

    O comando final gera aleatoriamente uma senha para o login de administrador do PostgreSQL. Copie-o para um local seguro para usar mais tarde ao se conectar ao seu servidor flexível PostgreSQL.

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

1. No [portal do Azure](https://portal.azure.com/), navegue até o servidor flexível do Banco de Dados do Azure para PostgreSQL.

2. No menu de recursos, em **Configurações**, selecione **Bancos de dados** e selecione **Conectar** para o banco de dados`rentals`.

    ![Captura de tela da página de banco de dados do Banco de Dados do Azure para PostfreSQL. Os bancos de dados e o Connect para o banco de dados de aluguéis são destacados por caixas vermelhas.](media/12-postgresql-rentals-database-connect.png)

3. No prompt "Senha para o usuário pgAdmin" no Cloud Shell, insira a senha gerada aleatoriamente para o logon do **pgAdmin**.

    Uma vez conectado, o prompt `psql` do banco de dados `rentals` é exibido.

4. Durante o restante deste exercício, você continuará trabalhando no Cloud Shell, portanto, pode ser útil expandir o painel na janela do navegador selecionando o botão **Maximizar** no canto superior direito do painel.

    ![Captura de tela do painel do Azure Cloud Shell com o botão Maximizar realçado por uma caixa vermelha.](media/12-azure-cloud-shell-pane-maximize.png)

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

## Revisar os objetos contidos na extensão `azure_ai`

Revisar os objetos contidos na extensão `azure_ai` pode fornecer uma melhor compreensão de seus recursos. Nesta tarefa, você inspecionará os vários esquemas, UDFs (funções definidas pelo usuário) e tipos compostos adicionados ao banco de dados pela extensão.

1. Ao trabalhar com `psql` no Cloud Shell, habilitar a exibição estendida para resultados de consulta pode ser útil, pois melhora a legibilidade da saída para comandos subsequentes. Execute o seguinte comando para permitir que a exibição estendida seja aplicada automaticamente.

    ```sql
    \x auto
    ```

2. O [metacomando `\dx`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DX-LC) é usado para listar objetos contidos em uma extensão. Execute o seguinte no prompt de comando `psql` para exibir os objetos na extensão `azure_ai`. Pode ser necessário pressionar a barra de espaço para visualizar a lista completa de objetos.

    ```psql
    \dx+ azure_ai
    ```

    A saída do metacomando mostra que a extensão `azure_ai` cria quatro esquemas, várias UDFs (funções definidas pelo usuário), vários tipos de composição no banco de dados e a tabela `azure_ai.settings`. Além dos esquemas, todos os nomes de objeto são precedidos pelo esquema ao qual pertencem. Os esquemas são usados para agrupar funções e tipos relacionados que a extensão adiciona aos buckets. A tabela a seguir lista os esquemas adicionados pela extensão e fornece uma breve descrição de cada um.

    | Esquema      | Descrição                                              |
    | ----------------- | ------------------------------------------------------------------------------------------------------ |
    | `azure_ai`    | O esquema principal onde reside a tabela de configuração e as UDFs que interagem com a extensão. |
    | `azure_openai`  | Contém as UDFs que habilitam a chamada de um ponto de extremidade do OpenAI do Azure.                    |
    | `azure_cognitive` | Fornece as UDFs e os tipos de composição relacionados à integração do banco de dados dos Serviços de IA do Azure.     |
    | `azure_ml`    | Inclui as UDFs para integrar os serviços do Azure Machine Learning (ML).                |

### Explorar o esquema da IA do Azure

O esquema `azure_ai` fornece a estrutura para interagir diretamente com os serviços de ML e IA do Azure do banco de dados. Ele contém funções para configurar conexões com esses serviços e recuperá-los da tabela `settings` que também está hospedada no mesmo esquema.  A tabela `settings` fornece armazenamento seguro no banco de dados para pontos de extremidade e chaves associadas aos serviços de ML e IA do Azure.

1. Para examinar as funções definidas no esquema, use o [metacomando `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC), especificando o esquema cujas funções devem ser exibidas. Execute o seguinte para exibir as funções no esquema `azure_ai`:

    ```sql
    \df azure_ai.*
    ```

    A saída do comando deve ser uma tabela semelhante a seguinte:

    ```sql
                  List of functions
     Schema |  Name  | Result data type | Argument data types | Type 
    ----------+-------------+------------------+----------------------+------
     azure_ai | get_setting | text      | key text      | func
     azure_ai | set_setting | void      | key text, value text | func
     azure_ai | version  | text      |           | func
    ```

    A função `set_setting()` permite definir o ponto de extremidade e a chave dos serviços de ML e IA do Azure para que a extensão possa se conectar a eles. Ela aceita uma **chave** e o **valor** para atribuir a ela. A função `azure_ai.get_setting()` fornece uma maneira de recuperar os valores definidos com a função `set_setting()`. Ela aceita a **chave** da configuração que você deseja visualizar e retorna o valor atribuído a ela. Para ambos os métodos, a chave deve ser uma das seguintes:

    | Key | Descrição |
    | --- | ----------- |
    | `azure_openai.endpoint` | Um ponto de extremidade OpenAI com suporte (por exemplo, <https://example.openai.azure.com>). |
    | `azure_openai.subscription_key` | Uma chave de assinatura para um recurso do OpenAI do Azure. |
    | `azure_cognitive.endpoint` | Um ponto de extremidade dos Serviços de IA do Azure com suporte (por exemplo, <https://example.cognitiveservices.azure.com>). |
    | `azure_cognitive.subscription_key` | Uma chave de assinatura para um recurso dos Serviços de IA do Azure. |
    | `azure_ml.scoring_endpoint` | Um ponto de extremidade de pontuação do Azure ML com suporte (por exemplo, <https://example.eastus2.inference.ml.azure.com/score>) |
    | `azure_ml.endpoint_key` | Uma chave de ponto de extremidade para uma implantação do Azure ML. |

    > Importante
    >
    > Como as informações de conexão dos serviços de IA do Azure, incluindo as chaves de API, são armazenadas em uma tabela de configuração no banco de dados, a extensão `azure_ai` define uma função chamada `azure_ai_settings_manager` para garantir que essas informações sejam protegidas e acessíveis somente aos usuários atribuídos à função. Essa função permite a leitura e gravação de configurações relacionadas à extensão. Somente os membros da função `azure_ai_settings_manager` podem invocar as funções `azure_ai.get_setting()` e `azure_ai.set_setting()`. Em um servidor flexível do Banco de Dados do Azure para PostgreSQL, todos os usuários administradores (com a função `azure_pg_admin` atribuída) também recebem a função `azure_ai_settings_manager`.

2. Para demonstrar como você usa as funções `azure_ai.set_setting()` e `azure_ai.get_setting()`, configure a conexão com sua conta OpenAI do Azure. Usando a mesma guia do navegador em que o Cloud Shell está aberto, minimize ou restaure o painel do Cloud Shell e navegue até o recurso OpenAI do Azure no [portal do Azure](https://portal.azure.com/). Quando estiver na página de recursos do OpenAI do Azure, no menu de recursos, na seção **Gerenciamento de Recursos**, selecione **Chaves e Ponto de Extremidade** e copie o ponto de extremidade e uma das chaves disponíveis.

    ![Captura de tela da página Chaves e Pontos de Extremidade do serviço OpenAI do Azure sendo exibida, com os botões de cópia KEY 1 e Ponto de Extremidade realçados por caixas vermelhas.](media/12-azure-openai-keys-and-endpoints.png)

    Você pode usar `KEY 1` ou `KEY 2`. Ter sempre duas chaves permite girar e regenerar chaves com segurança, sem causar interrupção de serviço.

3. Depois de ter o ponto de extremidade e a chave, maximize o painel do Cloud Shell novamente e use os comandos abaixo para adicionar seus valores à tabela de configuração. Certifique-se de substituir os tokens `{endpoint}` e `{api-key}` pelos valores copiados do portal do Azure.

    ```sql
    SELECT azure_ai.set_setting('azure_openai.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_openai.subscription_key', '{api-key}');
    ```

4. Você pode verificar as configurações gravadas na tabela `azure_ai.settings` usando a função `azure_ai.get_setting()` nas seguintes consultas:

    ```sql
    SELECT azure_ai.get_setting('azure_openai.endpoint');
    SELECT azure_ai.get_setting('azure_openai.subscription_key');
    ```

    A extensão `azure_ai` agora está conectada à sua conta do OpenAI do Azure.

### Examinar o esquema do OpenAI do Azure

O esquema `azure_openai` fornece a capacidade de integrar a criação de inserção vetorial de valores de texto em seu banco de dados usando o OpenAI do Azure. Usando esse esquema, você pode [gerar inserções com o OpenAI do Azure](https://learn.microsoft.com/azure/ai-services/openai/how-to/embeddings) diretamente do banco de dados para criar representações vetoriais de texto de entrada, que podem ser usadas em pesquisas de similaridade de vetor e consumidas por modelos de aprendizado de máquina. O esquema contém uma única função, `create_embeddings()`, com duas sobrecargas. Uma sobrecarga aceita uma única cadeia de caracteres de entrada e a outra espera uma matriz de cadeias de caracteres de entrada.

1. Como você fez acima, você pode usar o [ lmetacomando `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) para visualizar os detalhes das funções no esquema `azure_openai`:

    ```sql
    \df azure_openai.*
    ```

    A saída mostra as duas sobrecargas da função `azure_openai.create_embeddings()`, permitindo que você examine as diferenças entre as duas versões da função e os tipos que elas retornam. A propriedade `Argument data types` na saída revela a lista de argumentos que a sobrecarga da função espera:

    | Argumento    | Tipo       | Padrão | Descrição                                                          |
    | --------------- | ------------------ | ------- | ------------------------------------------------------------------------------------------------------------------------------ |
    | deployment_name | `text`      |    | Nome da implantação no estúdio do OpenAI do Azure que contém o modelo `text-embedding-ada-002`.               |
    | input     | `text` ou `text[]` |    | Texto de entrada (ou matriz de texto de entrada) para o qual as inserções são criadas.                                |
    | batch_size   | `integer`     | 100  | Somente para as sobrecargas que esperam uma entrada de `text[]`. Especifica o número de registros a serem processados por vez.          |
    | timeout_ms   | `integer`     | 3600000 | Tempo limite em milissegundos após o qual a operação é interrompida.                                 |
    | throw_on_error | `boolean`     | true  | Sinalizador que indica se a função deve, em caso de erro, gerar uma exceção que resulte em uma reversão das transações de encapsulamento. |
    | max_attempts  | `integer`     | 1   | Número de novas tentativas de chamar o serviço Azure OpenAI em caso de falha.                     |
    | retry_delay_ms | `integer`     | 1000  | Tempo, em milissegundos, para aguardar antes de tentar chamar novamente o ponto de extremidade de serviço do Azure OpenAI.        |

2. Para fornecer um exemplo simplificado de uso da função, execute a consulta a seguir, que cria uma inserção de vetor para o campo `description` na tabela `listings`. O parâmetro `deployment_name` na função é definido como `embedding`, que é o nome da implantação do modelo `text-embedding-ada-002` no serviço OpenAI do Azure (ele foi criado com esse nome pelo script de implantação do Bicep):

    ```sql
    SELECT
      id,
      name,
      azure_openai.create_embeddings('embedding', description) AS vector
    FROM listings
    LIMIT 1;
    ```

    A saída deve ser semelhante a esta:

    ```sql
     id |      name       |              vector
    ----+-------------------------------+------------------------------------------------------------
      1 | Stylish One-Bedroom Apartment | {0.020068742,0.00022734122,0.0018286322,-0.0064167166,...}
    ```

    Para resumir, as incorporações vetoriais são abreviadas na saída acima.

    As [inserções](https://learn.microsoft.com/azure/postgresql/flexible-server/generative-ai-overview#embeddings) são um conceito em aprendizado de máquina e NLP (processamento de linguagem natural) que envolvem a representação de objetos, como palavras, documentos ou entidades, como [vetores](https://learn.microsoft.com/azure/postgresql/flexible-server/generative-ai-overview#vectors) em um espaço multidimensional. As inserções permitem que os modelos de aprendizado de máquina avaliem o quão estreitamente as informações estão relacionadas entre si. Essa técnica permite a identificação eficiente de relações e semelhanças entre dados, permitindo que algoritmos identifiquem padrões e façam previsões precisas.

    A extensão `azure_ai` permite que você gere inserções para o texto de entrada. Para permitir que os vetores gerados sejam armazenados junto com o restante dos dados no banco de dados, você deve instalar a extensão `vector` seguindo as diretrizes na documentação [habilitar suporte de vetor do banco de dados](https://learn.microsoft.com/azure/postgresql/flexible-server/how-to-use-pgvector#enable-extension). No entanto, isso está fora do escopo deste exercício.

### Examinar o esquema azure_cognitive

O esquema `azure_cognitive` fornece a estrutura para interagir diretamente com os Serviços de IA do Azure do banco de dados. As integrações de serviços de IA do Azure incluídas no esquema fornecem um conjunto avançado de recursos da Linguagem de IA do Azure acessíveis diretamente do banco de dados. As funcionalidades incluem análise de sentimento, detecção de idioma, extração de frases-chave, reconhecimento de entidade, resumo de texto e tradução. O acesso a esses recursos é habilitado por meio do [serviço de Linguagem de IA do Azure](https://learn.microsoft.com/azure/ai-services/language-service/overview).

1. Para revisar todas as funções definidas em um esquema, você pode usar o [metacomando `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) como fez anteriormente. Para exibir as funções no esquema `azure_cognitive`, execute:

    ```sql
    \df azure_cognitive.*
    ```

2. Existem inúmeras funções definidas neste esquema, portanto, a saída do [metacomando `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) pode ser difícil de ler, por isso é melhor dividi-la em partes menores. Execute o seguinte para examinar apenas a função `analyze_sentiment()`:

    ```sql
    \df azure_cognitive.analyze_sentiment
    ```

    Na saída, observe que a função tem três sobrecargas, com uma aceitando uma única cadeia de caracteres de entrada e as outras duas esperando matrizes de texto. A saída mostra o esquema, o nome, o tipo de dados de resultado e os tipos de dados de argumento da função. Essas informações podem ajudá-lo a entender como usar a função.

3. Repita o comando acima, substituindo o nome da função `analyze_sentiment` por cada um dos seguintes nomes de função, para inspecionar todas as funções disponíveis no esquema:

   - `detect_language`
   - `extract_key_phrases`
   - `linked_entities`
   - `recognize_entities`
   - `recognize_pii_entities`
   - `summarize_abstractive`
   - `summarize_extractive`
   - `translate`

    Para cada função, inspecione as várias formas da função e suas entradas esperadas e tipos de dados resultantes.

4. Além das funções, o esquema `azure_cognitive` também contém vários tipos compostos usados como tipos de dados de retorno das várias funções. É imperativo entender a estrutura do tipo de dados que uma função retorna para que você possa lidar corretamente com a saída em suas consultas. Por exemplo, execute o seguinte comando para inspecionar o tipo `sentiment_analysis_result`:

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
       Column  |   Type   | Collation | Nullable | Default | Storage | Description 
    ----------------+------------------+-----------+----------+---------+----------+-------------
     sentiment   | text      |     |     |    | extended | 
     positive_score | double precision |     |     |    | plain  | 
     neutral_score | double precision |     |     |    | plain  | 
     negative_score | double precision |     |     |    | plain  |
    ```

    O tipo `azure_cognitive.sentiment_analysis_result` é um composto contendo as previsões de sentimento do texto de entrada. Inclui o sentimento — que pode ser positivo, negativo, neutro ou misto —, e as pontuações para os aspectos positivos, neutros e negativos encontrados no texto. As pontuações são representadas por números reais entre 0 e 1. Por exemplo, em (neutro: 0,26, 0,64, 0,09), o sentimento é neutro, com uma pontuação positiva de 0,26, neutra de 0,64 e negativa de 0,09.

6. Assim como acontece com as funções `azure_openai`, para fazer chamadas com êxito nos serviços de IA do Azure usando a extensão `azure_ai`, você deve fornecer o ponto de extremidade e uma chave para o serviço de Linguagem de IA do Azure. Usando a mesma guia do navegador em que o Cloud Shell está aberto, minimize ou restaure o painel do Cloud Shell e navegue até o recurso de serviço de linguagem no [portal do Azure](https://portal.azure.com/). No menu de recursos, em **Gerenciamento de Recursos**, selecione **Chaves e Ponto de extremidade**.

    ![Captura de tela da página Chaves e pontos de extremidade do serviço de linguagem do Azure, com os botões de cópia CHAVE 1 e Ponto de extremidade realçados por caixas vermelhas.](media/12-azure-language-service-keys-and-endpoints.png)

7. Copie os valores de ponto de extremidade e chave de acesso e substitua os tokens `{endpoint}` e `{api-key}` pelos valores copiados do portal do Azure. Maximize o Cloud Shell novamente e execute os comandos no prompt de comando `psql` no Cloud Shell para adicionar seus valores à tabela de configuração.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '{api-key}');
    ```

8. Agora, execute a seguinte consulta para analisar o sentimento de algumas avaliações:

    ```sql
    SELECT
      id,
      comments,
      azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment
    FROM reviews
    WHERE id IN (1, 3);
    ```

    Observe os valores `sentiment` na saída `(mixed,0.71,0.09,0.2)` e `(positive,0.99,0.01,0)`. Eles representam o `sentiment_analysis_result` retornado pela função `analyze_sentiment()` na consulta acima. A análise foi realizada sobre o campo `comments` da tabela `reviews`.

## Inspecionar o esquema do Azure ML

O esquema `azure_ml` permite que as funções se conectem aos serviços do Azure ML diretamente do banco de dados.

1. Para revisar as funções definidas em um esquema, você pode usar o [metacomando `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC). Para exibir as funções no esquema `azure_ml`, execute:

    ```sql
    \df azure_ml.*
    ```

    Na saída, observe que há duas funções definidas neste esquema `azure_ml.inference()` e `azure_ml.invoke()`, cujos detalhes são exibidos abaixo:

    ```sql
                  List of functions
    -----------------------------------------------------------------------------------------------------------
    Schema       | azure_ml
    Name        | inference
    Result data type  | jsonb
    Argument data types | input_data jsonb, deployment_name text DEFAULT NULL::text, timeout_ms integer DEFAULT NULL::integer, throw_on_error boolean DEFAULT true, max_attempts integer DEFAULT 1, retry_delay_ms integer DEFAULT 1000
    Type        | func
    ```

    A função `inference()` usa um modelo de aprendizado de máquina treinado para fazer previsões ou gerar saídas com base em dados novos e não vistos.

    Ao fornecer um ponto de extremidade e uma chave, você pode se conectar a um ponto de extremidade implantado do Azure ML da mesma forma que se conectou aos pontos de extremidade do OpenAI do Azure e dos Serviços de IA do Azure. A interação com o Azure ML requer ter um modelo treinado e implantado, portanto, está fora do escopo deste exercício e você não está configurando essa conexão para experimentá-lo aqui.

## Limpar

Depois de concluir este exercício, exclua os recursos do Azure que você criou. Você é cobrado pela capacidade configurada, não por quanto do banco de dados é utilizado. Siga estas instruções para excluir seu grupo de recursos e todos os recursos que você criou para este laboratório.

1. Abra um navegador da Web e navegue até o [portal do Azure](https://portal.azure.com/) e, na home page, selecione **Grupos de recursos** em Serviços do Azure.

    ![Captura de tela de grupos de recursos realçados por uma caixa vermelha em Serviços do Azure no portal do Azure.](media/12-azure-portal-home-azure-services-resource-groups.png)

2. Na caixa de pesquisa de filtro para qualquer campo, insira o nome do grupo de recursos que você criou para este laboratório e selecione seu grupo de recursos na lista.

3. Na página **Visão geral** do grupo de recursos, selecione **Excluir grupo de recursos**.

    ![Captura de tela da folha Visão geral do grupo de recursos com o botão Excluir grupo de recursos realçado por uma caixa vermelha.](media/12-resource-group-delete.png)

4. Na caixa de diálogo de confirmação, insira o nome do seu grupo de recursos que deseja excluir e selecione **Excluir**.
