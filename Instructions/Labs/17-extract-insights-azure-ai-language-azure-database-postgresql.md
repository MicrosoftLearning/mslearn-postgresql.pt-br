---
lab:
  title: Extrair insights usando o serviço de Linguagem de IA do Azure
  module: Extract insights using the Azure AI Language service with Azure Database for PostgreSQL
---

# Extrair insights usando o serviço de Linguagem de IA do Azure com o Banco de Dados do Azure para PostgreSQL

Lembre-se de que a empresa de listagens deseja analisar as tendências do mercado, como as frases ou lugares mais populares. A equipe também pretende aprimorar as proteções para PII (informações de identificação pessoal). Os dados atuais estão armazenados em um servidor flexível do Banco de Dados do Azure para PostgreSQL. O orçamento do projeto é pequeno, portanto, minimizar os custos iniciais e os custos contínuos na manutenção de palavras-chave e rótulos é essencial. Os desenvolvedores estão cautelosos com quantas formas as PII podem assumir e preferem uma solução econômica e verificada em vez de uma correspondência interna de expressões regulares.

Você integrará o banco de dados aos serviços de Linguagem de IA do Azure usando a extensão `azure_ai`. A extensão fornece APIs de função SQL definidas pelo usuário para várias APIs do Serviço Cognitivo do Azure, incluindo:

- como extração de frases-chave
- reconhecimento de entidade
- Reconhecimento de PII

Essa abordagem permitirá que a equipe de ciência de dados se junte rapidamente aos dados de popularidade da listagem para determinar as tendências do mercado. Ele também fornecerá aos desenvolvedores de aplicativos um texto seguro para PII para apresentar em situações que não exigem acesso. O armazenamento de entidades identificadas permite a revisão humana em caso de consulta ou de um falso positivo no reconhecimento de PII (pensar que algo é PII, mas que não é).

No final, você terá quatro novas colunas na tabela `listings` com insights extraídos:

- `key_phrases`
- `recognized_entities`
- `pii_safe_description`
- `pii_entities`

## Antes de começar

