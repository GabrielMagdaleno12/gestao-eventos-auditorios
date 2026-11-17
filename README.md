# Sistema de Gestão de Eventos e Auditórios

Implementação do projeto da disciplina (Mackenzie – Sistemas de Informação), conforme os requisitos RF01–RF11, casos de uso UC01–UC16 e regras RN01–RN10 da especificação.

## Arquitetura

```
 React (Vite, TS) ──token JWT──▶ Spring Boot (API REST, Java 21) ──JDBC──▶ PostgreSQL (Supabase)
        │                               │
        └───── login (OIDC/PKCE) ───▶ Keycloak ◀── valida JWT (issuer/JWKS)
```

| Camada | Tecnologia | Pasta |
|---|---|---|
| Front-end | React 18 + Vite + TypeScript, `keycloak-js` | `frontend/` |
| Back-end | Spring Boot 3.3, Spring Security (OAuth2 Resource Server), JDBC | `backend/` |
| Identidade | Keycloak 26 (realm `eventos`, roles `ADMIN` e `PARTICIPANTE`) | `keycloak/`, `docker-compose.yml` |
| Banco | PostgreSQL 17 no Supabase (projeto `gestao-eventos-auditorios`, região sa-east-1) | `db/` |

> Ajuste este quadro se o diagrama de arquitetura do Moodle pedir algo diferente (por exemplo, outro desenho de camadas).

### Onde cada regra mora
As regras de negócio ficam no **banco**, em funções transacionais (`db/migrations/002_funcoes.sql`), para valer mesmo com acessos simultâneos (RNF04):

| Regra | Implementação |
|---|---|
| RN01/RN02 – sem conflito de local/horário | trigger `eventos_validar` + *exclusion constraints* (`gist`) |
| RN03/RN04 – confirmar enquanto houver vaga; lotou → fila | `fn_inscrever` (trava a linha do evento com `FOR UPDATE`) |
| RN05 – ordem da fila preservada | coluna `inscricoes.ordem` (identity) + `fn_promover_fila` |
| RN06 – sem inscrição duplicada | índice único parcial `inscricoes_unica_ativa` |
| RN07/RN08 – check-in só do inscrito, uma vez | `fn_registrar_checkin` + `unique(inscricao_id)` |
| RNF07 – check-in imutável | trigger `checkins_imutavel` |
| RN09/RN10 – certificado só com presença, código único | `fn_emitir_certificado`, `certificados.codigo_verificacao` (64 hex aleatórios) |
| RNF06 – permissão no servidor | `@PreAuthorize("hasRole('ADMIN')")` + RLS ligado e sem acesso para `anon`/`authenticated` |

O back-end traduz os erros do banco (`EVENTO_LOTADO`, `CONFLITO_AGENDA`, …) em respostas HTTP (`ApiExceptionHandler`).

## Como executar

### 1. Banco (Supabase) – já criado
As migrations de `db/migrations/` **já foram aplicadas** no projeto `gestao-eventos-auditorios` (`mirmpnbpbdvwixqxyptm`) e os 3 locais iniciais já foram cadastrados. Para recriar em outro projeto, rode `001`, `002`, `003` e `seed.sql`, nessa ordem, no SQL Editor.

Defina/anote a **senha do banco** em *Project Settings → Database → Reset database password*.

### 2. Keycloak
```bash
docker compose up -d        # http://localhost:8180  (admin / admin)
```
O realm `eventos` é importado automaticamente, com dois usuários de teste:

| Usuário | Senha | Perfil |
|---|---|---|
| `organizador` | `admin123` | Administrador/Organizador |
| `aluno` | `aluno123` | Participante |

Novos participantes podem se cadastrar na própria tela de login.

### 3. Back-end
```bash
cd backend
cp .env.example .env        # preencha DB_PASSWORD (e DB_URL, se usar o pooler)
export $(grep -v '^#' .env | xargs)
mvn spring-boot:run         # http://localhost:8080
```
Requer Java 21 e Maven.

### 4. Front-end
```bash
cd frontend
cp .env.example .env
npm install
npm run dev                 # http://localhost:5173
```

## Principais endpoints (todos exigem token JWT)

| Método e rota | Perfil | RF/UC |
|---|---|---|
| `GET /api/me` | todos | RF01 / UC01 |
| `GET/POST/PUT /api/locais`, `PATCH /api/locais/{id}/ativo` | ADMIN | RF02 / UC02 |
| `GET/POST /api/locais/{id}/bloqueios`, `DELETE /api/locais/bloqueios/{id}` | ADMIN | RF02 / UC03 |
| `GET /api/locais/disponiveis?inicio&fim&capacidade` | ADMIN | RF03 / UC04 |
| `POST/PUT /api/eventos`, `POST /api/eventos/{id}/encerrar` | ADMIN | RF04 / UC05 |
| `GET /api/eventos`, `GET /api/eventos/{id}` | todos | RF05 / UC06 |
| `POST /api/eventos/{id}/inscricoes?fila=true\|false` | todos | RF06, RF07 / UC07, UC09 |
| `DELETE /api/inscricoes/{id}` | dono ou ADMIN | RF06 / UC08 (promove a fila: UC10) |
| `GET /api/inscricoes/minhas` | todos | RF08 / UC11 |
| `POST /api/eventos/{id}/checkin` | ADMIN | RF09 / UC12 |
| `POST/GET /api/inscricoes/{id}/certificado` | dono ou ADMIN | RF10 / UC13 |
| `GET /api/certificados/verificar/{codigo}` | ADMIN | RF10 / UC14 |
| `GET /api/painel`, `GET /api/eventos/{id}/inscritos`, `PATCH /api/eventos/{id}/vagas` | ADMIN | RF11 / UC15, UC16 |

