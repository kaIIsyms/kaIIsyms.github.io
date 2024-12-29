---
title: 'Fazendo consultas no LDAP sem ser detectado'
disqus: hackmd
---

Fazendo consultas no LDAP sem ser detectado
===

![ldap](https://wwwseeburgercom-160c6.kxcdn.com/fileadmin/_processed_/b/0/csm_Connector-LDAP_71c38695d9.png)
[toc]
# Introdução
Bom, resolvi escrever este pequeno e muito simples artigo pois novamente estou entediado, e o assunto da vez novamente é leigo porém não é inutil, então aproveita e lê o artigo ai \o/
# O que é o LDAP
Primeiro antes de falar sobre o assunto principal, vamos conhecer o LDAP.

O LDAP é um protocolo fundamental usado em quase todos os ambientes Windows, ele permite que os administradores acessem o AD e gerencie usuários e grupos nele, além de permitir que recursos externos consultem dados do AD.

O ponto principal do LDAP é que um servidor configurado para o LDAP pode ser completamente *standalone*, além de suportar outros OS q não são o windows, tipo o OSX e qualquer outro OS unix-like, como os OS q usam o kernel Linux, por exemplo.
## Como funciona a auditoria do LDAP
O LDAP gera uma quantidade de logs muito intensa, o que pode até dificultar o processo de identificação de atividades maliciosas durante uma análise, isso acontece porque como eu disse ali em cima, o LDAP permite que outros recursos externos (aplicações etc) consultem coisas do AD, daí já sabe né?

Por exemplo pode ocorrer de no teu alvo o Outlook ter sido configurado pra ser integrado com o LDAP, e pode ter certeza que isso vai gerar muitas logs, já que o Outlook (quando integrado com o LDAP) faz diversas consultas no LDAP pra obter informações de contatos do Outlook que não foram salvados por uma certa maquina, e também pra carregar informações de uma conta.

Pro azar do hacker leigo, o Windows pode promover uma auditoria host-based do LDAP com o ETW (**Event Tracing for Windows**) facilmente:

- Usando o provedor `Microsoft-Windows-LDAP-Client`:
```powershell
wevtutil set-log "Microsoft-Windows-LDAP-Client/Debug" /enabled:true /quiet:true /retention:false /maxsize:100032
```
- O provedor `Microsoft-Windows-LDAP-Client` do ETW faz quando a `wldap32.dll` (api do LDAP) ser chamada (geralmente usam justamente a `wldap32.dll` pra fazer consultas no LDAP), gerar uma log, futuramente vou mostrar como burlar isso de 2 formas.
---
- É possivel fazer uma auditoria do LDAP sniffando todo o tráfego do LDAP, também com o ETW.
---
- E pode fazer uma auditoria do LDAP habilitando o `Microsoft-Windows-ActiveDirectory_DomainService` no Event Viewer.
## Executando consultas sem ser detectado
- O primeiro método que eu vou apresentar, contra o provedor `Microsoft-Windows-LDAP-Client`, vai ser patchear o ETW (no nosso caso user-mode), sim, isso mesmo. A ideia é localizar o memory address de uma função especifica do provedor q vc queira patchear (`EtwEventWrite`, `NtTraceEvent`, etc...), e adicionar uma instrução específica.

- O segundo método contra o `Microsoft-Windows-LDAP-Client` é usar uma tool como o [BloodHound.py](https://github.com/dirkjanm/BloodHound.py), que não usa a `wldap32.dll`

- Agora pra burlar o packet tracing do ETW (sniffar as conexões do LDAP), você pode usar essa tool, muito boa inclusive, chamada de: [bloodyAD](https://github.com/CravateRouge/bloodyAD).
- Para burlar a auditoria do `Microsoft-Windows-ActiveDirectory_DomainService` você também pode usar o [bloodyAD](https://github.com/CravateRouge/bloodyAD).
