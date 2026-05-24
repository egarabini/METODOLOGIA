...# Diagnóstico de Segurança: Migração para Keycloak — CarePlanner

**Data:** 2026-05-23  
**Autor:** Claude Code (com base nos documentos da pasta Drive e no código dos repositórios)  
**Escopo:** Repositório `careplanner/` · Feature F1.0

## 1. Estado Atual da Segurança (Baseline)

### 1.1 Mecanismo de Autenticação

O CarePlanner usa autenticação monolítica nativa do Django (`django.contrib.auth`):

- Login via `username + password` tratado em `paineladmin/views.py:login_page_view()`, que chama `authenticate()` e `login()` do Django.
- Sessão gerenciada por cookie de sessão (`SESSION_COOKIE_SECURE=True` em produção).
- Não há JWT, OIDC nem qualquer token externo no fluxo atual.
- A tabela `auth_user` do Django é a única fonte de identidade.

### 1.2 Modelo de Autorização

O sistema usa um modelo híbrido RBAC/ABAC baseado em sessão com três camadas:

| Camada | Onde fica | O que faz |
|---|---|---|
| Papel (`VsPapelProfissional`) | Banco de dados (tabela `vs_papel_profissional`) | Define o "o que é" o profissional (ex: `enf-h`, Coordenador) |
| Lotação (`LotacaoProfissional`) | Banco de dados (tabela `lotacao_profissional`) | Define "onde" ele exerce o papel |
| Permissões de sessão (`permissoes_ativas`) | Session cookie | Lista de permissões funcionais lidas de `PAPEL_PERMISSOES` em `settings.py` |

O vínculo entre `auth_user` e `Profissional` é a FK `Profissional.usuario_id → auth_user.id`.

### 1.3 Fluxo de Acesso Atual (resumo)

    Usuário → POST /login/ (username+password)

      → Django authenticate() → auth_user

      → ProfileSetupMiddleware → Profissional.objects.get(usuario_id=request.user)

      → get_papeis_info() → session['modo_admin'], session['permissoes_ativas']

      → Se não-admin: /autenticacao/selecionar_perfil/ → session['active_servico_id'], etc.

      → Acesso liberado

### 1.4 Controles de Segurança Existentes

- `@login_required` em todas as views de Kanban e painel de gestão.
- `ProfileSetupMiddleware`: impede acesso de usuário logado sem perfil selecionado.
- `BloqueioApiAdminMiddleware`: bloqueia `/api/paineladmin/` sem permissão de sessão.
- `NoCacheMiddleware`: evita que dados de um usuário apareçam para outro via cache.
- Validação de perfil no POST de `selecionar_perfil_view` (segurança contra privilege escalation).
- `PAPEL_PERMISSOES` em `settings.py` como mapa estático de papéis → permissões.
- Cabeçalhos de segurança em produção (`HSTS`, SSL redirect, `X-Frame-Options: DENY`).

### 1.5 Problemas e Limitações Identificados no Modelo Atual

- **Acoplamento identidade/profissional:** a tabela `auth_user` e a tabela `profissional` estão acopladas via FK direta. Todo `Profissional` com acesso ao sistema precisa ter um `User` do Django associado — mas nem todo profissional deveria ser usuário (ex: médicos plantonistas).
- **Gerenciamento de senhas no Django:** o sistema hoje gerencia hashes de senha, reset por e-mail, validadores de complexidade etc. Isso aumenta a superfície de ataque e a carga operacional.
- **Permissões hardcoded em `settings.py`:** `PAPEL_PERMISSOES` e `ADMIN_PAPEL_NOMES` são listas estáticas que exigem re-deploy para qualquer mudança de papel ou permissão.
- **Sem suporte a SSO:** não há suporte a Single Sign-On, MFA, ou qualquer federação de identidade.
- **Sem visibilidade de sessões ativas:** não é possível auditar ou revogar sessões ativas sem intervenção no banco de dados.
- **Multi-tenancy futura impossível no modelo atual:** o campo `usuario_id` é uma FK para `auth_user` sem qualquer escopo de tenant — incompatível com a roadmap SaaS.

## 2. Arquitetura Alvo: Keycloak + Django

