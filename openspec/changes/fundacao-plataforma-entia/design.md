## Context

A ENT.IA é um projeto greenfield. A motivação e o escopo funcional estão em `proposal.md`; os contratos de comportamento estão nas nove especificações desta mudança.

As principais restrições que moldam a arquitetura são:

- múltiplas organizações compartilham uma instalação, um banco PostgreSQL e um schema de aplicação;
- dados organizacionais são segregados por `organization_id`;
- definições de entidades são globais, compartilhadas e possuem uma única versão ativa;
- cada entidade dinâmica possui tabela física própria, compartilhada entre organizações;
- autenticação é delegada ao Keycloak e autorização de negócio pertence à ENT.IA;
- auditoria começa no mesmo PostgreSQL, precisa ser append-only e permitir detectar adulteração;
- interfaces e APIs são interpretadas em tempo de execução a partir do catálogo, sem geração de código por entidade;
- uma LLM futura interage com os registros somente por APIs REST controladas;
- a equipe possui maior experiência em Java e prioriza tecnologias abertas, sem assumir que toda edição comunitária possui licença OSI.

## Goals / Non-Goals

**Goals:**

- estabelecer limites modulares que permitam iniciar como monólito e evoluir componentes de maior risco ou escala;
- aplicar isolamento organizacional e autorização no servidor em todos os caminhos, inclusive IA;
- definir uma evolução segura e auditável dos metadados e das tabelas físicas;
- manter os contratos de catálogo, runtime, REST e interface consistentes com a mesma versão ativa;
- evitar dependência estrutural de um provedor de identidade ou de LLM específico;
- permitir implantação e recuperação previsíveis, sem apagar automaticamente dados publicados.

**Non-Goals:**

- criar microsserviços para cada domínio ou introduzir Kafka na fundação;
- permitir definições ou versões de entidade diferentes por organização;
- executar SQL, scripts ou código arbitrário fornecido por administradores ou pela LLM;
- gerar código-fonte Java ou React para cada entidade;
- automatizar alterações destrutivas de tabelas e colunas no MVP;
- garantir resistência absoluta da auditoria contra o superusuário do banco sem armazenamento ou âncora externa;
- definir agora o fornecedor da LLM, RAG, agentes autônomos ou banco vetorial;
- representar regras de negócio específicas e arbitrárias dentro do núcleo inicial.

## Decisions

### 1. Arquitetura de alto nível

O núcleo será um monólito modular Java. Keycloak, PostgreSQL, proxy/WAF e o provedor futuro de LLM são dependências externas. O orquestrador de IA terá uma fronteira que permita isolamento ou implantação separada, mas não força a decomposição prematura do restante da plataforma.

```text
+----------------------+       +----------------------+
| React SPA            |------>| BFF / REST API       |
+----------------------+       +----------+-----------+
                                           |
                           +---------------+---------------+
                           | Spring Boot modular monolith  |
                           +---------------+---------------+
                                           |
              +----------------------------+-------------------------+
              |                            |                         |
              v                            v                         v
       +-------------+              +-------------+           +-------------+
       | Keycloak    |              | PostgreSQL  |           | AI          |
       | private     |              | shared      |           | orchestrator|
       +-------------+              +-------------+           +------+------+
                                                                      |
                                                                      v
                                                               +-------------+
                                                               | LLM provider|
                                                               +-------------+
```

Alternativa considerada: microsserviços por capacidade. Foi rejeitada para a fundação por aumentar coordenação, observabilidade, consistência distribuída e custo operacional antes de existir demanda que justifique a separação.

### 2. Stack base

- OpenJDK 25 LTS;
- Spring Boot 4.1;
- Spring Modulith para limites, eventos e testes entre módulos;
- Spring Security para BFF, OIDC e proteção das APIs;
- jOOQ para SQL estático e construção segura das operações dinâmicas;
- PostgreSQL como banco principal;
- Liquibase para schema estático, sujeito à revisão de licença da versão adotada;
- Maven para build;
- Testcontainers para testes de integração com infraestrutura real;
- React 19, TypeScript e Vite no frontend;
- MUI Core, TanStack Table, TanStack Query e React Hook Form;
- Vitest, React Testing Library e Playwright para testes do frontend.

