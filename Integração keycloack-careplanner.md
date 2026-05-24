# Integração Keycloak + CarePlanner (Base FHIR)

**Versão:** 1.2  
**Data:** 24/05/2026  
**Abordagem:** Spec Driven Development  
**Estratégia:** Modelo Híbrido de Autenticação e Autorização

---

## 1. Visão Geral

Este documento descreve a integração entre o **Keycloak** (Identity Provider) e o **CarePlanner** (plataforma baseada em FHIR).

**Modelo Híbrido adotado:**

- **Keycloak**: Responsável por **Autenticação** + **Papéis grossos** (Coarse-grained)
- **CarePlanner**: Responsável pelas **Permissões granulares** (Fine-grained) e regras de negócio específicas do domínio da saúde.

### Papéis no Keycloak (simplificados)

| Papel          | Descrição                              |
|----------------|----------------------------------------|
| `ADMIN`        | Administrador global do sistema        |
| `GESTOR`       | Gestor de Organização / Clínica        |
| `USUARIO`      | Profissional de saúde padrão           |

> Todas as permissões finas (ex: `VIEW_PATIENT`, `EDIT_PRESCRIPTION`, `SCHEDULE_APPOINTMENT`, `MANAGE_ORGANIZATION_USERS`, etc.) serão gerenciadas exclusivamente dentro do CarePlanner.

---

## 2. Estado Atual (Baseline)

- Autenticação monolítica do Django (`username + password` → `auth_user`)
- Vinculação: `Profissional.usuario_id → auth_user.id`
- Permissões: Hardcoded em `settings.py`
- Sem suporte a SSO, MFA, OIDC ou multi-tenant
- Senhas gerenciadas diretamente pelo Django

**Principais problemas identificados:**
- Forte acoplamento entre `auth_user` e `Profissional`
- Superfície de ataque desnecessária
- Dificuldade de evolução e escalabilidade

---

## 3. Ponto Crítico da Arquitetura Alvo

- **Binding principal**: `Profissional.objects.get(email=jwt_email)`
- **Ação urgente**: Alterar o campo `profissional.email` para `UNIQUE NOT NULL`
- Manter o `BaseModel.usuario_id` (FK para `auth_user`) como **stub temporário** (Opção A) durante a migração para evitar grande impacto nas tabelas de auditoria.

---

## 4. Arquitetura da Integração

flowchart TD
    A[Usuário / Frontend] --> B[CarePlanner]
    B --> C[Keycloak - OIDC Login]
    C --> D[Access Token + Roles Grossas]
    D --> E[KeycloakJWTAuthentication]
    E --> F[Binding por email + keycloak_user_id]
    F --> G[Busca Profissional + Organization]
    G --> H[Carrega Permissões Granulares]
    H --> I[PermissionService + Regras de Negócio]

---

## 5. Modelo de Dados Recomendado

erDiagram
    KC_USER {
        string keycloak_user_id PK
        string username
        string email
    }
    PROFISSIONAL {
        int id PK
        string keycloak_user_id FK
        string fhir_id
        string email UK "UNIQUE NOT NULL"
        int organization_id FK
    }
    ORGANIZATION {
        int id PK
        string fhir_id
        string name
        string type
    }
    PERMISSAO {
        int id PK
        string code "VIEW_PATIENT, EDIT_PRESCRIPTION..."
        string description
    }
    PROFISSIONAL_PERMISSAO {
        int profissional_id FK
        int organization_id FK
        int permissao_id FK
        string scope "OWN|ORGANIZATION|ALL"
    }

    KC_USER ||--o{ PROFISSIONAL : "vincula"
    PROFISSIONAL ||--o{ PROFISSIONAL_PERMISSAO : "possui"
    ORGANIZATION ||--o{ PROFISSIONAL : "pertence"
    PROFISSIONAL_PERMISSAO }o--|| PERMISSAO : "referencia"

---

## 6. Fases de Implementação

**Fase 0: Preparação**
- Auditar e corrigir emails duplicados/inválidos
- Tornar `profissional.email` → `UNIQUE NOT NULL`
- Criar Realm + Client no Keycloak
- Configurar os 3 papéis grossos

**Fase 1: Autenticação Dual**
- Implementar `KeycloakJWTAuthentication`
- Modo híbrido: JWT + Sessão Django legada
- Binding por email + armazenamento do `keycloak_user_id`

**Fase 2: Frontend**
- Integrar `keycloak-js` no Kanban
- Receber e tratar o Access Token

**Fase 3: Autorização Granular**
- Criar `PermissionService`
- Migrar permissões do `settings.py` para o banco
- Implementar regras baseadas em Profissional + Organização

**Fase 4: Migração Final**
- Remover autenticação Django legada
- Migrar Painel Admin
- Desativar `auth_user` (opcional)

---

## 7. Vantagens da Abordagem Híbrida

- Keycloak mais leve e fácil de manter
- Maior flexibilidade para regras específicas do domínio FHIR
- Token JWT menor (melhor performance)
- Redução significativa da dívida técnica
- Preparação natural para SSO, MFA e multi-tenant futuro

---

**Próximos Passos Imediatos:**
1. Realizar auditoria dos emails (`profissional.email`)
2. Criar o Realm e Client no Keycloak
3. Iniciar implementação da Fase 1

---

Documento alinhado com a visão discutida pelo time (modelo híbrido simplificado).