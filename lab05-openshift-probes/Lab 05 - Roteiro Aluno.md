# Lab 05 - Operação Resiliente, Segredos e Persistência no Red Hat OpenShift: Health Probes e Storage PVC

* **Programa:** MBA em MultiCloud Strategy & Architecture
* **Ambiente / Plataforma:** Red Hat Academy (DO180 v4.14 / Red Hat OpenShift Container Platform 4.14)
* **Stack Técnica:** OpenShift CLI (`oc`), OpenShift Web Console (Health Checks / Storage), Kubernetes Secrets, Liveness & Readiness Probes, PersistentVolumeClaims (PVC)
* **Duração Estimada:** 35 a 40 minutos

---

## 🎯 Objetivo do Lab

Capacitar o aluno a operar microsserviços em produção no Red Hat OpenShift com padrões avançados de resiliência, governança de credenciais e persistência de dados, desacoplando parâmetros de ambiente da imagem de contêiner, blindando senhas com Kubernetes Secrets, configurando sondas autônomas de monitoramento e autocura (*Health Probes*) e anexando volumes corporativos persistentes (*PVCs*).

**Cenário Corporativo (FinCorp Resiliência e Operações):**  
A equipe de operações da *FinCorp* está homologando o microsserviço de saúde e catálogo (`health-app`) para execução em ambiente de produção. Para atender às normas de compliance do Banco Central e auditorias de segurança, a aplicação não pode conter senhas em texto puro nem exigir recompilação de código para alteração de parâmetros. Além disso, o cluster deve ser capaz de detectar travamentos internos de processo de forma autônoma (reiniciando o pod automaticamente), impedir o roteamento de tráfego para contêineres ainda em inicialização e garantir a durabilidade de arquivos gerados em disco através de armazenamento persistente corporativo.

**Habilidades Conquistadas:**
1. Desacoplar parâmetros de configuração da imagem de contêiner utilizando variáveis de ambiente (metodologia *12-Factor App*).
2. Criar e gerenciar objetos *Secret* no OpenShift, blindando credenciais confidenciais em Base64 e injetando-as de forma transparente em *Deployments*.
3. Auditar a injeção de variáveis confidenciais dentro do contêiner em tempo de execução via terminal interativo (`oc rsh`).
4. Configurar e calibrar sondas de *Liveness Probe*, permitindo que o Kubelet detecte deadlocks e execute a autocura do contêiner.
5. Configurar sondas de *Readiness Probe*, garantindo que o roteador HAProxy apenas envie tráfego para réplicas 100% prontas.
6. Provisionar e acoplar volumes persistentes (*PersistentVolumeClaim - PVC*) de 1Gi, garantindo imunidade à efemeridade do ciclo de vida dos Pods.
7. Homologar a saúde e os recursos da aplicação através do OpenShift Web Console.

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
* Console Web:
  * `https://console-openshift-console.apps.ocp4.example.com`

---

## 🚀 Passo a Passo Guiado

### Passo 1: Autenticação e Preparação do Ambiente

Na estação de gerenciamento (`workstation`), realize a autenticação com o cluster OpenShift para estabelecer a sessão e o arquivo de configuração local (`~/.kube/config`):

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

Em seguida, garanta que o seu projeto dedicado esteja ativo e limpo de execuções anteriores (substitua `seunome` pelo seu identificador pessoal):

```bash
oc project lab-open-shift-seunome || oc new-project lab-open-shift-seunome
oc delete all --all
```

> **Saída Esperada:**
> ```text
> Now using project "lab-open-shift-seunome" on server "https://api.ocp4.example.com:6443".
> pod "my-web-app-..." deleted
> service "my-web-app" deleted
> deployment.apps "my-web-app" deleted
> route.route.openshift.io "my-web-app" deleted
> ```

---

### Passo 2: Deploy da Aplicação Base com Injeção de Variáveis (12-Factor)

Em arquiteturas *cloud-native*, parâmetros operacionais nunca devem ser fixados no código-fonte ou na imagem. Dispare o deploy da aplicação base Node.js e injete uma variável de ambiente corporativa:

```bash
oc new-app https://github.com/sclorg/nodejs-ex.git --name=health-app
oc set env deployment/health-app APP_MSG="MBA MultiCloud FIAP - Operacao e Resiliencia"
```

