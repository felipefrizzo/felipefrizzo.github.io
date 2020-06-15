+++
Tags = ["terraform", "ansible", "packer", "ci/cd", "self-hosted"]
Categories = ["Iac", "Terraform", "Ansible", "Packer", "CI/CD"]
date = "2020-06-15T15:15:32-03:00"
title = "Gitlab Runner de alta disponibilidade rodando na sua infraestrutura - Parte 1"
author = "Felipe Frizzo"
Description = ""
+++

Neste artigo irei abordar como rodar os runners do gitlab na sua infraestrutura tendo alta disponibilidade e tudo isso com código usando ansible, packer e terraform. Este artigo será dividido em duas partes onde nessa primeira parte irei contextualizar as ferramentas citadas acima e num próximo post onde iremos colocar a mão na massa.

### Gitlab Runner

O [gitlab-runner](https://docs.gitlab.com/runner/) é um projeto open-source escrito em golang que é usado para rodar os jobs (pipelines) e enviar o resultado para o gitlab. Foi projetado para rodar na maioria dos sistemas operacionais, são eles, GNU/Linux, macOS e Windows.

### Ansible

O [ansible](https://www.ansible.com/) é um projeto open-source escrito em python que é usado para provisionamento, controle de configurações, mas no nosso caso, iremos usar apenas para controle de configurações e instalação de pacotes. Ele também foi projetado para rodar na maioria dos sistemas operacionais.

### Packer

O [packer](https://www.packer.io/) é um projeto open-source escrito em golang que é usado para a automação da criação de imagens, como por exemplo, AMI (Amazon machine images). Ele utiliza gerenciamento de configurações para que você possa usar scripts automatizados para realizar a instalação e configuração das dependências, no nosso caso iremos utilizar a ferramenta citada a cima o ansible. Nesse artigo iremos utilizar json para escrever o manifesto do packer, mas nas novas versões já está disponível o uso da linguagem HCL.

### Terraform

O [terraform](http://terraform.io/) é um projeto open-source escrito em golang que é usado para fazer o provisionamento e versionamento da infraestrutura usando uma linguagem de alto nível chamada HCL (hashicorp configuration language).

### Mãos a obra

Vamos começar com a estrutura de pastas do projeto, vocês podem acompanhar pelo [repositório](https://github.com/felipefrizzo/gitlab-runner), onde centralizei os código para facilitar o a visualização:

```shell
❯ tree
.
├── README.md
├── ansible
├── packer
└── terraform

3 directories, 1 file
```

Na pasta do ansible iremos criar um arquivo chamado `playbook_packer.yml`, onde irá conter algumas das instruções para execução inicial, e é esse arquivo que o packer irá chamar quando ele executar, mas deixamos para falar disso no próximo passo. Primeiramente iremos validar se o python está instalado na distribuição em questão, no nosso caso iremos usar ubuntu, após validar iremos executar os comandos para atualizar os pacotes e após executaremos as roles.

A primeira role será responsável pela instalação do docker e da docker-machine, por que os runners serão executados através deles. A segunda role será responsável pela instalação do runner e suas dependências.

```yml
---
- hosts: runner
  gather_facts: yes
  become: yes

  pre_tasks:
    - name: Dist Upgrade
      include: tasks/dist_upgrade.yml

  roles:
    - role: docker-machine
    - role: gitlab-runner
```

Agora iremos criar a pasta roles, ela será responsável por "armazenar" os playbooks que iremos criar e utilizar. Para isso é só executar o seguinte comando: `mkdir -p roles`, dentro desta pasta iremos criar os nossos dois playbooks citados acima, para isso basta executar:

```shell
❯ ansible-galaxy init docker-machine
- Role docker-machine was created successfully

❯ ansible-galaxy init gitlab-runner
- Role gitlab-runner was created successfully
```

Agora dentro da pasta ansible nós teremos a seguinte estrutura:

```shell
❯ tree
.
├── playbook_packer.yml
├── roles
│   ├── docker-machine
│   │   ├── README.md
│   └── gitlab-runner
│       ├── README.md
└── tasks
    └── dist_upgrade.yml

20 directories, 18 files
```

Dentro de um [playbook](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html#playbooks-intro) ansible nós teremos a seguinte estrutura:

```shell
├── roles
│   ├── docker-machine
│   │   ├── README.md
│   │   ├── defaults
│   │   │   └── main.yml
│   │   ├── files
│   │   ├── handlers
│   │   │   └── main.yml
│   │   ├── meta
│   │   │   └── main.yml
│   │   ├── tasks
│   │   │   └── main.yml
│   │   ├── templates
│   │   ├── tests
│   │   │   ├── inventory
│   │   │   └── test.yml
│   │   └── vars
│   │       └── main.yml
│   └── gitlab-runner
```

* Defaults:  
  É um arquivo de configuração que você pode usar para definir valores padrão para variáveis usadas em sua função.  
* Files:  
  Esta pasta contém todos os arquivos extras necessários para realizar a tarefa de função.  
* Handlers:  
  É usado principalmente para o gerenciamento de serviços para aplicar uma alteração na configuração executada por outra tarefa.  
* Tasks:  
  O código desses arquivos executa as principais tarefas da função chamando todos os outros elementos definidos da função.  
* Templates:  
  Esta pasta contém os arquivos de modelo usados pela função para criar os arquivos de configuração reais.  
* Tests:  
  Esta pasta contém arquivos para execução de testes.  
* Vars:  
  Esta pasta contém as variáveis que seu projeto terá.  

Porém para esse nosso projeto iremos precisa apenas das seguintes pastas: `tasks, handlers`.

Para não ficar muito extenso eu vou comentar sobre a instalação do runner, sobre a do docker e docker-machine você irá encontrar várias explicações pela internet. Lembrando você pode acompanhar o código pelo [repositório](https://github.com/felipefrizzo/gitlab-runner) la terá todo o código.

Como estamos utilizando linux, no caso ubuntu podemos instalar pelo repositório oficial do gitlab para isso basta executar os seguintes comandos, para maiores informações de como instalar você pode acessar o site do [gitlab](https://docs.gitlab.com/runner/install/):

```shell
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
apt install -y gitlab-runner
```

O playbook ansible ficaria assim:

```yml
# roles/gitlab-runner/tasks/main.yml
---
- name: Add gitlab official repository
 shell: |
   set -o pipefail
   curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | /bin/bash
 changed_when: false

- name: Install gitlab runner
 apt:
   name: gitlab-runner
   update_cache: yes
 notify: enable gitlab-runner

# roles/gitlab-runner/handlers/main.yml
---
- name: enable gitlab-runner
 systemd:
   name: gitlab-runner.service
   state: started
   enabled: true
   daemon-reload: yes
```

Primeiramente adicionamos o pacote oficial do gitlab runner e após o instalamos com o apt, após o gitlab runner ser instalado, irá notificar um dos handlers para o habilitar e para que inicie automaticamente na inicialização. E pronto nosso playbook está pronto agora falaremos sobre o packer, que será responsável por criar uma imagem para nós.

Mas isso é assunto para um próximo post. [Parte 2](http://felipefrizzo.github.io/post/self-hosted-gitlab-runner-with-iac-part-2/)
