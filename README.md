# CTR Ark-nology

Sistema de gestao de procedimentos (SOP) baseado em Laravel, com controle de acesso por perfil e setor, versionamento, aprovacao, publicacao e auditoria.

## Visao Geral

O projeto evoluiu da base administrativa LaraSaaS e hoje atende principalmente:

- cadastro e manutencao de procedimentos por setor
- fluxo de aprovacao com trilha de auditoria
- versionamento e restauracao de conteudo
- dashboard operacional com metricas reais
- notificacoes dinamicas baseadas em eventos do sistema

## Funcionalidades Principais

### Procedimentos

- CRUD completo de procedimentos
- vinculacao com multiplos setores
- versionamento automatico a cada atualizacao
- comparacao entre versoes
- restauracao de versoes anteriores
- publicacao controlada por permissao

### Fluxo de aprovacao

- status internos no backend: `draft`, `in_review`, `approved`, `published`
- filtro de UI simplificado: `Revisao`, `Aprovado`, `Publicado`
- retorno ao ciclo de revisao em caso de reprovacao
- trilha completa de alteracoes e aprovacoes

### Editor Markdown com imagens

- upload protegido de imagens
- preview em tempo real
- suporte a largura customizada por imagem

```md
![Minha imagem](http://localhost:8080/painel/procedures/temp-images/arquivo.jpg){width=420}
```

### Dashboard e notificacoes

- cards de metricas em layout operacional
- graficos por setor e status
- ranking de procedimentos mais acessados
- dropdown de notificacoes com dados reais de auditoria

### ACL e seguranca

- ACL por modulos, permissoes, roles e usuarios
- escopo por setor para usuarios nao administradores
- middleware de autenticacao e autorizacao por rota

## Stack

### Backend

- PHP 8.4
- Laravel 12
- MySQL 8

### Frontend

- Blade
- Alpine.js
- Tailwind CSS
- Chart.js
- Toast UI Editor

### DevOps e qualidade

- Docker Compose com Laravel Sail
- Pest
- Laravel Pint
- Larastan / PHPStan

## Requisitos

- Docker
- Docker Compose
- Git

## Inicio Rapido

Use o guia dedicado para provisionamento inicial:

- [SETUP_NOVA_VM.md](docs/SETUP_NOVA_VM.md)

Passos basicos:

```bash
git clone <repo>
cd ctr-process
cp .env.example .env

docker run --rm \
  -u "$(id -u):$(id -g)" \
  -v "$(pwd):/var/www/html" \
  -w /var/www/html \
  laravelsail/php84-composer:latest \
  composer install --ignore-platform-reqs
```

O projeto usa `docker-entrypoint.sh`, que automatiza parte do bootstrap no container `laravel.test`.

## Modos de Ambiente

O projeto suporta dois modos principais com Docker Compose.

### 1. Desenvolvimento local com MySQL em container

Use o `compose.yaml` padrao:

```bash
docker compose up -d --build
```

Esse modo sobe:

- `laravel.test`
- `scheduler`
- `mysql`
- `mysql.test`

### 2. Aplicacao com banco externo

Use o arquivo de override [compose.external-db.yaml](compose.external-db.yaml):

```bash
docker compose -f compose.yaml -f compose.external-db.yaml up -d --build --no-deps laravel.test scheduler
```

Esse modo sobe apenas:

- `laravel.test`
- `scheduler`

Os containers `mysql` e `mysql.test` nao sao iniciados.

## Configuracao de Banco de Dados

### Banco local em container

No modo padrao, a aplicacao usa o servico `mysql` definido no `compose.yaml`. Nesse caso, o `.env` normalmente fica com:

```env
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=larasaas
DB_USERNAME=sail
DB_PASSWORD=password
```

### Banco externo

Para usar um banco externo, ajuste o `.env` com os dados reais do servidor:

```env
DB_CONNECTION=mysql
DB_HOST=seu-host-ou-ip
DB_PORT=3306
DB_DATABASE=seu_banco
DB_USERNAME=seu_usuario
DB_PASSWORD=sua_senha
```

Pontos importantes:

- `DB_CONNECTION` deve ser o driver do Laravel, por exemplo `mysql`, e nao o nome do container
- `DB_HOST` deve ser o host, IP ou DNS do banco
- `DB_HOST=mysql` so faz sentido quando a aplicacao usa o container `mysql` do Compose
- se o banco estiver na maquina host e a aplicacao rodar em container, normalmente o host sera `host.docker.internal`

Antes do primeiro `php artisan migrate`, o banco externo ja precisa estar preparado. O Laravel cria as tabelas, mas nao cria automaticamente:

- o banco de dados
- o usuario do banco
- as permissoes de acesso

Exemplo de preparacao no MySQL externo:

```sql
CREATE DATABASE arknowledge
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE USER 'ctr_user'@'%' IDENTIFIED BY 'sua_senha_forte';

GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, ALTER, INDEX, DROP
ON arknowledge.* TO 'ctr_user'@'%';

FLUSH PRIVILEGES;
```

Depois disso, ajuste o `.env` para usar os mesmos valores:

```env
DB_CONNECTION=mysql
DB_HOST=seu-host-ou-ip
DB_PORT=3306
DB_DATABASE=arknowledge
DB_USERNAME=ctr_user
DB_PASSWORD=sua_senha_forte
```

Depois de ajustar o `.env`, limpe o cache de configuracao:

```bash
docker compose exec -T laravel.test php artisan config:clear
```

Se estiver em ambiente novo, crie a estrutura das tabelas com:

```bash
docker compose exec -T laravel.test php artisan migrate
```

Em banco novo, executar apenas `migrate` nao basta para liberar o acesso administrativo. Depois das migrations, rode tambem:

```bash
docker compose exec -T laravel.test php artisan db:seed
```

Esse passo cria ou atualiza:

- usuario administrador padrao
- roles e permissoes
- modulos
- menus

As credenciais iniciais do admin sao lidas do `.env`:

```env
ADMIN_NAME="Administrador"
ADMIN_EMAIL="admin@larasaas.com"
ADMIN_PASSWORD="P@ssw0rd"
```

Se preferir outro usuario inicial, altere esses valores antes de rodar o `db:seed`.

## Acesso Local

- Aplicacao: `http://localhost:8080`
- Vite: `http://localhost:5173`

## Testes

O projeto usa um banco de testes dedicado em container (`mysql.test`) no fluxo padrao.

### Rodar testes

```bash
docker compose exec -T laravel.test php artisan test
```

### Rodar uma suite especifica

```bash
docker compose exec -T laravel.test php artisan test tests/Feature/Http/Controllers/Admin/DashboardControllerTest.php
```

### Banco externo e testes

O arquivo `phpunit.xml` esta configurado para usar `mysql.test`. Se voce quiser rodar testes com banco externo, ajuste nele:

- `DB_CONNECTION`
- `DB_HOST`
- `DB_PORT`
- `DB_DATABASE`

Se isso nao for alterado, o ambiente de testes continuara esperando o container `mysql.test`.

Documentacao complementar:

- [TESTING_DATABASE.md](docs/TESTING_DATABASE.md)

## Comandos Uteis

```bash
docker compose exec -T laravel.test php artisan route:list
docker compose exec -T laravel.test php artisan migrate --force
docker compose exec -T laravel.test php artisan db:seed
docker compose exec -T laravel.test vendor/bin/pint
docker compose exec -T laravel.test vendor/bin/phpstan analyse
```

## Estrutura do Projeto

```text
app/
database/
resources/
routes/
docs/
compose.yaml
compose.external-db.yaml
```

## Documentacao

- [Guia do desenvolvedor](docs/DEVELOPER_GUIDE.md)
- [ACL](docs/ACL.md)
- [Qualidade](docs/QUALITY.md)
- [Scheduler](docs/SCHEDULER.md)
- [SonarQube](docs/SONARQUBE.md)

## Observacoes

- Evite rodar testes diretamente no host quando o alvo for banco containerizado.
- Para consistencia do time, prefira comandos via `docker compose exec -T laravel.test ...`.

## Producao

Esta secao resume um caminho objetivo para deploy em VM Linux usando Docker Compose.

### 1. Preparacao

```bash
git clone <repo>
cd ctr-process
cp .env.example .env
```

Minimo recomendado no `.env`:

```env
APP_ENV=production
APP_DEBUG=false
APP_URL=https://seu-dominio.com

DB_CONNECTION=mysql
DB_HOST=seu-host-ou-ip
DB_PORT=3306
DB_DATABASE=ctr_process
DB_USERNAME=seu_usuario
DB_PASSWORD=senha_forte

CACHE_STORE=database
SESSION_DRIVER=database
QUEUE_CONNECTION=database
```

### 2. Subida dos containers

Com banco externo:

```bash
docker compose -f compose.yaml -f compose.external-db.yaml up -d --build --no-deps laravel.test scheduler
```

Com banco local do Compose:

```bash
docker compose up -d --build
```

### 3. Inicializacao da aplicacao

```bash
docker compose exec -T laravel.test php artisan key:generate
docker compose exec -T laravel.test php artisan migrate --force
docker compose exec -T laravel.test php artisan db:seed --force
docker compose exec -T laravel.test php artisan config:cache
docker compose exec -T laravel.test php artisan route:cache
docker compose exec -T laravel.test php artisan view:cache
```

### 4. Build de frontend

```bash
docker compose exec -T laravel.test npm ci
docker compose exec -T laravel.test npm run build
```

### 5. Scheduler e filas

O servico `scheduler` ja existe no `compose.yaml`.

Se usar filas assincronas, rode tambem um worker dedicado:

```bash
docker compose exec -d laravel.test php artisan queue:work --sleep=1 --tries=3 --timeout=90
```

### 6. Operacao

Boas praticas recomendadas:

- exponha apenas o proxy reverso na internet
- mantenha TLS habilitado
- faca backup de banco e `storage/app`
- teste restauracao periodicamente

Exemplo de dump manual quando o MySQL estiver em container local:

```bash
docker compose exec -T mysql mysqldump -u sail -p ctr_process > backup-$(date +%F).sql
```

### 7. Atualizacao

```bash
git pull
docker compose up -d --build
docker compose exec -T laravel.test php artisan migrate --force
docker compose exec -T laravel.test php artisan config:cache
docker compose exec -T laravel.test php artisan route:cache
docker compose exec -T laravel.test php artisan view:cache
```

### 8. Checklist pos-deploy

- login funcionando
- criacao e edicao de procedimento OK
- upload de imagem markdown OK
- dashboard carregando sem erro
- notificacoes exibindo auditoria
- scheduler ativo
- backup validado
