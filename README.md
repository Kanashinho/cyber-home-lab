# Laboratório Isolado de Cibersegurança — Wazuh (SIEM), Ubuntu Server e Metasploitable

Este repositório documenta a arquitetura, implementação e resolução de problemas (Troubleshooting) do meu laboratório isolado de cibersegurança, utilizando Wazuh (SIEM), Ubuntu Server e Metasploitable, com operação remota via SSH a partir de um notebook dedicado.

## Objetivo

Construir um ambiente controlado e segmentado para praticar operações de SOC (Security Operations Center), monitoramento de logs, detecção de intrusão e testes de invasão sem expor ativos vulneráveis à internet.

## Arquitetura e Topologia de Rede

```
[Notebook Ubuntu (Atacante/Analista)]
       │  Interface: eno2 (10.0.0.2/30)
       │  Conexão física: Cabo Ethernet + Adaptador USB GbE
       ▼
[PC Principal - Windows Host (Roteador L3 / NetNat)]
       ├── Interface USB GbE: 10.0.0.1/30 (Forwarding Enabled)
       ├── VirtualBox Host-Only Adapter: 192.168.56.1/24
       └── Disco secundário (D:) → VDIs dedicados do Wazuh (/var/ossec/queue e /var/ossec/logs)
                │
                ├── Wazuh SIEM (OVA Oficial)  ◄── eventos ── agente (Ubuntu Server)
                ├── Ubuntu Server (Alvo Monitorado, com wazuh-agent)
                └── Metasploitable 2 (Alvo Legado)
```

> **Nota de OpSec:** Todos os endereços IP documentados neste repositório são ilustrativos (RFC 1918) para fins didáticos.

- **PC Principal (Windows, 16GB RAM):** Hospeda as VMs no Oracle VirtualBox e atua como roteador de Camada 3 entre o enlace físico dedicado e o switch virtual *Host-Only*.
- **Notebook Ubuntu:** Máquina de operação e ataque conectada fisicamente via cabo Ethernet e adaptador USB.
- **Isolamento:** As máquinas virtuais comunicam-se na rede *Host-Only*, isoladas da rede doméstica principal.
- **Monitoramento:** O Ubuntu Server reporta ao Wazuh via `wazuh-agent`; é o `auth.log` do SSH que alimenta as regras do grupo `sshd`.

---

## Diário de Troubleshooting e Engenharia de Rede

Durante a evolução do laboratório, enfrentei e solucionei seis desafios reais de infraestrutura, redes e operação de SIEM:

### 1. Roteamento Camada 3 e Assimetria de Rota entre Notebook e VMs

**Problema:** O acesso SSH funcionava a partir do PC Principal, mas apresentava `100% packet loss` e falha de resolução ARP (`FAILED`) a partir do notebook conectado via cabo Ethernet USB.

**Diagnóstico:**

1. O adaptador *Host-Only* do VirtualBox opera como um switch virtual isolado no host, sem ponte de Camada 2 automática para a placa física USB.
2. Após alinhar o enlace físico em uma sub-rede ponto-a-ponto dedicada (`/30`) e ativar o encaminhamento de pacotes no Windows (`IPEnableRouter`), os pacotes ICMP chegavam às VMs, mas o retorno era perdido por **assimetria de rota** (as VMs não possuíam rota de retorno para a sub-rede `/30` do cabo).

**Solução:**

- **No Host Windows:** Ativação do roteamento IP no registro (`IPEnableRouter = 1`), habilitação do `Forwarding` nas interfaces via PowerShell e configuração de tradução de endereços com `New-NetNat` para a sub-rede `/30`, além de regras específicas de Firewall:

  ```
  Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" -Name "IPEnableRouter" -Value 1
  Set-NetIPInterface -InterfaceAlias "Ethernet 3","Ethernet 2" -Forwarding Enabled
  New-NetNat -Name "Lab_HostOnly_NAT" -InternalIPInterfaceAddressPrefix 10.0.0.0/30
  ```

- **No Notebook Ubuntu:** Padronização da persistência de rede via **Netplan** (`/etc/netplan/01-network-manager-all.yaml`) definindo explicitamente o `renderer: NetworkManager`, IP estático `/30` na interface cabeada e rota estática para a rede das VMs (`192.168.56.0/24 via 10.0.0.1`), eliminando conflitos no boot com o `systemd-networkd`:

  ```
  network:
    version: 2
    renderer: NetworkManager
    ethernets:
      eno2:
        dhcp4: no
        addresses:
          - 10.0.0.2/30
        routes:
          - to: 192.168.56.0/24
            via: 10.0.0.1
  ```

---

### 2. Inicialização da Interface `eth1` e Acesso ao Dashboard no Wazuh (OVA)

**Problema:** Perda recorrente de acesso SSH e ao Dashboard web do Wazuh após reiniciar a VM. Embora o arquivo `/etc/sysconfig/network-scripts/ifcfg-eth1` estivesse configurado com `ONBOOT=yes`, o script não subia o IP do laboratório durante o boot, exigindo ativação manual via terminal local da VM. Além disso, o sistema operacional da OVA oficial não utiliza `NetworkManager`.

