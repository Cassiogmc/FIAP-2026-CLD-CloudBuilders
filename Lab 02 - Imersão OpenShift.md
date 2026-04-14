# Lab Autoguiado: Missão OpenShift Cloud-Native 🚀

Neste laboratório, você assumirá o controle total da infraestrutura OpenShift para realizar o deploy, escala e exposição de sua primeira aplicação enterprise.

---

## 🎯 Objetivo Final
Realizar o deploy de uma aplicação web, garantir sua resiliência e expô-la para a internet.
**Critério de Sucesso:** Enviar a URL funcional no chat da aula.

---

## 🏗️ Passo 1: Acesso ao Ambiente
1. Abra o terminal da sua workstation.
2. Acesse o servidor de utilitários:
   ```bash
   ssh lab@utility
   ```
3. Execute o script de monitoramento e aguarde o ambiente ficar "UP":
   ```bash
   ./wait.sh
   ```
   *Nota: O cluster pode levar de 10 a 20 minutos para estabilizar. Quando terminar, saia do Utility (`exit`).*

4. Realize o login no cluster OpenShift via CLI:
   ```bash
   oc login -u developer -p developer https://api.ocp4.example.com:6443
   ```
4. Verifique a URL do Console Web e acesse pelo browser, selecionar Red Hat Identity Management (Use o login `admin` e senha `redhatocp`):
   ```bash
   oc whoami --show-console
   ```

---

## 📂 Passo 2: Criando seu Espaço de Trabalho (Namespace)
Crie um projeto exclusivo para você (substitua `seunome` pelo seu nome/RM):
```bash
oc new-project lab-open-shift-seunome
```

---

## 🚢 Passo 3: Deploy Ágil da Aplicação
Vamos subir uma aplicação pronta diretamente de uma imagem de container:
```bash
oc new-app --name=my-web-app bitnami/nginx:latest
```
*Aguarde alguns segundos e verifique se o Pod está rodando:*
```bash
oc get pods
```

---

## 🌐 Passo 4: Exposição para o Mundo (Rotas)
Neste momento, a aplicação está rodando mas não é acessível externamente. Vamos criar a Rota:
```bash
oc expose svc/my-web-app
```
**Descubra sua URL pública:**
```bash
oc get routes
```

---

## 🔄 Passo 5: Teste de Resiliência (Self-Healing)
O OpenShift cuida da saúde da sua app. Vamos forçar uma "falha" para ver a mágica:
1. Deleta o seu Pod manualmente:
   ```bash
   oc delete pod my-web-app-<ID_DO_SEU_POD>
   ```
2. Digite imediatamente:
   ```bash
   oc get pods
   ```
*Observe que um novo Pod já nasceu para substituir o anterior. Isso é Cloud-Native!*

---

## ✅ VALIDAÇÃO FINAL
1. Copie a URL gerada no **Passo 4**.
2. Cole no browser e verifique se a página do Nginx carregou.
3. **Missão Cumprida:** Cole a URL no chat da aula para validação do instrutor.

---
*Dúvidas? Utilize o chat da aula. Tempo estimado: 30 minutos.*
