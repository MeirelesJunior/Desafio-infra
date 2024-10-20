# Desafio de Infraestrutura como Código com Terraform

## 1. Descrição Técnica da Infraestrutura

O código Terraform define a criação de uma infraestrutura básica na AWS, que inclui os seguintes recursos:

- **VPC (Virtual Private Cloud)**: Criada para isolar a infraestrutura em um ambiente virtual. A CIDR block é definida como `10.0.0.0/16`, permitindo uma ampla gama de endereços IP.
- **Subnet**: Uma sub-rede foi criada dentro da VPC com CIDR `10.0.1.0/24`, o que permite um total de 256 endereços IP.
- **Internet Gateway**: Um gateway de internet foi associado à VPC para permitir o acesso à internet.
- **Route Table**: Uma tabela de rotas foi configurada para permitir o tráfego da sub-rede para a internet através do gateway.
- **Security Group**: Um grupo de segurança foi criado para permitir acesso SSH (porta 22) de um IP específico e todo o tráfego de saída.
- **Instância EC2**: Uma instância EC2 foi lançada com a imagem do Debian 12, configurada para permitir SSH e com Nginx instalado automaticamente.

### Observações

O código foi escrito de forma a seguir as melhores práticas de organização e segurança. As variáveis de projeto e candidato foram utilizadas para nomear os recursos de forma dinâmica.

## 2. Código `main.tf` Modificado

```hcl
provider "aws" {
  region = "us-east-1"
}

variable "projeto" {
  description = "Nome do projeto"
  type        = string
  default     = "VExpenses"
}

variable "candidato" {
  description = "Nome do candidato"
  type        = string
  default     = "SeuNome"
}

resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048

  tags = {
    Owner      = var.candidato
    Project    = var.projeto
    Environment = "Development"
  }
}

resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "${var.projeto}-${var.candidato}-key"
  public_key = tls_private_key.ec2_key.public_key_openssh

  tags = {
    Owner      = var.candidato
    Project    = var.projeto
    Environment = "Development"
  }
}

resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name        = "${var.projeto}-${var.candidato}-vpc"
    Owner       = var.candidato
    Project     = var.projeto
    Environment = "Development"
  }
}

resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name        = "${var.projeto}-${var.candidato}-subnet"
    Owner       = var.candidato
    Project     = var.projeto
    Environment = "Development"
  }
}

resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name        = "${var.projeto}-${var.candidato}-igw"
    Owner       = var.candidato
    Project     = var.projeto
    Environment = "Development"
  }
}

resource "aws_route_table" "main_route_table" {
  vpc_id = aws_vpc.main_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main_igw.id
  }

  tags = {
    Name        = "${var.projeto}-${var.candidato}-route_table"
    Owner       = var.candidato
    Project     = var.projeto
    Environment = "Development"
  }
}

resource "aws_route_table_association" "main_association" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_route_table.id

  tags = {
    Name        = "${var.projeto}-${var.candidato}-route_table_association"
    Owner       = var.candidato
    Project     = var.projeto
    Environment = "Development"
  }
}

resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH de um IP específico e todo o tráfego de saída"
  vpc_id      = aws_vpc.main_vpc.id

  # Regras de entrada
  ingress {
    description      = "Allow SSH from specified IP"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["X.X.X.X/32"]  # Substitua por seu IP específico
  }

  # Regras de saída
  egress {
    description      = "Allow all outbound traffic"
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.projeto}-${var.candidato}-sg"
    Owner       = var.candidato
    Project     = var.projeto
    Environment = "Development"
  }
}

data "aws_ami" "debian12" {
  most_recent = true

  filter {
    name   = "name"
    values = ["debian-12-amd64-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["679593333241"]
}

resource "aws_instance" "debian_ec2" {
  ami             = data.aws_ami.debian12.id
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.main_subnet.id
  key_name        = aws_key_pair.ec2_key_pair.key_name
  security_groups = [aws_security_group.main_sg.name]

  associate_public_ip_address = true

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    delete_on_termination = true
  }

  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
              apt-get install nginx -y
              systemctl start nginx
              systemctl enable nginx
              EOF

  tags = {
    Name        = "${var.projeto}-${var.candidato}-ec2"
    Owner       = var.candidato
    Project     = var.projeto
    Environment = "Development"
  }
}

output "private_key" {
  description = "Chave privada para acessar a instância EC2"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}

output "ec2_public_ip" {
  description = "Endereço IP público da instância EC2"
  value       = aws_instance.debian_ec2.public_ip
}
```