**Diagnóstico:** Sistemas baseados em CentOS/Amazon Linux utilizados na OVA do Wazuh dependem do serviço legado `network.service` e sofrem interferência do `cloud-init`, que pode redefinir configurações de interfaces secundárias se o endereço físico (MAC Address) não estiver explicitamente vinculado ao script de boot.

**Solução:**

1. Vinculação forçada do MAC Address (`HWADDR`) e desativação do controle pelo NetworkManager diretamente no `/etc/sysconfig/network-scripts/ifcfg-eth1`:

   ```
   DEVICE=eth1
   BOOTPROTO=static
   ONBOOT=yes
   NM_CONTROLLED=no
   HWADDR=08:00:27:XX:XX:XX
   IPADDR=192.168.56.10
   NETMASK=255.255.255.0
   ```

2. Bloqueio de sobrescrita de rede pelo `cloud-init` criando `/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg`:

   ```
   network: {config: disabled}
   ```

3. Habilitação persistente do serviço legado de rede no boot:

   ```
   sudo systemctl enable network
   sudo systemctl restart network
   ```

---

### 3. Conflito de IP Duplicado na Interface `eth0`

**Problema:** Ao configurar os adaptadores no Wazuh (Adaptador 1 Host-Only e Adaptador 2 NAT), a interface `eth0` reteve dois endereços IP simultaneamente.

**Solução:** Limpeza da tabela de endereços da interface e reinicialização do serviço de rede:

```
sudo ip addr flush dev eth0
sudo systemctl restart network
```

---

### 4. Compatibilidade Criptográfica SSH com Sistemas Legados (Metasploitable)

**Problema:** O cliente OpenSSH moderno do Ubuntu recusava conexão com o Metasploitable (2008) por depreciação dos algoritmos antigos de troca de chaves e cifras.

**Solução:** Habilitação seletiva dos algoritmos legados para o host específico e criação de atalhos no `~/.ssh/config`:

```
Host wazuh
    HostName 192.168.56.10
    User wazuh-user

Host alvo
    HostName 192.168.56.11
    User usuario

Host metasploitable
    HostName 192.168.56.12
    User msfadmin
    KexAlgorithms +diffie-hellman-group1-sha1
    Ciphers +aes128-cbc
    HostKeyAlgorithms +ssh-rsa
```

Agora o acesso a qualquer VM é feito diretamente via:

```
ssh wazuh
ssh alvo
ssh metasploitable
```

---

### 5. Esgotamento de Disco no Wazuh e Indexer em Modo Somente Leitura

**Problema:** O disco principal da VM (`/dev/sda1`, 25 GB) chegou a 100% de uso. O principal ofensor foi o diretório `/var/ossec/queue`, que acumulou cerca de 20 GB. Sem espaço, o Wazuh Indexer ativou a trava de segurança (*flood stage watermark*) e colocou os índices em modo somente leitura, impedindo a ingestão de novos dados. Os gráficos de Events, MITRE ATT&CK e Compliance ficaram zerados.

**Causa raiz:** o crescimento veio do módulo *Vulnerability Detector*. Medido no volume novo (`du -sh /var/ossec/queue/*`), de 23 GB consumidos em `/var/ossec/queue`, 12 GB estão em `vd_updater` e 9,4 GB em `vd` (21,4 GB, cerca de 93% do total). O restante: 1,1 GB em `indexer`, 75 MB em `harvester` e menos de 9 MB somados em `db`, `fim`, `syscollector` e `alerts`. Esses diretórios guardam localmente os feeds de vulnerabilidades (CVEs) usados para cruzar com os pacotes dos agentes. As bases são grandes e atualizadas constantemente, o que é inviável em um disco primário de 25 GB sem partição dedicada.

**Diagnóstico (camadas do problema):**

1. **Shadowing de montagem:** ao montar um disco novo de 50 GB (`/dev/sdc`) sobre `/var/ossec/queue`, os 20 GB originais ficaram "soterrados" sob o ponto de montagem. O disco raiz continuava 100% cheio, mas os arquivos não eram visíveis aos comandos normais (`du`/`ls`).
2. **Permissões:** tentativas de apagar os dados com o usuário padrão (`wazuh-user`) falharam por falta de permissão de leitura na pasta do Wazuh.
3. **Arquivos fantasma:** mesmo após a deleção, processos ativos mantinham descritores abertos para arquivos já removidos, e o espaço físico não era devolvido.

**Solução:**

1. **Segmentação de armazenamento:** novo disco VDI de 50 GB criado no disco secundário (`D:`) do hospedeiro, anexado à VM e formatado em `ext4` para absorver exclusivamente a carga de `/var/ossec/queue`.
   Persistência no boot via `/etc/fstab`, com um volume dedicado para `/var/ossec/queue` e outro para `/var/ossec/logs`:

   ```
   # /etc/fstab
   /dev/sdb  /var/ossec/logs   ext4  defaults  0  2
   /dev/sdc  /var/ossec/queue  ext4  defaults  0  2
   ```

   Validação: `findmnt /var/ossec/queue` confirma a montagem ativa de `/dev/sdc` (`ext4`).

