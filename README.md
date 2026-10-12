# postgres-dev

PostgreSQL + pgAdmin via Docker Compose para projetos que não possuem configuração própria de banco. Uma instância compartilhada, com um banco separado para cada projeto.

**Somente para desenvolvimento local.** As portas ficam disponíveis apenas em `127.0.0.1`. O usuário inicial do PostgreSQL é superusuário; esta configuração não é adequada para produção.

## Requisitos

- Docker com Docker Compose V2 (`docker compose`).

## Iniciar

```bash
cp .env.example .env
```

Edite o `.env` para definir suas senhas e, se necessário, alterar as portas. Não versione esse arquivo.

```bash
docker compose up -d
```

As imagens utilizadas são `postgres:17` e `dpage/pgadmin4:9.18`, sem depender da tag `latest`. Os dados do PostgreSQL e as configurações do pgAdmin ficam em volumes persistentes.

## Configurar o pgAdmin

1. Acesse <http://localhost:5050> (ou a porta configurada em `PGADMIN_PORT`).
2. Entre com `PGADMIN_DEFAULT_EMAIL` e `PGADMIN_DEFAULT_PASSWORD` do `.env`.
3. Em **Register > Server**, escolha um nome, como `PostgreSQL local`.
4. Na aba **Connection**, preencha:

| Campo | Valor |
| --- | --- |
| Host name/address | `postgres` |
| Port | `5432` |
| Maintenance database | Valor de `POSTGRES_DB` |
| Username | Valor de `POSTGRES_USER` |
| Password | Valor de `POSTGRES_PASSWORD` |

O pgAdmin usa o nome do serviço `postgres`, não `localhost`, pois roda em outro container. A porta interna continua sendo `5432`, mesmo se você alterar `POSTGRES_PORT`.

## Um banco por projeto

No pgAdmin, clique com o botão direito em **Databases > Create > Database** e crie, por exemplo, `meu_projeto`.

Ou crie pelo terminal (substitua `meu_projeto` pelo nome desejado):

```bash
docker compose exec postgres sh -c 'createdb -U "$POSTGRES_USER" meu_projeto'
```

Cada aplicação usa seu próprio banco, mas compartilha o servidor e, por simplicidade, o usuário. Isso separa os dados, não as permissões de acesso entre projetos.

Para aplicações executadas diretamente na sua máquina:

| Parâmetro | Valor |
| --- | --- |
| Host | `localhost` |
| Porta | Valor de `POSTGRES_PORT` (padrão: `5432`) |
| Banco | Nome do banco do projeto |
| Usuário | Valor de `POSTGRES_USER` |
| Senha | Valor de `POSTGRES_PASSWORD` |

Exemplo de URI (substitua as credenciais; caracteres especiais precisam de URL encoding):

```text
postgresql://dev:SUA_SENHA@localhost:5432/meu_projeto
```

Aplicações em outros containers precisam de configuração adicional de rede: `localhost` dentro de um container aponta para ele próprio. Esta configuração prioriza aplicações executadas na máquina.

## Comandos úteis

```bash
# Validar a configuração sem imprimir as credenciais
docker compose config --quiet

# Ver o estado dos serviços
docker compose ps

# Acompanhar os logs
docker compose logs -f

# Parar e remover os containers, preservando os dados
docker compose down

# Iniciar novamente
docker compose up -d
```

Os serviços usam `restart: unless-stopped`: podem iniciar novamente com o Docker, a menos que tenham sido parados manualmente.

### Apagar todos os dados

**Atenção: o comando abaixo remove os volumes, todos os bancos e as configurações do pgAdmin. Não há recuperação sem backup.**

```bash
docker compose down -v
```

## Observações

- Se a porta `5432` ou `5050` estiver ocupada, altere a respectiva variável no `.env` e execute `docker compose up -d` novamente.
- `POSTGRES_DB`, `POSTGRES_USER` e `POSTGRES_PASSWORD` inicializam apenas um volume vazio. Alterar o `.env` não cria outro banco nem atualiza usuários ou senhas existentes; faça essas alterações no PostgreSQL.
- As credenciais iniciais do pgAdmin também não redefinem automaticamente uma conta já persistida no volume.
- Não altere a versão principal do PostgreSQL sobre um volume existente sem planejar uma migração, como backup/restauração. A imagem 17 usa o volume em `/var/lib/postgresql/data`; a partir da versão 18, o layout é diferente.
- Este repositório não precisa ser copiado para cada projeto: mantenha a infraestrutura em execução e configure a conexão em cada aplicação.
