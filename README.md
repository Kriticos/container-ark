# CTR Akr

Sistema de gestão de procedimentos (SOP) baseado em Laravel, com controle de acesso por perfil/setor, versionamento, aprovação, publicação e auditoria.

## Visão Geral

O projeto evoluiu da base administrativa LaraSaaS e hoje é focado em:

- cadastro e manutenção de procedimentos por setor
- fluxo de aprovação (`Revisão` -> `Aprovado` -> `Publicado`)
- histórico de versões e restauração
- auditoria de ações (quem fez, quando, em qual procedimento)
- dashboard operacional com métricas reais
- notificações dinâmicas no topo (alimentadas por auditoria)

## Funcionalidades Principais

### Procedimentos
- CRUD completo de procedimentos
- vínculo de procedimento com múltiplos setores
- versionamento automático a cada atualização
- comparação entre versões
- restauração de versões anteriores
- publicação controlada por permissão

### Fluxo de aprovação
- status internos no backend: `draft`, `in_review`, `approved`, `published`
- filtro de UI simplificado: `Revisão`, `Aprovado`, `Publicado`
- `Revisão` agrupa `draft + in_review`
- ao reprovar, o item retorna ao ciclo de revisão (com trilha registrada)

### Editor Markdown com imagens
- upload protegido de imagens no editor
- preview em tempo real (render seguro)
- suporte a tamanho por imagem no markdown:

```md
![Minha imagem](http://localhost:8080/painel/procedures/temp-images/arquivo.jpg){width=420}
```

### Dashboard
- cards de métricas em layout 3x2
- card “Sem aprovação” abre lista filtrada em revisão
- gráficos por setor/status
- ranking de procedimentos mais acessados
- movimentações recentes de auditoria

### Notificações
- dropdown de notificações no topo com dados reais de auditoria
- respeita escopo de setor para usuários não-admin
- links diretos para o procedimento afetado

### ACL / Segurança
- ACL por módulos, permissões, roles e usuários
- escopo por setor (admin, gestor, editor, leitor)
- middleware de autenticação + autorização por rota

## Stack

### Backend
- PHP 8.4
- Laravel 12
- MySQL 8

### Frontend
- Blade + Alpine.js
- Tailwind CSS
- Chart.js
- Toast UI Editor (markdown)

### DevOps / Qualidade
- Docker Compose (Sail runtime)
- Pest (testes)
- Laravel Pint
- Larastan/PHPStan

## Requisitos

- Docker + Docker Compose
- Git

## Setup Rápido (Nova VM)

Use o guia dedicado:

- [SETUP_NOVA_VM.md](docs/SETUP_NOVA_VM.md)

Resumo dos passos:

```bash
git clone <repo>
cd ctr-process
cp .env.example .env

# instalar dependências PHP (se necessário no primeiro setup)
docker run --rm \
  -u "$(id -u):$(id -g)" \
  -v "$(pwd):/var/www/html" \
  -w /var/www/html \
  laravelsail/php84-composer:latest \
  composer install --ignore-platform-reqs

# subir serviços
docker compose up -d
```

O projeto usa `docker-entrypoint.sh`, que automatiza tarefas de bootstrap no container `laravel.test`.

## Acesso

- Aplicação: `http://localhost:8080`
- Vite (dev): `http://localhost:5173`

## Banco e Testes

Este projeto usa banco de testes em container dedicado (`mysql.test`).

### Rodar testes (recomendado)

```bash
docker compose exec -T laravel.test php artisan test
```

### Rodar suíte específica

```bash
docker compose exec -T laravel.test php artisan test tests/Feature/Http/Controllers/Admin/DashboardControllerTest.php
```

Documentação complementar:

- [TESTING_DATABASE.md](docs/TESTING_DATABASE.md)

## Comandos Úteis

```bash
# Artisan
docker compose exec -T laravel.test php artisan route:list

# Migrations
docker compose exec -T laravel.test php artisan migrate --force

# Seed
docker compose exec -T laravel.test php artisan db:seed

# Pint
docker compose exec -T laravel.test vendor/bin/pint

# PHPStan
docker compose exec -T laravel.test vendor/bin/phpstan analyse
```

## Estrutura (resumo)