2. **Acesso ao disco original via *bind mount*:** com `sudo -i` e `mount --bind / /mnt`, foi possível enxergar o disco raiz sem a sobreposição do novo ponto de montagem:

   ```
   sudo -i
   mount --bind / /mnt
   # os dados soterrados ficam visíveis em /mnt/var/ossec/queue
   ```

3. **Limpeza:** remoção da "âncora" de 20 GB pelo diretório espelhado (`/mnt/var/ossec/queue`). O uso do disco principal caiu de 100% para 22%.
4. **Reset da stack:** reinício de todos os serviços (manager, indexer, dashboard e filebeat) para liberar os descritores órfãos e permitir que o sistema desarmasse a trava de segurança automaticamente, retomando a comunicação com o agente do Ubuntu Server.

**Lição:** montar um volume novo sobre um diretório que já contém dados não move nada; os dados antigos continuam ocupando o disco de origem, invisíveis. O `bind mount` da raiz é a forma de alcançá-los.

---

### 6. Dashboard Exibindo "No results" Após a Correção do Disco

**Problema:** Mesmo com o indexer funcionando, os painéis continuaram mostrando "No results".

**Causa:** cliques nos gráficos ativaram filtros de tempo minúsculos (ex.: zoom em 1 segundo específico) e criaram regras de busca restritivas, forçando a consulta a um período em que o motor estava desligado.

**Solução:** remoção das etiquetas (tags) de filtro acumuladas na barra superior e ajuste do relógio global para "Last 24 hours". Boa prática adotada: para investigar incidentes, usar os menus superiores de intervalo de tempo em vez de clicar na linha do tempo dos gráficos.

---

## Validação do Agente Wazuh no Ubuntu Server (pré-requisito do ataque)

Antes de qualquer teste de força bruta, confirmo que o Ubuntu Server está realmente reportando ao SIEM. Sem isso, o ataque não gera evidência nenhuma.

- **No Ubuntu Server:**

  ```
  sudo systemctl status wazuh-agent
  ls -l /var/log/auth.log        # fonte dos eventos de SSH
  ```

- **No Wazuh Manager:**

  ```
  sudo /var/ossec/bin/agent_control -l
  ```

  O agente deve aparecer como `Active`. No Dashboard: *Agents* → status **Active**.

- **Checagem de ponta a ponta:** um login SSH com senha errada, feito manualmente, deve aparecer em *Threat Hunting* em instantes (regras do grupo `sshd`).

- **Versão do agente:** 4.14.7, rodando no Ubuntu Server 26.04.1 LTS.
- **Instalação:** via repositório APT oficial do Wazuh (chave GPG + repositório), com `apt-get install wazuh-agent` apontando para o IP do manager.

---

## Principais Aprendizados

- Diagnóstico estruturado em camadas (Modelo OSI: Enlace/ARP, Rede/Roteamento IP, Transporte/Firewall e Aplicação/SSH).
- Configuração de roteamento L3 no Windows (`IPEnableRouter`, `NetIPInterface Forwarding` e `NetNat`) integrando adaptadores físicos USB a redes virtuais *Host-Only*.
- Gerenciamento de persistência de rotas no Linux com `Netplan`, `NetworkManager` e scripts `ifcfg` legados sem `NetworkManager` (vinculação de `HWADDR`/MAC).
- Resolução de assimetria de rotas e conflitos entre `NetworkManager` e `systemd-networkd`.
- Capacity planning de SIEM: monitorar uso de disco do manager/indexer, entender o *flood stage watermark* do indexer e separar o armazenamento de eventos em volume dedicado.
- Diagnóstico de armazenamento no Linux: shadowing de ponto de montagem, `bind mount` para enxergar dados soterrados e descritores de arquivos deletados mantidos por processos.
- Higiene operacional no Dashboard: filtros e intervalo de tempo podem simular "ausência de dados".

## Próximos Passos

Este laboratório é a base do meu próximo projeto: **mini-SOC com detecção de força bruta SSH em tempo real, regras customizadas no Wazuh e resposta ativa automatizada em Python**.

Plano de execução:

1. Confirmar o agente do Ubuntu Server como `Active` (seção acima).
2. Executar o ataque de força bruta SSH a partir do notebook contra o Ubuntu Server (ambiente próprio e isolado).
3. Registrar quais alertas das regras nativas do grupo `sshd` já disparam (ID da regra, nível, tempo até o alerta).
4. Identificar lacunas: o que o ataque fez que nenhuma regra nativa cobriu, e só então escrever regras customizadas para essas lacunas.
5. Implementar a resposta ativa automatizada em Python.
6. Definir rotina de monitoramento de uso de disco do manager (alerta antes de atingir o *flood stage* do indexer).

## Aviso Ético

Todos os testes foram realizados exclusivamente em ambiente próprio, isolado e autorizado. Nenhum sistema de terceiros foi acessado ou testado.
