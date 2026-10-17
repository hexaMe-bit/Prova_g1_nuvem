@'
# Arquitetura do sistema

```mermaid
flowchart TB
    usuario["Computador do usuário"]
    subgraph docker["Ambiente Docker"]
        subgraph rede["Rede bridge: rede_interna"]
            web["Contêiner web<br/>Flask + Gunicorn<br/>Porta 5000"]
            db["Contêiner db<br/>PostgreSQL 16<br/>Porta 5432 interna"]
        end

        fotos[("Volume nomeado: fotos<br/>/app/uploads")]
        dados[("Volume nomeado: dados_banco<br/>/var/lib/postgresql/data")]
        init["db/init.sql<br/>Montagem somente leitura"]
    end

    usuario -->|"HTTP :8080 → :5000"| web
    web -->|"SQL / TCP 5432"| db
    web --- fotos
    db --- dados
    init -.->|"Bind mount :ro"| db