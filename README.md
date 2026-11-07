# Laboratório Isolado de Cibersegurança — Wazuh (SIEM), Ubuntu Server e Metasploitable

Este repositório documenta a arquitetura, implementação e resolução de problemas (Troubleshooting) do meu laboratório isolado de cibersegurança, utilizando Wazuh (SIEM), Ubuntu Server e Metasploitable, com operação remota via SSH a partir de um notebook dedicado.

## Objetivo

Construir um ambiente controlado e segmentado para praticar operações de SOC (Security Operations Center), monitoramento de logs, detecção de intrusão e testes de invasão sem expor ativos vulneráveis à internet.

## Arquitetura e Topologia de Rede

```text
[Notebook Ubuntu (Atacante/Analista)]
       │  Interface: eno2 (10.0.0.2/30)
       │  Conexão física: Cabo Ethernet + Adaptador USB GbE
       ▼
[PC Principal - Windows Host (Roteador L3 / NetNat)]
       ├── Interface USB GbE: 10.0.0.1/30 (Forwarding Enabled)
       └── VirtualBox Host-Only Adapter: 192.168.56.1/24
                │
                ├── Wazuh SIEM (OVA Oficial)
                ├── Ubuntu Server (Alvo Monitorado)
                └── Metasploitable 2 (Alvo Legado)
```

> **Nota de OpSec:** Todos os endereços IP documentados neste repositório são ilustrativos (RFC 1918) para fins didáticos.

- **PC Principal (Windows, 16GB RAM):** Hospeda as VMs no Oracle VirtualBox e atua como roteador de Camada 3 entre o enlace físico dedicado e o switch virtual *Host-Only*.
- **Notebook Ubuntu:** Máquina de operação e ataque conectada fisicamente via cabo Ethernet e adaptador USB.
- **Isolamento:** As máquinas virtuais comunicam-se na rede *Host-Only*, isoladas da rede doméstica principal.

---

## Diário de Troubleshooting e Engenharia de Rede

Durante a evolução do laboratório, enfrentei e solucionei quatro desafios reais de infraestrutura e redes:

### 1. Roteamento Camada 3 e Assimetria de Rota entre Notebook e VMs

**Problema:** O acesso SSH funcionava a partir do PC Principal, mas apresentava `100% packet loss` e falha de resolução ARP (`FAILED`) a partir do notebook conectado via cabo Ethernet USB.

**Diagnóstico:**
1. O adaptador *Host-Only* do VirtualBox opera como um switch virtual isolado no host, sem ponte de Camada 2 automática para a placa física USB.
2. Após alinhar o enlace físico em uma sub-rede ponto-a-ponto dedicada (`/30`) e ativar o encaminhamento de pacotes no Windows (`IPEnableRouter`), os pacotes ICMP chegavam às VMs, mas o retorno era perdido por **assimetria de rota** (as VMs não possuíam rota de retorno para a sub-rede `/30` do cabo).

**Solução:**
- **No Host Windows:** Ativação do roteamento IP no registro (`IPEnableRouter = 1`), habilitação do `Forwarding` nas interfaces via PowerShell e configuração de tradução de endereços com `New-NetNat` para a sub-rede `/30`, além de regras específicas de Firewall:
  ```powershell
  Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" -Name "IPEnableRouter" -Value 1
  Set-NetIPInterface -InterfaceAlias "Ethernet 3","Ethernet 2" -Forwarding Enabled
  New-NetNat -Name "Lab_HostOnly_NAT" -InternalIPInterfaceAddressPrefix 10.0.0.0/30
  ```
- **No Notebook Ubuntu:** Padronização da persistência de rede via **Netplan** (`/etc/netplan/01-network-manager-all.yaml`) definindo explicitamente o `renderer: NetworkManager`, IP estático `/30` na interface cabeada e rota estática para a rede das VMs (`192.168.56.0/24 via 10.0.0.1`), eliminando conflitos no boot com o `systemd-networkd`:
  ```yaml
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

**Diagnóstico:**
Sistemas baseados em CentOS/Amazon Linux utilizados na OVA do Wazuh dependem do serviço legado `network.service` e sofrem interferência do `cloud-init`, que pode redefinir configurações de interfaces secundárias se o endereço físico (MAC Address) não estiver explicitamente vinculado ao script de boot.

**Solução:**
1. Vinculação forçada do MAC Address (`HWADDR`) e desativação do controle pelo NetworkManager diretamente no `/etc/sysconfig/network-scripts/ifcfg-eth1`:
   ```ini
   DEVICE=eth1
   BOOTPROTO=static
   ONBOOT=yes
   NM_CONTROLLED=no
   HWADDR=08:00:27:XX:XX:XX
   IPADDR=192.168.56.10
   NETMASK=255.255.255.0
   ```
2. Bloqueio de sobrescrita de rede pelo `cloud-init` criando `/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg`:
   ```yaml
   network: {config: disabled}
   ```
3. Habilitação persistente do serviço legado de rede no boot:
   ```bash
   sudo systemctl enable network
   sudo systemctl restart network
   ```

---

### 3. Conflito de IP Duplicado na Interface `eth0`

**Problema:** Ao configurar os adaptadores no Wazuh (Adaptador 1 Host-Only e Adaptador 2 NAT), a interface `eth0` reteve dois endereços IP simultaneamente.

**Solução:** Limpeza da tabela de endereços da interface e reinicialização do serviço de rede:
```bash
sudo ip addr flush dev eth0
sudo systemctl restart network
```

---

### 4. Compatibilidade Criptográfica SSH com Sistemas Legados (Metasploitable)

**Problema:** O cliente OpenSSH moderno do Ubuntu recusava conexão com o Metasploitable (2008) por depreciação dos algoritmos antigos de troca de chaves e cifras.

**Solução:** Habilitação seletiva dos algoritmos legados para o host específico e criação de atalhos no `~/.ssh/config`:

```text
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
```bash
ssh wazuh
ssh alvo
ssh metasploitable
```

---

## Principais Aprendizados

- Diagnóstico estruturado em camadas (Modelo OSI: Enlace/ARP, Rede/Roteamento IP, Transporte/Firewall e Aplicação/SSH).
- Configuração de roteamento L3 no Windows (`IPEnableRouter`, `NetIPInterface Forwarding` e `NetNat`) integrando adaptadores físicos USB a redes virtuais *Host-Only*.
- Gerenciamento de persistência de rotas no Linux com `Netplan`, `NetworkManager` e scripts `ifcfg` legados sem `NetworkManager` (vinculação de `HWADDR`/MAC).
- Resolução de assimetria de rotas e conflitos entre `NetworkManager` e `systemd-networkd`.

## Próximos Passos

Este laboratório é a base do meu próximo projeto: **mini-SOC com detecção de força bruta SSH em tempo real, regras customizadas no Wazuh e resposta ativa automatizada em Python**.

## Aviso Ético

Todos os testes foram realizados exclusivamente em ambiente próprio, isolado e autorizado. Nenhum sistema de terceiros foi acessado ou testado.
````[cite: 1, 4, 6, 7]