### 2.1 Divisão de Responsabilidades

    ┌──────────────────────────────────────────────────────────┐
    │  Keycloak (IDP externo)                                 │
    │  - Gerencia credenciais (senhas, MFA, SSO)              │
    │  - Emite JWT (Access Token) com: email, roles           │
    │    (macroperfis)                                        │
    │  - Realm: CarePlanner                                   │
    │  - Roles: admin_plat, admin_cp, coord_prg_*, coord_srv, │
    │           enf-h, gca                                    │
    └───────────────────────────┬──────────────────────────────┘
                                │ JWT via Authorization: Bearer
                                ▼
    ┌──────────────────────────────────────────────────────────┐
    │  Django / DRF (Backend GC Cuidado)                      │
    │  - Valida assinatura JWT via JWKS                       │
    │    (chave pública Keycloak)                             │
    │  - Binding: Profissional.objects.get(email=jwt_email)   │
    │  - Autorização ABAC/RBAC via lotacao_profissional       │
    │  - Não gerencia mais senhas                             │
    └──────────────────────────────────────────────────────────┘
                                │
                                ▼
    ┌──────────────────────────────────────────────────────────┐
    │  Frontend (Kanban + futura POA)                         │
    │  - keycloak-js: Authorization Code Flow + PKCE          │
    │  - Interceptor Axios: injeta Bearer token em toda req   │
    └──────────────────────────────────────────────────────────┘

### 2.2 Mecanismo de Binding (Vínculo de Identidade)

O documento técnico (Doc 07) define o binding por e-mail:

O JWT contém o claim `email` → Django executa `Profissional.objects.get(email=jwt_email)` → `request.user` passa a ser a instância local do `Profissional`.

**Implicação de implementação:** o campo `Profissional.email` (atualmente `null=True`, `blank=True`) deve ser promovido a obrigatório e único (`unique=True`, `null=False`).

## 3. Especificação da Nova API de Acesso (Backend)

Este backend (repositório `plataformacp/` ou novo serviço) não existe ainda. Abaixo a especificação mínima necessária:

### 3.1 Novo Authentication Backend Django

Criar `core/authentication.py`:

    # Pseudocódigo da especificação

    class KeycloakJWTAuthentication(BaseAuthentication):

        def authenticate(self, request):

            token = extrair_bearer_token(request.headers)

            if not token:
                return None

            payload = validar_jwt_com_jwks(token)  # Cache da chave pública do Keycloak

            email = payload.get('email')

            try:
                profissional = Profissional.objects.get(email=email)
                return (profissional, token)

            except Profissional.DoesNotExist:
                raise AuthenticationFailed("Profissional não cadastrado (HTTP 403)")

### 3.2 Configuração DRF

    REST_FRAMEWORK = {
        'DEFAULT_AUTHENTICATION_CLASSES': [
            'core.authentication.KeycloakJWTAuthentication',
        ],
        'DEFAULT_PERMISSION_CLASSES': [
            'rest_framework.permissions.IsAuthenticated',
        ],
    }

### 3.3 Endpoints necessários na nova API

| Endpoint | Método | Descrição |
|---|---|---|
| `/api/auth/token/verify/` | POST | Valida token JWT e retorna dados do Profissional |
| `/api/auth/profile/` | GET | Retorna perfis disponíveis para o Profissional autenticado |
| `/api/auth/profile/select/` | POST | Seleciona perfil ativo (equivalente à `selecionar_perfil_view`) |

### 3.4 Dependência a instalar

    python-keycloak>=3.0  # ou mozilla-django-oidc>=4.0
    PyJWT>=2.8
    cryptography>=42.0

## 4. Impacto no Código Legado do CarePlanner

### 4.1 Arquivos que precisam ser modificados

| Arquivo | Impacto | Tipo de mudança |
|---|---|---|
| `careplanner/settings.py` | Alto | Remover `django.contrib.auth` do fluxo principal; adicionar `KEYCLOAK_*` settings; remover `PAPEL_PERMISSOES` estático |
| `core/middleware.py` | Alto | `ProfileSetupMiddleware`: trocar `Profissional.objects.get(usuario_id=request.user)` por `Profissional.objects.get(email=request.user.email)` (onde `request.user` passa a ser o objeto `Profissional`) |
| `core/authentication.py` | Novo | Implementar `KeycloakJWTAuthentication` |
| `core/models.py` | Médio | `BaseModel.usuario_id` (FK para `auth_user`) precisa ser revisado — após migração, o campo de auditoria deve referenciar o profissional ou ser substituído |
| `autenticacao/views.py` | Alto | `selecionar_perfil_view` e `custom_logout_view` devem ser adaptados para o fluxo OIDC; logout deve chamar endpoint de logout do Keycloak |
| `autenticacao/urls.py` | Médio | Adicionar rota de callback OIDC |
| `paineladmin/views.py` | Alto | `login_page_view` com `authenticate(username, password)` deve ser removido — substituído pelo redirecionamento ao Keycloak |
| `kanban/views.py` | Baixo | As views já usam `IsAuthenticated` do DRF — compatíveis com o novo backend de autenticação sem mudança de lógica |
| `core/access_control.py` | Baixo | `gerar_perfis_de_acesso` usa `LotacaoProfissional` — não precisa mudar |

