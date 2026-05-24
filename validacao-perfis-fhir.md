# Validação de Perfis FHIR no CarePlanner - Guia Técnico

**Versão:** 1.0  
**Data:** 24/05/2026  
**Abordagem:** Spec Driven Development

---

## 1. Objetivo

Estabelecer uma estratégia robusta de **validação de perfis FHIR** (StructureDefinition) para garantir que todos os recursos FHIR criados ou recebidos no CarePlanner estejam em conformidade com as regras de negócio da plataforma, antes de serem persistidos ou enviados para outros sistemas.

---

## 2. Conceitos Fundamentais

- **StructureDefinition**: Define o "perfil" (regras customizadas) sobre um recurso base do FHIR (ex: Practitioner, Patient).
- **Profile**: Restrições adicionais (cardinalidade, valores permitidos, extensions obrigatórias, slicing, etc.).
- **Validation**: Processo de verificar se um recurso JSON/XML está conforme o perfil.

**Benefícios:**
- Garantia de qualidade dos dados
- Interoperabilidade com outros sistemas FHIR
- Detecção precoce de erros
- Suporte a auditoria e conformidade regulatória (ex: LGPD, ANS)

---

## 3. Perfis Recomendados para CarePlanner

| Recurso FHIR       | Nome do Perfil                          | Extensões Obrigatórias              | Restrições Principais |
|--------------------|-----------------------------------------|-------------------------------------|-----------------------|
| Practitioner       | `CarePlannerPractitioner`               | `keycloak-user-id`                  | CPF obrigatório, email verificado |
| Organization       | `CarePlannerOrganization`               | `cnpj`, `keycloak-org-id`           | CNPJ válido, tipo de organização |
| Patient            | `CarePlannerPatient`                    | -                                   | CPF ou CNS obrigatório |
| Appointment        | `CarePlannerAppointment`                | -                                   | Practitioner + Patient presentes |
| Encounter          | `CarePlannerEncounter`                  | -                                   | Vínculo com Organization |

---

## 4. Ferramentas de Validação

**Recomendação Principal: HAPI FHIR Validator + Python**

- **HAPI FHIR Validator** (Java) → Mais completo
- **fhir.resources + pydantic** (Python) → Mais integrado ao Django
- **Inferno Validator** ou **FHIR Validator CLI**

**Biblioteca Python Recomendada:**

```bash
pip install fhir.resources pydantic fhir-validator