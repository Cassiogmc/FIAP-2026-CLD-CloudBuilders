# Lab Autoguiado: Operação e Saúde da Aplicação (Lab 04) 🛡️

Neste laboratório, você aprenderá a gerenciar configurações sem mexer no código e a configurar o OpenShift para monitorar e curar sua aplicação automaticamente.

---

## 🏗️ Passo 0: Acesso e Preparação do Ambiente
Se você está iniciando uma nova sessão, siga estes passos para preparar o ambiente:

1. Acesse o servidor de utilitários e aguarde o ambiente ficar "UP":
   ```bash
   ssh lab@utility
   ./wait.sh
   exit
   ```
2. Realize o login no cluster OpenShift via CLI:
   ```bash
   oc login -u developer -p developer https://api.ocp4.example.com:6443
   ```
3. Garanta que o seu projeto esteja criado e selecionado (substitua `seunome` pelo seu nome/RM):
   ```bash
   oc new-project lab-open-shift-seunome || oc project lab-open-shift-seunome
   ```

---

## 🏗️ Passo 1: Limpeza e Preparação
Para evitar conflitos com laboratórios anteriores, vamos garantir que o terreno esteja limpo:
```bash
oc delete all --all
```

---

## 🏗️ Passo 2: Personalização Rápida (Environment Variables)
Muitas aplicações leem configurações de variáveis de ambiente. Vamos subir uma nova app e mudar seu comportamento:
```bash
oc new-app https://github.com/sclorg/nodejs-ex.git --name=health-app
oc set env deployment/health-app APP_MSG="MBA Cloud FIAP - Operação e Resiliência"
```
*Acesse o Console Web (Login: `admin` / `redhatocp` via Red Hat Identity Management).*

---

## 🔐 Passo 3: Gerenciamento de Segredos (Secrets)
Para dados sensíveis (senhas, tokens), usamos Secrets. Vamos simular a injeção de uma senha de banco de dados:
1. Crie o Secret:
   ```bash
   oc create secret generic db-pass --from-literal=password=P@ssw0rd123
   ```
2. Injete o Secret na sua aplicação:
   ```bash
   oc set env deployment/health-app --from=secret/db-pass
   ```
3. Aguarde o novo deploy terminar e verifique se o segredo "chegou":
   ```bash
   oc rollout status deployment/health-app
   oc rsh deployment/health-app env | grep -i password
   ```

---

## 🩺 Passo 4: Autocura Inteligente (Health Checks)
O OpenShift precisa saber se sua app está "viva" (Liveness) e "pronta" (Readiness).
Vamos configurar um check que testa se a porta 8080 está respondendo:

1. Configure o **Liveness Probe** (Se falhar, o cluster mata e cria um novo Pod):
   ```bash
   oc set probe deployment/health-app --liveness --get-url=http://:8080/ --initial-delay-seconds=30
   ```
2. Configure o **Readiness Probe** (Se falhar, o cluster tira o Pod do balanceador até ele se recuperar):
   ```bash
   oc set probe deployment/health-app --readiness --get-url=http://:8080/ --initial-delay-seconds=5
   ```

---

## 📂 Passo 5: Persistência de Dados (PVC)
Vamos garantir que seus arquivos não sumam se o Pod for reiniciado.
1. Adicione um volume de armazenamento à aplicação:
   ```bash
   oc set volume deployment/health-app --add --name=my-storage -t pvc --claim-size=1Gi --mount-path=/var/lib/storage
   ```
2. Verifique se o volume foi montado:
   ```bash
   oc get pvc
   ```

---

## ✅ VALIDAÇÃO FINAL
1. Acesse o **Console Web do OpenShift**: `https://console-openshift-console.apps.ocp4.example.com`
2. **Login:** Selecione o método **"Red Hat Identity Management"** e utilize as credenciais:
   - **User:** `admin`
   - **Password:** `redhatocp`
3. No menu lateral, garanta que você está na perspectiva **"Administrator"**.
4. Navegue em: **Workloads -> Deployments -> health-app -> YAML**.
5. Localize no arquivo YAML a seção `livenessProbe` e confirme as configurações que você aplicou.
6. **Missão Cumprida:** Tire um print da aba **"Health Checks"** (ao lado de YAML) mostrando os checks verdes e envie no chat.

---
*Dúvidas? Utilize o chat da aula. Tempo estimado: 30 minutos.*
