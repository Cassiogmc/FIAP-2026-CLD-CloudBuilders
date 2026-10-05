# Lab 03 - Decolagem Cloud-Native no Red Hat OpenShift: Deploy Declarativo e Governança Visual

* **Programa:** MBA em Cloud Strategy & Architecture
* **Ambiente / Plataforma:** Red Hat Academy (DO180 v4.14 / Red Hat OpenShift Container Platform 4.14)
* **Stack Técnica:** OpenShift CLI (`oc`), OpenShift Web Console (Topology View), CRI-O, Kubelet
* **Duração Estimada:** 35 a 40 minutos

---

## 🎯 Objetivo do Lab

Capacitar o aluno a operar a plataforma corporativa Red Hat OpenShift, compreendendo o modelo de governança multi-tenant baseado em namespaces e executando o deploy declarativo de microsserviços a partir de imagens de contêiner seguras (*non-root*).

**Cenário Corporativo (FinCorp Platform Engineering):**  
A equipe de engenharia de plataformas da *FinCorp* está migrando seus microsserviços legados de VMs no OpenStack para a plataforma PaaS Red Hat OpenShift. O seu primeiro desafio é provisionar um ambiente isolado (*namespace*) para a squad financeira e realizar o deploy do microsserviço de catálogo web (`my-web-app`), validando a conformidade com as políticas corporativas de segurança (*Security Context Constraints*) e homologando visualmente a carga na interface gráfica do OpenShift Web Console.

**Habilidades Conquistadas:**
1. Autenticar no cluster OpenShift tanto pela interface de linha de comando (`oc login`) quanto pelo Web Console corporativo.
2. Criar e gerenciar projetos isolados (*namespaces*) com controle de acesso integrado.
3. Realizar o deploy ágil de aplicações a partir de registros de imagens corporativas com `oc new-app`.
4. Compreender a imposição de segurança do perfil `restricted-v2` e execução em portas desprivilegiadas.
5. Inspecionar o ciclo de vida, status e eventos de execução de Pods no terminal.
6. Operar a perspectiva *Developer* do Web Console, dominando a visão de **Topology** para monitoramento visual.

---

## 📋 Pré-requisitos & Materiais

* Acesso à estação de gerenciamento `workstation` do curso DO180 na Red Hat Academy.
* Cluster OpenShift inicializado via script de espera na máquina `utility`:
  ```bash
  ssh lab@utility
  ./wait.sh
  exit
  ```
* Credenciais de acesso ao OpenShift:
  * **Acesso CLI:** Usuário `developer` / Senha `developer`
  * **Acesso Web Console (IdM):** Usuário `admin` / Senha `redhatocp`
* Endpoint da API do cluster:
  * `https://api.ocp4.example.com:6443`

---

## 🚀 Passo a Passo Guiado

### Passo 1: Autenticação no Cluster via CLI

Abra o terminal na estação `workstation` e realize a autenticação com a API do cluster OpenShift:

```bash
oc login -u developer -p developer https://api.ocp4.example.com:6443
```

> **Saída Esperada:**
> ```text
> Login successful.
> 
> You have access to the following projects and can switch between them with 'oc project <projectname>':
> 
> Using project "default".
> ```

Descubra a URL oficial do Console Web corporativo:

```bash
oc whoami --show-console
```

> **Saída Esperada:**
> ```text
> https://console-openshift-console.apps.ocp4.example.com
> ```

---

### Passo 2: Acesso ao OpenShift Web Console

1. No navegador **Firefox** da `workstation`, abra a URL obtida no passo anterior:  
   `https://console-openshift-console.apps.ocp4.example.com`
2. Na tela de seleção de provedor de identidade, clique em **Red Hat Identity Management**.
3. Autentique-se com as credenciais administrativas da plataforma:
   * **Username:** `admin`
   * **Password:** `redhatocp`
4. Observe a barra superior com o seletor de projetos e o menu lateral esquerdo.

---

### Passo 3: Provisionamento de Tenant Isolado (Namespace)

No terminal da `workstation`, crie um projeto dedicado para o laboratório (substitua `seunome` pelo seu primeiro nome ou identificador pessoal):

```bash
oc new-project lab-open-shift-seunome
```

> **Saída Esperada:**
> ```text
> Now using project "lab-open-shift-seunome" on server "https://api.ocp4.example.com:6443".
> 
> You can add applications to this project with the 'new-app' command. For example, try:
> 
>     oc new-app rails-postgresql-example
> 
> to build a new example application in Ruby.
> ```

*Nota de Arquitetura:* Um `Project` no OpenShift é uma extensão corporativa do `Namespace` padrão do Kubernetes, adicionando anotações de governança, controle de acesso baseado em papéis (RBAC) e isolamento de rede virtual através do CNI OVN-Kubernetes.

