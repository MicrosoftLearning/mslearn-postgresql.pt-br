---
lab:
  title: Llistar alterações de tabela com decodificação lógica
  module: Understand write-ahead logging
---

# Llistar alterações de tabela com decodificação lógica

Neste exercício, você configurará a replicação lógica, que é nativa do PostgreSQL. Você criará dois servidores, que atuam como publicador e assinante. Os dados no zoodb serão replicados entre eles.

## Antes de começar

Você precisa ter uma assinatura própria do Azure para concluir este exercício. Se você não tiver uma assinatura do Azure, crie uma [conta de avaliação gratuita do Azure](https://azure.microsoft.com/free).

## Criar grupo de recursos

1. Entre no portal do Azure. Sua conta de usuário precisa ter função de Proprietário ou Colaborador para a assinatura do Azure.
1. Selecione **Grupos de recursos** e **+ Criar**.
1. Selecione sua assinatura.
1. No grupo de recursos, insira **rg-PostgreSQL_Replication**.
1. Selecione uma região próxima à sua localização.
1. Selecione **Examinar + criar**.
1. Selecione **Criar**.

## Criar um servidor publicador

1. Em Serviços do Azure, selecione **+ Criar um recurso**. Em **Categorias**, selecione **Bancos de dados**. Em **Banco de Dados do Azure para PostgreSQL**, selecione **Criar**.
1. Na guia **Noções básicas** do servidor flexível, insira cada campo da seguinte maneira:
    1. Assinatura – a sua assinatura.
    1. Grupo de recursos – selecione **rg-PostgreSQL_Replication**.
    1. Nome do servidor – **psql-postgresql-pub9999** (o nome precisa ser globalmente exclusivo, portanto, substitua 9999 por números aleatórios).
    1. Região – selecione a mesma região que a do grupo de recursos.
    1. Versão do PostgreSQL – selecione 16.
    1. Tipo de carga de trabalho – **Desenvolvimento**.
    1. Computação + armazenamento – **Com capacidade de intermitência**. Selecione **Configurar servidor** e examine as opções de configuração. Não faça nenhuma alteração e feche a seção.
    1. Zona de disponibilidade – 1. Se não houver suporte para zonas de disponibilidade, deixe como Nenhuma preferência.
    1. Alta disponibilidade – Desabilitado.
    1. Método de autenticação – Somente autenticação PostgreSQL.
    1. Em **nome de usuário do administrador**, insira **`demo`**.
    1. Em **senha**, insira uma senha complexa adequada.
    1. Selecione **Avançar: Rede >**.
1. Na guia **Rede** do servidor flexível, insira cada campo da seguinte maneira:
    1. Método de conectividade: (o) Acesso público (endereços IP permitidos).
    1. **Permitir o acesso público de qualquer serviço do Azure de dentro do Azure para esse servidor** – marcada. Essa opção precisa ser marcada para que os bancos de dados do publicador e do assinante possam se comunicar entre si.
    1. Em Regras de firewall, selecione **+ Adicionar endereço IP do cliente atual**. O endereço IP atual é adicionado como uma regra de firewall. Opcionalmente, você pode dar um nome significativo para essa regra.
1. Selecione **Examinar + criar**. Em seguida, selecione **Criar**.
1. Como a criação de um Banco de Dados do Azure para PostgreSQL pode levar alguns minutos, comece com a próxima etapa assim que essa implantação estiver em andamento. Lembre-se de abrir uma nova janela ou guia do navegador para continuar.

## Criar um servidor assinante

1. Em Serviços do Azure, selecione **+ Criar um recurso**. Em **Categorias**, selecione **Bancos de dados**. Em **Banco de Dados do Azure para PostgreSQL**, selecione **Criar**.
1. Na guia **Noções básicas** do servidor flexível, insira cada campo da seguinte maneira:
    1. Assinatura – a sua assinatura.
    1. Grupo de recursos – selecione **rg-PostgreSQL_Replication**.
    1. Nome do servidor – **psql-postgresql-sub9999** (o nome precisa ser globalmente exclusivo, portanto, substitua 9999 por números aleatórios).
    1. Região – selecione a mesma região que a do grupo de recursos.
    1. Versão do PostgreSQL – selecione 16.
    1. Tipo de carga de trabalho – **Desenvolvimento**.
    1. Computação + armazenamento – **Com capacidade de intermitência**. Selecione **Configurar servidor** e examine as opções de configuração. Não faça nenhuma alteração e feche a seção.
    1. Zona de disponibilidade – 2. Se não houver suporte para zonas de disponibilidade, deixe como Nenhuma preferência.
    1. Alta disponibilidade – Desabilitado.
    1. Método de autenticação – Somente autenticação PostgreSQL.
    1. Em **nome de usuário do administrador**, insira **`demo`**.
    1. Em **senha**, insira uma senha complexa adequada.
    1. Selecione **Avançar: Rede >**.
1. Na guia **Rede** do servidor flexível, insira cada campo da seguinte maneira:
    1. Método de conectividade: (o) Acesso público (endereços IP permitidos)
    1. **Permitir o acesso público de qualquer serviço do Azure de dentro do Azure para esse servidor** – marcada. Essa opção precisa ser marcada para que os bancos de dados do publicador e do assinante possam se comunicar entre si.
    1. Em Regras de firewall, selecione **+ Adicionar endereço IP do cliente atual**. O endereço IP atual é adicionado como uma regra de firewall. Opcionalmente, você pode dar um nome significativo para essa regra.
1. Selecione **Examinar + criar**. Em seguida, selecione **Criar**.
1. Aguarde até que os dois servidores do Banco de Dados do Azure para PostgreSQL sejam implantados.

## Configurar a replicação

Para ** os servidores editor e assinante:

1. No portal do Azure, navegue até o servidor e, em Configurações, selecione **Parâmetros de servidor**.
1. Usando a barra de pesquisa, localize cada parâmetro e faça as seguintes alterações:
    1. `wal_level` = LOGICAL
    1. `max_worker_processes` = 24
1. Selecione **Salvar**. Em seguida, selecione **Salvar e Reiniciar**.
1. Aguarde a reinicialização de ambos os servidores.

    > Observação
    >
    > Depois que os servidores forem reimplantados, talvez seja necessário atualizar as janelas do navegador para observar que os servidores foram reiniciados.

## Antes de continuar

Certifique-se de ter:

1. Instalado e iniciado os servidores flexíveis do Banco de Dados do Azure para PostgreSQL.
1. Ter clonado os scripts de laboratório do [PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql.git). Caso ainda não tenha feito isso, clone o repositório localmente:
    1. Abra uma linha de comando/terminal.
    1. Execute o comando:
       ```bash
       md .\DP3021Lab
       git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git .\DP3021Lab
       ```
       > OBSERVAÇÃO
       > 
       > Se o **git** não estiver instalado, [baixe e instale o aplicativo ***git***](https://git-scm.com/download) e tente executar os comandos anteriores novamente.
1. Você instalou o Azure Data Studio. Se você ainda não tiver feito isso, [baixe e instale o ***Azure Data Studio***](https://go.microsoft.com/fwlink/?linkid=2282284).
1. Instale a extensão do **PostgreSQL** no Azure Data Studio.

## Configurar o publicador

1. Abra o Azure Data Studio e conecte-se ao servidor do publicador. (Copie o nome do servidor da seção Visão geral.)
1. Selecione **Arquivo**, **Abrir arquivo** e navegue até a pasta em que você salvou os scripts.
1. Abra o script **../Allfiles/Labs/06/Lab6_Replication.sql** e conecte-se ao servidor.
1. Realce e execute a seção **Conceder a permissão de replicação ao usuário administrador**.
1. Realce e execute a seção **Criar banco de dados zoodb**.
1. Selecione zoodb como o banco de dados atual usando a lista suspensa na barra de ferramentas. Verifique se o zoodb é o banco de dados atual executando a instrução **SELECT**.
1. Realce e execute a seção **Criar tabelas** e **Restrições de chave estrangeira** no zoodb.
1. Realce e execute a seção **Preencher as tabelas no zoodb**.
1. Realce e execute a seção **Criar uma publicação**. Quando você executar a instrução SELECT, ela não listará nada, pois a replicação ainda não estará ativa.

## Configurar o Assinante

1. Abra uma segunda instância do Azure Data Studio e conecte-se ao servidor do assinante.
1. Selecione **Arquivo**, **Abrir arquivo** e navegue até a pasta em que você salvou os scripts.
1. Abra o script **../Allfiles/Labs/06/Lab6_Replication.sql** e conecte-se ao servidor assinante. (Copie o nome do servidor da seção Visão geral.)
1. Realce e execute a seção **Conceder a permissão de replicação ao usuário administrador**.
1. Realce e execute a seção **Criar banco de dados zoodb**.
1. Selecione zoodb como o banco de dados atual usando a lista suspensa na barra de ferramentas. Verifique se o zoodb é o banco de dados atual executando a instrução **SELECT**.
1. Realce e execute a seção **Criar tabelas** e **Restrições de chave estrangeira** no zoodb.
1. Role para baixo até a seção **Criar uma assinatura**.
    1. Edite a instrução **CREATE SUBSCRIPTION** para que ela tenha o nome correto e a senha forte do servidor publicador. Realce e execute a instrução.
    1. Realce e execute a instrução **SELECT**. Isso mostra a assinatura "sub" que você criou.
1. Na seção **Exibir as tabelas**, realce e execute cada instrução **SELECT**. As tabelas foram preenchidas por replicação, realizada pelo publicador.

## Fazer alterações no banco de dados publicador

- Na primeira instância do Azure Data Studio (*sua instância do publicador*), em **Inserir mais animais** realce e execute a instrução **INSERT**. *Certifique-se de **não** executar essa instrução INSERT no assinante*.

## Exibir as alterações no banco de dados do assinante

- Na segunda instância do Azure Data Studio (assinante), em **Exibir as tabelas de animais**, realce e execute a instrução **SELECT**.

Agora você criou dois servidores flexíveis do Banco de Dados do Azure para PostgreSQL e configurou um como um publicador e outro como assinante. No banco de dados publicador, você criou e preencheu o banco de dados do zoológico. No banco de dados do assinante, você criou um banco de dados vazio, que foi preenchido por replicação de streaming.

## Limpeza

1. Depois de concluir o exercício, exclua o grupo de recursos que contém os dois servidores. Você será cobrado pelos servidores, a menos que você os interrompa ou exclua.
1. Se necessário, exclua a pasta .\DP3021Lab.
