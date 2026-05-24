# Validação de Perfis HL7 FHIR no CarePlanner - Guia Técnico Completo

**Versão:** 1.0  
**Data:** 24/05/2026  
**Abordagem:** Spec Driven Development

---

## 1. Objetivo

Explorar e definir a estratégia completa de **validação de perfis HL7 FHIR** no CarePlanner, garantindo conformidade com os padrões HL7, interoperabilidade e qualidade dos dados clínicos.

---

## 2. Contexto HL7 FHIR

- **HL7 FHIR** (Fast Healthcare Interoperability Resources) é o padrão mais moderno da HL7.
- **Perfis (Profiles)**: Restrições aplicadas sobre os recursos base do FHIR (StructureDefinition).
- **Validação**: Verificação se um recurso está conforme o perfil declarado (`meta.profile`).

**Níveis de Conformidade HL7:**
- **Base Resource** → Validação mínima
- **Perfil Nacional/Internacional** (ex: BR Core, IPS)
- **Perfil Local (CarePlanner)** → Regras específicas do domínio

---

## 3. Perfis HL7 Recomendados para CarePlanner

| Recurso FHIR          | Perfil HL7 Base                  | Perfil CarePlanner Customizado              | Extensões Obrigatórias |
|-----------------------|----------------------------------|---------------------------------------------|------------------------|
| Practitioner          | HL7 Practitioner                 | CarePlannerPractitioner                     | keycloak-user-id       |
| Organization          | HL7 Organization                 | CarePlannerOrganization                     | cnpj, keycloak-org-id  |
| Patient               | HL7 Patient                      | CarePlannerPatient                          | CNS/CPF                |
| Appointment           | HL7 Appointment                  | CarePlannerAppointment                      | -                      |
| Encounter             | HL7 Encounter                    | CarePlannerEncounter                        | -                      |
| Observation           | HL7 Observation                  | CarePlannerObservation                      | -                      |

**Exemplo de declaração de perfil no recurso:**

```json
{
  "resourceType": "Practitioner",
  "meta": {
    "profile": ["https://careplanner.com/fhir/StructureDefinition/CarePlannerPractitioner"]
  },
  ...
}