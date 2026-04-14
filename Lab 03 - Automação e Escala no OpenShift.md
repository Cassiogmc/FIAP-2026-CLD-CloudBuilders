# Lab Autoguiado: Automação e Escala no OpenShift (Lab 03) ⚡

Neste laboratório, você verá o verdadeiro poder do OpenShift: transformar código fonte em uma aplicação rodando em segundos e escalar para suportar milhares de acessos.

---

## 🏗️ Passo 1: Limpeza do Terreno
Para evitar confusão, vamos deletar a aplicação do lab anterior (se ainda estiver no mesmo projeto):
```bash
oc delete all --selector app=my-web-app
```

---

## 🚢 Passo 2: Deploy Direto do Git (S2I)
O OpenShift vai ler o código, identificar a linguagem e "buildar" a imagem sozinho.
Substitua o link abaixo por um repositório de exemplo (ou use este de Node.js):
```bash
oc new-app https://github.com/sclorg/nodejs-ex.git --name=scaling-app
```

---

## 🔍 Passo 3: Acompanhando a "Mágica" (Build Logs)
Como não enviamos uma imagem pronta, o OpenShift está compilando agora. Vamos espiar:
```bash
oc get builds
oc logs -f build/scaling-app-1
```
*Aguarde o build terminar (Status: Complete).*

---

## 📈 Passo 4: Escalabilidade Infinita
Sua aplicação nasceu com apenas 1 Pod. Vamos escalar para 3 instâncias agora:
```bash
oc scale deployment/scaling-app --replicas=3
```
**Verifique os novos Pods nascendo:**
```bash
oc get pods
```

---

## 🌐 Passo 5: Exposição e Teste de Carga
Vamos expor a aplicação e ver o balanceamento:
```bash
oc expose svc/scaling-app
oc get routes
```
1. Abra a URL no browser.
2. Atualize a página várias vezes (F5). 
3. No console web, observe os gráficos de tráfego sendo distribuídos entre os 3 Pods.

---

## ✅ VALIDAÇÃO FINAL
1. No console web, tire um print da tela **"Topology"** mostrando os 3 Pods rodando ao redor do círculo da aplicação.
2. **Missão Cumprida:** Envie o print ou a URL no chat da aula.

---
*Dúvidas? Utilize o chat da aula. Tempo estimado: 30 minutos.*