> **Saída Esperada:**
> ```text
> --> Creating resources ...
>     deployment.apps "health-app" created
>     service "health-app" created
> --> Success
> deployment.apps/health-app updated
> ```

O comando `oc set env` atualizou o manifesto do `Deployment`. O controlador do Kubernetes detecta a mutação na especificação do Pod e dispara automaticamente um novo *rolling update* com a nova variável injetada.

---

### Passo 3: Criação de Segredos Corporativos (Secrets) e Injeção Blindada

Dados confidenciais como chaves de API e senhas de bancos de dados exigem blindagem com objetos do tipo `Secret`.

1. Crie o objeto `Secret` corporativo contendo a senha simulada do banco de dados:
   ```bash
   oc create secret generic db-pass --from-literal=password=P@ssw0rd123
   ```

   > **Saída Esperada:**
   > ```text
   > secret/db-pass created
   > ```

2. Injete todas as chaves do Secret como variáveis de ambiente no Deployment da aplicação:
   ```bash
   oc set env deployment/health-app --from=secret/db-pass
   ```

   > **Saída Esperada:**
   > ```text
   > deployment.apps/health-app updated
   > ```

3. Acompanhe a conclusão do rollout e confirme a injeção da variável dentro do contêiner via terminal interativo (`oc rsh`):
   ```bash
   oc rollout status deployment/health-app
   oc rsh deployment/health-app env | grep -i password
   ```

   > **Saída Esperada:**
   > ```text
   > deployment "health-app" successfully rolled out
   > password=P@ssw0rd123
   > ```

*Fundamentação Técnica:* O segredo foi desacoplado da imagem e injetado pelo Kubelet durante o provisionamento do Pod no nó worker, garantindo que imagens idênticas possam rodar em desenvolvimento, homologação e produção apenas alternando o valor do Secret.

---

### Passo 4: Autocura Inteligente com Health Probes (Liveness & Readiness)

Para que o OpenShift monitore a saúde do processo e proteja o balanceador de tráfego, configuramos duas sondas HTTP na porta 8080:

1. **Liveness Probe (Sonda de Sobrevivência):**  
   Verifica se o processo da aplicação continua vivo e responsivo. Se esta sonda falhar repetidamente, o Kubelet reinicia o contêiner automaticamente para destravar a aplicação.
   ```bash
   oc set probe deployment/health-app --liveness --get-url=http://:8080/ --initial-delay-seconds=30
   ```

   > **Saída Esperada:**
   > ```text
   > deployment.apps/health-app updated
   > ```

2. **Readiness Probe (Sonda de Prontidão):**  
   Verifica se o contêiner concluiu o carregamento de rotas e conexões e está apto a receber conexões de rede externas. Se falhar, o Pod é isolado temporariamente dos endpoints do *Service* e do roteador HAProxy, sem sofrer reinício.
   ```bash
   oc set probe deployment/health-app --readiness --get-url=http://:8080/ --initial-delay-seconds=5
   ```

   > **Saída Esperada:**
   > ```text
   > deployment.apps/health-app updated
   > ```

3. Acompanhe a estabilização do novo Pod com as duas sondas ativas:
   ```bash
   oc rollout status deployment/health-app
   oc get pods -l deployment=health-app
   ```

---

### Passo 5: Anexação de Armazenamento Corporativo Persistente (PVC)

Por padrão, o sistema de arquivos de um contêiner é efêmero: se o contêiner for reiniciado pelo Kubelet ou migrado de nó, qualquer arquivo gravado em disco será destruído. Para viabilizar persistência real, requisitaremos um volume corporativo através de um *PersistentVolumeClaim (PVC)*.

1. Aloque e conecte um volume persistente de 1Gi montado no diretório `/var/lib/storage`:
   ```bash
   oc set volume deployment/health-app --add --name=my-storage -t pvc --claim-size=1Gi --mount-path=/var/lib/storage
   ```

   > **Saída Esperada:**
   > ```text
   > persistentvolumeclaim/health-app-claim created
   > deployment.apps/health-app volume updated
   > ```

2. Verifique o status da requisição de armazenamento:
   ```bash
   oc get pvc
   ```

   > **Saída Esperada:**
   > ```text
   > NAME               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
   > health-app-claim   Bound    pvc-7a2e5d8f-4b1c-4392-a9b1-5e8c1b9f0d3a   1Gi        RWO            gp2            25s
   > ```

   *Conceito de Storage:* O status `Bound` comprova que o subsistema de armazenamento do OpenShift atendeu à solicitação declarativa do PVC, vinculando um `PersistentVolume (PV)` real fornecido pelo provedor de infraestrutura.

