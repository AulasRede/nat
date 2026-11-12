# Roteiro de Atividade – NAT (Network Address Translation)

## 1) Objetivos de Aprendizagem
- Compreender o que é NAT, seus tipos e por que ele é usado em redes.
- Identificar como sockets (IP:porta) mudam antes e depois da tradução.
- Observar, na prática, a ação de **PAT/NAPT** (tradução por porta) em conexões de saída.
- Configurar uma **regra de encaminhamento de portas (DNAT/Port Forwarding)** em um roteador doméstico simulado.
- Registrar evidências e documentar mapeamentos NAT.

---

## 2) Resumo Teórico: Como o NAT Funciona

### 2.1 O problema que o NAT resolve  
- Escassez de endereços IPv4 e necessidade de **compartilhar um único IP público** entre vários dispositivos internos.  
- Ocultação/isolamento parcial da rede interna (efeito colateral útil para segurança).

### 2.2 Conceitos fundamentais  
- **Endereços privados** (RFC 1918): não roteáveis na Internet (ex.: 192.168.0.0/16).  
- **IP público**: endereços roteáveis na Internet, atribuídos por ISPs.  
- **Socket**: par `IP:porta` que identifica uma extremidade de conexão TCP/UDP.  
- **Tabela de tradução NAT**: mapeia sockets “internos” ↔ “públicos”.

### 2.3 Tipos de NAT (visão prática)
- **SNAT (Source NAT)**: altera o **IP de origem** (geralmente de privado → público) em **conexões de saída**.  
- **DNAT (Destination NAT / Port Forwarding)**: altera o **IP/porta de destino** para alcançar um host **interno** a partir da Internet (acessos de entrada).  
- **PAT / NAPT (Port Address Translation)**: múltiplos clientes internos compartilham **um único IP público**, diferenciados por **portas de origem traduzidas** (também chamado “NAT Overload”).

### 2.4 Exemplo (tráfego de saída – PAT/NAPT)
```
Antes do NAT (LAN)             Roteador NAT                Internet
192.168.0.20:53124  ──►  200.100.50.10:49152  ──►  203.0.113.80:443
          origem:porta         origem:porta                     destino:porta
```
- O host interno (192.168.0.20:53124) tem seu **socket de origem** traduzido para (200.100.50.10:49152).  
- A tabela NAT registra esse mapeamento para que as **respostas** voltem ao cliente correto.

### 2.5 Exemplo (tráfego de entrada – DNAT/Port Forwarding)
```
Internet         Roteador NAT (regra DNAT)          Host Interno
x.y.z.w:3389  ──► 200.100.50.10:3389 ──► 192.168.0.10:3389
```
- Uma porta do IP público (3389/TCP) é **encaminhada** para um host interno específico (192.168.0.10:3389).

### 2.6 Benefícios e limitações  
**Benefícios**  
- Conserva IPv4; facilita compartilhamento do acesso; adiciona uma camada de “ofuscação” da topologia interna.  

**Limitações/efeitos colaterais**  
- Complica conexões **inbound** (é preciso DNAT/port forwarding).  
- Pode exigir **ALGs**/recursos especiais para aplicativos P2P, VoIP, IPSec.  
- **CGNAT** (Carrier-Grade NAT no provedor) pode impedir **port forwarding** no roteador do cliente.

> **Nota sobre CGNAT:** Se o seu IP “público” no roteador não for roteável na Internet (ou mudar após reboot/duplo NAT), você pode estar atrás de CGNAT do ISP. Nesse cenário, acessos de entrada pela Internet **não** funcionarão apenas com port forwarding local.

---

## 3) Pré-requisitos e materiais
- Computador com acesso à Internet e navegador.
- Computador Windows com o **Windows PowerShell** e os comandos `ping`, `netstat`, `findstr` e `curl.exe` disponíveis. A alternativa com navegador requer acesso ao site pelo navegador.
- Emulador de roteador **TP-Link AC1750** (fornecido no enunciado/orientação da aula).

---

## 4) Atividade 1 — Observando a Tradução NAT (Saída)

### 4.1 O que você deve fazer
1. Abra uma janela do **Windows PowerShell** e descubra o IPv4 do servidor:

   ```powershell
   ping -4 nat.aulasrede.com.br
   ```

   Anote o endereço IPv4 entre colchetes na primeira linha (por exemplo, `203.0.113.80`). Mesmo que o ping não receba resposta ou indique que o destino está inacessível, o endereço resolvido entre colchetes ainda pode aparecer.