# 3. Descrição Técnica das Melhorias
Melhorias de Segurança
As regras de segurança do grupo foram ajustadas para permitir acesso SSH apenas de um IP específico, reduzindo a exposição da instância a ataques externos.

Automação da Instalação do Nginx
A instância EC2 foi configurada para instalar e iniciar automaticamente o Nginx após a criação, utilizando o user_data.

Adição de Tags
Tags consistentes foram adicionadas a todos os recursos, melhorando a organização e o gerenciamento da infraestrutura. As tags incluem Owner, Project e Environment.

Resultados Esperados
Melhoria na segurança, facilidade na automação da instalação do Nginx e melhor gerenciamento através de tagging.

4. Instruções de Uso
Pré-requisitos
Certifique-se de ter o Terraform instalado.
Tenha uma conta na AWS e as credenciais configuradas no seu ambiente local.
Um terminal com acesso à internet.


Claro! Aqui está o README.md reorganizado, mantendo o código e estruturando as informações de maneira legível:

markdown
Copy code
# Desafio de Infraestrutura como Código com Terraform

## 1. Descrição Técnica da Infraestrutura

O código Terraform define a criação de uma infraestrutura básica na AWS, que inclui os seguintes recursos:

- **VPC (Virtual Private Cloud)**: Criada para isolar a infraestrutura em um ambiente virtual. A CIDR block é definida como `10.0.0.0/16`, permitindo uma ampla gama de endereços IP.
- **Subnet**: Uma sub-rede foi criada dentro da VPC com CIDR `10.0.1.0/24`, o que permite um total de 256 endereços IP.
- **Internet Gateway**: Um gateway de internet foi associado à VPC para permitir o acesso à internet.
- **Route Table**: Uma tabela de rotas foi configurada para permitir o tráfego da sub-rede para a internet através do gateway.
- **Security Group**: Um grupo de segurança foi criado para permitir acesso SSH (porta 22) de um IP específico e todo o tráfego de saída.
- **Instância EC2**: Uma instância EC2 foi lançada com a imagem do Debian 12, configurada para permitir SSH e com Nginx instalado automaticamente.

### Observações

O código foi escrito de forma a seguir as melhores práticas de organização e segurança. As variáveis de projeto e candidato foram utilizadas para nomear os recursos de forma dinâmica.

## 2. Código `main.tf` Modificado