Alternativas consideradas: NoSQL como armazenamento primário e uma arquitetura frontend com SSR. PostgreSQL foi escolhido pela integridade relacional e capacidade de DDL, enquanto a aplicação autenticada e intensiva em interação não necessita de SSR inicialmente.

### 3. Limites dos módulos do backend

O backend será separado ao menos nos seguintes módulos lógicos:

- `identity`: integração OIDC, sessão BFF e identidade global;
- `tenancy`: organizações e vínculos de usuários;
- `authorization`: perfis e avaliação de permissões;
- `catalog`: entidades, campos, relações, versões e publicação;
- `schema-management`: análise de impacto, planos e execução de DDL;
- `dynamic-records`: validação e persistência dos registros;
- `dynamic-api`: contratos REST e OpenAPI do runtime;
- `audit`: gravação, consulta e verificação da trilha;
- `ai-orchestration`: conversas, tool calling, aprovações e cliente REST;
- `platform-observability`: correlação técnica, métricas e integração com logs/traces.

Módulos não acessam tabelas pertencentes a outro módulo diretamente. Colaboração síncrona usa interfaces públicas do módulo; efeitos assíncronos locais usam eventos de aplicação e outbox quando precisam sobreviver a falhas.

Alternativa considerada: uma camada compartilhada de serviços e repositórios. Foi rejeitada porque dissolveria os limites necessários para extrair componentes e testar isolamento.

### 4. Modelo multi-organização

Uma instalação usa um banco e schema de aplicação compartilhados. Todas as tabelas que contenham dados de cliente possuem `organization_id` obrigatório. O contexto é obtido da sessão autenticada e validado no servidor; valores enviados pelo cliente não são tratados como autoridade.

Registros dinâmicos usam UUID global como identificador técnico. Restrições únicas de negócio incluem `organization_id`, e relacionamentos entre registros não atravessam organizações. O formato lógico é equivalente a:

```text
UNIQUE (organization_id, business_key)

(organization_id, referenced_id)
    --> target (organization_id, id)
```

Definições e versões do catálogo não possuem proprietário organizacional. A relação `organization_entity` controla somente habilitação local; permissões continuam independentes.

Alternativas consideradas: database/schema por organização e definição personalizada por organização. Foram rejeitadas para reduzir custo operacional, divergência de versões e complexidade de migrations.

### 5. Identidade, Keycloak e sessão BFF

O Keycloak self-hosted será o provedor padrão e broker de identidades. Haverá um realm `ent-ia`, identidades globais e vínculos organizacionais mantidos pela aplicação. Cada organização poderá associar um provedor OIDC ou SAML; quem não possuir um usará o provedor padrão, incluindo login social habilitado.

O Keycloak ficará em rede interna. O proxy/WAF publicará em `auth.ent.ia.br` somente o frontchannel necessário, incluindo `/realms/ent-ia/*`, `/resources/*` e `/.well-known/*`, após validação final das rotas. Console administrativo, Admin REST API, realm `master`, health, métricas e porta de gerenciamento permanecerão internos ou sob VPN.

O React não armazena tokens. O BFF executa Authorization Code Flow, mantém tokens no backend e entrega ao navegador apenas cookie de sessão `HttpOnly`, `Secure` e `SameSite` apropriado. Backchannel usa rede interna com issuer e hostname OIDC consistentes.

Senhas, hashes e recuperação de credenciais não pertencem ao banco da ENT.IA. Dados do Keycloak ficam logicamente separados dos dados da aplicação.

Alternativas consideradas: autenticação própria e tokens mantidos pela SPA. Foram rejeitadas pelo custo de implementar controles maduros e pelo maior risco de exposição de credenciais e tokens.