### 4.2 O ponto crítico: `BaseModel.usuario_id`

Todo modelo de auditoria herda `usuario_id = models.ForeignKey(User, ...)`. Essa FK aponta para `django.contrib.auth.User`. Existem duas opções:

- **Opção A (recomendada):** manter a tabela `auth_user` para fins de auditoria durante período de transição, mas desconectá-la da autenticação. O `auth_user` passa a ser apenas um registro de rastreabilidade gerado automaticamente no primeiro login via Keycloak.
- **Opção B:** trocar o campo `usuario_id` de FK para `auth_user` para FK para `Profissional` — requer migration em todas as tabelas de domínio.

A **Opção A** é mais segura para a migração incremental.

### 4.3 Frontend (Kanban)

O Kanban atual é renderizado server-side (Django templates + `@login_required`). A integração com `keycloak-js` exige:

- Remover os decoradores `@login_required` das views de template (que hoje redirecionam para `/login/`).
- Na inicialização do JS da página, iniciar o fluxo OIDC com `keycloak.init({ onLoad: 'login-required' })`.
- Configurar interceptor Axios/Fetch para injetar `Authorization: Bearer <token>`.
- As chamadas de API DRF passam a ser autenticadas via JWT em vez de cookie de sessão.

## 5. Impacto no Banco de Dados

### 5.1 Mudanças necessárias na tabela `profissional`

    -- Tornar email obrigatório e único (binding por email)
    ALTER TABLE profissional ALTER COLUMN email SET NOT NULL;
    ALTER TABLE profissional ADD CONSTRAINT profissional_email_unique UNIQUE (email);

**Risco:** existem registros com `email IS NULL` na tabela `profissional`. Antes da migration, é necessário auditar e preencher ou remover esses registros.

### 5.2 Nova tabela necessária: `usuario_keycloak` (opcional, Opção A)

Para manter rastreabilidade sem depender do `auth_user` do Django:

    CREATE TABLE usuario_keycloak (
        id              SERIAL PRIMARY KEY,
        keycloak_sub    UUID NOT NULL UNIQUE,   -- 'sub' do JWT
        email           VARCHAR(255) NOT NULL UNIQUE,
        nome            VARCHAR(255),
        criado_em       TIMESTAMPTZ DEFAULT NOW(),
        ultimo_login    TIMESTAMPTZ
    );

Isso permite que o `BaseModel.usuario_id` continue funcionando com mínimo de mudança.

### 5.3 Tabelas que NÃO precisam mudar

As tabelas de negócio (`lotacao_profissional`, `executor_servico`, `vs_papel_profissional`, `servico_cuidado`, `plano`) permanecem intactas. O modelo ABAC/RBAC interno do CarePlanner é preservado — o Keycloak apenas autentica, e o Django continua fazendo a autorização via hierarquia de negócio.

### 5.4 Tabelas do Django que podem ser descontinuadas (a longo prazo)

| Tabela Django | Status pós-migração |
|---|---|
| `auth_user` | Mantida temporariamente (auditoria) |
| `auth_user_groups` | Pode ser removida — grupos migram para Roles do Keycloak |
| `auth_user_user_permissions` | Pode ser removida |
| `authtoken_token` | Remover — substituído por JWT do Keycloak |
| `django_session` | Avaliar — sessão de seleção de perfil ainda pode ser necessária |

## 6. Fases e Passos para a Migração

### Fase 0 — Preparação (sem mudança em produção)

- Auditar e corrigir emails na tabela `profissional`: garantir que todo profissional com acesso tenha email preenchido e único.
- Configurar Realm no Keycloak: criar o Realm `CarePlanner`, as Roles (macroperfis) e cadastrar usuários-piloto.
- Documentar mapeamento `auth_user → profissional.email`: levantar todos os usuários ativos e confirmar que há email válido correspondente.
- Instalar dependências (`python-keycloak` ou `mozilla-django-oidc`, `PyJWT`) no `requirements.txt` — sem ativar ainda.

