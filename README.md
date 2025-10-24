# Projeto: CI/CD com GitHub Actions - Aplicação FastAPI

Este repositório faz parte do Programa de Bolsas - DevSecOps e contém o código-fonte de uma aplicação FastAPI simples. O objetivo deste projeto é demonstrar um ciclo completo de automação (CI/CD).

Este repositório é a "fonte da verdade" para o **código-fonte da aplicação**.

**Repositório de Manifestos (GitOps):** `https://github.com/Henrique-Versiani/projeto-manifests.git`

## Objetivo do Projeto

O objetivo é automatizar o ciclo completo de desenvolvimento, build, deploy e execução da aplicação. Para isso, utilizamos:
* **Aplicação:** FastAPI.
* **CI/CD:** GitHub Actions.
* **Registry:** Docker Hub.
* **Deploy (GitOps):** ArgoCD.
* **Cluster:** Kubernetes local via Rancher Desktop.

---

## Passo a Passo da Construção

Este projeto foi construído seguindo 5 etapas principais para criar um pipeline GitOps funcional.

### Etapa 1: Aplicação FastAPI e Repositórios
1.  **Criação dos Repositórios:** Foram criados dois repositórios no GitHub:
    * `projeto-app` (este): Para o código-fonte da aplicação FastAPI.
    * `projeto-manifests`: Para os manifestos do Kubernetes que o ArgoCD irá monitorar.
2.  **Código da Aplicação:** O repositório `projeto-app` foi clonado localmente.
3.  Foram criados os arquivos `main.py` (com o código FastAPI), `requirements.txt` (com `fastapi` e `uvicorn[standard]`) e `Dockerfile` (usando uma base `python:3.10-slim`, copiando os arquivos, instalando dependências e rodando `uvicorn`).
4.  **Commit Inicial:** Os arquivos da aplicação foram enviados para o GitHub.

### Etapa 2: GitHub Actions (Pipeline CI/CD)
1.  **Criação dos Segredos:** No repositório `projeto-app`, foram configurados os "Actions Secrets" em `Settings > Secrets and variables > Actions`:
    * `DOCKER_USERNAME`: Nome de usuário do Docker Hub.
    * `DOCKER_PASSWORD`: Token de Acesso gerado no Docker Hub.
    * `SSH_PRIVATE_KEY`: Uma chave SSH privada.
2.  **Geração da Chave SSH:** Um novo par de chaves foi gerado localmente para permitir que o Actions escreva no `projeto-manifests`.
    ```bash
    ssh-keygen -t ed25519 -C "github-actions-key" -f ./github-actions-key
    ```
3.  O conteúdo de `github-actions-key` (chave privada) foi copiado para o segredo `SSH_PRIVATE_KEY`.
4.  **Configuração da Chave SSH:** O conteúdo de `github-actions-key.pub` (chave pública) foi adicionado como "Deploy Key" em `projeto-manifests` via `Settings > Deploy keys > Add deploy key`, marcando a opção "Allow write access".
5.  **Criação do Workflow:** No repositório `projeto-app`, o arquivo `.github/workflows/cicd.yml` foi criado. Ele contém dois jobs:
    * **`build_and_push` (CI):** Faz o checkout, login no Docker Hub, builda e publica a imagem.
    * **`update_manifest` (CD):** Depende do job anterior, faz checkout do `projeto-manifests` (usando `SSH_PRIVATE_KEY`), usa `sed` para substituir a tag da imagem no `deployment.yaml` e faz `git push` da alteração.

### Etapa 3: Manifestos do ArgoCD
1.  O repositório `projeto-manifests` foi clonado localmente.
2.  Foram criados os manifestos Kubernetes:
    * **`deployment.yaml`**: Define o `Deployment` da `projeto-app`, com 2 réplicas e a imagem `SEU_USUARIO_DOCKERHUB/projeto-app:latest` como placeholder.
    * **`service.yaml`**: Define o `Service` `projeto-app-service` do tipo `ClusterIP`, expondo a porta `8080` e mirando a `targetPort: 8000` do container.
3.  **Commit dos Manifestos:** Os arquivos foram enviados para o GitHub, onde o ArgoCD irá lê-los.

### Etapa 4: Configuração do ArgoCD
1.  **Acesso à Interface:** O acesso à interface web do ArgoCD foi feito via port-forward.
    ```bash
    kubectl port-forward svc/argocd-server -n argocd 8080:443
    ```
2.  **Login:** A senha inicial de `admin` foi obtida com o comando:
    ```bash
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
    ```