### 6. Autorização RBAC contextual

Keycloak autentica; ENT.IA autoriza. Perfis pertencem a uma organização e concedem combinações de entidade e operação. Um vínculo pode possuir vários perfis, e sua permissão efetiva é a união das concessões. Ausência de concessão implica negação.

Não haverá permissões diretamente no usuário nem regras explícitas de negação no MVP. O backend repete a avaliação em toda operação, independentemente do que a interface exibiu.

As permissões administrativas do catálogo são separadas desde a fundação:

- `CATALOG_EDIT`;
- `CATALOG_VALIDATE`;
- `CATALOG_PUBLISH`;
- `CATALOG_ACTIVATE`.

O mesmo administrador global pode acumular todas no MVP. Não há aprovação obrigatória por segunda pessoa, mas toda ativação exige relatório de impacto e confirmação explícita.

Alternativa considerada: armazenar papéis e permissões de entidade no Keycloak. Foi rejeitada porque entidades e operações são metadados dinâmicos e sua autorização precisa ser transacionalmente coerente com o catálogo.

### 7. Catálogo global e persistência física por entidade

Cada entidade tem identidade estável e tabela física própria. Todas as organizações usam a mesma tabela, segregada por `organization_id`. Tabelas de grande volume poderão ser particionadas quando métricas justificarem.

Cada registro dinâmico inclui ao menos:

- UUID técnico;
- `organization_id`;
- versão do registro para concorrência otimista;
- informações de criação e última alteração;
- campos definidos no catálogo.

O catálogo armazena snapshots completos de cada versão. Deltas são derivados apenas para análise de impacto; não são a fonte de reconstrução. Entidades e campos possuem identificadores internos imutáveis, chaves lógicas estáveis e mapeamento físico independente de rótulos editáveis.

Alternativas consideradas: uma tabela JSONB única e tabela por organização. A primeira concentra volume e reduz as garantias físicas por campo; a segunda multiplica tabelas e migrations pelo número de clientes. A tabela por entidade mantém isolamento lógico e permite índices e constraints adequados ao domínio.

### 8. Ciclo de vida das versões

O ciclo global de uma versão é:

```text
DRAFT --> VALIDATED --> PUBLISHED --> ACTIVATING --> ACTIVE --> SUPERSEDED
  ^          |                            |
  +----------+                            +--> ACTIVATION_FAILED
   editou                                      |
                                               +--> retry ACTIVATING
```

- `DRAFT`: mutável e invisível ao runtime;
- `VALIDATED`: validação corresponde à revisão e ao hash atuais; editar retorna a `DRAFT`;
- `PUBLISHED`: snapshot imutável com número e hash, sem efeito no runtime;
- `ACTIVATING`: plano físico em execução;
- `ACTIVE`: única versão usada por todas as organizações;
- `SUPERSEDED`: versão ativa anterior preservada;
- `ACTIVATION_FAILED`: tentativa falhou e pode ser repetida com o mesmo plano.

Há no máximo um rascunho por entidade. Rascunhos têm UUID e revisão otimista, não consomem número. Na publicação recebem número inteiro sequencial por entidade e hash criptográfico. Nova evolução copia a versão ativa para um novo rascunho.

Publicação não executa DDL. Ativação é global, assíncrona, serializada por entidade, idempotente e auditada. A versão anterior permanece ativa até a conclusão e continua ativa se houver falha.

Alternativas consideradas: editar versões publicadas e publicar/ativar em uma única operação. Foram rejeitadas porque eliminariam reprodutibilidade, revisão do impacto e recuperação previsível.

### 9. Classificação e migração das mudanças

O comparador classifica alterações como:

- `COMPATIBLE`: metadado visual, descrição ou adição física compatível, como campo opcional;
- `CONDITIONAL`: exige inspeção ou transformação dos dados, como obrigatoriedade, unicidade ou mudança de tipo;
- `INCOMPATIBLE`: destrutiva ou sem conversão segura definida, como remoção física e redução de tipo.

