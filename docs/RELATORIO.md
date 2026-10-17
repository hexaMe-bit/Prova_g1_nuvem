# Relatório técnico — Achados & Perdidos

## 1. Objetivo

Este relatório descreve a arquitetura da aplicação Achados & Perdidos, sua conteinerização com Docker e a integração entre a aplicação Flask e o banco de dados PostgreSQL por meio do Docker Compose.

## 2. Construção da imagem da aplicação

O arquivo `app/Dockerfile` define como a imagem do serviço `web` é construída.

### 2.1. Imagem-base

```dockerfile
FROM python:3.12-slim
```

Utiliza uma imagem oficial do Python 3.12 baseada em uma distribuição Linux enxuta. Isso fornece o ambiente necessário para executar a aplicação, evitando incluir ferramentas desnecessárias de uma imagem maior.

### 2.2. Diretório de trabalho

```dockerfile
WORKDIR /app
```

Define `/app` como diretório de trabalho das instruções seguintes. Assim, os arquivos da aplicação são organizados nesse diretório dentro da imagem.

### 2.3. Instalação das dependências

```dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
```

Primeiro, copia o arquivo de dependências para a imagem e instala os pacotes Python necessários.

Essa ordem favorece o cache de camadas do Docker: se o código da aplicação mudar, mas `requirements.txt` permanecer igual, a etapa de instalação das dependências poderá ser reutilizada no próximo build.

A opção `--no-cache-dir` evita manter o cache de downloads do pip na imagem, reduzindo arquivos desnecessários.

### 2.4. Cópia do código

```dockerfile
COPY app.py .
```

Copia o arquivo principal da aplicação para o diretório `/app` dentro da imagem.

### 2.5. Criação de usuário e diretório de uploads

```dockerfile
RUN groupadd --system app \
 && useradd --system --gid app --no-create-home app \
 && mkdir -p /app/uploads \
 && chown app:app /app/uploads
```

Cria um grupo e um usuário de sistema chamados `app`, cria o diretório de uploads e atribui sua propriedade a esse usuário.

O diretório precisa permitir a gravação das fotografias recebidas pela API.

### 2.6. Execução sem privilégios de root

```dockerfile
USER app
```

Define o usuário `app` como o usuário utilizado pelas instruções de execução posteriores e, consequentemente, pelo processo principal do contêiner.

Essa configuração segue o princípio do menor privilégio: a aplicação não precisa executar como administrador do contêiner.

### 2.7. Porta da aplicação

```dockerfile
EXPOSE 5000
```

Documenta a porta 5000 como a porta utilizada pela aplicação dentro do contêiner. Essa instrução, isoladamente, não publica a porta no computador host.

### 2.8. Inicialização com Gunicorn

```dockerfile
CMD ["gunicorn", "app:app", "--bind", "0.0.0.0:5000"]
```

Define o comando padrão executado quando o contêiner inicia.

- `gunicorn`: inicia o servidor de aplicação.
- `app:app`: indica o objeto Flask chamado `app`, definido no módulo Python `app.py`.
- `--bind 0.0.0.0:5000`: faz o servidor escutar na porta 5000 em todas as interfaces de rede do contêiner.

O formato JSON utilizado em `CMD` é a forma exec, que executa o processo diretamente, sem precisar de um shell intermediário.

## 3. Configuração dos serviços no Docker Compose

O arquivo `docker-compose.yml` descreve dois serviços: `db` e `web`.

### 3.1. Serviço `db`

O serviço `db` utiliza a imagem `postgres:16-alpine`, que fornece o PostgreSQL 16 em uma imagem baseada no Alpine Linux.

As variáveis `POSTGRES_DB`, `POSTGRES_USER` e `POSTGRES_PASSWORD` recebem os valores de `DB_NAME`, `DB_USER` e `DB_PASSWORD` definidos no ambiente do projeto.

As credenciais são mantidas fora do arquivo Compose, por meio do arquivo local `.env`.

O serviço utiliza o volume nomeado `dados_banco`, montado em `/var/lib/postgresql/data`, para persistir os dados do banco.

Também monta `./db/init.sql` em `/docker-entrypoint-initdb.d/init.sql`, com a opção `:ro`, que indica montagem somente leitura. O script é disponibilizado para o mecanismo de inicialização da imagem oficial do PostgreSQL.

### 3.2. Verificação de saúde do banco

A configuração `healthcheck` executa `pg_isready` para verificar se o PostgreSQL está aceitando conexões.

Os parâmetros configurados definem:

- intervalo de cinco segundos entre verificações;
- tempo limite de cinco segundos por verificação;
- até cinco tentativas;
- período inicial de dez segundos antes de considerar as verificações.

O `depends_on` do serviço `web`, com `condition: service_healthy`, faz o Compose aguardar o banco ser considerado saudável antes de iniciar a aplicação web.

Isso não significa que o banco permanecerá sempre disponível; a aplicação ainda deve lidar com eventuais falhas posteriores.