```hcl
provider "aws" {
  region = "us-east-1"
}

variable "projeto" {
  description = "Nome do projeto"
  type        = string
  default     = "VExpenses"
}

variable "candidato" {
  description = "Nome do candidato"
  type        = string
  default     = "SeuNome"
}

resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048

  tags = {
    Owner      = var.candidato
    Project    = var.projeto
    Environment = "Development"
  }
}

resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "${var.projeto}-${var.candidato}-key"
  public_key = tls_private_key.ec2_key.public_key_openssh

  tags = {
    Owner      = var.candidato
    Project    = var.projeto
    Environment = "Development"
  }
}

resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name        = "${var.projeto}-${var.candidato}-vpc"
    Owner       = var.candidato
    Project     = var.projeto
    Environment = "Development"
  }
}

resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name        = "${var.projeto}-${var.candidato}-subnet"
    Owner       = var.candidato
    Project     = var.projeto
    Environment = "Development"
  }
}

resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name        = "${var.projeto}-${var.candidato}-igw"
    Owner       = var.candidato
    Project     = var.projeto
    Environment = "Development"
  }
}

resource "aws_route_table" "main_route_table" {
  vpc_id = aws_vpc.main_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main_igw.id
  }

  tags = {
    Name        = "${var.projeto}-${var.candidato}-route_table"
    Owner       = var.candidato
    Project     = var.projeto
    Environment = "Development"
  }
}

resource "aws_route_table_association" "main_association" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_route_table.id

  tags = {
    Name        = "${var.projeto}-${var.candidato}-route_table_association"
    Owner       = var.candidato
    Project     = var.projeto
    Environment = "Development"
  }
}

resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH de um IP específico e todo o tráfego de saída"
  vpc_id      = aws_vpc.main_vpc.id

  # Regras de entrada
  ingress {
    description      = "Allow SSH from specified IP"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["X.X.X.X/32"]  # Substitua por seu IP específico
  }

  # Regras de saída
  egress {
    description      = "Allow all outbound traffic"
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.projeto}-${var.candidato}-sg"
    Owner       = var.candidato
    Project     = var.projeto
    Environment = "Development"
  }
}

data "aws_ami" "debian12" {
  most_recent = true

  filter {
    name   = "name"
    values = ["debian-12-amd64-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["679593333241"]
}

resource "aws_instance" "debian_ec2" {
  ami             = data.aws_ami.debian12.id
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.main_subnet.id
  key_name        = aws_key_pair.ec2_key_pair.key_name
  security_groups = [aws_security_group.main_sg.name]

  associate_public_ip_address = true

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    delete_on_termination = true
  }

  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
              apt-get install nginx -y
              systemctl start nginx
              systemctl enable nginx
              EOF

  tags = {
    Name        = "${var.projeto}-${var.candidato}-ec2"
    Owner       = var.candidato
    Project     = var.projeto
    Environment = "Development"
  }
}

output "private_key" {
  description = "Chave privada para acessar a instância EC2"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}

output "ec2_public_ip" {
  description = "Endereço IP público da instância EC2"
  value       = aws_instance.debian_ec2.public_ip
}
3. Descrição Técnica das Melhorias
Melhorias de Segurança
As regras de segurança do grupo foram ajustadas para permitir acesso SSH apenas de um IP específico, reduzindo a exposição da instância a ataques externos.

Automação da Instalação do Nginx
A instância EC2 foi configurada para instalar e iniciar automaticamente o Nginx após a criação, utilizando o user_data.

Adição de Tags
Tags consistentes foram adicionadas a todos os recursos, melhorando a organização e o gerenciamento da infraestrutura. As tags incluem Owner, Project e Environment.

Resultados Esperados
Melhoria na segurança, facilidade na automação da instalação do Nginx e melhor gerenciamento através de tagging.

4. Instruções de Uso
Pré-requisitos
Certifique-se de ter o Terraform instalado.
Tenha uma conta na AWS e as credenciais configuradas no seu ambiente local.
Um terminal com acesso à internet.
Passos para Inicializar e Aplicar a Configuração Terraform
Clone o Repositório ou Crie um Diretório:
Clone o Repositório ou Crie um Diretório:


Clone o Repositório ou Crie um Diretório:
mkdir meu_projeto_terraform
cd meu_projeto_terraform

Inicialize o Terraform:
Execute o seguinte comando para inicializar o Terraform:
terraform init

Verifique o Plano de Execução:
Execute o comando para verificar o que será criado:
terraform plan


Aplique a Configuração:
Para criar a infraestrutura, execute:
terraform apply

Acessar a Instância EC2 e Verificar o Nginx
Após o Terraform criar a instância EC2, você pode verificar a instalação do Nginx de duas maneiras:

Conectar via SSH na Instância: Copie o IP público da instância (disponível na saída do Terraform ou no console da AWS) e use o comando a seguir para se conectar à instância. Certifique-se de que o arquivo de chave privada (.pem) gerado esteja salvo corretamente.


ssh -i /caminho/para/sua/chave.pem ec2-user@IP_PUBLICO_DA_INSTANCIA
Uma vez conectado à instância, você pode verificar o status do Nginx rodando o comando:


sudo systemctl status nginx
Verificar no Navegador: Abra o navegador e acesse o IP público da instância EC2. Se o Nginx estiver instalado corretamente, você verá a página padrão de boas-vindas do Nginx.
http://IP_PUBLICO_DA_INSTANCIA

5. Destruir a Infraestrutura (Opcional)
Após testar tudo, se você quiser destruir os recursos criados (VPC, EC2, etc.), execute o comando:
terraform destroy
Isso removerá todos os recursos da AWS que foram criados pelo Terraform.

Considerações Finais
Certifique-se de acompanhar a saída dos comandos e monitorar a criação da infraestrutura na AWS para verificar se tudo está funcionando conforme o esperado.






