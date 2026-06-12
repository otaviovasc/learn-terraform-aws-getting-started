# Terraform + LocalStack — Infraestrutura como Código (IaC)

Este repositório documenta a execução passo a passo do tutorial de IaC com **Terraform**, utilizando **LocalStack** (via Docker) como ambiente AWS local — sem dependência de conta AWS real.

---

## Sumário

- [Pré-requisitos](#pré-requisitos)
- [Passo a Passo](#passo-a-passo)
  - [1. Instalação das Ferramentas](#1-instalação-das-ferramentas)
  - [2. Estrutura do Projeto](#2-estrutura-do-projeto)
  - [3. Formatação dos Arquivos](#3-formatação-dos-arquivos)
  - [4. Inicialização do Workspace](#4-inicialização-do-workspace)
  - [5. Validação da Configuração](#5-validação-da-configuração)
  - [6. Plano de Execução](#6-plano-de-execução)
  - [7. Criação da Infraestrutura](#7-criação-da-infraestrutura)
  - [8. Inspeção do Estado](#8-inspeção-do-estado)
- [Recursos Provisionados no LocalStack](#recursos-provisionados-no-localstack)
- [Arquivos do Projeto](#arquivos-do-projeto)
- [Comandos Utilizados](#comandos-utilizados)
- [Limpeza](#limpeza)
- [Troubleshooting](#troubleshooting)

---

## Pré-requisitos

| Ferramenta | Versão | Instalação |
|---|---|---|
| macOS | Darwin arm64 | — |
| Docker | `v29.2.0` | [Docker Desktop](https://www.docker.com/products/docker-desktop/) |
| Terraform | `v1.15.6` | `brew install hashicorp/tap/terraform` |
| LocalStack CLI | `v2026.5.0` | `brew install localstack/tap/localstack-cli` |

---

## Passo a Passo

### 1. Instalação das Ferramentas

**Terraform:**

```bash
brew install hashicorp/tap/terraform
terraform --version
```

> ![Terraform install](images/01-terraform-install.png)
>
> *Terraform v1.15.6 instalado via Homebrew*

**LocalStack CLI e Docker:**

```bash
brew install localstack/tap/localstack-cli
docker run --rm -d --name localstack \
  -p 4566:4566 -p 4510-4559:4510-4559 \
  -e LOCALSTACK_AUTH_TOKEN=seu-token \
  localstack/localstack:latest
```

Verifique com `curl localhost:4566/_localstack/health` — o serviço `ec2` deve aparecer como `available`.

---

### 2. Estrutura do Projeto

```
ponderada-romualdo-8/
├── main.tf              # Provider + resource EC2 (LocalStack)
├── terraform.tf         # required_providers + required_version
├── .gitignore           # Exclusão de .terraform/, *.tfstate
├── .terraform.lock.hcl  # Lock file de providers
└── README.md            # Esta documentação
```

**Arquivo `terraform.tf`** — define a versão do Terraform e o provider AWS:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.92"
    }
  }

  required_version = ">= 1.2"
}
```

**Arquivo `main.tf`** — provider apontando para LocalStack e resource EC2:

```hcl
provider "aws" {
  region                      = "us-east-1"
  access_key                  = "test"
  secret_key                  = "test"
  skip_credentials_validation = true
  skip_requesting_account_id  = true
  skip_metadata_api_check     = true

  endpoints {
    ec2 = "http://localhost:4566"
  }
}

resource "aws_instance" "app_server" {
  ami           = "ami-12345678" # LocalStack mock AMI
  instance_type = "t2.micro"

  tags = {
    Name = "learn-terraform-localstack"
  }
}
```

> **Nota:** O provider é configurado com `endpoints` customizados para o LocalStack (`localhost:4566`). As validações de credenciais são desabilitadas (`skip_credentials_validation`, `skip_requesting_account_id`, `skip_metadata_api_check`). A AMI `ami-12345678` é um mock fornecido pelo LocalStack.

> ![Project structure](images/02-project-structure.png)
>
> *Arquivos do projeto*

---

### 3. Formatação dos Arquivos

```bash
terraform fmt
```

> ![Terraform fmt](images/03-terraform-fmt.png)
>
> *Formatação aplicada nos arquivos .tf*

---

### 4. Inicialização do Workspace

```bash
terraform init
```

**Saída:**

```
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.92"...
- Installing hashicorp/aws v5.100.0...
- Installed hashicorp/aws v5.100.0 (signed by HashiCorp)

Terraform has been successfully initialized!
```

> ![Terraform init](images/04-terraform-init.png)
>
> *Provider AWS v5.100.0 instalado*

---

### 5. Validação da Configuração

```bash
terraform validate
```

**Saída:**

```
Success! The configuration is valid.
```

> ![Terraform validate](images/05-terraform-validate.png)
>
> *Configuração válida*

---

### 6. Plano de Execução

```bash
terraform plan
```

**Resumo do plano:**

| Campo | Valor |
|---|---|
| Ação | `+ create` |
| Recurso | `aws_instance.app_server` |
| AMI | `ami-12345678` (mock LocalStack) |
| Instance Type | `t2.micro` |
| Região | `us-east-1` |
| Tag | `Name = learn-terraform-localstack` |

> ![Terraform plan](images/06-terraform-plan.png)
>
> *Plano de execução: 1 recurso a ser criado*

---

### 7. Criação da Infraestrutura

```bash
terraform apply -auto-approve
```

**Saída:**

```
aws_instance.app_server: Creating...
aws_instance.app_server: Creation complete after 11s [id=i-abdd4450dcf967a96]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> ![Terraform apply](images/07-terraform-apply.png)
>
> *Instância EC2 criada com sucesso no LocalStack*

---

### 8. Inspeção do Estado

```bash
terraform state list
terraform show
```

**Recursos no state:**

```
aws_instance.app_server
```

**Detalhes da instância:**

```
resource "aws_instance" "app_server" {
    id                    = "i-abdd4450dcf967a96"
    ami                   = "ami-12345678"
    instance_type         = "t2.micro"
    instance_state        = "running"
    private_ip            = "10.120.58.121"
    public_ip             = "54.214.29.121"
    subnet_id             = "subnet-231dd16ab367d3bd3"
    availability_zone     = "us-east-1a"
    tags                  = { "Name" = "learn-terraform-localstack" }
}
```

> ![Terraform state](images/08-terraform-state.png)
>
> *Estado do Terraform após provisionamento*

---

## Recursos Provisionados no LocalStack

Abaixo os itens provisionados via Terraform no ambiente LocalStack.

### Instância EC2 (Mock)

| Atributo | Valor |
|---|---|
| **Instance ID** | `i-abdd4450dcf967a96` |
| **Nome** | `learn-terraform-localstack` |
| **AMI** | `ami-12345678` (mock) |
| **Tipo** | `t2.micro` |
| **Estado** | `running` |
| **Região** | `us-east-1` |
| **Zona** | `us-east-1a` |
| **Private IP** | `10.120.58.121` |
| **Public IP** | `54.214.29.121` |

### Infraestrutura de Rede (gerada pelo LocalStack)

| Recurso | ID |
|---|---|
| **Subnet** | `subnet-231dd16ab367d3bd3` |
| **Network Interface** | `eni-06014836ac768cfcd` |
| **Root Volume (EBS)** | `vol-547c636039e10d296` (8 GB gp2) |

### Resumo

| Recurso | Tipo Terraform | ID | Status |
|---|---|---|---|
| EC2 Instance | `aws_instance` | `i-abdd4450dcf967a96` | ✅ Running |
| Subnet | (gerado pelo LocalStack) | `subnet-231dd16ab367d3bd3` | ✅ Active |
| ENI | (gerado pelo LocalStack) | `eni-06014836ac768cfcd` | ✅ Attached |
| EBS Volume | (root_block_device) | `vol-547c636039e10d296` | ✅ Attached |

---

## Arquivos do Projeto

```
ponderada-romualdo-8/
├── .gitignore           # Arquivos ignorados pelo git
├── .terraform/          # Provider plugins (não versionado)
├── .terraform.lock.hcl  # Lock file de providers
├── main.tf              # Configuração principal (LocalStack)
├── terraform.tf         # Requisitos de versão
├── README.md            # Esta documentação
└── terraform.tfstate    # Estado da infraestrutura (não versionado)
```

---

## Comandos Utilizados

| Comando | Propósito |
|---|---|
| `docker run -d --name localstack ...` | Inicia o LocalStack em container |
| `curl localhost:4566/_localstack/health` | Verifica saúde do LocalStack |
| `terraform fmt` | Formata arquivos `.tf` |
| `terraform init` | Inicializa workspace, baixa providers |
| `terraform validate` | Valida sintaxe e consistência |
| `terraform plan` | Gera preview das mudanças |
| `terraform apply -auto-approve` | Aplica mudanças sem confirmação |
| `terraform state list` | Lista recursos no state |
| `terraform show` | Exibe o estado completo |
| `terraform destroy` | Remove toda a infraestrutura |

---

## Limpeza

Para remover a infraestrutura do LocalStack e parar o container:

```bash
# Remover recursos gerenciados pelo Terraform
terraform destroy -auto-approve

# Parar e remover o container LocalStack
docker stop localstack
```

> ⚠️ Como o LocalStack roda localmente, **não há custos** de AWS. Mas é boa prática limpar os recursos após o uso.

---

## Troubleshooting

### Erro de permissão IAM na AWS real

Ao tentar executar contra a AWS real, o erro abaixo ocorreu devido à falta de permissões EC2 no usuário IAM:

```
Error: reading EC2 Instance Type (t2.micro):
UnauthorizedOperation: You are not authorized to perform this operation.
User: arn:aws:iam::934990476107:user/otavio is not authorized to perform:
ec2:DescribeInstanceTypes
```

**Solução adotada:** Substituir AWS real pelo **LocalStack** — um ambiente AWS local que não requer credenciais nem permissões IAM.

### LocalStack não inicia (erro de mount no macOS)

```
Error: failed to create shim task: OCI runtime create failed
```

**Causa:** Docker Desktop com virtiofs e conflito de paths.

**Solução:** Usar `docker run` diretamente ao invés de `localstack start`:

```bash
docker run --rm -d --name localstack \
  -p 4566:4566 -p 4510-4559:4510-4559 \
  localstack/localstack:latest
```

---

## Referências

- [HashiCorp — AWS Get Started Tutorial](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-create)
- [LocalStack Documentation](https://docs.localstack.cloud/overview/)
- [Terraform AWS Provider — Custom Endpoints](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/guides/custom-service-endpoints)
