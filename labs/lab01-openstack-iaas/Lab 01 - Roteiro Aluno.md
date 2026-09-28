# Lab 01 - Decolagem IaaS no OpenStack: Provisionamento e Governança Enterprise

* **Programa:** MBA em Cloud Computing & Infrastructure
* **Ambiente / Plataforma:** Red Hat Academy (CL110 / Red Hat OpenStack Platform)
* **Stack Técnica:** OpenStack CLI (`python-openstackclient`), Horizon Dashboard, QEMU/KVM
* **Duração Estimada:** 40 a 45 minutos

---

## 🎯 Objetivo do Lab

Capacitar o aluno a operar uma nuvem privada corporativa, compreendendo o ciclo de vida completo de uma instância IaaS (computação, redes SDN e armazenamento em blocos). 

**Cenário Corporativo (FinCorp):**  
A equipe de tesouraria da *FinCorp* necessita de uma instância Linux isolada (`finance-server1`) para executar rotinas noturnas de conciliação bancária. A máquina precisa ser instanciada na rede interna da controladoria (`finance-network1`) e homologada tanto via linha de comando quanto via console visual corporativa (Horizon).

**Habilidades Conquistadas:**
1. Inicializar cenários de laboratório controlados na workstation de gerenciamento.
2. Autenticar na control plane do OpenStack exportando variáveis de ambiente de projeto (`developer1-finance-rc`).
3. Auditar catálogos de recursos computacionais (Flavors) e imagens de sistema operacional (Glance).
4. Provisionar instâncias virtuais via CLI garantindo vinculação correta à SDN (Neutron).
5. Inspecionar o console de boot e diagnosticar eventos do hipervisor (Nova).
6. Validar graficamente a topologia de instâncias no Horizon Dashboard.

---

## 📋 Pré-requisitos & Materiais

* Acesso à estação gráfica `workstation` do curso CL110 na Red Hat Academy.
* Credenciais de acesso ao sistema operacional:
  * **Usuário:** `student`
  * **Senha:** `student`
* Credenciais de acesso ao OpenStack (CLI e Horizon):
  * **Domínio:** `Example`
  * **Usuário:** `developer1` (ou `admin` conforme a etapa)
  * **Senha:** `redhat`
* URL do Horizon Dashboard (via Firefox da workstation):
  * `http://dashboard.overcloud.example.com`

---

## 🚀 Passo a Passo Guiado

### Passo 1: Inicialização do Laboratório no Terminal

Abra o terminal na estação `workstation` e dispare o script automatizado da Red Hat Academy que prepara as redes e os catálogos necessários para o exercício:

```bash
lab intro-launching start
```

> **Saída Esperada:**
> ```text
> Starting lab exercise...
> Preparing environment for intro-launching...
> SUCCESS: Environment is ready for the lab.
> ```

---

### Passo 2: Autenticação na Control Plane via CLI

No OpenStack, comandos de CLI comunicam-se com a API REST do serviço **Keystone**. Para que a ferramenta `openstack` saiba qual endpoint e credencial utilizar, carregamos o arquivo com as variáveis de ambiente gerado pelo script para o seu usuário:

```bash
source ~/developer1-finance-rc
```

Valide se o token de sessão foi emitido consultando o seu perfil autenticado:

```bash
openstack token issue
```

> **Por que isso é necessário?**  
> Em produção, engenheiros de nuvem utilizam arquivos RC (`openrc.sh`) ou credenciais baseadas em Application Credentials para autenticação automatizada em esteiras CI/CD e Terraform.

---

### Passo 3: Auditoria dos Catálogos de Recursos

Antes de disparar a criação de qualquer máquina virtual, precisamos inspecionar os recursos disponíveis:

1. **Catálogo de Imagens (Glance):**
   ```bash
   openstack image list
   ```
   *Identifique a imagem corporativa `rhel8` que será a base da nossa instância.*

2. **Dimensionamento de Hardware Virtual (Nova Flavors):**
   ```bash
   openstack flavor list
   ```
   *Observe o flavor `default` verificando alocação de vCPUs, memória RAM e disco raiz.*

3. **Segmentação de Rede Privada (Neutron Networks):**
   ```bash
   openstack network list
   ```
   *Copie o nome ou UUID da rede `finance-network1` destinada ao projeto financeiro.*

---

### Passo 4: Provisionamento da Instância Virtual (`finance-server1`)

Execute a ordem de provisionamento na API do Nova, informando imagem, perfil de máquina e a rede isolada:

```bash
openstack server create \
  --image rhel8 \
  --flavor default \
  --nic net-id=finance-network1 \
  --wait finance-server1
```

> **O que o `--wait` faz?**  
> A flag bloqueia o terminal exibindo a transição de estado da VM de `BUILD` para `ACTIVE`, garantindo que o hipervisor KVM concluiu o agendamento (*Nova Scheduler*) e o anexo do disco.

---

### Passo 5: Inspeção do Boot e Diagnóstico Operacional

Verifique os detalhes da instância recém-criada e inspecione a saída do console serial para comprovar a subida do kernel Linux:

1. **Listar Instâncias Ativas:**
   ```bash
   openstack server list
   ```

2. **Inspecionar Log de Boot da Máquina:**
   ```bash
   openstack console log show finance-server1 | tail -n 25
   ```

> **Saída Esperada:**
> Mensagens do `cloud-init` e o prompt final de login do Red Hat Enterprise Linux 8 indicando que a VM concluiu o boot com sucesso.

---

### Passo 6: Validação Visual no Horizon Dashboard

1. Na `workstation`, abra o navegador web **Firefox**.
2. Acesse o endereço institucional:  
   `http://dashboard.overcloud.example.com`
3. Preencha os parâmetros de autenticação:
   * **Domain:** `Example`
   * **User Name:** `developer1`
   * **Password:** `redhat`
4. No menu lateral esquerdo, navegue até: **Project $\rightarrow$ Compute $\rightarrow$ Instances**.
5. Localize a máquina `finance-server1`, confirme o status `Active`, a imagem associada e o IP privado alocado.
6. Clique sobre o nome da máquina e explore a aba **Console** para ver o terminal gráfico da VM em tempo real.

---

## 🧪 Validação & Critérios de Aceite

O laboratório é considerado concluído com sucesso quando:
* O comando `openstack server list` retornar a máquina `finance-server1` com status `ACTIVE` e `Power State: Running`.
* O Horizon Dashboard exibir a VM vinculada à rede `finance-network1`.
* O log de console comprovar a execução do `cloud-init` sem erros de alocação de storage.

---

## 🧹 Cleanup & Liberação de Recursos

Para liberar os recursos computacionais alocados e restaurar as redes do projeto para o estado inicial:

```bash
lab intro-launching finish
```

> **Saída Esperada:**
> ```text
> Cleaning up the lab on workstation:
> 
> . Deleting instances: ........................ SUCCESS
> . Deleting subnet: finance-subnet1............. SUCCESS
> . Deleting network: finance-network1........... SUCCESS
> ```

---

## 💡 Desafios Complementares (Para Alunos Avançados)

1. **Atribuição de Floating IP:**  
   Descubra como alocar um IP público da rede externa (`public-network`) e associá-lo a uma nova instância para permitir conectividade SSH direta fora do cluster.
2. **Inspeção de Portas SDN:**  
   Utilize o comando `openstack port list --server finance-server1` para auditar a interface virtual (vNIC), o endereço MAC e as regras de segurança (*Security Groups*) vinculadas à máquina.
