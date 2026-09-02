# Handoff de contexto — ENT.IA

Última consolidação: 2026-09-02

Este documento permite que uma nova sessão do Codex retome o trabalho sem depender do histórico da conversa. Os artefatos OpenSpec continuam sendo a fonte formal dos requisitos; este arquivo reúne decisões arquiteturais, estado do trabalho e pendências ainda não registradas no `design.md`.

## Objetivo atual

Estruturar a proposta e a arquitetura inicial da ENT.IA, uma plataforma web multi-organização para modelar entidades de negócio por metadados e oferecer dinamicamente persistência, cadastro, visualização e consultas.

Ainda não há implementação da aplicação. O trabalho atual está na fase de especificação e desenho arquitetural.

## Identidade do produto

- **Nome:** ENT.IA
- **Domínio:** `ent.ia.br`
- **Significado:** **ENT**idades de negócio + **IA**
- **Descrição:** Plataforma inteligente para modelagem e personalização de entidades de negócio.
- **Slogans:**
  - **Entidades moldadas ao seu negócio.**
  - **Modele entidades. A IA faz o resto.**

A fundação inclui uma área de chat para consultas e alterações de registros por meio de uma LLM. O provedor e o modelo serão definidos futuramente; não assumir geração automática de entidades, agentes autônomos ou modelos específicos antes de novas decisões.

## Decisões confirmadas

### Multi-organização e dados

- Uma mesma instalação atenderá múltiplas organizações/clientes.
- O PostgreSQL usará banco e schema compartilhados para os dados da aplicação.
- Todo dado organizacional deverá possuir `organization_id` e ser isolado pelo contexto da organização.
- As definições das entidades serão globais e compartilhadas entre as organizações.
- Cada organização poderá habilitar ou desabilitar uma entidade publicada, sem personalizar sua definição.
- Haverá somente uma versão global ativa por entidade; não haverá uma versão de schema diferente por organização.
- Versões publicadas das definições serão imutáveis. Alterações deverão produzir nova versão.

### Catálogo e persistência dinâmica

- Administradores globais da ENT.IA criarão, validarão, publicarão e ativarão definições de entidades.
- Administradores organizacionais habilitarão entidades e administrarão acessos locais.
- A persistência seguirá o modelo de tabela física por entidade, compartilhada entre organizações e com `organization_id` obrigatório.
- Tabelas de volume muito elevado poderão receber particionamento posteriormente.
- O catálogo de metadados deverá descrever atributos, tipos, obrigatoriedade, validações, relacionamentos e informações de apresentação.
- A plataforma interpretará metadados em tempo de execução; não deverá depender da geração de código-fonte React ou Java para cada entidade.

### Identidade e autenticação

- A identidade do usuário será global, e um usuário poderá pertencer a múltiplas organizações.
- Cada organização poderá usar seu próprio provedor de identidade OIDC ou SAML.
- Quando a organização não possuir provedor próprio, será usado o provedor padrão da instalação.
- Keycloak self-hosted será o provedor padrão e o broker de identidade.
- Será adotado um único realm Keycloak chamado `ent-ia`.
- Senhas e hashes de senha não serão armazenados no banco da aplicação ENT.IA.
- Recuperação de credenciais e políticas de senha ficarão sob responsabilidade do provedor de identidade.
- Os dados do Keycloak ficarão logicamente separados dos dados da aplicação.

### Exposição e integração do Keycloak

- A instância do Keycloak permanecerá em rede interna e não será exposta diretamente à internet.
- Um proxy reverso ou WAF publicará somente os endpoints indispensáveis ao frontchannel de autenticação, incluindo os callbacks necessários ao login social.
- O hostname público previsto para autenticação é `auth.ent.ia.br`.
- A exposição pública deverá ser limitada a `/realms/ent-ia/*`, `/resources/*` e `/.well-known/*`, sujeita à validação final durante a implantação.
- O console administrativo, a Admin REST API, o realm `master`, métricas, health checks e a porta de gerenciamento permanecerão acessíveis somente pela rede interna ou VPN.
- O Keycloak deverá aceitar conexões somente dos proxies e serviços internos autorizados.
- A comunicação de backchannel entre o backend ENT.IA e o Keycloak utilizará a rede interna, mantendo hostname e issuer públicos consistentes com o fluxo OIDC.
- Será adotado o padrão BFF: o React iniciará o login pelo backend ENT.IA, o backend fará a troca do authorization code e manterá os tokens fora do navegador.
- O navegador receberá somente uma sessão protegida por cookie `HttpOnly`, `Secure` e com política `SameSite` apropriada.
- Não será usado fluxo de autenticação que envie usuário e senha à API da ENT.IA; autenticação, MFA e login social continuarão centralizados no Keycloak e nos provedores federados.
- TLS, sobrescrita segura de cabeçalhos encaminhados, lista de proxies confiáveis, rate limiting e proteção contra bots deverão ser aplicados no gateway.

