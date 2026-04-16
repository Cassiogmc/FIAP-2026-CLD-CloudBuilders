# Lab Integrador: A Missão Final (Lab 06) 🚀

Parabéns! Você chegou à etapa final. O objetivo aqui é unir tudo o que você aprendeu (infraestrutura, resiliência, automação e persistência) em um único projeto de **Nível Enterprise**.

---

## 🎯 O Desafio
Deploy de um **Guestbook (Livro de Visitas)** completo no OpenShift.
A aplicação consiste em um **Frontend Node.js** que se conecta a um **Banco de Dados MariaDB**.

---

## 📋 Requisitos Técnicos (Checklist)

### 1. Infraestrutura do Banco de Dados
- [ ] Criar um **Secret** chamado `db-pass` com a chave `password` (Ex: `P@ssw0rd123`).
- [ ] Realizar o deploy do MariaDB:
  ```bash
  oc new-app mariadb-ephemeral --name=guestbook-db -e MYSQL_PASSWORD=$(oc get secret db-pass -o jsonpath='{.data.password}' | base64 -d) -e MYSQL_USER=guestbook -e MYSQL_DATABASE=guestbook
  ```
  *(Dica: Se quiser persistência real entre restarts do cluster, use `mariadb-persistent` e adicione o PVC de 1Gi que você aprendeu no Lab 04).*

### 2. Fábrica de Software (S2I)
- [ ] No seu **Gitea** (Lab 05), crie uma nova migração para o repositório:
  `https://github.com/IBM/node-s2i-openshift.git`
- [ ] Realizar o deploy via S2I no OpenShift apontando para o SEU Gitea:
  ```bash
  oc new-app http://URL_DO_SEU_GITEA/USUARIO/node-s2i-openshift.git --name=guestbook-app
  ```

### 3. Integração e Conectividade
- [ ] Configurar a App para falar com o Banco de Dados. A aplicação Node.js busca o banco via nome de host.
- [ ] Injetar as credenciais (Secrets) e o Host do Banco na Aplicação via **Environment Variables**:
  - `DB_HOST` = `guestbook-db`
  - `DB_PASS` = (vinda do Secret `db-pass`)

### 4. Resiliência (Health Checks)
- [ ] Configurar **Liveness** e **Readiness Probes** na sua `guestbook-app` (Porta 8080).
- [ ] Garanta que o Readiness Probe tenha um delay inicial para dar tempo da aplicação conectar no banco.

---

## ⚡ O Teste de Caos (Validação)
1. Exponha sua aplicação (`oc expose svc/guestbook-app`) e acesse pelo navegador.
2. Escreva seu nome no Guestbook.
3. **Simule uma Falha:** Delete o Pod do MariaDB (`oc delete pod -l deployment=guestbook-db`).
4. **Verifique:** O OpenShift deve subir um novo Pod do banco. Se você configurou o PVC, seu nome continuará lá. Se usou a versão ephemeral, ele sumirá (o que é esperado nesse caso).

---

## ✅ ENTREGA FINAL
- Tire um print do Guestbook funcionando com o seu nome e poste no chat da aula.
- **Destaque:** Poste também o log do build da sua aplicação (`oc logs -f bc/guestbook-app`).

---
*Dúvidas? Utilize os labs 04 e 05 como referência. Tempo estimado: 60-90 minutos.*
