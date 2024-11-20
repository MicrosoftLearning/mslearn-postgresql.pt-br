---
lab:
  title: Explorar o Banco de Dados do Azure para PostgreSQL
  module: Explore PostgreSQL architecture
---

# Explorar o Banco de Dados do Azure para PostgreSQL

Neste exercício, você criará um servidor flexível do Banco de Dados do Azure para PostgreSQL e configurará o período de retenção de backup.

## Antes de começar

Você precisa ter uma assinatura própria do Azure para concluir este exercício. Se você não tiver uma assinatura do Azure, crie uma [conta de avaliação gratuita do Azure](https://azure.microsoft.com/free).

### Criar um grupo de recursos

1. Em um navegador da Web, navegue até o [portal do Azure](https://portal.azure.com). Entre usando uma conta de proprietário ou colaborador.
2. Em Serviços do Azure, selecione **Grupos de recursos** e **+ Criar**.
3. Verifique se a assinatura correta é exibida e insira o nome do grupo de recursos como **rg-PostgreSQL_Flexi**. Selecione uma **Região**.
4. Selecione **Examinar + criar**. Em seguida, selecione **Criar**.

### Criar um servidor flexível do Banco de Dados do Azure para PostgreSQL

1. Em Serviços do Azure, selecione **+ Criar um recurso**.
    1. Em **Pesquisar no Marketplace**, digite **`azure database for postgresql flexible server`**, escolha **Servidor flexível do Banco de Dados do Azure para PostgreSQL** e clique em **Criar**.
1. Na guia **Noções básicas** do servidor flexível, insira cada campo da seguinte maneira:
    1. Assinatura – a sua assinatura.
    1. Grupo de recursos – **rg-PostgreSQL_Flexi**.
    1. Nome do servidor – **psql-postgresql-fx9999** (o nome do servidor precisa ser globalmente exclusivo, portanto, substitua 9999 por números aleatórios).
    1. Região – selecione a mesma região que a do grupo de recursos.
    1. Versão do PostgreSQL – selecione 16.
    1. Tipo de carga de trabalho – **Desenvolvimento**.
    1. Computação + armazenamento – **Com capacidade de intermitência, B1ms**. Selecione **Configurar servidor** e examine as opções de configuração. Não faça nenhuma alteração e feche a folha no lado esquerdo.
    1. Zona de disponibilidade – sem preferência.
    1. Alta disponibilidade – Desabilitado.
    1. Método de autenticação – **Somente autenticação PostgreSQL**
    1. Em **nome de usuário do administrador**, insira **`demo`**.
    1. Em **senha**, insira uma senha adequadamente complexa.
    1. Selecione **Avançar: Rede >**.
1. Na guia **Rede** do servidor flexível, insira cada campo da seguinte maneira:
    1. Método de conectividade: (o) Acesso público (endereços IP permitidos) e ponto de extremidade privado.
    1. Acesso púlico, selecione **Permitir acesso público a esse recurso por meio da Internet usando um endereço IP público**
    1. De acordo com as regras do Firewall, selecione **+ Adicionar endereço IP do cliente atual** para adicionar seu endereço IP atual como uma regra de firewall. Opcionalmente, você pode nomear essa regra de firewall como algo significativo.
1. Selecione **Examinar + criar**. Examine suas configurações, então,selecione **Criar** para criar seu servidor flexível do Banco de Dados do Azure para PostgreSQL. Quando a implantação for concluída, selecione **Ir para o recurso** pronto para a próxima etapa.

## Examinar parâmetros do servidor

1. Em **Configurações**, selecione **Parâmetros do servidor**.
1. Na caixa **Pesquisar para filtrar itens...**, insira **`connections`**. Parâmetros de servidor relacionados a conexões são exibidos. Observe o valor de **max_connections**. Não faça nenhuma alteração.
1. No menu à esquerda, selecione **Visão geral** para sair dos **Parâmetros do servidor**.

## Alterar o período de retenção de backup

1. Navegue até a folha **Visão geral** e, em **Configurações**, selecione **Computação + armazenamento**. Esta seção exibe a camada de computação atual e a opção de atualizá-la. Ela também exibe a quantidade de armazenamento que você provisionou e a opção de aumentar o armazenamento.
1. Em **backups**, o período de retenção de backup em dias é exibido. Usando a barra de controle deslizante, altere o período de retenção de backup para 14 dias. Selecione **Salvar** para gravar suas alterações.
1. Quando você tiver concluído este exercício, navegue até a folha **Visão geral** e clique em **PARAR**, que interromperá o servidor.
    1. Você não será cobrado enquanto o servidor estiver parado, mas lembre-se de que dentro de sete dias o servidor será reiniciado se você não o tiver excluído.

## Exercício opcional – configurar um servidor de alta disponibilidade

1. Em Serviços do Azure, selecione **+ Criar um recurso**.
    1. Em **Pesquisar no Marketplace**, digite**`azure database for postgresql flexible server`** , escolha **Servidor flexível do Banco de Dados do Azure para PostgreSQL** e clique em **Criar**.
1. Na guia **Noções básicas** do servidor flexível, insira cada campo da seguinte maneira:
    1. Assinatura – a sua assinatura.
    1. Grupo de recursos – **rg-PostgreSQL_Flexi**.
    1. Nome do servidor – **psql-postgresql-fx8888** (o nome do servidor precisa ser globalmente exclusivo, portanto, substitua 8888 por números aleatórios).
    1. Região – selecione a mesma região que a do grupo de recursos.
    1. Versão do PostgreSQL – selecione 16.
    1. Tipo de carga de trabalho – **Produção (pequena/média)**
    1. Computação + armazenamento – Deixe como **Uso geral**.
    1. Zona de disponibilidade – Você pode deixar isso em "Sem preferência" e o Azure escolherá automaticamente zonas de disponibilidade para seus servidores primário e secundário. Como alternativa, especifique uma zona de disponibilidade a ser co-localizada com o seu aplicativo.
    1. Habilitar alta disponibilidade – verifique isso. Observe os custos estimados quando essa opção é selecionada.
    1. Modo de alta disponibilidade – Escolha **Com redundância de zona – um servidor em espera está sempre disponível dentro de outra zona na mesma região do servidor principal**
    1. Método de autenticação – **Somente autenticação PostgreSQL**
    1. Em **nome de usuário do administrador**, insira **demo**.
    1. Em **senha**, insira uma senha adequadamente complexa.
    1. Selecione **Avançar: Rede >**.
1. Na guia **Rede** do servidor flexível, insira cada campo da seguinte maneira:
    1. Método de conectividade: (o) Acesso público (endereços IP permitidos) e ponto de extremidade privado.
    1. Acesso púlico, selecione **Permitir acesso público a esse recurso por meio da Internet usando um endereço IP público**
    1. De acordo com as regras do Firewall, selecione **+ Adicionar endereço IP do cliente atual** para adicionar seu endereço IP atual como uma regra de firewall. Opcionalmente, você pode nomear essa regra de firewall como algo significativo.
1. Selecione **Examinar + criar**. Examine suas configurações, e selecione **Criar** para criar seu servidor flexível do Banco de Dados do Azure para PostgreSQL. Quando a implantação for concluída, selecione **Ir para o recurso** pronto para a próxima etapa.

### Inspecionar o novo servidor

1. Em **Configurações**, selecione **Alta disponibilidade**. A alta disponibilidade está habilitada e a zona de disponibilidade primária deve ser 1.
    1. A zona de disponibilidade em espera foi alocada automaticamente e será diferente da zona de disponibilidade primária e normalmente é a zona 2.

### Forçar um failover

1. Na folha **Alta disponibilidade**, no menu superior, selecione **Failover Forçado**. Uma mensagem é exibida. Selecione **OK**.
1. O processo de failover é iniciado. Uma mensagem é exibida quando a operação de failover é concluída com êxito.
1. Na folha **Alta disponibilidade**, você pode ver que a zona primária agora é 2 e a zona de disponibilidade em espera é 1. Talvez seja necessário atualizar seu navegador para ver as informações mais recentes.
1. Quando você concluir este exercício, exclua o servidor.

## Limpar

O servidor deste exercício de laboratório incorrerá em encargos. Exclua o grupo de recursos **rg-PostgreSQL_Flexi** depois de concluir o exercício. Isso removerá o servidor e quaisquer outros recursos que você tenha implantado neste exercício de laboratório.
