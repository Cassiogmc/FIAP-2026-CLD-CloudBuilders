# Lab 02 - Sob o Capô do IaaS: Auditoria de Serviços e Microsserviços no Controller

* **Programa:** MBA em Cloud Computing & Infrastructure
* **Ambiente / Plataforma:** Red Hat Academy (CL110 / Red Hat OpenStack Platform)
* **Stack Técnica:** Red Hat Enterprise Linux 8, Podman Engine, Systemd, TripleO Controller
* **Duração Estimada:** 30 a 35 minutos

---

## 🎯 Objetivo do Lab

Compreender a arquitetura interna de uma control plane moderna de IaaS empresarial, desmistificando como os serviços de infraestrutura (APIs de computação, rede e autenticação) foram conteinerizados para garantir alta disponibilidade e isolamento operacional.

**Cenário Corporativo (SRE Cloud Audit):**  
Antes de liberar o cluster de nuvem privada para homologação dos times de produto, a equipe de SRE e Arquitetura de Plataforma deve realizar uma auditoria de resiliência no nó mestre (`controller0`). Sua missão é acessar a control plane, inspecionar a esteira de contêineres gerenciados pelo Podman e auditar a integridade das APIs do Nova e Neutron.

**Habilidades Conquistadas:**
1. Realizar conexão administrativa segura via chave SSH da workstation para os nós do Overcloud.
2. Inspecionar o ecossistema de microsserviços conteinerizados gerenciados pelo **Podman**.
3. Filtrar e diagnosticar o ciclo de vida dos daemons core (Nova, Neutron, Keystone, Cinder).
4. Rastrear chamadas e exceções em tempo real inspecionando logs de contêineres de sistema.
5. Analisar o mecanismo de supervisão e self-healing via unidades de serviço do **Systemd**.
6. Executar o encerramento canônico de laboratórios no ambiente Red Hat Academy.

---

## 📋 Pré-requisitos & Materiais

* Acesso ao terminal da estação gráfica `workstation` (CL110).
* Chave SSH pré-configurada para o usuário administrativo `heat-admin`.
* Privilégios de superusuário (`sudo`) no nó `controller0`.

---

## 🚀 Passo a Passo Guiado

### Passo 1: Acesso Administrativo ao Nó Controller

A partir do terminal da `workstation`, estabeleça uma conexão SSH segura com o nó controladora do Overcloud (`controller0`):

```bash
ssh heat-admin@controller0
```

Eleve seus privilégios para o usuário `root` para ter visibilidade total sobre o runtime do Podman:

```bash
sudo -i
```

> **Por que `heat-admin`?**  
> No Red Hat OpenStack Platform (RHOSP), o provisionador mestre (Director / Undercloud) utiliza o Heat e o Ansible para injetar a chave do usuário `heat-admin` em todos os nós da infraestrutura, eliminando logins diretos com senhas compartilhadas.

---

### Passo 2: Auditoria Geral de Contêineres de Sistema (Podman)

Diferente de instalações antigas de OpenStack onde os serviços rodavam como daemons soltos no sistema operacional do host, o RHOSP moderno encapsula cada serviço dentro de contêineres OCI via **Podman**.

Liste todos os contêineres em execução no nó:

```bash
podman ps --format "table {{.Names}} {{.Status}} {{.Image}}"
```

Observe a quantidade de microsserviços especializados que compõem uma control plane: bancos relacionais (MariaDB/Galera), filas de mensagens assíncronas (RabbitMQ) e APIs de infraestrutura.

---

### Passo 3: Filtragem Específica dos Daemons Core

Para auditar apenas os subsistemas críticos responsáveis por computação, rede e identidade, aplique uma filtragem por expressão regular:

```bash
podman ps --format "table {{.Names}}  {{.Status}}" | grep -E "nova|neutron|keystone"
```

> **Saída Esperada:**
> ```text
> nova_api_cron          Up 29 minutes ago
> nova_metadata          Up 29 minutes ago
> nova_api               Up 29 minutes ago
> nova_vnc_proxy         Up 29 minutes ago
> nova_scheduler         Up 29 minutes ago
> nova_conductor         Up 28 minutes ago
> neutron_api            Up 29 minutes ago
> keystone               Up 29 minutes ago
> ```

