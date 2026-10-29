# Laboratório Isolado de Cibersegurança — Wazuh, Ubuntu Server e Metasploitable

Este repositório documenta a montagem do meu primeiro laboratório isolado de cibersegurança, usando Wazuh (SIEM), Ubuntu Server e Metasploitable, com acesso via SSH a partir de um notebook Ubuntu.

## Objetivo

Criar uma rede isolada para praticar conceitos de SOC, monitoramento de eventos e análise de segurança, sem expor nada à internet.

## Arquitetura do laboratório

```text
[Notebook Ubuntu] --cabo Ethernet + adaptador USB--> [PC principal com VMs]
                                                          ├── Wazuh (OVA oficial)
                                                          ├── Ubuntu Server (alvo)
                                                          └── Metasploitable (alvo legado)
```

- **PC principal (Windows, 16GB RAM):** hospeda as VMs no Oracle VirtualBox.
- **Notebook Ubuntu:** máquina de ataque, conectada por cabo Ethernet via adaptador USB.
- **Rede:** modo *host-only*, totalmente isolada da internet.
- **Acesso:** SSH a partir do notebook para todas as VMs.

## Configuração de rede no Wazuh

Na VM do Wazuh, adicionei dois adaptadores de rede no VirtualBox:

- **Adaptador 1 — Host-only:** para comunicação na rede isolada.
- **Adaptador 2 — NAT:** para acesso à internet durante a instalação e atualizações.

### Problema encontrado: IP duplicado no eth0

Ao configurar o IP e a máscara no arquivo `ifcfg-eth0`, percebi que o adaptador `eth0` estava com dois IPs simultaneamente.

Criei então um arquivo separado, `ifcfg-eth1`, para o segundo adaptador, mas o problema persistiu.

### Solução

Limpei os IPs extras do `eth0` e reiniciei o serviço de rede:

```bash
sudo ip addr flush dev eth0
sudo systemctl restart network
```

Depois disso, a conexão via SSH do notebook funcionou corretamente.

## Compatibilidade SSH com Metasploitable

A VM Metasploitable é de 2008 e meu notebook roda uma versão moderna do Ubuntu. Na tentativa de conectar via SSH, o servidor recusava a conexão por incompatibilidade de algoritmos de criptografia e troca de chaves.

### Solução

Reativei algoritmos legados diretamente no comando SSH:

```bash
ssh -oKexAlgorithms=+diffie-hellman-group1-sha1 \
    -oCiphers=+aes128-cbc \
    -oHostKeyAlgorithms=+ssh-rsa \
    usuario@192.168.56.x
```

> **Nota:** os IPs acima são exemplos. Use os IPs reais da sua rede isolada.

## Atalhos com SSH config

Para facilitar o acesso, criei o arquivo `~/.ssh/config`:

```text
Host wazuh
    HostName 192.168.56.x
    User usuario

Host alvo
    HostName 192.168.56.x
    User usuario

Host metasploitable
    HostName 192.168.56.x
    User msfadmin
```

Agora acesso qualquer VM com:

```bash
ssh wazuh
ssh alvo
ssh metasploitable
```

## O que aprendi

- Diferença entre modos de rede em VMs (host-only vs NAT).
- Como diagnosticar e corrigir IPs duplicados em interfaces de rede.
- Como lidar com incompatibilidades de criptografia em sistemas legados.
- Como organizar acessos SSH de forma prática e segura.
- Importância de manter ambientes de teste isolados.

## Próximos passos

Este laboratório é a base do meu próximo projeto: **mini-SOC com detecção de força bruta SSH, regras customizadas no Wazuh e automação em Python**.

## Aviso ético

Todos os testes foram realizados exclusivamente em ambiente próprio, isolado e autorizado. Nenhum sistema de terceiros foi acessado ou testado.