3. Aguarde o rollout e confirme a montagem do ponto de persistência no contêiner:
   ```bash
   oc rollout status deployment/health-app
   oc rsh deployment/health-app df -h /var/lib/storage
   ```

   > **Saída Esperada:**
   > ```text
   > Filesystem      Size  Used Avail Use% Mounted on
   > /dev/xvda1      1.0G   33M  991M   4% /var/lib/storage
   > ```

---

### Passo 6: Validação Visual no OpenShift Web Console

1. No navegador Firefox da `workstation`, acesse o **OpenShift Web Console**:  
   `https://console-openshift-console.apps.ocp4.example.com`
2. Faça login com o método **Red Hat Identity Management**:
   * **Username:** `admin`
   * **Password:** `redhatocp`
3. Alterne para a perspectiva **Administrator** (menu superior esquerdo).
4. No seletor de projetos, confirme que está em `lab-open-shift-seunome`.
5. Navegue até **Workloads -> Deployments** e clique sobre `health-app`.
6. Valide os seguintes painéis:
   * **Aba Health Checks:** Confirme os dois checks ativos (*Liveness* com delay de 30s e *Readiness* com delay de 5s) exibindo status verde/saudável.
   * **Aba Volumes:** Confirme o volume `my-storage` associado ao claim `health-app-claim` montado no caminho `/var/lib/storage`.
   * **Aba YAML:** Localize as seções `livenessProbe`, `readinessProbe` e `volumeMounts` no manifesto declarativo.

---

## 🧪 Validação & Critérios de Aceite

O laboratório é considerado concluído com sucesso quando:
* O comando `oc get pods -l deployment=health-app` retornar o Pod no estado `Running` com `1/1` contêineres prontos.
* O comando `oc rsh deployment/health-app env | grep -i password` confirmar a injeção transparente da credencial do Secret `db-pass`.
* O comando `oc get pvc` exibir o volume com status `Bound` e capacidade `1Gi`.
* A aba **Health Checks** no Web Console comprovar o funcionamento simultâneo de *Liveness* e *Readiness Probes*.
* **Quick Win de Sala:** Capturar o print da aba **Health Checks** do Console Web ou a saída do terminal comprovando o Pod `1/1 Running` com o PVC `Bound` e compartilhar no chat da turma.

---

## 🧹 Cleanup & Próximos Passos

Para preparar o cluster para a montagem da Fábrica S2I no **Lab 06**, execute a limpeza dos recursos temporários criados neste laboratório:

```bash
oc delete all -l app=health-app
oc delete secret db-pass
oc delete pvc health-app-claim
```

> **Saída Esperada:**
> ```text
> deployment.apps "health-app" deleted
> service "health-app" deleted
> secret "db-pass" deleted
> persistentvolumeclaim "health-app-claim" deleted
> ```

---

## 💡 Desafios Complementares (Para Alunos Avançados)

1. **Simulação de Falha de Liveness e Auditoria do Kubelet:**  
   Altere deliberadamente a porta ou caminho do Liveness Probe para um endpoint inexistente (`/falha-critica`) utilizando o comando:
   ```bash
   oc set probe deployment/health-app --liveness --get-url=http://:8080/falha-critica
   ```
   Acompanhe o comportamento do Pod com `oc get pods -w` e audite os eventos gerados com `oc describe pod -l deployment=health-app`. Observe o Kubelet registrando falhas repetidas (`Unhealthy`), matando o contêiner e incrementando o contador `RESTARTS`.

2. **Auditoria e Decodificação de Segredos via CLI:**  
   Extraia a definição declarativa bruta do Secret em formato YAML:
   ```bash
   oc get secret db-pass -o yaml
   ```
   Localize o campo `data.password` e decodifique o payload em Base64 utilizando o terminal:
   ```bash
   oc get secret db-pass -o jsonpath='{.data.password}' | base64 -d; echo
   ```
   Explique por que o armazenamento de Secrets em Base64 requer a ativação de criptografia em repouso (*Encryption at Rest*) no banco etcd do OpenShift para conformidade corporativa.