2. Na mesma janela do PowerShell, inicie o monitoramento **antes** de abrir o site. No comando abaixo, substitua `203.0.113.80` pelo IPv4 anotado no passo 1:

   ```powershell
   while ($true) {
       netstat -ano -p tcp | findstr /C:"203.0.113.80:443"
       Start-Sleep -Seconds 1
   }
   ```

   Deixe essa janela aberta. O laço executa uma consulta `netstat` por vez, filtra sua saída e aguarda um segundo antes de repetir; assim, cada amostra filtrada aparece durante a execução. Mostra somente as linhas que contêm o IP do servidor e a porta HTTPS `443`. É normal não aparecer nada antes de acessar o site.

3. **Enquanto o monitoramento continua**, gere a conexão em uma segunda janela ou no navegador, escolhendo **uma** destas alternativas:
   - **Navegador:** acesse **https://nat.aulasrede.com.br**. A página mostra o IP público e a porta de origem após a tradução pelo NAT (PAT/NAPT).
   - **curl:** em uma segunda janela do PowerShell, execute o comando abaixo, substituindo `203.0.113.80` pelo IPv4 anotado no passo 1:

     ```powershell
     curl.exe -4 --http1.1 --resolve "nat.aulasrede.com.br:443:203.0.113.80" -i https://nat.aulasrede.com.br/api/getIpInfo
     ```

     `--resolve` direciona esta conexão para o IPv4 anotado, mesmo que a resolução normal do curl escolha outro endereço. A URL permanece com o nome do site, preservando HTTPS e a validação do certificado. Confirme o status HTTP **200** e o cabeçalho `Content-Type: text/plain; charset=utf-8`. A resposta contém os campos de IP e porta, cada um em uma linha; um campo indisponível aparece como `N/A`, sem mensagem nem timestamp. Exemplo (valores ilustrativos):

     ```text
     IP: 203.0.113.42
     Porta: 54321
     ```

4. Observe a saída na primeira janela. Como `findstr` mostra somente a linha correspondente, o cabeçalho original do `netstat` não aparece. Leia cada linha pela posição dos campos; por exemplo:

   ```text
   TCP    192.168.1.25:50123    203.0.113.80:443    ESTABLISHED    1234
   ```

   Da esquerda para a direita: `TCP` é o protocolo; `192.168.1.25:50123` é o **socket local** (IP e porta locais); `203.0.113.80:443` é o **destino** (IP e porta remotos); `ESTABLISHED` é o estado; `1234` é o PID do processo. O `netstat` local não mostra o socket público após o NAT: esse valor deve vir da tela do site ou da resposta do curl.

   `ESTABLISHED` indica uma conexão ativa; `TIME_WAIT` indica uma conexão fechada recentemente, ainda visível no sistema. Outros estados de encerramento, como `FIN_WAIT_1`, também podem aparecer; o PID pode ser `0` em algumas linhas. Várias linhas podem corresponder a conexões distintas, então não associe uma delas à porta pública sem evidência suficiente. Uma resposta do endpoint com campo `N/A` deve ser registrada como exibida; não invente um valor para completar a tabela.

   Uma página pode abrir várias conexões, e o navegador pode escolher outro IP ou usar outro protocolo. Portanto, uma linha do `netstat` não identifica necessariamente, de forma inequívoca, a conexão exibida na página. A opção curl acima força IPv4 e HTTP/1.1 para facilitar a comparação com o filtro TCP. Mesmo com o monitoramento ativo, uma conexão muito curta pode não coincidir com uma amostra; se necessário, repita o acesso enquanto o laço continua rodando.

5. Depois de observar ou capturar a saída, volte à primeira janela do PowerShell e pressione **Ctrl+C** para encerrar o monitoramento.

### 4.2 O que você deve entregar
- **Print 1 — socket externo:** escolha uma alternativa: tela do site **nat.aulasrede.com.br** mostrando o IP/porta após o NAT **ou** saída do `curl.exe` mostrando a resposta com IP/porta.
- **Print 2 — socket local:** captura da primeira janela do PowerShell enquanto o monitoramento está ativo, mostrando a linha correspondente do `netstat`. Se houver várias linhas, inclua-as sem presumir que uma única conexão foi identificada com certeza.
- **Tabela das associações NAT**, seguindo o modelo a seguir:

#### Modelo de Tabela – Associações NAT

| Associação NAT | Socket antes do NAT (local) | Socket após o NAT (público) | Destino |
|---|---|---|---|
| 1 | 192.168.0.20:53124 | **200.100.50.10:49152** | x.y.z.w:443 |
| 2 | 192.168.0.21:60412 | **200.100.50.10:49153** | x.y.z.w:443 |

> **Dica:** o “antes do NAT” é o **socket local** visto pelo seu sistema (LAN); o “após o NAT” é o **socket público** exibido no site ou no corpo da resposta do curl.

