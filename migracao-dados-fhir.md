# Migração de Dados para FHIR no CarePlanner

**Versão:** 1.0  
**Data:** 24/05/2026  
**Abordagem:** Spec Driven Development

---

## 1. Objetivo

Definir a estratégia de migração dos dados legados do CarePlanner (modelos Django) para o padrão **FHIR** (Fast Healthcare Interoperability Resources), mantendo consistência com a integração Keycloak já planejada.

---

## 2. Contexto Atual

- Dados atualmente armazenados em modelos Django (`Profissional`, `Paciente`, `Organizacao`, agendamentos, etc.).
- Necessidade de expor e armazenar recursos no formato FHIR (R4 ou R5).
- Integração com Keycloak usando `email` como principal binding.

---

## 3. Estratégias de Migração Recomendadas

### Opção 1: Migração Big Bang (Recomendada para MVP)
- Migrar todos os dados de uma vez em uma janela de manutenção.
- **Vantagens**: Simples, consistência imediata.
- **Desvantagens**: Alto risco, downtime maior.

### Opção 2: Migração Incremental / Dual Write (Recomendada)
- Manter os modelos Django como fonte da verdade inicialmente.
- Criar **sincronização bidirecional** ou **event-driven** para manter recursos FHIR atualizados.
- Gradualmente migrar tabelas uma a uma.

### Opção 3: FHIR First (para novas funcionalidades)
- Novas entidades já são criadas diretamente como recursos FHIR.
- Migração retroativa dos dados históricos.

---

## 4. Mapeamento Principais Recursos FHIR

| Entidade Django       | Recurso FHIR              | Campos Críticos |
|-----------------------|---------------------------|-----------------|
| `Profissional`        | `Practitioner`            | `id`, `name`, `telecom`, `identifier` (CPF), `keycloak_user_id` |
| `Organizacao`         | `Organization`            | `id`, `name`, `type`, `address` |
| `Paciente`            | `Patient`                 | `id`, `name`, `gender`, `birthDate`, `identifier` |
| Agendamento           | `Appointment`             | `status`, `participant`, `start`, `end` |
| Prontuário            | `Encounter` + `Observation` | Histórico clínico |
| Usuário Keycloak      | Extension em Practitioner | `keycloak_user_id` |

---

## 5. Ferramentas e Tecnologias Sugeridas

- **ETL Tools**: Apache NiFi, Talend, ou scripts Python + `fhir.resources`
- **Biblioteca Python**: `fhir.resources`, `pydantic`
- **FHIR Server**: HAPI FHIR, Microsoft FHIR Server, ou customizado no Django
- **Validação**: FHIR Validator + StructureDefinitions customizadas
- **ID Strategy**: Usar UUIDs ou manter IDs legados como `identifier`

---

## 6. Passos da Migração (Fases)

**Fase 0: Preparação**
- Mapear todos os modelos Django para recursos FHIR (criar planilha de mapeamento)
- Definir StructureDefinitions / Profiles FHIR do CarePlanner
- Criar `fhir_id` em todas as tabelas legadas

**Fase 1: Migração de Referência**
- Migrar `Organization` e `Practitioner` primeiro (base para tudo)
- Vincular com Keycloak via `email` + `keycloak_user_id`

**Fase 2: Migração de Dados Transacionais**
- Pacientes, Agendamentos, Prontuários

**Fase 3: Dual Mode + Cutover**
- Implementar sincronização em tempo real (Django signals → FHIR)
- Testes de consistência
- Desligar modelos legados gradualmente

---

## 7. Desafios e Mitigações

- **Referências entre recursos**: Usar `Reference` do FHIR + resolver por identifier
- **Dados incompletos**: Estratégia de enriquecimento progressivo
- **Performance**: Migração em batches + índices temporários
- **Auditoria**: Manter `BaseModel` com `usuario_id` durante transição
- **Versão de recursos**: Usar `meta.versionId` e `meta.lastUpdated`

---

## 8. Próximos Passos Imediatos

1. Criar planilha de mapeamento detalhada (Django → FHIR)
2. Escolher estratégia (Incremental é mais segura)
3. Implementar conversor `Django Model → FHIR Resource`
4. Integrar com a migração de autenticação Keycloak (Fase 1)

---

**Recomendação Final:**  
Adotar **migração incremental com dual write** + foco inicial em `Organization` e `Practitioner`, alinhado com a integração Keycloak.

---

Documento complementar ao "Integração Keycloak + CarePlanner".