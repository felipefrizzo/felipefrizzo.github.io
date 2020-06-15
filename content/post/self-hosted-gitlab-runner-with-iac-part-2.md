+++
Tags = ["terraform", "ansible", "packer", "ci/cd", "self-hosted"]
Categories = ["Iac", "Terraform", "Ansible", "Packer", "CI/CD"]
date = "2020-06-18T15:15:32-03:00"
title = "Gitlab Runner de alta disponibilidade rodando na sua infraestrutura - Parte 2"
author = "Felipe Frizzo"
Description = ""
+++

Na primeira parte desta série sobre como rodar o [gitlab runner e ter alta disponibilidade](http://felipefrizzo.github.io/post/self-hosted-gitlab-runner-with-iac-part-1/), falamos sobre as tecnologia empregadas no projeto e também abordamos como utilizar o ansible para fazer a configuração e a instalação do runner e suas dependencias. Agora iremos falar sobre packer e terraform, lembrando você pode acompanhar o código pelo [repositório](https://github.com/felipefrizzo/gitlab-runner).

### Packer

Como já falamos na primeira parte o packer é uma ferramenta para automação da criação de imagens, como por exemplo AMI (Amazon machine images). A estrutura de arquivos do packer é bem simples, é apenas um arquivo json que irá conter as instruções para executar.

```shell
❯ tree
.
├── README.md
├── ansible
├── packer
│   └── gitlab-runner.json
└── terraform

11 directories, 8 files
```

No manifesto do packer iremos ter os seguintes seções: `variables, builders e provisioners`

* Variables:  
  As variáveis permitem parametrizar seus manifestos para que você possa manter tokens secretos, dados específicos do ambiente e outros tipos de informações fora de seus manifestos.  
* Builders:  
  A seção de builder contém um array com instruções que o packer irá usar para gerar as imagens de máquina.  
* Provisioners:  
  A seção de provisioners contém um array com instruções que o packer irá usar para instalar e configurar as máquinas antes de transformá-las em imagens.  

Segue abaixo o manifesto do packer:

```json
{
 "variables": {
   "ssh_username": "ubuntu",
   "profile": "AWS_PROFILE",
   "aws_region": "us-east-1",
   "vpc_id": "vpc-xxxxxx",
   "subnet_id": "subnet-xxxxxx",
   "ami_name": "ami-gitlab-runner"
 },
 "builders": [
   {
     "type": "amazon-ebs",
     "region": "{{user `aws_region`}}",
     "vpc_id": "{{user `vpc_id`}}",
     "subnet_id": "{{user `subnet_id`}}",
     "instance_type": "t2.small",
     "ssh_username": "{{user `ssh_username`}}",
     "ami_virtualization_type": "hvm",
     "ami_name": "{{user `ami_name`}}-{{isotime | clean_resource_name}}",
     "ami_description": "AMI for run gitlab-runner with docker-machine.",
     "ami_regions": [
       "{{user `aws_region`}}"
     ],
     "source_ami_filter": {
       "owners": [
         "099720109477"
       ],
       "most_recent": true,
       "filters": {
         "virtualization-type": "hvm",
         "name": "*ubuntu-bionic-18.04-amd64-server-*",
         "root-device-type": "ebs"
       }
     }
   }
 ],
 "provisioners": [
   {
     "type": "ansible",
     "host_alias": "runner",
     "user": "{{user `ssh_username`}}",
     "playbook_file": "../ansible/playbook_packer.yml",
     "ansible_env_vars": [
       "ANSIBLE_HOST_KEY_CHECKING=False",
       "AWS_PROFILE={{user `profile`}}"
     ]
   }
 ]
}
```

Na seção de builders nós especificamos qual imagem base queremo utilizar, no nosso caso, a imagem mais recente do ubuntu bionic 18.04, em qual região requeremos criar, o nome final da ami, e algumas informações para que o packer possa subir uma EC2 e executar os provisioners, como vpc_id, subnet_id entre outros.

Na seção de provisioners nós especificamos o que queremos executar e qual o tipo de provisioner, no nosso caso será o ansible, especificamos o caminho do playbook que irá executar, aquele `playbook_packer.yml` que criamos no post passado.

Para executar e criar a nossa imagem é bem simples, basta executar o seguinte comando:

```shell
❯ packer build gitlab-runner.json
amazon-ebs: output will be in this color.

==> amazon-ebs: Prevalidating any provided VPC information
==> amazon-ebs: Prevalidating AMI Name: ami-gitlab-runner-2020-06-15T22-16-30Z
...
```

### Terraform

Agora que temos o nosso playbook criado, o nosso manifesto packer criado e nossa imagem pronta na AWS, podemos provisionar a infra para que possamos rodar nossos runners. Para fazer os passos a seguir vou presumir que sua infra já estava configurada e tenha vpc’s, subnet’s.

Nossa estrutura de arquivos terraform será a seguinte:

```shell
❯ tree
.
├── bucket.tf
├── iam.tf
├── instance.tf
├── main.tf
├── output.tf
├── security_group.tf
├── subnet.tf
├── user_data
│   └── template.tpl
├── variables.tf
└── vpc.tf

1 directory, 10 files
```

Por convenção e boa práticas nós separamos os resources por arquivos, assim facilitando a leitura dos arquivos, abaixo vou descrever os principais resources que utilizaremos para subir nossa infraestrutura.

Primeiro vamos criar um user_data para executar assim que a instância executar e rodar o comando para que o runner se registre no repositório ou na conta do gitlab, esse user_data além de executar o comando para registrar o runner crie antes um template para com algumas informações para que as docker-machines possam ser executadas, são elas, vpc_id, subnet_id, security_group_id entre outros, você pode conferir [aqui](https://github.com/felipefrizzo/gitlab-runner/blob/master/terraform/user_data/template.tpl).

Em seguida nós vamos declarar o resource para subir a instância mestre, a que instância que será responsável por registrar o runner, receber os jobs e executar as docker-machines, essa declaração consiste em um data source para buscar o id da imagem que criamos utilizando o packer, e o resource que é responsável por subir a instância:

```hcl
data "aws_ami" "runner" {
 most_recent = true
 owners      = ["self"]

 filter {
   name   = "name"
   values = ["ami-gitlab-runner-*"]
 }
}

resource "aws_instance" "runner" {
 ami                    = data.aws_ami.runner.id
 instance_type          = "t3a.small"
 vpc_security_group_ids = [aws_security_group.runner.id]
 subnet_id              = element(tolist(data.aws_subnet_ids.public_subnet.ids), 1)
 key_name               = var.key_name
 monitoring             = true

 associate_public_ip_address = true

 user_data = data.template_file.runner.rendered

 iam_instance_profile = aws_iam_instance_profile.runner.name

  root_block_device {
    volume_size = "8"
    volume_type = "gp2"
  }

  tags = {
    Name        = local.project_name
    ManagedBy   = "Terraform"
    Environment = var.environment
  }

  lifecycle {
    ignore_changes        = [subnet_id]
    create_before_destroy = true
  }
}
```

Após ter todos os resources configurados, agora é só criar um arquivo com a variaveis chamado `terraform.tfvars`, não se esqueça de nunca mandar esse arquivo para o git. Com esses passos feito agora podemos executar o terraform, com os seguintes comandos:

Primeiro inicializamos o terraform, onde ele vai baixar a dependências:

```shell
❯ terraform init

Initializing the backend...

Initializing provider plugins...
- Checking for available provider plugins...
- Downloading plugin for provider "template" (hashicorp/template) 2.1.2...
- Downloading plugin for provider "aws" (hashicorp/aws) 2.66.0...

The following providers do not have any version constraints in configuration,
so the latest version was installed.

To prevent automatic upgrades to new major versions that may contain breaking
changes, it is recommended to add version = "..." constraints to the
corresponding provider blocks in configuration, with the constraint strings
suggested below.

* provider.template: version = "~> 2.1"

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

Executamos o "plano de ação" do terraform, onde ele irá nos mostrar o que ele irá criar, editar e/ou excluir.

```shell
❯ terraform plan
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

data.aws_region.default: Refreshing state...
data.aws_vpc.default: Refreshing state...
data.aws_ami.runner: Refreshing state...
data.aws_iam_policy_document.runner_role: Refreshing state...
data.aws_subnet_ids.public_subnet: Refreshing state...
data.aws_subnet_ids.private_subnet: Refreshing state...

------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create
 <= read (data resources)

Terraform will perform the following actions:

...

  # aws_instance.runner will be created
  + resource "aws_instance" "runner" {
      + ami                          = "ami-0ffca72010b6035f2"
      + arn                          = (known after apply)
      + associate_public_ip_address  = true
      + availability_zone            = (known after apply)
      + cpu_core_count               = (known after apply)
      + cpu_threads_per_core         = (known after apply)
      + get_password_data            = false
      + host_id                      = (known after apply)
      + iam_instance_profile         = "post-gitlab-runner-test-instance-profile"
      + id                           = (known after apply)
      + instance_state               = (known after apply)
      + instance_type                = "t3a.small"
      + ipv6_address_count           = (known after apply)
      + ipv6_addresses               = (known after apply)
      + key_name                     = "KEY_PAIR_NAME"
      + monitoring                   = true
      + network_interface_id         = (known after apply)
      + outpost_arn                  = (known after apply)
      + password_data                = (known after apply)
      + placement_group              = (known after apply)
      + primary_network_interface_id = (known after apply)
      + private_dns                  = (known after apply)
      + private_ip                   = (known after apply)
      + public_dns                   = (known after apply)
      + public_ip                    = (known after apply)
      + security_groups              = (known after apply)
      + source_dest_check            = true
      + subnet_id                    = "subnet-xxxxxx"
      + tags                         = {
          + "Environment" = "test"
          + "ManagedBy"   = "Terraform"
          + "Name"        = "post-gitlab-runner-test"
        }
      + tenancy                      = (known after apply)
      + user_data                    = (known after apply)
      + volume_tags                  = (known after apply)
      + vpc_security_group_ids       = (known after apply)

      + ebs_block_device {
          + delete_on_termination = (known after apply)
          + device_name           = (known after apply)
          + encrypted             = (known after apply)
          + iops                  = (known after apply)
          + kms_key_id            = (known after apply)
          + snapshot_id           = (known after apply)
          + volume_id             = (known after apply)
          + volume_size           = (known after apply)
          + volume_type           = (known after apply)
        }

      + ephemeral_block_device {
          + device_name  = (known after apply)
          + no_device    = (known after apply)
          + virtual_name = (known after apply)
        }

      + metadata_options {
          + http_endpoint               = (known after apply)
          + http_put_response_hop_limit = (known after apply)
          + http_tokens                 = (known after apply)
        }

      + network_interface {
          + delete_on_termination = (known after apply)
          + device_index          = (known after apply)
          + network_interface_id  = (known after apply)
        }

      + root_block_device {
          + delete_on_termination = true
          + device_name           = (known after apply)
          + encrypted             = (known after apply)
          + iops                  = (known after apply)
          + kms_key_id            = (known after apply)
          + volume_id             = (known after apply)
          + volume_size           = 8
          + volume_type           = "gp2"
        }
    }

...

Plan: 8 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.
```

Após analisarmos o "plano de ação" podemos aplicar ele e assim irá criar a nossa infra, e em instantes teremos nosso runner rodando.

```shell
❯ terraform apply
data.aws_region.default: Refreshing state...
data.aws_vpc.default: Refreshing state...
data.aws_iam_policy_document.runner_role: Refreshing state...
data.aws_ami.runner: Refreshing state...
data.aws_subnet_ids.public_subnet: Refreshing state...
data.aws_subnet_ids.private_subnet: Refreshing state...

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create
 <= read (data resources)

Terraform will perform the following actions:

…
Plan: 8 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value:
```