Você precisa de uma [assinatura do Azure](https://azure.microsoft.com/free) com direitos administrativos e deve ter permissão de acesso ao OpenAI do Azure nessa assinatura. Caso precise ter acesso ao OpenAI do Azure, solicite-o na página [Acesso limitado do OpenAI do Azure](https://learn.microsoft.com/legal/cognitive-services/openai/limited-access).

### Implantar recursos na assinatura do Azure

Esta etapa orienta você no uso de comandos da CLI do Azure do Azure Cloud Shell para criar um grupo de recursos e executar um script Bicep para implantar os serviços do Azure necessários para concluir este exercício em sua assinatura do Azure.

1. Abra um navegador da Web e acesse o [portal do Azure](https://portal.azure.com/).

2. SElecione o ícone do **Cloud Shell** na barra de ferramentas do portal do Azure para abrir um novo painel do [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) na parte inferior da janela do navegador.

    ![Captura de tela da barra de ferramentas do Azure com o ícone do Cloud Shell realçado por uma caixa vermelha.](media/17-portal-toolbar-cloud-shell.png)

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

    ![Captura de tela da página de banco de dados do Banco de Dados do Azure para PostfreSQL. Os bancos de dados e o Connect para o banco de dados de aluguéis são destacados por caixas vermelhas.](media/17-postgresql-rentals-database-connect.png)

3. No prompt "Senha para o usuário pgAdmin" no Cloud Shell, insira a senha gerada aleatoriamente para o logon do **pgAdmin**.

    Uma vez conectado, o prompt `psql` do banco de dados `rentals` é exibido.

4. Durante o restante deste exercício, você continuará trabalhando no Cloud Shell, portanto, pode ser útil expandir o painel na janela do navegador selecionando o botão **Maximizar** no canto superior direito do painel.

    ![Captura de tela do painel do Azure Cloud Shell com o botão Maximizar realçado por uma caixa vermelha.](media/17-azure-cloud-shell-pane-maximize.png)

## Configuração: configurar extensões

Para armazenar e consultar vetores e gerar inserções, você precisa colocar na lista de permitidos e habilitar duas extensões para o servidor flexível do Banco de Dados do Azure para PostgreSQL: `vector` e `azure_ai`.

1. Para permitir ambas as extensões, adicione `vector` e `azure_ai` ao parâmetro do servidor `azure.extensions`, de acordo com as instruções fornecidas em [Como usar extensões do PostgreSQL](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions).

2. Para habilitar a extensão `vector`, execute o seguinte comando SQL. Para instruções detalhadas, consulte [Como habilitar e usar `pgvector` no Banco de Dados do Azure para PostgreSQL - servidor flexível](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-use-pgvector#enable-extension).

    ```sql
    CREATE EXTENSION vector;
    ```

3. Para habilitar a extensão `azure_ai`, execute o seguinte comando SQL. O ponto de extremidade e a chave de API do recurso do ***OpenAI do Azure***. Para obter instruções detalhadas, leia [Habilitar a extensão `azure_ai`](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/generative-ai-azure-overview#enable-the-azure_ai-extension).

    ```sql
    CREATE EXTENSION azure_ai;
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_openai.endpoint', 'https://<endpoint>.openai.azure.com');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_openai.subscription_key', '<API Key>');
    ```

4. Para fazer chamadas com êxito no serviços de ***Linguagem de IA do Azure*** usando a extensão `azure_ai`, você deve fornecer o ponto de extremidade do serviço e uma chave. Usando a mesma guia do navegador em que o Cloud Shell está aberto, navegue até o recurso de serviço de linguagem no [portal do Azure](https://portal.azure.com/) e selecione o item **Chaves e ponto de extremidade** em **Gerenciamento de recursos** no menu de navegação à esquerda.

5. Copie os valores do ponto de extremidade e da chave de acesso e, nos comandos abaixo, substitua os tokens `{endpoint}` e `{api-key}` pelos valores copiados do portal do Azure. Execute os comandos no prompt de comando `psql` no Cloud Shell para adicionar seus valores à tabela `azure_ai.settings`.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '{api-key}');
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

## Extrair frases-chave

1. As frases-chave são extraídas como `text[]`, conforme é revelado pela função `pg_typeof`:

    ```sql
    SELECT pg_typeof(azure_cognitive.extract_key_phrases('The food was delicious and the staff were wonderful.', 'en-us'));
    ```

    Crie uma coluna para conter os principais resultados.

    ```sql
    ALTER TABLE listings ADD COLUMN key_phrases text[];
    ```

1. Preencha a coluna em lotes. Dependendo da cota, você pode ajustar o valor `LIMIT`. *Sinta-se à vontade para executar o comando quantas vezes quiser*; você não precisa de todas as linhas preenchidas para este exercício.

    ```sql
    UPDATE listings
    SET key_phrases = azure_cognitive.extract_key_phrases(description)
    FROM (SELECT id FROM listings WHERE key_phrases IS NULL ORDER BY id LIMIT 100) subset
    WHERE listings.id = subset.id;
    ```

1. Consultar listagens por frases-chave:

    ```sql
    SELECT id, name FROM listings WHERE 'closet' = ANY(key_phrases);
    ```

    Você obterá resultados como este, dependendo de quais listagens têm frases-chave preenchidas:

    ```sql
       id    |                name                
    ---------+-------------------------------------
      931154 | Met Tower in Belltown! MT2
      931758 | Hottest Downtown Address, Pool! MT2
     1084046 | Near Pike Place & Space Needle! MT2
     1084084 | The Best of the Best, Seattle! MT2
    ```

## Reconhecimento de entidade nomeada

1. As entidades são extraídas como `azure_cognitive.entity[]`, conforme revelado pela função `pg_typeof`:

    ```sql
    SELECT pg_typeof(azure_cognitive.recognize_entities('For more information, see Cognitive Services Compliance and Privacy notes.', 'en-us'));
    ```

    Crie uma coluna para conter os principais resultados.

    ```sql
    ALTER TABLE listings ADD COLUMN entities azure_cognitive.entity[];
    ```

2. Preencha a coluna em lotes. Esse processo pode levar vários minutos. Você pode ajustar o valor `LIMIT` dependendo da cota ou retornar mais rapidamente com resultados parciais. *Sinta-se à vontade para executar o comando quantas vezes quiser*; você não precisa de todas as linhas preenchidas para este exercício.

    ```sql
    UPDATE listings
    SET entities = azure_cognitive.recognize_entities(description, 'en-us')
    FROM (SELECT id FROM listings WHERE entities IS NULL ORDER BY id LIMIT 500) subset
    WHERE listings.id = subset.id;
    ```

3. Agora você pode consultar todas as entidades de listagens para encontrar propriedades com porões:

    ```sql
    SELECT id, name
    FROM listings, unnest(entities) e
    WHERE e.text LIKE '%basement%'
    LIMIT 10;
    ```

    O que retorna algo semelhante a isso:

    ```sql
       id    |                name                
    ---------+-------------------------------------
      430610 | 3br/3ba. modern, roof deck, garage
      430610 | 3br/3ba. modern, roof deck, garage
     1214306 | Private Bed/bath in Home: green (A)
       74328 | Spacious Designer Condo
      938785 | Best Ocean Views By Pike Place! PA1
       23430 | 1 Bedroom Modern Water View Condo
      828298 | 2 Bedroom Sparkling City Oasis
      338043 | large modern unit & fab location
      872152 | Luxurious Local Lifestyle 2Bd/2+Bth
      116221 | Modern, Light-Filled Fremont Flat
    ```

## Reconhecimento de PII

1. As entidades são extraídas como `azure_cognitive.pii_entity_recognition_result`, conforme revelado pela função `pg_typeof`:

    ```sql
    SELECT pg_typeof(azure_cognitive.recognize_pii_entities('For more information, see Cognitive Services Compliance and Privacy notes.', 'en-us'));
    ```

    Esse valor é um tipo composto que contém texto redigido e uma matriz de entidades PII, conforme verificado por:

    ```sql
    \d azure_cognitive.pii_entity_recognition_result
    ```

    O que imprime:

    ```sql
         Composite type "azure_cognitive.pii_entity_recognition_result"
         Column    |           Type           | Collation | Nullable | Default 
    ---------------+--------------------------+-----------+----------+---------
     redacted_text | text                     |           |          | 
     entities      | azure_cognitive.entity[] |           |          | 
    ```

    Crie uma coluna para conter o texto redigido e outra para as entidades reconhecidas:

    ```sql
    ALTER TABLE listings ADD COLUMN description_pii_safe text;
    ALTER TABLE listings ADD COLUMN pii_entities azure_cognitive.entity[];
    ```

2. Preencha a coluna em lotes. Esse processo pode levar vários minutos. Você pode ajustar o valor `LIMIT` dependendo da cota ou retornar mais rapidamente com resultados parciais. *Sinta-se à vontade para executar o comando quantas vezes quiser*; você não precisa de todas as linhas preenchidas para este exercício.

    ```sql
    UPDATE listings
    SET
     description_pii_safe = pii.redacted_text,
     pii_entities = pii.entities
    FROM (SELECT id, description FROM listings WHERE description_pii_safe IS NULL OR pii_entities IS NULL ORDER BY id LIMIT 100) subset,
    LATERAL azure_cognitive.recognize_pii_entities(subset.description, 'en-us') as pii
    WHERE listings.id = subset.id;
    ```

3. Agora você pode exibir descrições de listagens com qualquer PII potencial editado:

    ```sql
    SELECT description_pii_safe
    FROM listings
    WHERE description_pii_safe IS NOT NULL
    LIMIT 1;
    ```

    O que exibe:

    ```sql
    A lovely stone-tiled room with kitchenette. New full mattress futon bed. Fridge, microwave, kettle for coffee and tea. Separate entrance into book-lined mudroom. Large bathroom with Jacuzzi (shared occasionally with ***** to do laundry). Stone-tiled, radiant heated floor, 300 sq ft room with 3 large windows. The bed is queen-sized futon and has a full-sized mattress with topper. Bedside tables and reading lights on both sides. Also large leather couch with cushions. Kitchenette is off the side wing of the main room and has a microwave, and fridge, and an electric kettle for making coffee or tea. Kitchen table with two chairs to use for meals or as desk. Extra high-speed WiFi is also provided. Access to English Garden. The Ballard Neighborhood is a great place to visit: *10 minute walk to downtown Ballard with fabulous bars and restaurants, great ****** farmers market, nice three-screen cinema, and much more. *5 minute walk to the Ballard Locks, where ships enter and exit Puget Sound
    ```

4. Você também pode identificar as entidades reconhecidas em PII, por exemplo, usando a listagem idêntica acima:

    ```sql
    SELECT entities
    FROM listings
    WHERE entities IS NOT NULL
    LIMIT 1;
    ```

    O que exibe:

    ```sql
                            pii_entities                        
    -------------------------------------------------------------
    {"(hosts,PersonType,\"\",0.93)","(Sunday,DateTime,Date,1)"}
    ```

## Verifique seu trabalho

Vamos garantir que as frases-chave extraídas, entidades reconhecidas e PII foram preenchidas:

1. Verifique as frases-chave:

    ```sql
    SELECT COUNT(*) FROM listings WHERE key_phrases IS NOT NULL;
    ```

    Você deve ver algo assim, dependendo de quantos lotes você executou:

    ```sql
    count 
    -------
     100
    ```

2. Verifique as entidades reconhecidas:

    ```sql
    SELECT COUNT(*) FROM listings WHERE entities IS NOT NULL;
    ```

    Você deve ver algo como:

    ```sql
    count 
    -------
     500
    ```

3. Verifique as PII editadas:

    ```sql
    SELECT COUNT(*) FROM listings WHERE description_pii_safe IS NOT NULL;
    ```

    Se você carregou um único lote de 100, deverá ver:

    ```sql
    count 
    -------
     100
    ```

    Você pode verificar quantas listagens tiveram PII detectadas:

    ```sql
    SELECT COUNT(*) FROM listings WHERE description != description_pii_safe;
    ```

    Você deve ver algo como:

    ```sql
    count 
    -------
        87
    ```

4. Verifique as entidades PII detectadas. De acordo com o acima, devemos ter 13 sem uma matriz PII vazia.

    ```sql
    SELECT COUNT(*) FROM listings WHERE pii_entities IS NULL AND description_pii_safe IS NOT NULL;
    ```

    Resultado:

    ```sql
    count 
    -------
        13
    ```

## Limpar

Depois de concluir este exercício, exclua os recursos do Azure que você criou. Você é cobrado pela capacidade configurada, não por quanto do banco de dados é utilizado. Siga estas instruções para excluir seu grupo de recursos e todos os recursos que você criou para este laboratório.

1. Abra um navegador da Web e navegue até o [portal do Azure](https://portal.azure.com/) e, na home page, selecione **Grupos de recursos** em Serviços do Azure.

    ![Captura de tela de grupos de recursos realçados por uma caixa vermelha em Serviços do Azure no portal do Azure.](media/17-azure-portal-home-azure-services-resource-groups.png)

2. Na caixa de pesquisa de filtro para qualquer campo, insira o nome do grupo de recursos que você criou para este laboratório e selecione seu grupo de recursos na lista.

3. Na página **Visão geral** do grupo de recursos, selecione **Excluir grupo de recursos**.

    ![Captura de tela da folha Visão geral do grupo de recursos com o botão Excluir grupo de recursos realçado por uma caixa vermelha.](media/17-resource-group-delete.png)

4. Na caixa de diálogo de confirmação, insira o nome do seu grupo de recursos que deseja excluir e selecione **Excluir**.
