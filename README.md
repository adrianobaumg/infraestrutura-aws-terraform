# Descrição do Código Terraform

O código Terraform fornecido provisiona uma infraestrutura na AWS, configurando uma VPC, uma sub-rede, um gateway de internet, uma tabela de rotas, um grupo de segurança, uma instância EC2 e uma chave SSH para acesso remoto.

## 1. Configuração do Provedor AWS
O bloco `provider "aws"` define a região onde os recursos serão criados: `us-east-1`.

## 2. Definição de Variáveis
- **projeto**: Nome do projeto (padrão: "VExpenses").
- **candidato**: Nome do candidato (padrão: "SeuNome").

## 3. Geração de Chave SSH
- `resource "tls_private_key" "ec2_key"`: Gera uma chave privada RSA.
- `resource "aws_key_pair" "ec2_key_pair"`: Cria um par de chaves na AWS usando a chave pública gerada.

## 4. Configuração da VPC e Sub-rede
- `resource "aws_vpc" "main_vpc"`: Cria uma VPC com bloco CIDR `10.0.0.0/16`.
- `resource "aws_subnet" "main_subnet"`: Cria uma sub-rede `10.0.1.0/24` na zona de disponibilidade `us-east-1a`.

## 5. Configuração do Acesso à Internet
- `resource "aws_internet_gateway" "main_igw"`: Cria um gateway de internet.
- `resource "aws_route_table" "main_route_table"`: Define uma tabela de rotas para permitir o tráfego da VPC para a internet.
- `resource "aws_route_table_association" "main_association"`: Associa a sub-rede à tabela de rotas.

## 6. Grupo de Segurança (Security Group)
- `resource "aws_security_group" "main_sg"`:
  - Permite conexões SSH (porta 22) de qualquer origem (`0.0.0.0/0`).
  - Permite todo o tráfego de saída (egress).

## 7. Obtenção da Imagem AMI
- `data "aws_ami" "debian12"`: Obtém a AMI mais recente do Debian 12.

## 8. Criação da Instância EC2
- `resource "aws_instance" "debian_ec2"`:
  - Utiliza a AMI Debian 12.
  - Tipo da instância: `t2.micro`.
  - Conectada à sub-rede criada.
  - Associada ao grupo de segurança criado.
  - Possui um volume raiz de 20 GB (`gp2`).
  - Executa um script de inicialização que realiza `apt-get update` e `apt-get upgrade`.

## 9. Saídas (Outputs)
- `private_key`: Exibe a chave privada gerada (sensível).
- `ec2_public_ip`: Exibe o endereço IP público da instância criada.

# 2° Parte: Melhorias

## 1. Salvamento da Chave Privada em um Arquivo Local

**Melhoria**: Adição do recurso `local_file` para salvar a chave privada gerada (`tls_private_key.ec2_key.private_key_pem`) em um arquivo local chamado `chave-privada.pem`.

**Resultado Esperado**: A chave privada é armazenada de forma persistente no sistema de arquivos local, permitindo que o usuário acesse a instância EC2 posteriormente sem precisar copiar manualmente o valor do output.

**Justificativa**: No código original, a chave privada era apenas exibida como output, o que exigia que o usuário a copiasse manualmente. Agora, o arquivo é criado automaticamente com permissões restritas (`0400`), garantindo segurança e praticidade.

## 2. Restrição de Acesso SSH a um IP Específico

**Melhoria**: No `aws_security_group`, a regra de entrada (`ingress`) para SSH foi alterada para permitir conexões apenas de um IP específico (`123.45.67.89/32`), em vez de permitir acesso de qualquer lugar (`0.0.0.0/0`).

**Resultado Esperado**: A instância EC2 só aceitará conexões SSH a partir do IP especificado, reduzindo significativamente a superfície de ataque.

**Justificativa**: Permitir SSH de qualquer IP é uma prática insegura, pois expõe a instância a tentativas de brute force e outros ataques. Restringir o acesso a um IP específico aumenta a segurança.

## 3. Adição de Regra para Tráfego HTTP

**Melhoria**: Foi adicionada uma nova regra de entrada (`ingress`) no `aws_security_group` para permitir tráfego HTTP na porta 80.

**Resultado Esperado**: A instância EC2 poderá receber requisições HTTP, o que é essencial para servir conteúdo web (por exemplo, um servidor Nginx).

**Justificativa**: Como o `user_data` instala e configura o Nginx, é necessário permitir tráfego HTTP para que o servidor web funcione corretamente.

## 4. Uso de `vpc_security_group_ids` em vez de `security_groups`

**Melhoria**: No recurso `aws_instance`, o parâmetro `security_groups` foi substituído por `vpc_security_group_ids`.

**Resultado Esperado**: A instância EC2 é associada ao security group criado anteriormente, mas agora de forma explícita e compatível com VPC.

**Justificativa**: O uso de `vpc_security_group_ids` é mais apropriado em ambientes VPC, pois garante que o security group seja aplicado corretamente à instância dentro da VPC especificada.

## 5. Instalação e Configuração do Nginx via `user_data`

**Melhoria**: O `user_data` da instância EC2 foi expandido para incluir a instalação e configuração do Nginx, além de desabilitar o login SSH como root.

**Resultado Esperado**: A instância EC2 será provisionada com o Nginx instalado e em execução, pronto para servir conteúdo web. Além disso, o login SSH como root será desabilitado, aumentando a segurança.

**Justificativa**: Automatizar a instalação do Nginx via `user_data` garante que a instância esteja pronta para uso imediatamente após a criação. Desabilitar o login SSH como root é uma prática recomendada de segurança.


## Instruções de Uso
### 1. Inicialização do Terraform
```bash
terraform init
```

### 2. Revisão do Plano de Execução
```bash
terraform plan
```

### 3. Aplicação da Configuração
```bash
terraform apply
```
Digite `yes` quando solicitado.

### 4. Conexão à Instância EC2
```bash
ssh -i chave-privada.pem admin@$(terraform output -raw ec2_public_ip)
```

### 5. Verificação do Nginx
Acesse no navegador:
```
http://<IP_PUBLICO_EC2>
```

### 6. Limpeza de Recursos
```bash
terraform destroy
```

## Referências
- [Documentação Oficial do Provedor AWS para Terraform](https://www.terraform.io/docs/providers/aws/index.html)
- [Documentação Oficial da AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)
- [Documentação do Debian 12](https://www.debian.org/releases/stable/amd64/)