### Autorização

- O Keycloak será responsável pela autenticação; a ENT.IA será responsável pela autorização de negócio.
- O modelo será RBAC contextual por organização.
- Perfis de acesso concederão permissões pela combinação de organização, entidade e operação.
- Um usuário poderá possuir múltiplos perfis na mesma organização.
- A permissão efetiva será a união das concessões dos perfis.
- Toda permissão não concedida será negada por padrão.
- No primeiro momento não haverá permissões diretas por usuário nem regras explícitas de negação.
- A autorização deverá ser aplicada no backend, independentemente das ações exibidas pelo frontend.

### Auditoria

- A trilha precisa ser append-only, imutável para a aplicação e resistente à adulteração.
- Inicialmente os eventos ficarão no mesmo PostgreSQL da aplicação.
- Não haverá armazenamento externo WORM ou serviço separado de auditoria no MVP.
- A trilha deverá permitir verificação de integridade e detecção de alterações ou remoções indevidas.
- A proteção inicial não é absoluta contra um DBA ou superusuário do banco; essa limitação deverá ser explicitada no design.
- Eventos relevantes incluem autenticação, administração, autorização, catálogo e manipulação dos registros dinâmicos.

### Backend e arquitetura

- O backend será implementado em Java, tecnologia em que o responsável pelo projeto possui maior experiência.
- A aplicação seguirá o modelo de monólito modular.
- Stack recomendada e aceita:
  - OpenJDK 25 LTS;
  - Spring Boot 4.1;
  - Spring Modulith;
  - Spring Security integrado ao Keycloak;
  - jOOQ para SQL estático e dinâmico;
  - PostgreSQL;
  - Liquibase para migrations do schema estático da plataforma, condicionado à revisão da licença da versão adotada;
  - Maven;
  - Testcontainers para testes de integração.
- Não introduzir Kafka inicialmente. Caso seja necessária execução assíncrona, começar com uma fila persistida no PostgreSQL.
- O Liquibase está confirmado como a ferramenta recomendada para evoluir as tabelas estáticas mantidas pelo código, como organizações, vínculos, perfis, catálogo, auditoria e conversas de IA.
- As tabelas físicas das entidades configuradas pelos usuários serão administradas por um motor próprio de schema da ENT.IA, e não por changelogs Liquibase gerados em tempo de execução.
- Em produção, as migrations estáticas deverão ser executadas uma única vez antes da disponibilização da nova versão, usando uma credencial de banco distinta e mais privilegiada que a credencial normal da aplicação.
- Antes de fixar a versão do Liquibase, deverá ser feita revisão técnica e jurídica da licença vigente. A linha Community 5.x usa FSL com conversão futura para Apache 2.0, enquanto versões anteriores possuíam condições diferentes; não atualizar a versão principal automaticamente sem essa revisão.

### Frontend

- React 19 com TypeScript e Vite.
- Aplicação autenticada no formato SPA; Next.js e SSR não são necessários inicialmente.
- MUI Core para os componentes visuais.
- TanStack Table para tabelas, filtros, ordenação e paginação dinâmicas.
- TanStack Query para estado de servidor e comunicação com APIs.
- React Hook Form para estado e validação dos formulários.
- JSON Schema 2020-12 como contrato de estrutura e validação, complementado por um UI Schema separado para layout, widgets e visibilidade.
- Será criado um renderer próprio da ENT.IA com registro de componentes para campos, formulários, detalhes, filtros e listagens.
- A validação no frontend servirá à experiência do usuário; o backend continuará sendo a autoridade final.
- Evitar Redux inicialmente. Estado de sessão e organização pode ser mantido em contextos pequenos.
- Testes previstos com Vitest, React Testing Library e Playwright.

### IA conversacional e operações REST