---

## 5) Atividade 2 — DNAT/Port Forwarding no Emulador TP-Link AC1750

### 5.1 O que você deve fazer
1. Acesse o [**Emulador AC1750**](https://emulator.tp-link.com/Archer_C7/Index.htm).  
2. Localize a opção **NAT** e verifique se está **habilitada**.  
   - **Print (1):** capture a tela que comprova a configuração.

3. Em **Forwarding** → **Virtual Server**, crie uma **regra de NAT estática (DNAT/Port Forwarding)** para permitir acesso externo via **RDP (Remote Desktop Protocol)** ao host **192.168.0.10**.  
   - **Protocolo/Porta do RDP:** **TCP 3389**.  
   - Sugestões de preenchimento:  
     - **Service Port (External/Public):** 3389  
     - **Internal IP:** 192.168.0.10  
     - **Internal Port:** 3389  
     - **Protocol:** TCP  
     - **Status/Enable:** Ativado

4. **Print (2):** capture a tela mostrando a **regra criada**.

> **Importante:** o emulador não grava as configurações. Não é necessário salvar para esta atividade; basta registrar as telas solicitadas enquanto a configuração estiver visível.

### 5.2 Boas práticas para equipamentos reais
As recomendações abaixo são para uso em um roteador real; não são etapas adicionais a configurar no emulador desta atividade.
- **Reserva de DHCP** para 192.168.0.10 (o IP do host interno não deve mudar).  
- Alterar a **porta externa** (ex.: 53389 → 3389 interno) pode reduzir varreduras automatizadas.  
- Aplicar **restrição de origem** (quando disponível) e **senhas fortes/MFA** no serviço de destino.  
- **UPnP:** manter **desativado**, salvo necessidade explícita.  
- **CGNAT:** se o seu ISP usa CGNAT, o port forwarding pode **não** ser alcançável a partir da Internet pública.

### 5.3 O que você deve entregar
- **Print (1):** NAT habilitado no emulador.  
- **Print (2):** tela da regra de **Virtual Server** para RDP em 192.168.0.10.

---

## 6) Checklist de entrega
Reúna **todas as imagens e a tabela** em **um único documento Word ou PDF** e envie esse documento pelo Teams.
- [ ] **Atividade 1 – Print 1:** site no navegador **ou** saída do `curl.exe`, mostrando o IP/porta após NAT.
- [ ] **Atividade 1 – Print 2:** janela do PowerShell com o monitoramento contínuo e a saída do `netstat` filtrada pelo IP do servidor.
- [ ] **Atividade 1 – Tabela** de associações NAT (modelo acima).
- [ ] **Atividade 2 – Print (1)**: NAT habilitado no emulador AC1750.  
- [ ] **Atividade 2 – Print (2)**: regra de DNAT/Port Forwarding para RDP.

---

## 7) Dicas de Troubleshooting
- **Sem saída no monitoramento?** Confira se substituiu o filtro pelo IPv4 do `ping -4` e se o monitoramento continua aberto; atualize a página ou repita a alternativa curl do passo 3 enquanto ele roda. Essa alternativa usa `--resolve` para conectar ao mesmo IPv4 anotado no ping. Uma conexão curta pode não coincidir com uma amostra. O navegador pode usar outro IP ou HTTP/3 (UDP), que não aparece no filtro TCP. Se aparecerem várias linhas, não presuma que uma delas corresponde inequivocamente à porta pública observada.
- **Portas efêmeras diferentes?** É esperado: o NAT pode escolher portas distintas para cada conexão (PAT).  
- **Não encontro o IP do servidor?** Execute `ping -4 nat.aulasrede.com.br` e anote o IPv4 exibido entre colchetes.
- **Acesso externo não funciona em um equipamento real?** As verificações abaixo aplicam-se a um roteador real, não ao emulador desta atividade.
  - Verifique se a **regra está ativa** e se o **serviço RDP está ouvindo** em 192.168.0.10:3389.  
  - Confirme se o **IP público do roteador** é realmente público (não CGNAT).  
  - Cheque **firewall** do host interno e regras do roteador (WAN → LAN).

---

## 8) Glossário Rápido
- **NAT:** Tradução de endereços de rede (IPv4).  
- **SNAT:** Tradução do endereço **de origem** (saída).  
- **DNAT / Port Forwarding:** Tradução do endereço/porta **de destino** (entrada).  
- **PAT/NAPT:** Tradução por **porta** para múltiplos clientes com um único IP público.  
- **ALG:** Adaptador de camada de aplicação (ex.: SIP-ALG).  
- **CGNAT:** NAT de grande porte no provedor (ISP), compartilhando IP público entre clientes.

---
