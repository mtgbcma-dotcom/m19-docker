# Bookstore - Django REST Framework no Docker

Atividade do Módulo 19 de Continuação Back-End Python da EBAC.

Criei um Dockerfile para executar a API Bookstore em um contêiner. A aplicação mantém o cadastro de categorias, produtos e pedidos, a paginação de 5 registros e a autenticação por token somente nos pedidos.


## Continuidade do projeto e do ambiente

Dei continuidade à entrega do [módulo 17 — Autenticação](https://github.com/mtgbcma-dotcom/m17-token/pull/1). Mantive os modelos, os serializers, a migração inicial e os testes de serializers do Bookstore, além das funcionalidades acrescentadas nas etapas anteriores.

Nos módulos 15 e 16, utilizei Poetry com `pyproject.toml` e `poetry.lock`. A partir do módulo 17, passei a declarar as dependências em `requirements.txt` e a instalar com pip. Nas etapas com Docker, o próprio Dockerfile executa essa instalação durante a construção da imagem. Documentei essa mudança de ferramenta para deixar explícita a continuidade do mesmo projeto.

Para reproduzir esta entrega, utilizei os comandos e os arquivos de dependências desta versão, descritos abaixo. A instalação com pip segue as faixas registradas em `requirements.txt`; o lock do Poetry das etapas anteriores continua disponível nos respectivos repositórios.

## Implementação

- Imagem base `python:3.13-slim`.
- Instalação das dependências com `requirements.txt` antes da cópia do código, aproveitando o cache de build.
- Variáveis `PYTHONDONTWRITEBYTECODE` e `PYTHONUNBUFFERED` configuradas.
- `.dockerignore` para excluir Git, ambiente virtual, caches, banco local e arquivos de ambiente da imagem.
- Execução por um usuário sem privilégios de administrador dentro do contêiner.
- Migrações aplicadas ao iniciar a aplicação, seguidas do servidor em `0.0.0.0:8000`.
- Banco SQLite em `/data/db.sqlite3`, com volume nomeado para manter os dados ao recriar o contêiner.

O servidor utilizado é o servidor de desenvolvimento do Django, para a execução local da atividade.

## Executar com Docker

Com o Docker Desktop aberto, use o PowerShell na pasta que contém `Dockerfile` e `manage.py`. Os comandos de build e execução também funcionam em um terminal Linux com Docker Engine.

### 1. Construir a imagem

```powershell
docker build -t m19-bookstore:latest .
```

O ponto no final indica a pasta atual como contexto do build.

### 2. Iniciar a aplicação

```powershell
docker run -d --name m19-api -p 127.0.0.1:8000:8000 --mount type=volume,source=m19-dados,target=/data m19-bookstore:latest
```

O Docker cria o volume `m19-dados` no primeiro uso. As migrações são executadas automaticamente antes de iniciar o servidor.

Confira a inicialização:

```powershell
docker logs m19-api
```

Se a porta 8000 estiver ocupada por outro projeto, pare esse servidor ou use `-p 127.0.0.1:8001:8000` e acesse a aplicação pela porta 8001.

### 3. Criar o administrador no banco do contêiner

```powershell
docker exec -it m19-api python manage.py createsuperuser
```

Escolha seu usuário, e-mail e senha. A senha não aparece enquanto é digitada. Os usuários criados em outros projetos não são copiados para este banco.

### 4. Criar dados para conferir a paginação

```powershell
docker exec m19-api python manage.py seed_demo
```

O comando cria 12 categorias e 12 produtos. Pode ser executado novamente sem duplicar os mesmos dados. Não cria usuários nem pedidos.

### 5. Abrir a API

- Raiz da API: http://127.0.0.1:8000/api/
- Administração: http://127.0.0.1:8000/admin/
- Categorias: http://127.0.0.1:8000/api/categories/
- Produtos: http://127.0.0.1:8000/api/products/
- Pedidos: http://127.0.0.1:8000/api/orders/

Para categorias e produtos, use **Log in** com o administrador criado no contêiner. A listagem possui `count`, `next`, `previous` e `results`.

Pedidos exigem token: abrir `/api/orders/` diretamente no navegador sem esse cabeçalho retorna HTTP 401, mesmo com uma sessão de login.

### 6. Conferir pedidos autenticados no PowerShell

```powershell
$credencialMD19 = Get-Credential -Message "Informe o administrador criado no conteiner MD19"
$corpoMD19 = @{
    username = $credencialMD19.UserName
    password = $credencialMD19.GetNetworkCredential().Password
} | ConvertTo-Json

$autenticacaoMD19 = Invoke-RestMethod -Uri "http://127.0.0.1:8000/api/token/" -Method Post -ContentType "application/json" -Body $corpoMD19 -ErrorAction Stop
$cabecalhosMD19 = @{ Authorization = "Token $($autenticacaoMD19.token)" }
Remove-Variable credencialMD19, corpoMD19

(Invoke-WebRequest -Uri "http://127.0.0.1:8000/api/orders/" -Headers $cabecalhosMD19 -UseBasicParsing).StatusCode

Invoke-RestMethod -Uri "http://127.0.0.1:8000/api/orders/" -Headers $cabecalhosMD19 | ConvertTo-Json -Depth 8
```

Resultado esperado: HTTP 200 e a resposta paginada. Sem pedidos cadastrados, `count` será 0 e `results` será uma lista vazia. O token fica nas variáveis do terminal; não deve ser incluído em prints ou commits.

## Verificações dentro do contêiner

```powershell
docker exec m19-api python manage.py check
docker exec m19-api python manage.py makemigrations --check --dry-run
docker exec m19-api python manage.py test --verbosity 2
```

A suíte contém 86 testes de serializers, operações da API, permissões, paginação e autenticação por token. O Django usa um banco temporário para os testes e preserva os dados cadastrados para a demonstração.

## Parar e iniciar novamente

```powershell
docker stop m19-api
docker start m19-api
```

`docker start` reinicia o mesmo contêiner, com o mesmo banco e as credenciais já cadastradas.

## Recriar após alterar o código

```powershell
docker build -t m19-bookstore:latest .
docker stop m19-api
docker rm m19-api
docker run -d --name m19-api -p 127.0.0.1:8000:8000 --mount type=volume,source=m19-dados,target=/data m19-bookstore:latest
```

O código está na imagem. Após modificá-lo, é necessário reconstruir a imagem e recriar o contêiner. O volume `m19-dados` mantém usuários, tokens, categorias, produtos e pedidos entre essas execuções. Excluir o volume também exclui esses dados.

## Tecnologias

Python 3.13, Django 5.2, Django REST Framework, SQLite e Docker.

## Referências

- [Dockerfile - documentação oficial](https://docs.docker.com/reference/dockerfile/)
- [Volumes - documentação oficial](https://docs.docker.com/engine/storage/volumes/)
- [Dockerfile de referência da atividade](https://gist.github.com/drsantos20/bacf3eaede5be27c82c788c7261bd954)

## Autor

Bruno Costa Macedo de Almeida