### 3.3. Serviço `web`

O serviço `web` utiliza `build: ./app`, que indica que a imagem deve ser construída com o Dockerfile localizado no diretório `app`.

As variáveis `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER` e `DB_PASSWORD` fornecem à aplicação os dados de conexão com o banco. O valor `DB_HOST: "db"` utiliza o nome do serviço PostgreSQL como endereço na rede do Compose.

A variável `UPLOAD_DIR: /app/uploads` define o diretório usado pela aplicação para salvar fotografias.

### 3.4. Publicação da porta

```yaml
ports:
  - "8080:5000"
```

Essa configuração encaminha conexões recebidas na porta 8080 do computador host para a porta 5000 do contêiner `web`.

A aplicação pode, assim, ser acessada em `http://localhost:8080`.

O serviço `db` não possui uma configuração `ports`. Portanto, a porta 5432 do PostgreSQL não é publicada diretamente no host por esse Compose.

### 3.5. Rede interna

```yaml
networks:
  rede_interna:
    driver: bridge
```

Cria uma rede Docker do tipo bridge. Os serviços conectados a ela podem se comunicar por seus nomes de serviço.

Por isso, a aplicação utiliza `db` como endereço do PostgreSQL, em vez de precisar conhecer o endereço IP do contêiner.

### 3.6. Volumes nomeados

O Compose declara dois volumes:

- `dados_banco`: persiste os dados do PostgreSQL.
- `fotos`: persiste as fotografias em `/app/uploads`.

Os volumes têm um ciclo de vida diferente dos contêineres. Remover e recriar os contêineres com `docker compose down` e `docker compose up -d` não remove esses volumes nomeados.

O comando `docker compose down -v`, por outro lado, solicita a remoção dos volumes nomeados gerenciados pelo projeto e pode apagar os dados armazenados neles.

### 3.7. Política de reinicialização

A configuração `restart: unless-stopped` permite que o Docker reinicie os serviços após determinadas falhas ou reinicializações do daemon, exceto quando eles tiverem sido explicitamente parados e essa condição continuar válida.

## 4. Fluxo de cadastro de um item com fotografia

O endpoint `POST /itens` recebe os campos `descricao` e `local`, além do campo opcional `foto`, por meio de uma requisição `multipart/form-data`.

O fluxo ocorre na seguinte ordem:

1. O cliente envia a requisição HTTP para `http://localhost:8080/itens`.
2. A publicação de portas encaminha a requisição à porta 5000 do contêiner `web`.
3. O Flask executa a função `cadastrar()`.
4. A aplicação verifica se `descricao` e `local` foram fornecidos. Se faltarem, retorna HTTP 400.
5. Se houver uma fotografia, a aplicação gera um nome com UUID e salva o arquivo no diretório configurado em `UPLOAD_DIR`, que corresponde a `/app/uploads`.
6. Como `/app/uploads` está associado ao volume nomeado `fotos`, o arquivo é armazenado nesse volume.
7. A aplicação executa um `INSERT` no PostgreSQL com a descrição, o local e o nome da fotografia.
8. O banco guarda os dados do item e a referência ao arquivo, enquanto o conteúdo da imagem permanece no volume `fotos`.
9. A API retorna HTTP 201 com o identificador do novo registro e o nome da fotografia, quando houver.

A rota `GET /itens` retorna os registros e monta o caminho `/fotos/<nome>` para os itens que possuem imagem. A rota `GET /fotos/<nome>` utiliza `send_from_directory` para disponibilizar o arquivo.

**Observação:** o arquivo é salvo antes de o registro ser inserido no banco. Portanto, se a gravação no banco falhar depois que a fotografia tiver sido salva, o arquivo poderá permanecer no volume sem um registro correspondente.

## 5. Teste de persistência

No teste realizado, a aplicação retornou seis registros e duas fotografias.

Após executar `docker compose down` e iniciar os serviços novamente com `docker compose up -d`, a consulta ao endpoint `/estatisticas` continuou retornando:

```json
{
  "fotos_no_volume": 2,
  "itens_no_banco": 6
}
```

O resultado demonstra que os registros e as fotografias permaneceram disponíveis após a remoção e a recriação dos contêineres.

Esse comportamento é esperado porque os dados são armazenados em volumes nomeados, e não apenas na camada gravável dos contêineres.

O resultado desse teste também foi registrado em `docs/evidencias/persistencia.txt`.

## 6. Conclusão

A arquitetura separa a aplicação web do banco de dados em dois serviços. A rede Docker permite a comunicação entre eles, enquanto a publicação da porta 8080 disponibiliza a API ao usuário.

Os volumes nomeados mantêm os dados do PostgreSQL e as fotografias independentes dos contêineres. O healthcheck e a condição `service_healthy` ajudam a ordenar a inicialização dos serviços, e a execução da aplicação como usuário não privilegiado contribui para reduzir riscos de segurança.

Os testes de saúde, consulta e persistência fornecem evidências do funcionamento integrado dos componentes.