- A ENT.IA oferecerá uma área de chat para interação em linguagem natural com uma LLM cujo provedor e modelo serão selecionados futuramente.
- A IA poderá consultar registros e propor criação, alteração e exclusão por meio das APIs REST da plataforma.
- A LLM não terá acesso direto ao PostgreSQL, repositórios internos, SQL livre ou URLs arbitrárias.
- Um orquestrador de IA interno converterá tool calls estruturadas em chamadas REST controladas.
- A API REST e seu contrato OpenAPI serão a fronteira oficial entre a IA e as operações de negócio.
- Somente operações explicitamente habilitadas e compatíveis com as permissões efetivas do usuário serão apresentadas à LLM como ferramentas.
- O contexto de usuário e `organization_id` será derivado da sessão confiável, nunca do conteúdo do prompt.
- As chamadas REST da IA deverão preservar o usuário iniciador e identificar o componente de IA como executor.
- Se o orquestrador for implantado separadamente, deverá utilizar um token delegado e restrito à audience da API; não deverá atuar apenas com uma conta de serviço privilegiada.
- Consultas autorizadas poderão ser executadas automaticamente, respeitando limites de campos e registros.
- Toda criação, alteração ou exclusão exigirá pré-visualização e confirmação humana explícita antes da chamada REST efetiva.
- Mutações deverão utilizar idempotência, correlação, prazo de validade da aprovação e controle de versão do registro.
- Campos e entidades deverão admitir política de uso pela IA, incluindo permitido, mascarado ou proibido.
- A auditoria deverá correlacionar conversa, usuário, organização, ferramenta, aprovação, chamada REST e resultado.
- O conteúdo integral das conversas não deverá ser copiado indiscriminadamente para a auditoria imutável; mensagens terão armazenamento e retenção próprios.
- O núcleo da ENT.IA continuará como monólito modular. O orquestrador será um cliente REST interno e deverá possuir uma fronteira que permita isolamento ou implantação separada quando necessário.
- Banco vetorial, RAG, MCP e agentes autônomos não são requisitos da primeira versão dessa capacidade.

## Estado do OpenSpec

- OpenSpec instalado: versão `1.11.0`.
- Mudança ativa: `fundacao-plataforma-entia`.
- Schema do workflow: `spec-driven`.
- Diretório: `openspec/changes/fundacao-plataforma-entia/`.
- `proposal.md`: concluído.
- `specs`: concluídas para as nove capacidades propostas.
- `design.md`: próximo artefato.
- `tasks.md`: bloqueado até a conclusão do design.
- Última validação: `openspec validate fundacao-plataforma-entia --strict` executada com sucesso.

Capacidades especificadas:

1. `identity/authentication`
2. `tenancy/organization-management`
3. `authorization/access-control`
4. `metadata/entity-catalog`
5. `runtime/dynamic-records`
6. `ui/dynamic-experiences`
7. `audit/tamper-resistant-audit`
8. `security/platform-security`
9. `ai/conversational-operations`

## Pontos ainda em aberto

Não transformar estes pontos em decisões sem discuti-los com o usuário:

- ciclo completo da entidade: rascunho, validação, publicação, ativação, depreciação e possível rollback;
- classificação de mudanças compatíveis e incompatíveis no schema;
- estratégia de migration, preenchimento de dados existentes e recuperação após falha de DDL;
- convenção e estabilidade dos nomes físicos de tabelas, colunas, índices e constraints;
- cardinalidades, integridade referencial e comportamento de exclusão dos relacionamentos;
- concorrência de atualização dos registros e controle de versão otimista;
- divisão entre isolamento aplicado pela aplicação e eventual PostgreSQL Row-Level Security;
- algoritmo de integridade da auditoria, encadeamento, checkpoints, retenção e verificação;
- contrato das APIs e estratégia de paginação, filtros, ordenação e exportação;
- roteamento de login quando uma identidade ou domínio puder corresponder a múltiplas organizações;
- topologia de implantação, gestão de segredos, backups, recuperação e observabilidade;
- limites funcionais e não funcionais do MVP;
- versão exata do Liquibase e compatibilidade de sua licença com distribuição, hospedagem e modelo comercial da ENT.IA;
- provedor e modelo de LLM, incluindo hospedagem, residência dos dados e possibilidade de configuração por organização;
- política de retenção, exportação e exclusão das conversas;
- limites de volume, custo e duração das interações com a LLM;
- escopo inicial de operações em lote e regras adicionais de confirmação;
- necessidade futura de RAG, embeddings, banco vetorial, MCP ou agentes especializados.

## Próximos passos recomendados

1. Criar o `design.md` pelo fluxo `openspec-continue-change`, consolidando as decisões técnicas deste handoff e resolvendo os pontos arquiteturais essenciais.
2. Validar o design contra a proposta e as nove especificações existentes.
3. Criar o `tasks.md`, dividindo a implementação em incrementos verificáveis.
4. Somente depois iniciar o scaffold e a implementação da aplicação.

Comandos úteis para a retomada:

```bash
openspec status --change "fundacao-plataforma-entia"
openspec instructions design --change "fundacao-plataforma-entia" --json
openspec validate fundacao-plataforma-entia --strict
```

Ao continuar, usar a habilidade `openspec-continue-change` e respeitar a criação de um artefato por vez.
