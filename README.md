# Achados & Perdidos

Aplicação web para registrar, consultar e acompanhar itens perdidos ou encontrados, incluindo fotos e o local onde foram encontrados.

## Objetivo

Este projeto demonstra a conteinerização de uma aplicação web com Docker Compose, integrando uma aplicação Flask com um banco de dados PostgreSQL e utilizando volumes para persistir os registros e as fotografias.

## Tecnologias

- **Python e Flask:** lógica da aplicação e API HTTP.
- **Gunicorn:** servidor que executa a aplicação Flask.
- **PostgreSQL 16:** armazenamento dos registros.
- **Docker:** criação e execução dos contêineres.
- **Docker Compose:** configuração e integração dos serviços.

## Arquitetura

O projeto é composto por dois serviços principais:

- **web:** aplicação Flask executada pelo Gunicorn, acessível pela porta `8080` do computador e pela porta `5000` dentro do contêiner.
- **db:** banco PostgreSQL, acessível pela rede interna do Docker. A porta do banco não é publicada diretamente no computador host.

O projeto utiliza volumes nomeados para manter os dados do banco e as fotos mesmo após a recriação dos contêineres.


## Pré-requisitos

Antes de executar o projeto, tenha instalado:

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- Docker Compose, disponibilizado pelo Docker Desktop.
- PowerShell ou outro terminal compatível.

## Configuração do ambiente

1. Copie o arquivo de exemplo para criar sua configuração local:

   ```powershell
   Copy-Item .env.example .env
   ```

2. Abra o arquivo `.env` e configure os valores de `DB_NAME`, `DB_USER` e `DB_PASSWORD`.

   **Importante:** se você já possui um `.env` funcional, não o sobrescreva. Confira apenas se os valores correspondem aos esperados pelo projeto.

3. Não compartilhe o arquivo `.env` nem envie suas credenciais reais ao repositório.

## Como executar

Na pasta raiz do projeto, execute:

```powershell
docker compose up -d --build
```

O comando constrói a imagem da aplicação, cria a rede e os volumes necessários e inicia os serviços em segundo plano.

Para verificar o estado dos serviços:

```powershell
docker compose ps
```

O serviço `db` deve aparecer como saudável (`healthy`), e o serviço `web` deve estar em execução.

A aplicação estará disponível em:

http://localhost:8080

## Como testar a API

Os exemplos abaixo utilizam `curl.exe`, disponível no Windows.

### Verificar a saúde da aplicação

```powershell
curl.exe http://localhost:8080/health
```

A resposta esperada deve indicar que a aplicação está funcionando e que o banco está conectado.

### Consultar os itens

```powershell
curl.exe http://localhost:8080/itens
```

Para filtrar os itens pelo local:

```powershell
curl.exe "http://localhost:8080/itens?local=biblio"
```

### Consultar estatísticas

```powershell
curl.exe http://localhost:8080/estatisticas
```

### Consultar os logs da aplicação

```powershell
docker compose logs web --tail 20
```

## Persistência dos dados

O projeto utiliza dois volumes nomeados:

- `dados_banco`: armazena os dados do PostgreSQL.
- `fotos`: armazena as fotografias enviadas à aplicação.

Os volumes são independentes do ciclo de vida dos contêineres. Por isso, os registros e as fotos devem continuar disponíveis após parar e recriar os serviços.

Para testar a persistência, consulte as estatísticas, pare os serviços e inicie-os novamente:

```powershell
curl.exe http://localhost:8080/estatisticas
docker compose down
docker compose up -d
curl.exe http://localhost:8080/estatisticas
```

Compare as respostas antes e depois da recriação.

**Atenção:** o comando `docker compose down -v` remove também os volumes nomeados do projeto e, consequentemente, pode apagar os dados persistidos.

## Como parar o projeto

Para parar e remover os contêineres e a rede criada pelo Compose, preservando os volumes nomeados:

```powershell
docker compose down
```

Para remover também os volumes e os dados persistidos, use somente quando desejar realmente reiniciar o armazenamento:

```powershell
docker compose down -v
```

## Estrutura do projeto

```text
G1-2N-Igor/
├── app/
│   ├── app.py
│   ├── requirements.txt
│   ├── Dockerfile
│   └── .dockerignore
├── db/
│   └── init.sql
├── docs/
│   └── evidencias/
├── docker-compose.yml
├── .env
├── .env.example
├── .gitignore
└── README.md
```

O arquivo `.env` contém a configuração local e não deve ser enviado ao repositório. O `.env.example` serve como modelo para configurar o ambiente.


## Referência da API

A API disponibiliza as seguintes rotas HTTP:

| Método | Rota | Descrição |
|---|---|---|
| GET | `/health` | Verifica se a aplicação está funcionando e se consegue consultar o PostgreSQL. |
| GET | `/itens` | Retorna os itens cadastrados, ordenados pelos mais recentes. |
| GET | `/itens?local=biblio` | Filtra os itens pelo local, com correspondência parcial e sem diferenciar maiúsculas de minúsculas. |
| POST | `/itens` | Cadastra um item e permite enviar uma fotografia. |
| GET | `/fotos/<nome>` | Retorna uma fotografia armazenada no diretório de uploads. |
| GET | `/estatisticas` | Retorna a quantidade de registros no banco e de arquivos no diretório de fotos. |

### Cadastrar um item com fotografia

O endpoint `POST /itens` recebe dados no formato `multipart/form-data`.

Exemplo no PowerShell:

```powershell
curl.exe -x POST http://localhost:8080/itens `
  -F "descricao=Garrafa azul" `
  -F "local=Biblioteca" `
  -F "foto=@C:\caminho\para\foto.png"
```

Substitua `C:\caminho\para\foto.png` pelo caminho de uma imagem que exista no seu computador.

Os campos `descricao` e `local` são obrigatórios. O campo `foto` é opcional. Quando o cadastro é realizado, a API retorna o identificador do item e o nome da fotografia, se houver, com o código HTTP `201`.

Se os campos obrigatórios estiverem ausentes, a API retorna o código HTTP `400`.

### Visualizar uma fotografia

A rota de consulta de itens retorna o caminho da foto no formato `/fotos/nome-do-arquivo`. Para visualizar uma fotografia cadastrada, abra a URL correspondente no navegador, usando o endereço base `http://localhost:8080`.

Exemplo:

```text
http://localhost:8080/fotos/nome-do-arquivo.png
```

Substitua o nome pelo valor retornado pela API.