O MVP ativa automaticamente apenas mudanças compatíveis. Mudanças condicionais exigem plano explícito, preenchimento e validação de todos os registros de todas as organizações. Alterações incompatíveis são bloqueadas.

O preenchimento inicial permite valor padrão fixo ou atualização em lote pela API REST. Scripts arbitrários não são aceitos. Endurecimento de validações ocorre em fases, por exemplo:

```text
add optional field --> backfill --> validate all rows --> make required
```

Mudança incompatível de tipo cria nova coluna, migra valores e descontinua a anterior. Colunas e tabelas publicadas não são removidas fisicamente no MVP.

Campos descontinuados deixam de aceitar escrita e de aparecer nos novos formulários, mas permanecem legíveis e exportáveis quando autorizado. Entidades descontinuadas não são habilitadas por novas organizações, continuam funcionando para organizações existentes e preservam dados e histórico. Desabilitação organizacional bloqueia novas operações sem apagar dados e pode ser revertida.

Alternativa considerada: permitir DDL livre e rollback destrutivo. Foi rejeitada porque rollback de estrutura não recupera dados e uma definição global amplia o impacto para todos os clientes.

### 10. Dependências entre entidades

Ativações são individuais no MVP. Uma relação nova só pode apontar para entidade globalmente ativa. Relacionamentos usam os identificadores estáveis e garantem igualdade de `organization_id`.

Relacionamento obrigatório exige migração prévia. Ciclos obrigatórios são bloqueados; ciclos opcionais podem existir quando todas as entidades já estão ativas. Não haverá pacote transacional de ativação de múltiplas entidades inicialmente.

Ao habilitar uma entidade, a organização recebe a lista de dependências e confirma a habilitação conjunta; dependências não são habilitadas silenciosamente. Desabilitar uma entidade requerida é bloqueado até que as dependentes sejam desabilitadas em conjunto.

Alternativa considerada: ativação atômica de um grafo de versões. Foi adiada pelo custo de coordenar DDL, validação e rollback entre várias tabelas.

### 11. Motor de schema dinâmico e Liquibase

Liquibase administra somente o schema estático versionado com o código: organizações, vínculos, perfis, catálogo, auditoria, sessões, conversas e registros operacionais. Em produção roda uma vez antes da nova aplicação, com credencial de migration separada da credencial normal de runtime.

Tabelas das entidades são administradas por um motor próprio. Ele gera um plano imutável e determinístico contendo versão de origem e destino, classificação, operações, precondições, pós-condições e hash. Administradores não editam o SQL gerado.

O motor mantém diário de execução com operação atual, tentativas, tempos, resultado, erro sanitizado, aprovador e verificação física. Um lock por entidade impede ativações concorrentes. Antes de executar, o motor compara catálogo e `pg_catalog`; drift bloqueia a ativação e gera auditoria.

No MVP, o caminho automático fica restrito a DDL seguro para a estratégia transacional adotada. Falha provoca rollback, mantém a versão anterior ativa e marca a nova tentativa como `ACTIVATION_FAILED`. Operações longas ou não transacionais exigirão um executor em fases posterior.

A versão exata do Liquibase não está fixada. A linha Community 5.x usa FSL com conversão futura para Apache 2.0; adoção ou atualização principal depende de revisão técnica e jurídica. Se a licença for incompatível, outra ferramenta de migration estática poderá substituir o Liquibase sem alterar o motor dinâmico.

Alternativa considerada: gerar changelogs Liquibase em runtime. Foi rejeitada porque as entidades não são conhecidas no build e possuem ciclo de aprovação, versionamento e recuperação próprio.

### 12. Corte de versão, concorrência e caches

`entityVersion` e `recordVersion` são controles distintos. A primeira identifica o contrato do schema; a segunda protege o registro contra perda de atualização.

