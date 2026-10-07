# Lab 04 - Resiliência Corporativa, Exposição Pública e Escala Elástica no Red Hat OpenShift

* **Programa:** MBA em Cloud Strategy & Architecture
* **Ambiente / Plataforma:** Red Hat Academy (DO180 v4.14 / Red Hat OpenShift Container Platform 4.14)
* **Stack Técnica:** OpenShift Routes (HAProxy Ingress L7), Kubernetes Services (ClusterIP), Self-Healing Controller, Kubelet
* **Duração Estimada:** 35 a 40 minutos

---

## 🎯 Objetivo do Lab

Capacitar o aluno a implementar padrões de resiliência e exposição corporativa na nuvem privada Red Hat OpenShift, dominando o roteamento L7 com *Routes*, testando o comportamento de autocura (*Self-Healing*) do cluster sob falhas e operando a escalabilidade elástica de réplicas.

**Cenário Corporativo (FinCorp Resiliência e Tráfego):**  
Após o deploy inicial realizado com sucesso no Lab 03, o comitê de arquitetura da *FinCorp* exige a conclusão da esteira de entrega: o frontend de catálogo (`my-web-app`) precisa ser publicado com um FQDN público acessível aos clientes externos, demonstrar imunidade a falhas inesperadas de processo através do mecanismo autônomo de *Self-Healing* do Kubernetes, e estar preparado para absorver um pico de acessos mediante escalonamento elástico para 3 réplicas balanceadas.

**Habilidades Conquistadas:**
1. Inspecionar o objeto *Service* e compreender o funcionamento do balanceamento interno L4 (*ClusterIP*).
2. Criar e gerenciar *OpenShift Routes* corporativas com `oc expose`, habilitando ingresso público via HAProxy nativo.
3. Validar a resolução de rotas corporativas e acesso HTTP no navegador web.
4. Executar testes de engenharia de caos (*Pod Failure*), auditando o loop de reconciliação contínua do Kubernetes em tempo real.
5. Escalar a capacidade da aplicação horizontalmente sob demanda com `oc scale`.
6. Auditar a distribuição de tráfego entre múltiplos Pods e validar visualmente o escalonamento na *Topology View*.

---

## 📋 Pré-requisitos & Materiais

* Conclusão do **Lab 03** com a aplicação `my-web-app` ativa no projeto `lab-open-shift-seunome`.
* Acesso à estação gráfica `workstation` do curso DO180 na Red Hat Academy.
* Sessão autenticada no terminal com usuário `developer`:
  ```bash
  oc project lab-open-shift-seunome
  ```
* Sessão administrativa ativa no Web Console via Firefox (`admin` / `redhatocp`).

---

## 🚀 Passo a Passo Guiado

### Passo 1: Inspeção do Service Interno

Antes de expor a aplicação para a rede externa, audite o objeto *Service* criado no laboratório anterior:

```bash
oc get svc my-web-app
```

> **Saída Esperada:**
> ```text
> NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
> my-web-app   ClusterIP   172.30.128.45    <none>        8080/TCP   10m
> ```

Inspecione os detalhes de roteamento interno e o endpoint do Pod:

```bash
oc describe svc my-web-app
```

> **Saída Esperada (Trecho Relevante):**
> ```text
> Selector:          deployment=my-web-app
> Type:              ClusterIP
> IP:                172.30.128.45
> Port:              8080-tcp  8080/TCP
> TargetPort:        8080/TCP
> Endpoints:         10.128.2.35:8080
> ```

*Insight de Arquitetura:* O *Service* possui um IP virtual imutável (`ClusterIP`). Ele utiliza o seletor `deployment=my-web-app` para direcionar o tráfego dinamicamente para o IP privado do Pod (`10.128.2.35`). Esse endereço só é alcançável por outros serviços dentro da rede SDN do cluster.

---

### Passo 2: Exposição Externa via OpenShift Route

Para tornar o microsserviço acessível aos clientes externos corporativos, crie uma **Route** a partir do serviço:

```bash
oc expose svc/my-web-app
```

> **Saída Esperada:**
> ```text
> route.route.openshift.io/my-web-app exposed
> ```

Inspecione a URL corporativa pública atribuída automaticamente pelo OpenShift Router (HAProxy):

```bash
oc get routes
```

> **Saída Esperada:**
> ```text
> NAME         HOST/PORT                                                         PATH   SERVICES     PORT       TERMINATION   WILDCARD
> my-web-app   my-web-app-lab-open-shift-seunome.apps.ocp4.example.com                  my-web-app   8080-tcp                 None
> ```

*Como funciona o OpenShift Router:* O OpenShift executa um conjunto de instâncias do HAProxy no nó de infraestrutura (*Ingress Controller*). Ao criar uma `Route`, o cluster atualiza as tabelas de roteamento do HAProxy em microssegundos, vinculando o domínio FQDN gerado ao serviço interno.

---

### Passo 3: Validação de Acesso Externo no Navegador

1. No navegador **Firefox** da `workstation`, abra uma nova aba.
2. Cole a URL pública obtida no passo anterior (ex: `http://my-web-app-lab-open-shift-seunome.apps.ocp4.example.com`).
3. Confirme o carregamento com sucesso da página padrão de boas-vindas do Nginx (*"Welcome to nginx!"*).
4. No terminal, execute uma requisição HTTP via CLI para confirmar a resposta de status:
   ```bash
   curl -I http://$(oc get route my-web-app -o jsonpath='{.spec.host}')
   ```
   > Confirme o retorno do cabeçalho `HTTP/1.1 200 OK`.

---

### Passo 4: O Teste do Caos (Simulação de Falha & Self-Healing)

