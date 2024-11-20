---
lab:
  title: Configurar permissões no Banco de Dados do Azure para PostgreSQL
  module: Secure Azure Database for PostgreSQL
---

# Configurar permissões no Banco de Dados do Azure para PostgreSQL

Neste exercício de laboratório, você atribuirá funções RBAC para controlar o acesso a recursos do Banco de Dados do Azure para PostgreSQL e o PostgreSQL GRANTS para controlar o acesso às operações de banco de dados.

## Antes de começar

Você precisa ter uma assinatura própria do Azure para concluir este exercício. Se você não tiver uma assinatura do Azure, crie uma [conta de avaliação gratuita do Azure](https://azure.microsoft.com/free).

Para concluir esses exercícios, você precisa instalar um servidor PostgreSQL conectado à ID do Microsoft Entra, anteriormente chamado Azure Active Directory

### Criar um grupo de recursos

1. Em um navegador da Web, navegue até o [portal do Azure](https://portal.azure.com). Entre usando uma conta de proprietário ou colaborador.
2. Em Serviços do Azure, selecione **Grupos de recursos** e **+ Criar**.
3. Verifique se a assinatura correta é exibida e insira o nome do grupo de recursos como **rg-PostgreSQL_Entra**. Selecione uma **Região**.
4. Selecione **Examinar + criar**. Em seguida, selecione **Criar**.

## Criar um servidor flexível do Banco de Dados do Azure para PostgreSQL

1. Em Serviços do Azure, selecione **+ Criar um recurso**.
    1. Em **Pesquisar no Marketplace**, digite**`azure database for postgresql flexible server`** , escolha **Servidor flexível do Banco de Dados do Azure para PostgreSQL** e clique em **Criar**.
1. Na guia **Noções básicas** do servidor flexível, insira cada campo da seguinte maneira:
    1. Assinatura – a sua assinatura.
    1. Grupo de recursos – **rg-PostgreSQL_Entra**.
    1. Nome do servidor – **psql-postgresql-fx7777** (o nome do servidor precisa ser globalmente exclusivo, portanto, substitua 7777 por números aleatórios).
    1. Região – selecione a mesma região que a do grupo de recursos.
    1. Versão do PostgreSQL – selecione 16.
    1. Tipo de carga de trabalho – **Desenvolvimento**.
    1. Computação + armazenamento – **Com capacidade de intermitência, B1ms**.
    1. Zona de disponibilidade – sem preferência.
    1. Alta disponibilidade – Deixe essa opção desmarcada.
    1. Método de autenticação – selecione **Autenticação PostgreSQL e Microsoft Entra**
    1. Definir administrador do Microsoft Entra – selecione **Definir administrador**
        1. Pesquise sua conta em **Selecionar administradores do Microsoft Entra** e (o) sua conta e clique em **Selecionar**
    1. Em **nome de usuário do administrador**, insira **`demo`**.
    1. Em **senha**, insira uma senha adequadamente complexa.
    1. Selecione **Avançar: Rede >**.
1. Na guia **Rede** do servidor flexível, insira cada campo da seguinte maneira:
    1. Método de conectividade: (o) Acesso público (endereços IP permitidos) e ponto de extremidade privado.
    1. Acesso púlico, selecione **Permitir acesso público a esse recurso por meio da Internet usando um endereço IP público**
    1. De acordo com as regras do Firewall, selecione **+ Adicionar endereço IP do cliente atual** para adicionar seu endereço IP atual como uma regra de firewall. Opcionalmente, você pode nomear essa regra de firewall como algo significativo. Adicione também **Adicionar 0.0.0.0 - 255.255.255.255** e clique em **Continuar**
1. Selecione **Examinar + criar**. Examine suas configurações, então,selecione **Criar** para criar seu servidor flexível do Banco de Dados do Azure para PostgreSQL. Quando a implantação for concluída, selecione **Acessar recurso**, para iniciar a próxima etapa.

## Instalar o Azure Data Studio

Para instalar o Azure Data Studio para uso com o Banco de Dados do Azure para PostgreSQL:

1. Em um navegador, navegue para [Baixar e instalar o Azure Data Studio](/sql/azure-data-studio/download-azure-data-studio) e, na plataforma Windows, selecione **Instalador de usuário (recomendado)**. O arquivo executável é baixado para a pasta Downloads.
1. Selecione **Abrir arquivo**.
1. O Contrato de licença é exibido. Revise e **aceite o contrato**, depois selecione **Avançar**.
1. Em **Selecionar Tarefas Adicionais**, selecione **Adicionar ao PATH** e as outras adições necessárias. Selecione **Avançar**.
1. A caixa de diálogo **Pronto para Instalar** é exibida. Examine suas configurações. Selecione **Voltar** para fazer alterações ou selecione **Instalar**.
1. A caixa de diálogo **Concluir o Assistente de Instalação do Azure Data Studio** é exibida. Selecione **Concluir**. O Azure Data Studio é iniciado.

### Instalar a extensão do PostgreSQL

1. Abra o Azure Data Studio se ainda não estiver aberto.
1. No menu esquerdo, selecione **Extensões** para exibir o painel Extensões.
1. Na barra de pesquisa, insira **PostgreSQL**. O ícone da extensão PostgreSQL para Azure Data Studio é exibido.
1. Selecione **Instalar**. A extensão é instalada.

### Conectar-se ao servidor flexível do Banco de Dados do Azure para PostgreSQL

1. Abra o Azure Data Studio se ainda não estiver aberto.
1. No menu à esquerda, selecione **Coleções**.
1. Selecione **Nova Conexão**.
1. Em **Detalhes da Conexão**, em **Tipo de conexão**, selecione **PostgreSQL** na lista suspensa.
1. No **Nome do servidor**, insira o nome completo do servidor como ele aparece no portal do Azure.
1. Mantenha o **Tipo de autenticação** como Senha.
1. Em Nome de usuário e senha, digite o nome de usuário **demo** e a senha complexa que você criou acima
1. Selecione [ x ] Lembrar senha.
1. Os campos restantes são opcionais.
1. Selecione **Conectar**. Você está conectado ao servidor de Banco de Dados do Azure para PostgreSQL.
1. Uma lista dos bancos de dados do servidor é exibida. Isso inclui bancos de dados de sistema e de usuário.

### Criar o banco de dados do zoológico

1. Navegue até a pasta com seus arquivos de script de exercício ou baixe o **Lab2_ZooDb.sql** do [MSLearn PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql/Allfiles/Labs/02).
1. Abra o Azure Data Studio se ainda não estiver aberto.
1. Selecione **Arquivo**, **Abrir Arquivo** e navegue até a pasta em que você salvou o script. Selecione **../Allfiles/Labs/02/Lab2_ZooDb.sql** e **Abrir**. Se um aviso de confiança for exibido, selecione **Abrir**.
1. Execute o script. O banco de dados zoodb é criado.

## Criar uma nova conta de usuário no Microsoft Entra ID

> [!NOTE]
> Na maioria dos ambientes de produção ou desenvolvimento, é muito possível que você não tenha os privilégios de conta de assinatura para criar contas em seu serviço Microsoft Entra ID.  Nesse caso, se permitido por sua organização, tente pedir ao administrador do Microsoft Entra ID para criar uma conta de teste para você. Se você não conseguir obter a conta do Entra de teste, ignore esta seção e continue na seção **CONCEDER acesso ao Banco de Dados do Azure para PostgreSQL**. 

1. No [portal do Azure](https://portal.azure.com), faça login usando uma conta de Proprietário e vá para Microsoft Entra ID.
1. Em **Gerenciar**, selecione **Usuários**.
1. No canto superior esquerdo, selecione **Novo usuário** e clique em **Criar usuário**.
1. Na página **Novo usuário**, insira os seguintes detalhes e selecione **Criar**:
    - **Nome UPN:** escolha um nome do princípio
    - **Nome de exibição:** escolha um nome de exibição
    - **Senha:** Desmarque **Gerar senha automaticamente** e insira uma senha forte. Registre o nome da entidade de usuário e a senha.
    - Clique em **Examinar + criar**

    > [!TIP]
    > Quando o usuário for criado, anote o **Nome UPN** completo para usá-lo posteriormente no logon.

### Atribuir a função Leitor

1. No portal do Azure, selecione **Todos os recursos** e selecione o recurso Banco de Dados do Azure para PostgreSQL.
1. Selecione **IAM (controle de acesso)** e escolha **Atribuições de função**. A nova conta de usuário aparece na lista.
1. Selecione **+ Adicionar** e **Adicionar atribuição de função**.
1. Selecione a função **Leitor** e clique em **Avançar**.
1. Escolha **+ Selecionar membros**, adicione a nova conta que você adicionou na etapa anterior à lista de membros e selecione **Avançar**.
1. Selecione **Examinar + Atribuir**.

### Testar a função Leitor

1. Na parte superior direita do portal do Azure, selecione a conta de usuário e selecione **Sair**.
1. Entre como o novo usuário, com o nome UPN e senha que anotou. Substitua a senha padrão se você for solicitado e anote a nova.
1. Escolha **Perguntar mais tarde** se for solicitada a autenticação multifator
1. Na página inicial do portal, selecione **Todos os recursos** e selecione o recurso Banco de Dados do Azure para PostgreSQL.
1. Selecione **Interromper**. Um erro é exibido, pois a função de Leitor permite exibir o recurso, mas não alterá-lo.

### Atribuir a função de Colaborador

1. Na parte superior direita do portal do Azure, selecione a conta de usuário e selecione **Sair**.
1. Entre usando sua conta de Proprietário original.
1. Navegue até o recurso do Banco de Dados do Azure para PostgreSQL e selecione **IAM (Controle de Acesso).**
1. Selecione **+ Adicionar** e **Adicionar atribuição de função**.
1. Selecione **Funções de administrador com privilégios**
1. Selecione a função **Colaborador** e clique em **Avançar**.
1. Adicione a nova conta que você adicionou anteriormente à lista de membros e selecione **Avançar**.
1. Selecione **Examinar + Atribuir**.
1. Selecione **Atribuições de função**. A nova conta agora tem as funções de Leitor e Colaborador.

## Testar a função Colaborador

1. Na parte superior direita do portal do Azure, selecione a conta de usuário e selecione **Sair**.
1. Entre com a nova conta, com o UPN e a senha que você anotou.
1. Na página inicial do portal, selecione **Todos os recursos** e clique no recurso do Banco de Dados do Azure para MySQL.
1. Selecione **Parar** e clique em **Sim**. Desta vez, o servidor para sem erros porque a nova conta tem a função necessária atribuída.
1. Selecione **Iniciar** para garantir que o servidor PostgreSQL esteja pronto para as próximas etapas.
1. Na parte superior direita do portal do Azure, selecione a conta de usuário e selecione **Sair**.
1. Entre usando sua conta de Proprietário original.

## CONCEDER acesso ao Banco de Dados do Azure para PostgreSQL

1. Abra o Azure Data Studio e conecte-se ao servidor do Banco de Dados do Azure para PostgreSQL usando o usuário de **demonstração** definido como administrador acima.
1. No painel de consulta, execute esse código no banco de dados postgres. Doze funções de usuário devem ser retornadas, incluindo a função de **demonstração** que você está usando para se conectar:

    ```SQL
    SELECT rolname FROM pg_catalog.pg_roles;
    ```

1. Para criar uma nova função, execute este código

    ```SQL
    CREATE ROLE dbuser WITH LOGIN NOSUPERUSER INHERIT CREATEDB NOCREATEROLE NOREPLICATION PASSWORD 'R3placeWithAComplexPW!';
    GRANT CONNECT ON DATABASE zoodb TO dbuser;
    ```
    > [!NOTE]
    > Certifique-se de substituir a senha no script acima por uma senha complexa.

1. Para listar a nova função, execute a consulta SELECT no **pg_catalog.pg_roles** novamente. Você deverá ver a função **dbuser** listada.
1. Para habilitar a nova função para consultar e modificar dados na tabela **animal** no banco de dados **zoodb**, execute este código no banco de dados zoodb:

    ```SQL
    GRANT SELECT, INSERT, UPDATE, DELETE ON animal TO dbuser;
    ```

## Testar a nova função

1. No Azure Data Studio, na lista de **CONEXÕES**, selecione o novo botão de conexão.
1. Na lista **Tipo de conexão**, selecione **PostgreSQL**.
1. Na caixa de texto **Nome do servidor**, digite o nome do servidor totalmente qualificado para o recurso do Banco de Dados do Azure para PostgreSQL. Você pode copiar no portal do Azure.
1. Na lista **Tipos de autenticação**, selecione a **Senha**.
1. Na caixa de texto **Nome de usuário**, digite **dbuser** e, na caixa de texto **Senha**, digite a senha complexa com a qual você criou a conta.
1. Marque a caixa de seleção **Lembrar senha** e selecione **Conectar**.
1. Selecione **Nova consulta** e execute este código:

    ```SQL
    SELECT * FROM animal;
    ```

1. Para testar se você tem o privilégio UPDATE, execute este código:

    ```SQL
    UPDATE animal SET name = 'Linda Lioness' WHERE ani_id = 7;
    SELECT * FROM animal;
    ```

1. Para testar se você tem o privilégio DROP, execute o código abaixo. Se ocorreu um erro, examine o código de erro:

    ```SQL
    DROP TABLE animal;
    ```

1. Para testar se você tem o privilégio GRANT, execute este código:

    ```SQL
    GRANT ALL PRIVILEGES ON animal TO dbuser;
    ```

Estes testes demonstram que o novo usuário pode executar comandos da DML (Linguagem de Manipulação de Dados) para consultar e modificar dados, mas não pode usar comandos da DDL (Linguagem de Definição de Dados) para alterar o esquema. Além disso, ele não pode conceder nenhum privilégio novo para contornar as permissões.

## Limpeza

Você não usará este servidor PostgreSQL novamente, portanto, exclua o grupo de recursos que você criou, o que removerá o servidor.