Durante a ativação, o runtime continua usando a versão anterior. Após DDL e pós-condições, a troca do ponteiro ativo e a gravação do evento `EntityVersionActivated` ocorrem atomicamente. O evento persistido em outbox invalida caches, contrato OpenAPI, interfaces e ferramentas da IA.

Cada requisição captura a versão ativa no início. Leituras já iniciadas podem terminar. Escritas baseadas em schema obsoleto recebem `409 Schema Version Mismatch`; escritas baseadas em registro obsoleto recebem conflito de concorrência. A interface recarrega metadados preservando valores compatíveis. Aprovações pendentes da IA expiram se a versão da entidade ou do registro mudar.

Caches são indexados por IDs de versões imutáveis. Nenhum cache compartilha uma representação de rascunho mutável com o runtime.

Alternativa considerada: aceitar silenciosamente payloads antigos quando aparentarem compatibilidade. Foi rejeitada no MVP porque esconderia condições de corrida entre interface, API e migrations.

### 13. API REST dinâmica e OpenAPI

A API REST é a fronteira oficial para interfaces, integrações e IA. Ela resolve a entidade pela chave estável, valida a versão ativa, aplica contexto organizacional, autorização, validação e auditoria antes de chegar ao executor jOOQ.

O contrato inclui operações de metadados, cadastro, detalhe, alteração, exclusão e consulta. Filtros, ordenação, projeção e paginação usam uma DSL estruturada; nunca recebem SQL. Erros são machine-readable e distinguem validação, autorização, versão de schema, versão de registro e indisponibilidade de migration.

OpenAPI será derivado dos contratos estáveis e dos metadados publicados. Operações adequadas à IA recebem metadados de habilitação, risco, confirmação e limites. O conjunto apresentado à LLM é uma allowlist filtrada pelas permissões do usuário, não a especificação completa.

Mutações aceitam chave de idempotência e controle otimista. O formato exato das URLs e da DSL permanece um ponto de design detalhado antes da implementação da API.

Alternativa considerada: GraphQL como contrato principal. REST foi escolhido por previsibilidade operacional, OpenAPI e adequação como fronteira de ferramentas da IA.

### 14. Frontend dinâmico

O React opera como SPA atrás do BFF. TanStack Query gerencia estado remoto, React Hook Form gerencia formulários e TanStack Table sustenta listagens. Redux não é introduzido inicialmente.

JSON Schema 2020-12 representa estrutura e validação; um UI Schema separado representa layout, widget, visibilidade e apresentação. Um registro próprio mapeia tipos de campo para componentes MUI. O renderer monta formulário, detalhe, filtros e tabela em runtime; validação no cliente melhora a experiência, mas o backend continua autoritativo.

Alternativa considerada: gerador genérico de formulários como núcleo ou geração de código React. Foi rejeitada para manter controle da experiência, evolução do contrato e compatibilidade com permissões dinâmicas.

### 15. Auditoria append-only e resistente à adulteração

Eventos ficam inicialmente no PostgreSQL da aplicação. A API não oferece update/delete e a role de aplicação recebe somente a permissão necessária para inserir e consultar conforme o caso. Identidade, organização, alvo, operação, instante, resultado e correlação são campos estruturados.

A estrutura reservará sequência, hash do evento e hash anterior para permitir encadeamento e verificação. O algoritmo, granularidade das cadeias e política de checkpoints serão definidos antes da implementação. A verificação gera evento e alerta quando encontra alteração, lacuna ou quebra de encadeamento.

Eventos globais e organizacionais têm visibilidade distinta. Conversas da IA mantêm retenção própria; a auditoria guarda correlação e hash/referência, evitando copiar indiscriminadamente prompts e respostas sensíveis.

A proteção no mesmo banco detecta e dificulta adulteração pela aplicação, mas não é WORM e não protege plenamente contra DBA/superusuário, especialmente para truncamento da cauda. Uma âncora externa poderá ser adicionada sem mudar o formato lógico dos eventos.

