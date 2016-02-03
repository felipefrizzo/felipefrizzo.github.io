+++
categories = ["DevOps", "Deploy", "Heroku", "Python", "Django"]
date = "2016-02-03T21:37:12-02:00"
image = "/img/home-bg.jpg"
tags = ["DevOps", "deploy", "heroku", "python", "django", "development"]
title = "Wercker - Build e Deploy de maneira facil no Heroku"
+++

O que é o [Wercker](http://wercker.com)?

Wercker é uma plataforma para facilitar as tarefas de build e deployment. A versão atual utiliza [Docker](http://docker.com) para executar seus builds, assim você pode usar o Wercker para fazer o build e deployment de qualquer linguagem basta utilizar uma imagem Docker e configurar os passos a serem executados pelo. Você pode customizar scripts para executar os passos tanto no build quanto no deployment assim evitando retrabalho quando precisar fazer o processo para outra aplicação que tenha a mesma estrutura.


Agora vamos a um exemplo, que vou utilizar as configurações que fiz para o build e deployment do projeto [Eventex](http://eventex-felipefrizzo.herokuapp.com/) do curso [Welcome to the Django](http://welcometothedjango.com.br/), que está no Heroku.

```
box: python:3.5
services:
    - postgres

build:
  steps:
    - pip-install:
        requeriments_file: "requeriments.txt"
    - script:
        name: Run Test
        code: |
            python manage.py test

deploy:
    steps:
        - heroku-deploy:
            instalar-toolbelt: true
            key: $HEROKU_KEY
            user: $HEROKU_USER
            app-name: $HEROKU_APP_NAME

```

Como o Wekcker utiliza Docker podemos utilizar qualquer imagem Docker, neste caso estou utilizando **python:3.5**.

**services** Serviços são boxes prontos disponibilizados pelo Wercker, como PostgreSql, MongoDB, etc. Em outras palavras, é um container configurado com um serviço ready to use.

Os passos para executar o build e criar os arquivos para deployment estão descritos em **build**.

Os Passos:

* **pip-install** instala todos os requeriments nessecessarios para rodar a aplicação **Python**.
* **script** a etapa de script permite que você pode executar um comando Shell.
* **Run Test** roda os testes da aplicação, caso de erro nos testes não passa para o depoy.

O deployment

* **heroku-deploy** Aqui preciso configurar **toolbelt**, o **HEROKU_KEY**, o **HEROKU_USER**, o **HEROKU_APP_NAME**.
  * **instalar-toolbelt** instala o toolbelt para poder utilizar o heroku.
  * **HEROKU_KEY** e **HEROKU_USER** e **HEROKU_APP_NAME** é uma variavel de ambiente que configuramos no settings do projeto. Mas esta mais especifamente esta configurada para estar disponível somento no momento em que os passos de deployment estão sendo executados.

Agora que temos o nosso projeto configurado com um arquivo **wercker.yml** vamos fazer as configurações no ambiente do Wercker.
Vamos criar uma nova aplicação
![alt text](/imgs/post/wercker/wercker.png)

No primeiro passo vamos selecionar se o nosso repositório esta no GitHub ou no BitBucket.
![alt text](/imgs/post/wercker/Wercker-1.png)

O segundo passo qual o repositório que vamos utilizar.
![alt text](/imgs/post/wercker/Wercker-2.png)

Quarto passo selecionamos a forma como o wercker vai baixar o repositório.
![alt text](/imgs/post/wercker/Wercker-3.png)

No quinto passo o wercker vai tentar identificar qual linguagem esta sendo utilizada no repositório e vai criar um arquivo **wercker.yml** inicial.
![alt text](/imgs/post/wercker/Wercker-4.png)

O sexto e ultimo passo é para confirmar a criação da aplicação e decidir se você quer tornar esse repositório publico.
![alt text](/imgs/post/wercker/Wercker-5.png)

No passo a seguir vamos configurar onde faremos o deployment.
![alt text](/imgs/post/wercker/Wercker-deploy.png)

Vamos adicionar um novo deploy target do tipo **Custom Deploy**.
* Em **Deploy Target Name** vamos colocar Heroku.
* Vamos selecionar o **Auto Deploy** assim toda vez que um novo commit for feito além do build também será executado os passos para deploy se o build ocorrer com sucesso. E também devemos indicar qual o branch será usado.

Vamos precisar adicionar algumas variáveis para ser utilizado pelo deploy. Faremos isso com **HEROKU_KEY** e **HEROKU_USER** e **HEROKU_APP_NAME**, todas marcadas como *Protected*.
![alt text](/imgs/post/wercker/Wercker-deploy-1.png)


E caso sua aplicação tenha variaveis de ambiente com está, basta adiciona-las na aba **Variaveis De Ambiente**

![alt text](/imgs/post/wercker/Wercker-Environment-variable.png)

E agora podemos alterar nosso código fazer pull para o repositório e deixar o Wercker fazer o resto do trabalho.

O Wercker também possui on cliente(OsX e Linux) que permitir baixarmos um container gerado por um build do wercker para rodar localmente com Docker. Isto é util quando precisamos descobrir porque nosso build não esta funcionando como deveria.

Para instalar o wercker no OsX.
```
curl https://s3.amazonaws.com/downloads.wercker.com/cli/stable/linux_amd64/wercker -o /usr/local/bin/wercker
```

No Linux
```
curl https://s3.amazonaws.com/downloads.wercker.com/cli/stable/linux_amd64/wercker -o /usr/local/bin/wercker
```

E por fim precisamos executar a seguinte linha tanto para OsX quanto para Linux.
```
chmod +x /usr/local/bin/wercker
```

Documentação http://devcenter.wercker.com/docs/index.html
Documentação da API http://devcenter.wercker.com/api/index.html