```text
app/
  Http/Controllers/Admin/
  Models/
database/
  migrations/
  seeders/
resources/views/
  admin/
  layouts/
routes/
  web.php
docs/
  SETUP_NOVA_VM.md
  DEVELOPER_GUIDE.md
  ACL.md
  TESTING_DATABASE.md
compose.yaml
```

## Documentação do Projeto

- [Guia do desenvolvedor](docs/DEVELOPER_GUIDE.md)
- [ACL](docs/ACL.md)
- [Qualidade](docs/QUALITY.md)
- [Scheduler](docs/SCHEDULER.md)
- [SonarQube](docs/SONARQUBE.md)

## Observações

- Evite rodar testes diretamente no host quando o alvo for banco containerizado.
- Para consistência do time, prefira comandos via `docker compose exec -T laravel.test ...`.

## Produção (Deploy)

Esta seção descreve um caminho objetivo para colocar o projeto em produção em uma VM Linux usando Docker Compose.

### 1. Pré-requisitos de infraestrutura

- VM Linux atualizada (Ubuntu 22.04+ recomendado)
- Docker Engine + Docker Compose plugin
- domínio apontando para a VM
- proxy reverso com TLS (Nginx, Traefik ou Cloudflare Tunnel)
- acesso a backup externo (S3, bucket compatível ou storage remoto)

### 2. Clonar e preparar ambiente

```bash
git clone <repo>
cd ctr-process
cp .env.example .env
```

Ajuste o `.env` para produção (mínimo):

```env
APP_ENV=production
APP_DEBUG=false
APP_URL=https://seu-dominio.com

DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=ctr_process
DB_USERNAME=sail
DB_PASSWORD=senha_forte

CACHE_STORE=database
SESSION_DRIVER=database
QUEUE_CONNECTION=database
```

Recomendações:
- use senha forte para banco
- configure SMTP real para recuperação de senha/notificações
- mantenha `APP_DEBUG=false`

### 3. Subir containers

```bash
docker compose up -d --build
```

### 4. Inicialização da aplicação

```bash
# gera chave (uma vez)
docker compose exec -T laravel.test php artisan key:generate

# migra banco em modo seguro para produção
docker compose exec -T laravel.test php artisan migrate --force

# seed inicial (roles/permissões/admin), se necessário
docker compose exec -T laravel.test php artisan db:seed --force

# cache de config/rotas/views
docker compose exec -T laravel.test php artisan config:cache
docker compose exec -T laravel.test php artisan route:cache
docker compose exec -T laravel.test php artisan view:cache
```

### 5. Build de frontend

```bash
docker compose exec -T laravel.test npm ci
docker compose exec -T laravel.test npm run build
```

### 6. Scheduler e filas

O serviço `scheduler` já está definido no `compose.yaml`.

Se usar filas assíncronas, rode também um worker dedicado (recomendado):

```bash
docker compose exec -d laravel.test php artisan queue:work --sleep=1 --tries=3 --timeout=90
```

Para produção, ideal é criar um serviço próprio de worker no `compose.yaml`.

### 7. Proxy reverso e SSL

- exponha somente o proxy (80/443) para internet
- mantenha containers internos em rede privada
- habilite TLS com Let's Encrypt
- redirecione HTTP -> HTTPS

### 8. Backup e recuperação

Mínimo recomendado:
- dump diário do MySQL
- cópia de `storage/app` (arquivos enviados)
- retenção de pelo menos 7-14 dias
- teste de restauração periódico

Exemplo de dump manual:

```bash
docker compose exec -T mysql mysqldump -u sail -p ctr_process > backup-$(date +%F).sql
```

### 9. Atualização de versão (rolling simples)

```bash
git pull
docker compose up -d --build
docker compose exec -T laravel.test php artisan migrate --force
docker compose exec -T laravel.test php artisan config:cache
docker compose exec -T laravel.test php artisan route:cache
docker compose exec -T laravel.test php artisan view:cache
```

### 10. Checklist pós-deploy

- login funcionando em `https://seu-dominio.com`
- criação/edição de procedimento OK
- upload de imagem markdown OK
- dashboard carregando sem erro
- notificações no topo exibindo auditoria
- scheduler ativo
- backup validado