Alternativa considerada: armazenamento externo imutável desde o MVP. Foi adiada para reduzir topologia e custo operacional, aceitando explicitamente a limitação inicial.

### 16. IA conversacional como cliente REST

O módulo de IA mantém conversas, mensagens, execuções, tool calls e aprovações em tabelas organizacionais próprias. Uma abstração de modelo impede tipos de um fornecedor de atravessarem o módulo; Spring AI é o candidato natural no ecossistema Java, mas a dependência e versão serão confirmadas junto com o provedor.

A LLM nunca recebe conexão com o banco, SQL livre ou cliente HTTP arbitrário. Ela solicita ferramentas tipadas. O orquestrador valida o JSON Schema, risco e autorização, e um cliente interno chama a API REST da ENT.IA.

O contexto confiável de usuário e organização vem da sessão, fora do prompt. Se o orquestrador for implantado separadamente, usa token delegado e limitado à audience da API, preservando sujeito usuário e identidade do executor; não usa somente conta de serviço ampla.

Consultas autorizadas podem executar automaticamente com minimização de campos e linhas. Criação, alteração e exclusão produzem pré-visualização e exigem confirmação humana. Aprovação tem prazo, hash do plano, idempotência, `entityVersion` e `recordVersion` esperadas. Apenas após confirmação ocorre a chamada REST de mutação.

Entidades e campos suportarão política de IA `allowed`, `masked` ou `forbidden`. Prompt e resultados de ferramentas são entradas não confiáveis. O orquestrador limita turnos, tool calls, tempo, tokens e volume de dados, e nunca informa sucesso antes da resposta positiva da API.

RAG, embeddings, banco vetorial, MCP e agentes autônomos não fazem parte da primeira versão. Os contratos de ferramentas poderão ganhar adaptador MCP futuramente sem substituir a API REST interna.

Alternativa considerada: permitir que o provedor de LLM chame diretamente a API ou gere SQL. Foi rejeitada por ampliar risco de prompt injection, vazamento entre organizações e excesso de autonomia.

### 17. Assincronia, outbox e observabilidade

Não haverá Kafka inicialmente. Ativações, preenchimentos e trabalhos longos usam fila persistida no PostgreSQL. Eventos que precisam sobreviver à transação usam outbox com consumidor idempotente.

Toda requisição recebe correlation ID propagado por módulos, chamadas REST internas, jobs, auditoria e IA. Métricas não incluem conteúdo sensível por padrão. Logs estruturados evitam tokens, segredos, prompts completos e valores de campos classificados.

Alternativa considerada: broker externo desde a fundação. Foi adiada porque o volume ainda não é conhecido e PostgreSQL já é uma dependência operacional obrigatória.

## Risks / Trade-offs

- [Falha de escopo organizacional expõe dados] -> centralizar contexto confiável, exigir `organization_id`, testar tentativas cruzadas e decidir RLS antes do runtime produtivo.
- [Tabela por entidade aumenta o número de objetos no PostgreSQL] -> impor limites, monitorar catálogo e particionar apenas mediante métricas.
- [Uma versão global pode ser bloqueada pelos dados de uma organização] -> relatório por organização, backfill explícito e ativação somente após conformidade total.
- [DDL bloqueia ou degrada tabelas grandes] -> classificar operações, medir impacto, executar em fases e exigir janela quando não houver técnica online segura.
- [Motor de schema próprio é uma área de alta complexidade] -> vocabulário fechado de operações, planos imutáveis, pre/pós-condições, testes reais e diário idempotente.
- [Drift por intervenção manual quebra o catálogo] -> credenciais separadas, comparação com `pg_catalog`, bloqueio e correção administrada.
- [Schema obsoleto causa perda de atualização] -> versionar schema e registro, rejeitar com conflito e invalidar caches por outbox.
- [Um realm compartilhado amplia colisões e roteamento de identidade] -> identidade por subject estável, vínculo controlado e política de descoberta ainda a definir.
- [Exposição pública do Keycloak amplia ataque] -> WAF, rotas mínimas, console interno, proxies confiáveis, TLS, rate limiting e BFF.
- [Auditoria no mesmo banco não é WORM] -> append-only, privilégios mínimos, encadeamento verificável e opção futura de âncora externa.
- [LLM pode sofrer prompt injection ou solicitar ação excessiva] -> allowlist de ferramentas, contexto fora do prompt, confirmação humana, limites e autorização repetida na API.
- [Dados sensíveis podem sair para um provedor de LLM] -> classificação por campo, minimização, mascaramento e seleção do provedor condicionada a requisitos de privacidade.
- [REST interno adiciona latência e autenticação ao orquestrador] -> cliente gerado/validado, conexão privada e fronteira que permite medir e extrair o componente.
- [Licença do Liquibase pode ser incompatível com o produto] -> revisão antes de fixar versão e possibilidade de substituição restrita ao migrador estático.

