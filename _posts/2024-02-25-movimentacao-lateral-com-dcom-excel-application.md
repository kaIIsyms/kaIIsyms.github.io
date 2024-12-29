---
title: 'Movimentação Lateral com DCOM/Excel.Application'
layout: post
---

Movimentação Lateral com DCOM/Excel.Application
===

![a](https://cdn.neow.in/news/images/uploaded/2019/03/1553427271_excel3_story.jpg)

[toc]

# Introdução
Escrevi esse artigo quando estava lendo um artigo de um conhecido meu explicando uma técnica parecida, mas com o `MMC20.Application`. Me interessei na técnica e estudei mais sobre, daí resolvi escrever esse artigo.

Antes de tudo, vou explicar os termos citados futuramente, se você já conhece (DCOM, COM, Excel, VBA, Lateral movement, DDE) pode pular pro tópico **Colocando a mão na massa**.

Agora chega de enrolação, vamos para os tópicos explicando.
## O  que é Movimentação Lateral?
A movimentação lateral consiste em técnicas que os atacantes usam pra entrar e controlar sistemas remotos em uma rede.

Para atingir o objetivo principal, muitas vezes é necessário explorar a rede para encontrar o alvo e, posteriormente, obter acesso a ele.

Os atacantes podem instalar suas próprias ferramentas de acesso remoto para realizar a movimentação lateral ou usar credenciais legítimas com ferramentas nativas de rede e sistema operacional, que podem ser mais furtivas.
## O que é COM?
**COM** (**Component Object Model**) é um sistema orientado a objetos, distribuído e independente de plataforma para a criação de componentes de software binários que podem interagir entre si. Esses objetos podem estar em um único processo, em outros processos e até mesmo em computadores remotos.

O **COM** foi introduzido pela Micro$oft em 1993. Ele é usado para IPC (**Inter Process Communication**, comunicação entre processos) em várias linguagens de programação.

O **Excel** usa o **COM** para permitir que os usuários criem/modifiquem/salvem/compartilhem arquivos do **Excel**.

Ao usar a **COM**, não precisamos entender o formato binário dos arquivos do **Excel** para realizar as diferentes operações.

Além disso, os objetos **COM** são registrados no sistema operacional para que possam ser carregados no futuro. A mágica por trás disso é o **CLSID** (**Class ID**).

Um **CLSID** é um identificador globalmente exclusivo que identifica um objeto de classe **COM**. Se o seu servidor ou contêiner permitir a vinculação a seus objetos incorporados, será necessário registrar um **CLSID** para cada classe suportada de objeto **COM**.
## O que é DCOM?
O **DCOM** é uma solução da Microsoft que permite que os componentes do software se comuniquem remotamente.

O **DCOM** foi introduzido pela Micro$oft em 1996, 3 anos depois de seu antecessor, o **COM**.

Seu antecessor, o **COM** (**Component Object Model**), não tinha funcionalidade de computação distribuída, por isso a Micro$oft introduziu o **DCOM** para atender à necessidade dos componentes de software de se comunicarem pela rede.

Basicamente, o **DCOM** permite que um cliente instancie remotamente um objeto de servidor **COM** em outra máquina e utilize seus métodos. Ele opera sobre o **RPC** (**Remote Procedure Call**) e usa a sequência de protocolo `ncacn_ip_tcp`.

O Registro do Windows armazena os dados de configuração do **DCOM** em três identificadores:

- **CLSID** - O identificador de classe (**CLSID**) é um identificador global exclusivo (**GUID**) que representa um ID exclusivo para qualquer componente de aplicativo no Windows; um exemplo de **CLSID** é `{00020812-0000-0000-C000-000000000046}`.
- **ProgID** - O identificador de programa (**ProgID**) é uma entrada de registro de identificador opcional que está vinculada ao **CLSID**, ao contrário do **CLSID**, o **ProgID** não é um formato **GUID** complexo, mas um formato legível por humanos, como `Excel.Application`.
- **APPID** - O identificador do aplicativo.
## O que são Macros do Excel?
As macros em **VBA** são há muito tempo uma das técnicas favoritas dos atacantes, normalmente o abuso de **VBA** envolve um e-mail de phishing com um documento do **Microsoft Office** contendo uma macro, juntamente com um texto feito pra induzir a vítima a ativar essa macro maliciosa.

A diferença é que vou usar macros pra dinamização e não pra acesso inicial, então não preciso me preocupar com as configurações de segurança das macros do **Office**.
# O que é DDE?
A **Dynamic Data Exchange** foi introduzida pela primeira vez em 1987, com o lançamento do Windows 2.0, como um método de comunicação entre processos, para que um programa pudesse se comunicar ou controlar outro programa, algo parecido com o **RPC**.

Na época, o único método de comunicação entre o sistema operacional e os aplicativos clientes era a "Camada de Mensagens do Windows".

A **DDE** estendeu esse protocolo para permitir a comunicação ponto a ponto entre os aplicativos clientes, por meio de transmissões de mensagens.

Uma célula no **Excel** era conhecida pela **DDE** por seu nome de "**aplicativo**". Cada aplicativo poderia organizar ainda mais as informações por grupos conhecidos como "**tópico**" e cada tópico poderia servir partes individuais de dados como um "**item**".

As alterações internas na célula devido às ações do **Excel** seriam então sinalizadas (em sentido inverso) para o aplicativo de chamada por meio de transmissões de mensagens adicionais.
## Colocando a mão na massa
Nós podemos realizar essa técnica com diversos métodos, como por exemplo:
- RegisterXLL
- Run
- ActivateMicrosoftApp
- ExcelDDE
- e outros...

Neste tópico eu usarei apenas o método **ExcelDDE**, mas antes de começarmos, há alguns pré-requisitos para esse ataque se você desejar realizá-lo:

- Requer privilégio de administrador local no alvo
- Requer o Microsoft Excel instalado no alvo
- Capacidade de gravar remotamente um arquivo no PATH do sistema

Agora podemos continuar;

![a](https://www.maisaprendizagem.com.br/wp-content/uploads/2019/10/m%C3%A3o-na-massa.png)

O mecanismo **DDE** pode ser utilizado pra realizar a execução de código sem macros em documentos do **Office**.

Essa técnica funciona fazendo com que o **Excel** avalie uma expressão (tipo `"=cmd|' /C calc'!A0"`) que exige que os dados sejam transmitidos via **DDE** de outro aplicativo.

Isso permite que um atacante especifique uma linha de comando arbitrária como o servidor **DDE** a ser executado, realizando, assim, uma execução arbitrária de código. \o/

Mas para realizar a técnica eu poderia usar o método `DDEInitiate`, citado por mim como `ExcelDDE`.

O `DDEInitiate` estabelece um canal **DDE** entre um aplicativo e um servidor **DDE**. Depois que um canal é estabelecido, o aplicativo pode solicitar dados do servidor fazendo referência ao canal nas funções **DDE** seguintes. O aplicativo atua como cliente, solicitando dados do aplicativo servidor por meio do canal.

Se o canal for estabelecido sem erros o `DDEInitiate` retornará o número do canal. Os números dos canais são não negativos e o número de canais que você pode estabelecer é limitado apenas pelos recursos do sistema.

> Para evitar que perguntem se o usuário deseja abrir o aplicativo, você pode definir a opção **SAFETY** do `DDESetOption`.

Exemplo:
```vba
channelNumber = Application.DDEInitiate( _ 
 app:="WinWord", _ 
 topic:="C:\WINWORD\DOCUMENTO.DOC") 
Application.DDEExecute channelNumber, "[FILEPRINT]" 
Application.DDETerminate channelNumber
```
O exemplo acima abre um canal para o **Word**, abre o documento do **Word** `documento.doc` e envia o comando **FilePrint** para o **WordBasic**.

Podemos fornecer `cmd` como o parâmetro do `DDEInitiate` enquanto o outro parâmetro pode ser qualquer argumento de linha de comando escolhido.

Na teoria é simples, agora vamos tentar executar o método:
```ps1
$Com = [Type]::GetTypeFromProgID("Excel.Application","192.168.2.10")
$Obj = [System.Activator]::CreateInstance($Com)
$Obj.DisplayAlerts = $false
$Obj.DDEInitiate("cmd", "/c calc.exe")
```
Explicando cada coisa: 
- `$Com = [Type]::GetTypeFromProgID("Excel.Application","192.168.2.10)` Pega as informações de tipo para o aplicativo **Excel** no computador especificado.
- `$Obj = [System.Activator]::CreateInstance($Com)` Cria uma instância do aplicativo **Excel** com as informações.
- `$Obj.DisplayAlerts = $false` Define a propriedade `DisplayAlerts` do objeto do aplicativo **Excel** como `false`, tirando os alertas e as mensagens de aviso.
- `$Obj.DDEInitiate("cmd", "/c calc.exe")` Inicia uma conversa **DDE** com o `calc.exe`.
# Conclusão
Essa técnica pode resultar em um impacto significativo porque permite que os atacantes rodem um executável mal-intencionado em qualquer máquina que tenha o **Microsoft Office** instalado desde que tenham direitos administrativos na máquina, o atacante poderia upar um malware, colocar no **PATH** e executar o malware por um método tipo `ActivateMicrosoftApp()`.

Essa técnica também pode ser usada pra persistência se o atacante ownou um computador com o **Microsoft Office** instalado, ele poderia escrever um script do **PowerShell** pra iniciar o `Excel.Application` via **DCOM** e chamar o método `ActivateMicrosoftApp()` ou outro método (você pode ler a lista de métodos no tópico **Colocando a mão na massa**) e por fim criar uma **scheduled task** pra executar esse script.