3.  **Conexão do Repositório:**
    * Na interface (acessada em `https://localhost:8080`), vá em `Settings > Repositories`.
    * Vá em `+ Connect repo using HTTPS`.
    * Preencha a URL HTTPS do `projeto-manifests` e vá em "Connect".
4.  **Criação do App:**
    * Vá em `Applications > + New App`.
    * **`General`**:
        * `Application Name`: `projeto-app`
        * `Project Name`: `default`
        * `Sync Policy`: `Automatic` (com `Prune Resources` e `Self Heal` marcados).
    * **`Source`**:
        * `Repository URL`: Selecione o repositório `projeto-manifests`.
        * `Revision`: `HEAD`
        * `Path`: `.`
    * **`Destination`**:
        * `Cluster URL`: `https://kubernetes.default.svc`
        * `Namespace`: `default`
    * Vá em `Create`. O ArgoCD automaticamente sincronizará a aplicação.

### Etapa 5: Teste do Fluxo Completo
1.  **Acesso Local:** O comando `kubectl port-forward` foi usado para expor o serviço.
    ```bash
    kubectl port-forward svc/projeto-app-service 8081:8080
    ```
2.  Acessei `http://localhost:8081/` no navegador e vi a mensagem `{"message": "Hello World"}`.
3.  **Teste de CI/CD:**
    * Alterei a mensagem de retorno no arquivo `main.py` do `projeto-app`.
    * Enviei a alteração para o GitHub:
    ```bash
    git add main.py
    git commit -m "feat: Update welcome message"
    git push origin main
    ```
4.  **Verificação:**
    * O pipeline do GitHub Actions foi acionado, publicou a nova imagem e atualizou o `projeto-manifests`.
    * O ArgoCD detectou a mudança, entrou em `OutOfSync` e se sincronizou automaticamente.
    * Recarreguei `http://localhost:8081/` e a nova mensagem apareceu, confirmando o sucesso do ciclo completo.

---

## Estrutura do Repositório

* `main.py`: O código-fonte da aplicação FastAPI.
* `Dockerfile`: As instruções para construir a imagem Docker da aplicação.
* `requirements.txt`: As dependências Python (FastAPI, Uvicorn).
* `.github/workflows/`: Contém o pipeline de CI/CD do GitHub Actions.

## Pré-requisitos (Para Replicar o Projeto)

* Conta no GitHub
* Conta no Docker Hub com token de acesso
* Rancher Desktop com Kubernetes habilitado
* `kubectl` configurado corretamente
* ArgoCD instalado no cluster local
* Git instalado
* Python 3 e Docker instalados

---

## Evidências da Entrega

Aqui estão as evidências do funcionamento completo do pipeline de CI/CD, conforme solicitado.

### 1. Build e Push no Docker Hub
Interface do Docker Hub mostrando a imagem sendo publicada com sucesso.

![Build e Push no Docker Hub](<img width="933" height="664" alt="Image" src="https://github.com/user-attachments/assets/25b51f96-9351-474e-8154-71457979e169" />)

### 2. Atualização Automática dos Manifestos
Commit automático feito pelo "GitHub Actions Bot" no repositório `projeto-manifests`, atualizando a tag da imagem.

![Atualização dos Manifestos](<img width="1359" height="670" alt="Image" src="https://github.com/user-attachments/assets/d6f205d0-81b5-4763-b657-d797a1767aa8" />)

### 3. ArgoCD Sincronizado
Captura de tela da interface do ArgoCD mostrando a aplicação com status `Healthy` (Saudável) e `Synced` (Sincronizado).

![ArgoCD Sincronizado](<img width="394" height="359" alt="Image" src="https://github.com/user-attachments/assets/15dae24d-6eb6-4d35-8840-a82051ea658d" />)

### 4. Pods em Execução no Kubernetes
Print do terminal com o comando `kubectl get pods` mostrando os pods da aplicação rodando.

![Pods em Execução](<img width="632" height="157" alt="Image" src="https://github.com/user-attachments/assets/847905aa-8d99-47e3-ae36-74b620447d62" />)

### 5. Resposta da Aplicação
Print do navegador (ou `curl`) acessando `http://localhost:8081/` e mostrando a mensagem atualizada após o ciclo de CI/CD.

![Resposta da Aplicação antes](<img width="595" height="257" alt="Image" src="https://github.com/user-attachments/assets/c48f5e4b-0f60-4e57-a33f-5e89fb6f7a45" />)

![Resposta da Aplicação depois](<img width="719" height="258" alt="Image" src="https://github.com/user-attachments/assets/33abdac1-c12e-4f90-ab49-2788e0e64786" />)
