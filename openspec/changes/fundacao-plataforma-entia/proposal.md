## Why

Organizações que precisam operar dados de negócio específicos normalmente dependem da criação repetitiva de bancos, APIs, telas de cadastro, consultas e controles de segurança para cada domínio. A ENT.IA propõe uma base web segura e configurável na qual novas entidades de negócio possam ser definidas por metadados e disponibilizadas dinamicamente, reduzindo esse esforço sem abrir mão de isolamento, governança e rastreabilidade.

## Product Identity

- **Nome:** ENT.IA
- **Domínio:** `ent.ia.br`
- **Significado:** **ENT**idades de negócio + **IA**
- **Descrição:** Plataforma inteligente para modelagem e personalização de entidades de negócio.
- **Slogans:**
  - **Entidades moldadas ao seu negócio.**
  - **Modele entidades. A IA faz o resto.**

## What Changes

- Criar a **ENT.IA**, uma plataforma web multi-organização para definição e operação inteligente de entidades de negócio configuráveis.
- Disponibilizar uma estrutura comum de identificação, login, autenticação e gerenciamento seguro de sessões.
- Isolar configurações e dados por organização/cliente dentro da mesma instalação da plataforma.
- Controlar permissões no contexto de organização, entidade e operação.
- Manter um catálogo de metadados para definição de entidades, atributos, relacionamentos, validações e demais características necessárias à sua manipulação.
- Disponibilizar persistência, operações de cadastro e consulta e suas respectivas interfaces de forma dinâmica a partir do catálogo publicado.
- Disponibilizar uma área de chat integrada a uma LLM, a ser selecionada futuramente, capaz de consultar e propor alterações nos registros por meio das APIs REST da plataforma.
- Mediar toda ação solicitada pela LLM por ferramentas controladas, mantendo autorização contextual, confirmação humana para escritas e rastreabilidade da execução.
- Registrar eventos relevantes de segurança, administração e manipulação de dados em uma trilha de auditoria imutável e resistente a adulteração.
- Estabelecer requisitos transversais de segurança para proteger identidades, dados organizacionais, configurações e operações dinâmicas.
- Manter regras de negócio específicas de cada domínio fora do núcleo fixo da plataforma; elas serão representadas futuramente por recursos configuráveis ou extensões explicitamente definidas.

## Capabilities

### New Capabilities

- `identity/authentication`: identificação de usuários, login, recuperação de acesso e gerenciamento seguro de sessões.
- `tenancy/organization-management`: cadastro de organizações e isolamento de suas configurações, usuários e dados.
- `authorization/access-control`: autorização contextual por organização, entidade e operação.
- `metadata/entity-catalog`: definição, validação, versionamento e publicação de entidades de negócio e seus metadados.
- `runtime/dynamic-records`: persistência e manipulação dinâmica dos registros das entidades publicadas.
- `ui/dynamic-experiences`: geração dinâmica de telas de cadastro, visualização, listagem e consulta.
- `ai/conversational-operations`: interação conversacional com uma LLM para consultar e alterar registros exclusivamente por operações REST autorizadas e controladas pela plataforma.
- `audit/tamper-resistant-audit`: trilha de auditoria imutável e resistente a adulteração para eventos relevantes da plataforma.
- `security/platform-security`: controles transversais para proteção da plataforma, segregação entre organizações e tratamento seguro de entradas e dados.

### Modified Capabilities

Nenhuma. A proposta estabelece as capacidades iniciais de um projeto novo.

## Impact

- Introduz a base conceitual de um novo produto web e seus futuros componentes de frontend, backend, persistência, identidade, autorização, IA e observabilidade.
- Torna o catálogo de metadados o contrato central entre configuração, armazenamento, APIs e interfaces dinâmicas.
- Torna a API REST um limite de segurança e integração para operações realizadas pela IA, sem acesso da LLM ao banco de dados.
- Exige abstração do provedor de LLM, governança dos dados enviados ao modelo, confirmação das mutações e auditoria correlacionada das chamadas de ferramentas.
- Exige que o isolamento entre organizações e a autorização contextual sejam aplicados de forma consistente em todas as camadas futuras.
- Exige mecanismos verificáveis de imutabilidade e resistência a adulteração para a auditoria, incluindo a própria administração do catálogo.
- As tecnologias, o modelo físico de persistência, a estratégia de extensibilidade e a topologia de implantação serão definidos no artefato de arquitetura após discussão específica.
