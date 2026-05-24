# Migração de Dados para FHIR no CarePlanner - Versão Técnica Detalhada

**Versão:** 1.1  
**Data:** 24/05/2026  
**Abordagem:** Spec Driven Development

---

## 1. Objetivo Técnico

Migrar os dados legados do CarePlanner (Django models) para recursos **FHIR R4**, garantindo:
- Consistência entre modelo relacional e FHIR
- Integração com Keycloak (binding por `email` + `keycloak_user_id`)
- Migração incremental com mínimo downtime
- Zero perda de dados e auditoria completa

---

## 2. Mapeamento Detalhado Django ↔ FHIR

| Django Model              | FHIR Resource         | Estratégia de ID                        | Campos Obrigatórios                     | Extension / Custom                  |
|---------------------------|-----------------------|-----------------------------------------|-----------------------------------------|-------------------------------------|
| `Profissional`            | `Practitioner`        | `fhir_id` (UUID) + Identifier (CPF)    | name, telecom, gender                   | `keycloak_user_id`                  |
| `Organizacao`             | `Organization`        | `fhir_id` (UUID)                        | name, type, address                     | `cnpj`, `keycloak_org_id`           |
| `Paciente`                | `Patient`             | `fhir_id` (UUID) + Identifier (CPF)    | name, birthDate, gender                 | -                                   |
| `Agendamento`             | `Appointment`         | `fhir_id` (UUID)                        | status, start, end, participant         | -                                   |
| Registro Clínico          | `Encounter` + `Observation` | `fhir_id` (UUID)                  | status, class, subject                  | -                                   |

**Exemplo de Extension Keycloak:**

```json
{
  "url": "https://careplanner.com/fhir/StructureDefinition/keycloak-user-id",
  "valueString": "a1b2c3d4-e5f6-..."
}