---

### Passo 4: Diagnóstico e Inspeção de Logs em Tempo Real

Caso uma requisição de criação de máquina falhe no Nova ou a criação de rede falhe no Neutron, o primeiro ponto de telemetria é o log do contêiner da respectiva API.

Inspecione as últimas 30 linhas de eventos da API do Nova:

```bash
podman logs --tail 30 nova_api
```

Em seguida, verifique os eventos de tráfego e requisições de rede no daemon do Neutron:

```bash
podman logs --tail 30 neutron_api
```

> **Conceito Chave:**  
> O isolamento de processos impede falhas em cascata: mesmo que a API do Horizon sofra um ataque ou pico de carga, os contêineres de virtualização (KVM) e as regras de firewall (OVS/Neutron) continuam operando sem degradação.

---

### Passo 5: Supervisão de Resiliência via Systemd

No RHOSP, os contêineres do Podman não rodam soltos; eles são gerenciados por unidades de serviço do **Systemd** (`tripleo_*`), garantindo inicialização ordenada e recuperação automática após reboots:

```bash
systemctl status tripleo_nova_api.service --no-pager
```

> **Saída Esperada:**
> ```text
> ● tripleo_nova_api.service - nova_api container
>    Loaded: loaded (/etc/systemd/system/tripleo_nova_api.service; enabled; vendor preset: disabled)
>    Active: active (running) since ...
>  Main PID: 5613 (conmon)
>    CGroup: /system.slice/tripleo_nova_api.service
>            └─5613 /usr/bin/conmon ...
> ```

> **O papel do `conmon` (Container Monitor):**  
> Observe que o processo gerenciado pelo Systemd é o `/usr/bin/conmon`. O Podman adota uma arquitetura *daemonless* (sem um processo central como o dockerd). Cada contêiner recebe um monitor `conmon` dedicado, permitindo que o Systemd gerencie o ciclo de vida, colete logs e realize auto-recuperação (self-healing) de forma isolada.

---

### Passo 6: Desconexão e Retorno à Workstation

Saia da sessão privilegiada de `root`, encerre a conexão SSH com o nó `controller0` e retorne ao terminal da sua `workstation`:

```bash
exit
exit
```

Valide que você retornou com segurança ao prompt da sua estação de gerenciamento:

```bash
whoami; hostname
```

> **Saída Esperada:**
> ```text
> student
> workstation.lab.example.com
> ```

---

## 🧪 Validação & Critérios de Aceite

O laboratório é considerado concluído com sucesso quando:
* O aluno estabelece com sucesso a sessão administrativa SSH no nó `controller0`.
* A inspeção via `podman ps` confirma que os daemons core (`nova_api`, `neutron_api` e `keystone`) estão com status `Up`.
* A telemetria de logs (`podman logs`) confirma que as APIs do Nova e Neutron estão ativas e processando chamadas sem exceções de runtime.
* O comando `systemctl status` valida a supervisão pelo Systemd e o papel do monitor de contêiner (`conmon`).
* As sessões remotas são devidamente finalizadas (`exit`), garantindo retorno limpo à workstation.

---

## 🧹 Cleanup

Como o Lab 02 realiza auditoria somente leitura (*read-only*) no nó mestre, não há alocação de recursos persistentes no cluster. O encerramento ordenado das sessões SSH (`exit`) conclui a liberação do ambiente.

---

## 💡 Desafios Complementares (Para Alunos Avançados)

1. **Inspeção de Volumes e Configurações Mapeadas:**  
   Execute `podman inspect nova_api | grep -A 10 Mounts` no `controller0` para entender como os arquivos de configuração do host (`/var/lib/config-data/...`) são injetados em modo somente leitura (*read-only*) dentro do contêiner.
2. **Auditoria de Nós de Computação:**  
   A partir da workstation, conecte-se via SSH em `compute0` (`ssh heat-admin@compute0`) e verifique quais contêineres rodam em um nó hipervisor (ex: `nova_compute` e `neutron_ovs_agent`).
