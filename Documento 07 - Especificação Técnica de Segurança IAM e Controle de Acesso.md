# Documento 07 — Especificação Técnica de Segurança IAM e Controle de Acesso

**Versão:** 1.0 (Integração Keycloak e Modelo GC Cuidado)  
**Status:** Homologado para Engenharia (Feature F1.0)

## 1. Objetivo da Especificação

Este documento detalha o "Como" da Feature F1.0. Ele traduz as diretrizes arquiteturais de separação entre **"Usuário"** (Identidade no Keycloak) e **"Profissional"** (Regras de Negócio no GC Cuidado), estabelecendo o mecanismo de vinculação e o fluxo de autorização baseado nas tabelas relacionais do sistema legado e do Core MCP.

## 2. O Mecanismo de Vinculação (Binding)

Para garantir que o Keycloak não assuma a complexidade do modelo de saúde, adotamos a regra de **Binding por E-mail**:

- **Keycloak (IDP):** autentica o usuário e emite o token JWT contendo o claim `email` e a macrocategoria de perfil (ex: `enf-h`).
- **Django (Backend):** intercepta o JWT, valida a assinatura e extrai o `email`. Em seguida, executa a query: `Profissional.objects.get(email=jwt_email)`.
- **Resultado:** o contexto da requisição no Django (o objeto `request.user`) passa a ser a instância local do **Profissional**, herdando todas as suas lotações e permissões estruturais (ABAC/RBAC nativo). Nenhuma sincronização de IDs UUID entre bases é necessária.

## 3. Arquitetura de Autorização (Modelo GC Cuidado)

Uma vez que o Django identifica o **Profissional** autenticado, a autorização de ações na plataforma (como acessar um bucket ou executar um serviço) deriva estritamente da seguinte hierarquia de tabelas do esquema de dados CarePlanner:

| Camada / Entidade | Tabela de Origem | Regra de Vínculo e Acesso |
|---|---|---|
| 1. PGCC (Programa) | (Futura) | Programa de Gestão (Ex: Linha de Cuidado Internação). Coordenado por um profissional com macroperfil `coord_prg` em determinados locais. |
| 2. PLGCC (Plano) | `plano` | Pertence a um PGCC. Define os planos táticos (Ex: Transição de Cuidado de Internação). |
| 3. SRC (Serviço) | `servico_cuidado` | Pertence a um Plano. (Ex: Planejamento de Alta). Coordenado por um `coord_srv`. |
| 4. Profissional & Papel | `profissional` / `papel_profissional` | O núcleo da identidade (ABAC). Define **QUEM** é a pessoa e **O QUE** ela é institucionalmente (Ex: `enf-h`, `gca`). |
| 5. Lotação & Execução | `lotacao_profissional` / `executor_servico` | Define **ONDE** o profissional exerce o papel (Ex: Ala E) e **SE** ele tem permissão para executar um SRC (Ex: `enf-h` executa Planejamento de Alta). |

## 4. Fluxo Técnico OIDC para as Aplicações Frontend (Kanban)

O frontend existente será refatorado para utilizar a biblioteca `keycloak-js`. O fluxo de autenticação seguirá o modelo **Authorization Code Flow with PKCE**:

1. **Bootstrap:** a aplicação SPA (Single Page Application) do Kanban inicia e verifica a presença de sessão.
2. **Redirect (Login):** caso não autenticado, redireciona o usuário ao Keycloak (Realm: `CarePlanner`).
3. **Captura do JWT:** o Keycloak retorna o Access Token (JWT) contendo as Roles associadas e o e-mail verificado.
4. **Injeção Automática:** o Frontend configura um Interceptor no Axios/Fetch, adicionando o cabeçalho `Authorization: Bearer <TOKEN_JWT>` em toda requisição ao Backend Django.

## 5. Validação Backend (Django REST Framework)

O Django não administrará senhas. A camada de segurança será reconstruída para confiar exclusivamente na assinatura do Keycloak:

- **JWKS Verification:** o Backend fará cache da chave pública (JWKS) exposta pelo Keycloak e validará matematicamente o JWT de cada requisição.
- **Policy Gate Interno:** após validar o JWT, um Custom Authentication Backend do DRF mapeará o `jwt_email` para o registro **Profissional**. Se o e-mail não existir na tabela `profissional`, a API retornará **HTTP 403 Forbidden** (usuário autenticado, porém sem registro institucional no CarePlanner).
