# Lab Autoguiado: Fábrica de Software e Sua Própria Infra de Git (Lab 05) 🏗️

Neste laboratório, você vai elevar seu nível técnico: em vez de usar serviços externos, você vai instalar seu próprio servidor Git (**Gitea**) dentro do OpenShift e usá-lo para alimentar sua fábrica de software (**S2I**).

---

## 🛠️ Passo 0: Acesso e Limpeza
Garanta que seu projeto está selecionado e limpo:
```bash
oc project lab-open-shift-seunome
oc delete all --all
```

---

## 🦊 Passo 1: Instalando seu Servidor Git (Gitea)
Vamos subir um servidor Git profissional em menos de 2 minutos.

*Dica Técnica: Por segurança, o OpenShift proíbe containers de rodarem como "root". Por isso, usaremos a versão **rootless** (sem root) do Gitea.*

1. No terminal, dispare o deploy do Gitea:
   ```bash
   oc new-app gitea/gitea:latest-rootless --name=meu-git
   ```
2. Crie a rota para acessar a interface web (forçando a porta 3000):
   ```bash
   oc expose svc/meu-git --port=3000
   ```
3. Acompanhe a subida do Pod:
   ```bash
   oc get pods -w
   ```
   *Aguarde o status ficar "Running" e o "Ready" 1/1.*

---

## ⚙️ Passo 2: Configuração Inicial do Git
1. Pegue a URL do seu Git: `oc get route meu-git`
2. Acesse pelo navegador e você verá a tela de instalação.
3. **Não mude nada:** As configurações padrão (SQLite) são perfeitas para o nosso lab.
4. Clique no botão azul lá embaixo: **"Install Gitea"**.
5. **Crie sua conta:** O primeiro usuário criado será o Administrador. Guarde bem o usuário e senha!

---

## 📦 Passo 3: Migrando o Código para o seu Git
Agora que você tem um Git privado, vamos trazer o código para ele.

1. No painel do Gitea, clique no **"+"** (topo direito) -> **New Migration**.
2. No campo **URL**, cole: `https://github.com/sclorg/nodejs-ex.git`
3. Nome do repositório: `meu-app-nodejs`
4. Clique em **Migrate Repository**.

---

## 🚀 Passo 4: Deploy "Source-to-Image" (S2I)
Vamos conectar o OpenShift ao SEU novo servidor Git interno.

1. Execute o deploy (substitua `SEU_USUARIO` e o `IP_DA_ROTA` conforme necessário):
   ```bash
   # Dica: Use a URL HTTP que o Gitea mostra no repositório
   oc new-app http://URL_DA_SUA_ROTA_GITEA/SEU_USUARIO/meu-app-nodejs.git --name=fabrica-app
   ```
2. **Domínio Técnico (Logs):** Acompanhe a construção da sua imagem:
   ```bash
   oc logs -f bc/fabrica-app
   ```

---

## ⚡ Passo 5: O "Show" - Webhooks e Automação
1. Exponha sua app: `oc expose svc/fabrica-app`.
2. No Gitea, edite o arquivo `public/index.html` e mude o título para: **"Minha Infra, Minhas Regras - [Seu Nome]"**.
3. Faça o **Commit**.
4. **Verifique:** Como o Gitea e o OpenShift estão na mesma rede, o build deve disparar automaticamente ou você pode forçar via:
   ```bash
   oc start-build fabrica-app
   ```

---

## ✅ VALIDAÇÃO FINAL
- Mostre a aplicação rodando com seu nome.
- **Bônus:** Mostre o log do Build no terminal.

---
*Dúvidas? Utilize o chat da aula. Tempo estimado: 35-40 minutos.*