O Kubernetes opera sob o princípio da **Reconciliação Contínua**: ele compara a todo momento o *Estado Atual* com o *Estado Desejado*. Vamos provocar uma falha deliberada para auditar o mecanismo de **Self-Healing**:

1. Identifique o nome do Pod atualmente em execução:
   ```bash
   oc get pods
   ```
   *(Anote o nome, por exemplo: `my-web-app-5d8f6b7c4d-x9q2p`)*

2. Force a destruição imediata do Pod simulando uma pane de sistema:
   ```bash
   oc delete pod -l deployment=my-web-app
   ```

3. Imediatamente após o comando, liste os pods com frequência rápida:
   ```bash
   oc get pods
   ```

> **Saída Esperada:**
> ```text
> NAME                          READY   STATUS              RESTARTS   AGE
> my-web-app-5d8f6b7c4d-k4v7z   1/1     Running             0          4s
> ```

*Análise Crítica:* Observe o nome do Pod (`-k4v7z`) e o campo `AGE` (poucos segundos). O Pod anterior foi destruído, mas o controlador do *Deployment/ReplicaSet* detectou a discrepância (`Atual: 0 != Desejado: 1`) e instruiu o Kubelet a instanciar uma nova réplica instantaneamente. Se você atualizar o navegador web, constatará que a aplicação permaneceu acessível sem interrupção.

---

### Passo 5: Escalonamento Elástico de Réplicas

A squad financeira prevê um aumento expressivo de requisições no fechamento contábil. Escale a aplicação para **3 réplicas**:

```bash
oc scale deployment/my-web-app --replicas=3
```

> **Saída Esperada:**
> ```text
> deployment.apps/my-web-app scaled
> ```

Acompanhe o provisionamento das novas instâncias em tempo real:

```bash
oc get pods -o wide
```

> **Saída Esperada:**
> ```text
> NAME                          READY   STATUS    RESTARTS   AGE   IP            NODE
> my-web-app-5d8f6b7c4d-k4v7z   1/1     Running   0          3m    10.128.2.36   compute1.ocp4.example.com
> my-web-app-5d8f6b7c4d-m8w2l   1/1     Running   0          12s   10.131.0.40   compute2.ocp4.example.com
> my-web-app-5d8f6b7c4d-t9r5p   1/1     Running   0          12s   10.129.2.18   compute1.ocp4.example.com
> ```

Audite a tabela de endpoints do *Service* para comprovar que as 3 réplicas foram vinculadas ao mesmo balanceador:

```bash
oc get endpoints my-web-app
```

> **Saída Esperada:**
> ```text
> NAME         ENDPOINTS                                               AGE
> my-web-app   10.128.2.36:8080,10.129.2.18:8080,10.131.0.40:8080   15m
> ```

---

### Passo 6: Auditoria Visual da Carga na Topologia do Console Web

1. Retorne ao navegador **Firefox** na tela do OpenShift Web Console.
2. Certifique-se de estar na perspectiva **Developer** e na aba **Topology**.
3. Observe a evolução visual do nó da aplicação:
   * O anel circular agora exibe o número **3** no centro, indicando **3 Pods** ativos.
   * O anel está segmentado em 3 partes proporcionais na cor azul/ciano.
   * À direita do círculo da aplicação, agora existe um ícone de seta apontando para a **Route** corporativa.
4. Clique no ícone de atalho da rota (o link no canto do card) e confirme que a aplicação abre diretamente em uma nova aba do browser.

---

## 🧪 Validação & Critérios de Aceite

O laboratório é considerado concluído com sucesso quando:
* O comando `oc get routes` listar a rota pública no status ativo com a URL corporativa acessível no navegador.
* O teste de exclusão de Pod comprovar a recriação autônoma do Pod pelo mecanismo de *Self-Healing*.
* O comando `oc get endpoints my-web-app` listar exatamente 3 endereços IPs de contêineres vinculados.
* A interface do Console Web na aba **Topology** exibir o anel com **3 Pods** saudáveis e link de rota ativo.

---

## 🧹 Cleanup & Liberação de Recursos

Ao término da validação, limpe todos os recursos criados no seu projeto para liberar memória do cluster:

```bash
oc delete all --all
```

> **Saída Esperada:**
> ```text
> pod "my-web-app-5d8f6b7c4d-k4v7z" deleted
> pod "my-web-app-5d8f6b7c4d-m8w2l" deleted
> pod "my-web-app-5d8f6b7c4d-t9r5p" deleted
> service "my-web-app" deleted
> deployment.apps "my-web-app" deleted
> route.route.openshift.io "my-web-app" deleted
> ```

---

## 💡 Desafios Complementares (Para Alunos Avançados)

1. **Auditoria de Balanceamento de Carga com Loop HTTP:**  
   No terminal, execute um loop disparando 15 requisições rápidas contra a rota corporativa:
   ```bash
   URL=$(oc get route my-web-app -o jsonpath='{.spec.host}')
   for i in {1..15}; do curl -s -I http://$URL | grep HTTP; done
   ```
   Inspecione os logs agregados de todos os pods com o comando `oc logs deployment/my-web-app --tail=5 --all-containers` e comprove que as requisições foram distribuídas entre os diferentes nós e réplicas.

2. **Drenagem Graciosa e Escala Regressiva:**  
   Reduza o número de réplicas de 3 para 1 (`oc scale deployment/my-web-app --replicas=1`). Observe com `oc get pods -w` como o OpenShift envia o sinal `SIGTERM` para os pods excedentes, aguarda o encerramento gracioso e remove seus respectivos IPs da tabela de *Endpoints* do serviço sem causar erros 502/503 nas conexões ativas.