## Migration Plan

Como o projeto é novo, o plano é de implantação incremental, não de migração de um sistema legado:

1. provisionar PostgreSQL da aplicação e armazenamento logicamente separado para Keycloak;
2. aplicar o schema estático inicial por migration controlada, com credencial exclusiva;
3. configurar realm `ent-ia`, clientes BFF/API e um provedor padrão;
4. implantar proxy/WAF com frontchannel mínimo e administração interna;
5. disponibilizar o monólito com módulos de identidade, tenancy, autorização, catálogo e auditoria;
6. validar o motor de schema em uma entidade piloto, cobrindo publicação, falha, retry e drift;
7. habilitar runtime REST e frontend dinâmico somente após testes de isolamento e concorrência;
8. introduzir o orquestrador de IA sem ferramentas de escrita; habilitar consultas após testes de minimização e autorização;
9. habilitar mutações pela IA somente após aprovação, idempotência e auditoria estarem verificadas;
10. expandir capacidades e limites com métricas reais.

Rollback de aplicação deve preservar compatibilidade com o schema estático já aplicado. Ativação dinâmica falha antes da troca do ponteiro e mantém a versão anterior. Estruturas e dados publicados não são removidos automaticamente; restauração de backup é último recurso operacional, não mecanismo normal de rollback.

## Open Questions

### Antes de decompor o trabalho de implementação

- Qual será a convenção exata e imutável para nomes físicos de tabelas, colunas, índices e constraints?
- PostgreSQL Row-Level Security será obrigatório no MVP, complementar aos filtros da aplicação, ou ficará preparado para uma fase posterior?
- Quais cardinalidades entram no MVP e qual é o comportamento padrão de exclusão: `RESTRICT`, `SET NULL` ou outro?
- Exclusão de registros de negócio será física, lógica ou configurável por entidade?
- Qual será o formato exato das URLs REST, DSL de filtros, paginação, ordenação, projeção e exportação?
- Qual algoritmo e granularidade de encadeamento serão usados na auditoria, e como será tratada a retenção?
- Como a organização será descoberta antes do login e como conflitos de e-mail entre provedores serão resolvidos?
- Quais limites iniciais de entidades, campos, registros, payload, consulta e tempo de resposta definem o MVP?

### Definíveis durante o planejamento de implantação ou de fases posteriores

- Qual ambiente de execução e topologia serão usados para containers, proxy, jobs, observabilidade, backup e disaster recovery?
- Qual versão do Liquibase é compatível com a política jurídica, de distribuição e de hospedagem da ENT.IA?
- Qual provedor e modelo de LLM atendem tool calling, privacidade, residência, latência e custo?
- O provedor de LLM será global ou poderá ser substituído por organização?
- Qual será a política de retenção, exportação e exclusão das conversas?
- Quais limites de tokens, custo, duração, chamadas de ferramentas e operações em lote serão adotados?
- Em que fase RAG, embeddings, pgvector, MCP ou agentes especializados passam a ter benefício comprovado?
