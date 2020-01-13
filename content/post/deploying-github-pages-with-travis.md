+++
Tags = ["devops", "travis", "ci", "cd", "deploy", "delivery"]
Categories = ["DevOps", "Travis", "Deploy"]
date = "2020-01-12T20:20:00-03:00"
title = "TravisCI para fazer o deploy do seu blog ao GitHub Pages"
author = "Felipe Frizzo"
Description = ""
+++

Travis CI é uma serviço de integração contínua usada para build, testes e deploy de software que estão no GitHub, para projetos open sources pode ser utilizado sem quaisquer cobranças.

Ao acessar a plataforma do Travis você terá a opção de ativar determinado repositório e quando o travis estiver ativado o GitHub passará a notificar quando houver novos commits.

![alt text](/img/post/deploying-github-pages-with-travis/enable-travis.png)

Quando você tiver ativado o repositório no travis, você precisará ter um arquivo chamado `.travis.yml`, esse arquivo contém instruções que informam ao travis o que fazer, como fazer e quando fazer. Nesse passo é onde entra a mágica para realizar o deploy para o GitHub Pages.

### Adicionando o `.travic.yml` ao seu projeto

Como exemplo utilizaremos um blog feito com [Hugo]('https://gohugo.io')

#### Vamos detalhar o código

```yml
# .travis.yml
language: python

env:
  global:
    - PRODUCTION=true

before_install:
  - sudo apt-get update -qq
  - sudo apt-get -yq install apt-transport-https curl

install:
  - curl -LO https://github.com/gohugoio/hugo/releases/download/v0.57.2/hugo_0.57.2_Linux-64bit.deb
  - sudo dpkg -i hugo_0.57.2_Linux-64bit.deb
  - hugo version
  - rm -rf public 2> /dev/null

# Build the website
script:
  - hugo

deploy:
  provider: pages
  github_token: $GITHUB_TOKEN
  skip_cleanup: true
  verbose: true
  keep_history: false
  local_dir: public
  target_branch: gh-pages
  on:
    branch: master
```

* `deploy`: Este campo é onde vamos declarar as instruções de como o travis irá executar o deploy.
* `provider`: Aqui dizemos ao travis para utilizar o provider do github.
* `github_token`: Aqui passamos o token de acesso ao github para que o travis possa realizar o deploy. [Como gerar o Token]('https://help.github.com/pt/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line').
* `skip_cleanup`: Mantenha esse campo como true para que o travis não delete os arquivos que foi criado durante o build, que provavelmente será os arquivos que você está tentando fazer o upload.
* `verbose`: Opcional, detalhar os passos que está sendo realizado, por padrão é false.
* `keep_history`: Opcional, cria commits incrementais ao invés de fazer um push force, por padrão é false
* `local_dir`: Diretório a ser enviado para o GitHub pages, por padrão é o diretório atual. Pode ser especificado path absoluto ou relativo.
* `target_branch`: Branch onde será enviado o conteudo do local_dir, por padrão é gh-pages
* `on/branch`: Branch que será a trigger para realizar o deploy.

Documentação:  
[https://docs.travis-ci.com/user/deployment/pages/]('https://docs.travis-ci.com/user/deployment/pages/')
