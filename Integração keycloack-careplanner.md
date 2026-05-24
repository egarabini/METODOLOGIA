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

```mermaid
flowchart TD
    A[Usuário / Frontend] --> B[CarePlanner]
    B --> C[Keycloak - OIDC Login]
    C --> D[Access Token + Roles Grossas]
    D --> E[KeycloakJWTAuthentication]
    E --> F[Binding por email + keycloak_user_id]
    F --> G[Busca Profissional + Organization]
    G --> H[Carrega Permissões Granulares]
    H --> I[PermissionService + Regras de Negócio]