## Testes feitos
O banco foi validado com 27 cenários em uma transação desfeita ao final (conflito de agenda, capacidade, bloqueios, lotação, fila FIFO, promoção ao cancelar, ajuste de vagas, check-in duplo/imutável/de outro evento, certificado sem presença, verificação de código, painel).
O front-end passa em `tsc --strict` e `vite build`.

---

## Publicar na internet (tudo pelo navegador, sem instalar nada)

Serviços: **Render** (Keycloak + API, Docker) e **Vercel** (front-end). O banco já está no Supabase. Tudo em planos gratuitos.

> Planos gratuitos "dormem" após ~15 min sem uso: o primeiro acesso pode levar 1–2 min (o Keycloak é o mais lento). Abra o link uns minutos antes de uma apresentação.

### 0. Pegar a conexão do banco (Supabase)
No projeto `gestao-eventos-auditorios`: botão **Connect** → **Session pooler**. O Render só tem IPv4, então use o *pooler* (o endereço direto `db.<ref>.supabase.co` é só IPv6). Anote:
- `DB_URL` = `jdbc:postgresql://<host-do-pooler>:5432/postgres?sslmode=require`
- `DB_USER` = `postgres.mirmpnbpbdvwixqxyptm`
- `DB_PASSWORD` = a senha definida em *Project Settings → Database → Reset database password*

### 1. Render – Keycloak e API
1. Em render.com, entre com o GitHub → **New + → Blueprint** → escolha este repositório. O Render lê o `render.yaml` e cria `eventos-keycloak` e `eventos-api`.
2. Preencha os valores pedidos:
   - Keycloak: `KC_BOOTSTRAP_ADMIN_PASSWORD` (invente uma) e `FRONTEND_URL` (coloque `https://placeholder.vercel.app` por enquanto).
   - API: `DB_URL`, `DB_USER`, `DB_PASSWORD` (passo 0), `KEYCLOAK_ISSUER` = `https://eventos-keycloak.onrender.com/realms/eventos` e `CORS_ORIGINS` = `https://placeholder.vercel.app`.
   - Se o nome `eventos-keycloak`/`eventos-api` já estiver ocupado, o Render acrescenta um sufixo: use o endereço real que ele mostrar.
3. Aguarde os dois ficarem **Live**. Teste abrir `https://<keycloak>.onrender.com/realms/eventos` (deve mostrar um JSON).

### 2. Vercel – front-end
1. Em vercel.com, entre com o GitHub → **Add New → Project** → importe o repositório.
2. **Root Directory**: `frontend`. Framework: Vite (detectado).
3. **Environment Variables**:
   - `VITE_API_URL` = `https://<eventos-api>.onrender.com`
   - `VITE_KEYCLOAK_URL` = `https://<eventos-keycloak>.onrender.com`
   - `VITE_KEYCLOAK_REALM` = `eventos`
   - `VITE_KEYCLOAK_CLIENT` = `eventos-frontend`
4. **Deploy**. Anote a URL final (ex.: `https://eventos-web.vercel.app`).

### 3. Amarrar as URLs
No Render, edite e salve (cada serviço reinicia sozinho):
- `eventos-keycloak` → `FRONTEND_URL` = URL da Vercel (sem barra no fim).
- `eventos-api` → `CORS_ORIGINS` = URL da Vercel.

Pronto: acesse a URL da Vercel e entre com `organizador` / `admin123` ou `aluno` / `aluno123`.

### Se o login der "Invalid redirect URI"
O realm é importado com a `FRONTEND_URL` informada. Se a importação não tiver aplicado o valor, abra `https://<keycloak>.onrender.com`, entre como `admin` → realm **eventos** → **Clients → eventos-frontend** e preencha *Valid redirect URIs* (`https://<vercel>/*`), *Web origins* (`https://<vercel>`) e *Valid post logout redirect URIs* (`https://<vercel>/*`).

### Limitações conhecidas
- O Keycloak usa o banco interno do container (modo demonstração): usuários cadastrados na tela de login somem quando ele reinicia. Os dois usuários de teste voltam sempre, pela importação do realm. Para persistir, o Keycloak pode usar um schema no próprio Postgres do Supabase.
- Não foi possível testar a publicação a partir do ambiente de desenvolvimento do Claude; os arquivos seguem a documentação de Render, Vercel e Keycloak 26.
