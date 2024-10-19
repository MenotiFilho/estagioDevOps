# Desafio DevOps - Infraestrutura como Código com Terraform

## Análise Técnica do Código Original

### Estrutura Geral
O código Terraform original cria uma infraestrutura básica na AWS, incluindo uma VPC, uma subnet, um Internet Gateway, um grupo de segurança e uma instância EC2.

### Componentes Principais

1. **Provider e Variáveis**
   - Usa o provider AWS na região us-east-1.
   - Define variáveis para o nome do projeto e do candidato.

2. **Key Pair**
   - Gera um par de chaves RSA para acesso SSH à instância EC2.

3. **Rede**
   - Cria uma VPC com CIDR 10.0.0.0/16.
   - Define uma subnet pública com CIDR 10.0.1.0/24.
   - Configura um Internet Gateway e uma tabela de rotas para acesso à internet.

4. **Segurança**
   - Cria um grupo de segurança permitindo SSH de qualquer lugar (0.0.0.0/0).
   - Permite todo o tráfego de saída.

5. **Instância EC2**
   - Usa uma AMI Debian 12.
   - Tipo de instância t2.micro.
   - Volume root de 20GB.
   - Executa um script básico de user data para atualizar o sistema.

6. **Outputs**
   - Fornece a chave privada e o IP público da instância EC2.

### Pontos de Atenção
1. O acesso SSH está aberto para qualquer IP, o que é um risco de segurança.
2. Não há instalação ou configuração de aplicativos específicos na instância EC2.
3. A instância EC2 não tem tags além do nome.
4. O script de user data é básico, apenas atualizando o sistema.

## Modificações Implementadas

1. **Adição de Variável para IP do Administrador**
   ```hcl
   variable "admin_ip" {
     description = "Endereço IP do administrador para acesso SSH"
     type        = string
     default     = "SEU_IP_AQUI/32"
   }
   ```
   Esta variável foi adicionada para restringir o acesso SSH a um IP específico, melhorando a segurança.

2. **Modificação no Grupo de Segurança**
   ```hcl
   resource "aws_security_group" "main_sg" {
     # ... (código existente)

     ingress {
       description = "SSH from admin IP"
       from_port   = 22
       to_port     = 22
       protocol    = "tcp"
       cidr_blocks = [var.admin_ip]
     }

     ingress {
       description = "HTTP from anywhere"
       from_port   = 80
       to_port     = 80
       protocol    = "tcp"
       cidr_blocks = ["0.0.0.0/0"]
     }

     # ... (resto do código)
   }
   ```
   - Modificamos a regra de ingresso SSH para usar a variável `admin_ip`.
   - Adicionamos uma nova regra para permitir tráfego HTTP (porta 80) de qualquer lugar.

3. **Atualização do User Data da Instância EC2**
   ```hcl
   resource "aws_instance" "debian_ec2" {
     # ... (código existente)

     user_data = <<-EOF
                 #!/bin/bash
                 apt-get update -y
                 apt-get upgrade -y
                 apt-get install -y nginx
                 systemctl enable nginx
                 systemctl start nginx
                 EOF

     # ... (resto do código)
   }
   ```
   O script de user data foi expandido para incluir a instalação e inicialização do Nginx.

4. **Adição de Novo Output**
   ```hcl
   output "nginx_url" {
     description = "URL para acessar o servidor Nginx"
     value       = "http://${aws_instance.debian_ec2.public_ip}"
   }
   ```
   Este novo output fornece a URL para acessar o servidor Nginx instalado.

## Instruções de Utilização

1. **Pré-requisitos**
   - Terraform instalado (versão 0.12+)
   - AWS CLI configurado com suas credenciais

2. **Configuração**
   - Clone este repositório:
     ```
     git clone [URL_DO_REPOSITORIO]
     cd [NOME_DO_DIRETORIO]
     ```
   - Abra o arquivo `main.tf` e atualize a variável `admin_ip` com seu endereço IP:
     ```hcl
     variable "admin_ip" {
       default = "SEU_IP_AQUI/32"
     }
     ```

3. **Inicialização do Terraform**
   ```
   terraform init
   ```

4. **Planejamento da Infraestrutura**
   ```
   terraform plan
   ```
   Revise as mudanças que serão aplicadas.

5. **Aplicação da Infraestrutura**
   ```
   terraform apply
   ```
   Digite 'yes' quando solicitado para confirmar a criação dos recursos.

6. **Acesso à Instância EC2**
   - Use o output `private_key` para salvar a chave privada em um arquivo:
     ```
     terraform output -raw private_key > private_key.pem
     chmod 400 private_key.pem
     ```
   - Conecte-se via SSH:
     ```
     ssh -i private_key.pem admin@[EC2_PUBLIC_IP]
     ```
   Substitua [EC2_PUBLIC_IP] pelo valor do output `ec2_public_ip`.

7. **Acesso ao Nginx**
   - Abra um navegador e acesse a URL fornecida no output `nginx_url`.

8. **Limpeza da Infraestrutura**
   Quando não precisar mais dos recursos:
   ```
   terraform destroy
   ```
   Digite 'yes' quando solicitado para confirmar a exclusão dos recursos.
