# Pipeline CI/CD com GitHub Actions

Este repositório demonstra uma pipeline CI/CD completa com GitHub Actions, incluindo build, análise, testes, publicação de imagem Docker e deploy em Kubernetes.

## Visão Geral da Pipeline

A pipeline principal está definida em `.github/workflows/main.yml` e orquestra os seguintes estágios:

1. `build` - checkout do código, análise do Dockerfile com Hadolint e build da solução .NET.
2. `testes` - invoca o workflow reutilizável `.github/workflows/testes.yml`, que executa:
   - testes unitários
   - testes de integração com PostgreSQL em serviço containerizado
   - análise SonarQube
3. `release` - publica uma imagem Docker no Docker Hub e executa uma análise de segurança com Trivy.
4. `deploy-homo` - chama o workflow reutilizável `.github/workflows/deploy.yml` para deploy em homologação no Kubernetes.
5. `teste-e2e` - executa testes end-to-end com Selenium em um container Chrome.
6. `deploy-producao` - realiza deploy em produção após aprovação dos testes E2E.

## Estrutura dos Workflows

### `.github/workflows/main.yml`

- `on: [workflow_dispatch, push.branches: [main]]`
- `permissions`: controla permissões mínimas para leitura de código, token OIDC e eventos de segurança.
- `build`: valida o `Dockerfile` e compila a solução com `dotnet build`.
- `testes`: reutiliza `.github/workflows/testes.yml` via `workflow_call`.
- `release`: faz login no Docker Hub, publica a imagem com tags `latest` e `v${{github.run_number}}` e gera relatório SARIF do Trivy.
- `deploy-homo`: executa deploy em homologação usando `.github/workflows/deploy.yml`.
- `teste-e2e`: executa testes de interface no ambiente homologação.
- `deploy-producao`: faz deploy em produção após os testes E2E.

### `.github/workflows/testes.yml`

Esse workflow reutilizável agrega:

- `unit-test`
  - checkout do código
  - setup do .NET 8.0
  - execução de `dotnet test` no projeto de testes unitários
- `integration-test`
  - checkout do código
  - setup do .NET 8.0
  - serviço PostgreSQL em container `postgres:alpine3.22`
  - execução de testes de integração com `ConnectionStrings__DefaultConnection`
- `sonarqube`
  - setup do JDK e do .NET
  - instalação e execução do `dotnet-sonarscanner`
  - verificação do Quality Gate do SonarQube

### `.github/workflows/deploy.yml`

Workflow reutilizável para deploy em Kubernetes:

- Login na AWS com `KoalaOps/login-aws@v1`
- configuração de contexto Kubernetes com `azure/k8s-set-context@v5`
- atualização dinâmica do `host` no manifest `k8s/deployment.yaml`
- deploy com `Azure/k8s-deploy@v5`

## Segredos e Variáveis de Ambiente

A pipeline usa variáveis de repositório (`vars`) e segredos (`secrets`):

- `DOCKERHUB_USER`
- `DOCKERHUB_PWD`
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `K8S_CONFIG`
- `K8S_NAMESPACES`
- `SONAR_HOST_URL`
- `SONAR_TOKEN`
- `BASE_URL`

## Como funciona

- Um push para `main` ou execução manual dispara `.github/workflows/main.yml`.
- O fluxo é fragmentado em jobs independentes e workflows reutilizáveis para manter o pipeline limpo e modular.
- A chamada de workflows reutilizáveis (`uses: .../.github/workflows/*.yml@main`) permite reaproveitar lógica de testes e deploy entre pipelines.
- O deploy em homologação acontece antes dos testes E2E, garantindo validação no ambiente real.
- O deploy em produção só é executado se todos os passos anteriores forem aprovados.

## Pontos importantes

- A pipeline publica imagens Docker versão `latest` e `v${{github.run_number}}` no Docker Hub.
- O scan de segurança Trivy gera SARIF para integração com GitHub Code Scanning.
- O deploy Kubernetes utiliza manifestos locais e atualização dinâmica de host via `sed`.
- O ambiente de testes de integração usa um serviço PostgreSQL isolado para dependências.

## Uso local e recomendações

- Para rodar os workflows localmente, você pode simular em um ambiente GitHub Actions com o `act` ou ajustar a execução direta dos comandos do `main.yml`.
- Garanta que os segredos necessários estejam configurados no repositório GitHub antes de habilitar o deploy automático.

---

**Observação:** este README é focado na construção da pipeline CI/CD com GitHub Actions. Para instruções de execução da aplicação, consulte a documentação de projeto ou adicione uma seção dedicada separada.

---

## Assista a apresentação do projeto:

https://youtu.be/zhXkw4fI0Bk