---

### Passo 4: Deploy Declarativo de Microsserviço Corporativo

Vamos instanciar a aplicação a partir de uma imagem de contêiner enterprise. Utilizaremos a imagem do Nginx mantida pela Bitnami, que é concebida especificamente para operar sob o padrão de segurança desprivilegiado (*non-root* na porta 8080):

```bash
oc new-app --name=my-web-app bitnami/nginx:latest
```

> **Saída Esperada:**
> ```text
> --> Found container image ... (bitnami/nginx:latest)
>     * An image stream tag will be created as "my-web-app:latest" that will track this image
> 
> --> Creating resources ...
>     deployment.apps "my-web-app" created
>     service "my-web-app" created
> --> Success
>     Application is not exposed. You can expose services to the outside world with the 'oc expose' command.
>     Run 'oc status' to view your app.
> ```

O OpenShift gerou automaticamente três objetos do Kubernetes:
* **Deployment:** Responsável por manter o estado desejado da aplicação.
* **ImageStream / Container:** Rastreamento do ciclo de vida da imagem base.
* **Service:** Criação de um endereço IP virtual estável (*ClusterIP*) para comunicação interna.

---

### Passo 5: Auditoria do Ciclo de Vida do Pod via CLI

Verifique a criação do contêiner e aguarde o status transicionar para `Running`:

```bash
oc get pods -o wide
```

> **Saída Esperada:**
> ```text
> NAME                          READY   STATUS    RESTARTS   AGE   IP            NODE
> my-web-app-5d8f6b7c4d-x9q2p   1/1     Running   0          35s   10.128.2.35   compute1.ocp4.example.com
> ```

Inspecione os detalhes e eventos de inicialização do Pod:

```bash
oc describe pod -l app=my-web-app
```

Verifique nos eventos finais que o Kubelet realizou o agendamento no nó worker, baixou a imagem via CRI-O e iniciou o contêiner com sucesso.

---

### Passo 6: Validação Visual no Console Web (Topology View)

1. Retorne ao navegador **Firefox** onde o Console Web está aberto.
2. No menu suspenso do canto superior esquerdo, mude da perspectiva **Administrator** para **Developer**.
3. No seletor de projeto no topo da página, selecione o seu projeto: `lab-open-shift-seunome`.
4. No menu lateral esquerdo, clique em **Topology**.
5. Observe a representação visual:
   * O nó circular central representando a aplicação `my-web-app`.
   * O anel em azul/ciano indicando que **1 Pod** está ativo e saudável.
   * O selo de identificação do serviço vinculado (`S`).
6. Clique sobre o círculo da aplicação para abrir a gaveta lateral direita.
7. Na aba **Details**, visualize as métricas básicas de CPU e memória. Na aba **Resources**, confirme a presença do *Deployment*, *Pod* e *Service*.

---

## 🧪 Validação & Critérios de Aceite

O laboratório é considerado concluído com sucesso quando:
* O comando `oc get pods` retornar o Pod da aplicação no estado `Running` com `1/1` contêineres prontos.
* O comando `oc get svc my-web-app` comprovar a alocação de um *ClusterIP* estável escutando na porta 8080.
* A visualização gráfica na aba **Topology** do Console Web exibir o círculo da aplicação em estado saudável (anel contínuo azul/ciano).

---

## 🧹 Cleanup & Próximos Passos

> [!NOTE]
> **Atenção:** **Não delete os recursos deste laboratório!**  
> A aplicação `my-web-app` e o namespace `lab-open-shift-seunome` serão utilizados diretamente no **Lab 04**, onde realizaremos a exposição pública corporativa através de *Routes*, executaremos testes de tolerância a falhas (*Self-Healing*) e realizaremos o escalonamento elástico.

---

## 💡 Desafios Complementares (Para Alunos Avançados)

1. **Inspeção do Manifesto Declarativo (YAML):**  
   Extraia a definição declarativa gerada pelo OpenShift com o comando:
   ```bash
   oc get deployment my-web-app -o yaml
   ```
   Localize a seção `spec.template.spec.containers` e identifique as portas configuradas, políticas de pull de imagem e o seletor de labels `app=my-web-app`.

2. **Terminal Interativo no Contêiner (`oc rsh`):**  
   Abra uma sessão interativa dentro do contêiner em execução:
   ```bash
   oc rsh deployment/my-web-app
   ```
   Dentro do contêiner, verifique o usuário não-root em execução com `whoami` (deve retornar um UID numérico arbitrário gerado pelo SCC, como `1000680000`), inspecione a porta de escuta do Nginx (`cat /opt/bitnami/nginx/conf/nginx.conf | grep listen`) e saia da sessão com `exit`.