### Fase 1 — Backend Parallel (Feature F1.0, escopo mínimo)

- Implementar `KeycloakJWTAuthentication` em `core/authentication.py`.
- Adicionar configurações Keycloak em `settings.py` (variáveis de ambiente: `KEYCLOAK_SERVER_URL`, `KEYCLOAK_REALM`, `KEYCLOAK_CLIENT_ID`, `KEYCLOAK_ALGORITHMS`).
- Configurar DRF para aceitar tanto JWT (novo) quanto cookie de sessão (legado) — modo dual durante transição.
- Testar com um endpoint não-crítico usando Postman + token emitido pelo Keycloak.
- Aplicar migration no banco: tornar `profissional.email` `UNIQUE NOT NULL`.

### Fase 2 — Frontend Kanban

- Adicionar `keycloak-js` ao `package.json`.
- Criar adaptador de inicialização do Keycloak (`keycloak-init.js`) nas páginas do Kanban.
- Configurar interceptor Axios para injetar Bearer token.
- Substituir redirecionamento de `@login_required` pelo redirecionamento OIDC do Keycloak.
- Testar com usuário `enf-h` do Keycloak: login → retorno ao Kanban → chamadas de API funcionando.

### Fase 3 — Remoção do Auth Legado

- Desativar `login_page_view` (`paineladmin`) — redirecionar para Keycloak.
- Adaptar `custom_logout_view` para chamar `GET /realms/{realm}/protocol/openid-connect/logout` do Keycloak.
- Remover `rest_framework.authtoken` dos `INSTALLED_APPS`.
- Remover `django.contrib.auth` dos `MIDDLEWARE` (manter apenas para admin Django interno, se necessário).
- Deprecar tabelas `auth_*` não utilizadas (com plano de exclusão futuro).

### Fase 4 — Painel Admin Keycloak

- Criar views para `admin_cp` (administrador de instância) usando autenticação JWT.
- Migrar o `paineladmin` para validar via macroperfil Keycloak em vez de `session['permissoes_ativas']`.
- Avaliar substituição do `BloqueioApiAdminMiddleware` por permission class DRF específica.

## 7. Riscos e Mitigações

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Profissionais sem email cadastrado travam no login | Alta | Alto | Auditar e corrigir antes da Fase 1 |
| Regress no fluxo de seleção de perfil | Média | Alto | Manter modo dual (sessão + JWT) na Fase 1-2 |
| Token JWT expira durante sessão ativa no Kanban | Média | Médio | Implementar refresh token automático no `keycloak-js` |
| `BaseModel.usuario_id` FK quebra com remoção de `auth_user` | Alta | Alto | Usar Opção A (manter `auth_user` como stub) até refatoração total |
| Keycloak fora do ar bloqueia todos os logins | Baixa | Crítico | Keycloak em HA; definir fallback de emergência |
| Usuários de automação (ex: "Geralda") sem fluxo definido | Média | Médio | Criar Service Account no Keycloak com Client Credentials Flow |

## 8. Dependências Externas Identificadas

- Servidor Keycloak já configurado (confirmado na reunião de 22/05/2026).
- Referência de implementação: integração realizada no Intellicare (a ser documentada por Eduardo Garabini).
- Nenhuma library de JWT está instalada hoje no `requirements.txt` do CarePlanner.

## 9. Conclusão

A migração é viável e bem fundamentada nos documentos produzidos. O modelo de binding por email é elegante e preserva toda a lógica de negócio do CarePlanner intacta. Os pontos de atenção críticos são:

- O campo `profissional.email` é o pivô de toda a arquitetura — deve ser auditado e tornado único antes de qualquer deploy.
- O `BaseModel.usuario_id` é a maior dívida técnica da migração — requer decisão de design (Opção A ou B) antes da Fase 3.
- O modo dual (sessão legada + JWT) é necessário durante as Fases 1 e 2 para garantir zero downtime.
- A seleção de perfil (`selecionar_perfil_view`) não desaparece — ela será mantida como etapa pós-autenticação Keycloak, pois o mapeamento de lotações e serviços continua no banco local do Django.

O sequenciamento proposto (Fases 0→4) permite entregas incrementais e reversíveis, com risco operacional controlado.