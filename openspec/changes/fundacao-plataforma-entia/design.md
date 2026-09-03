## Context

A ENT.IA é um projeto greenfield. A motivação e o escopo funcional estão em `proposal.md`; os contratos de comportamento estão nas nove especificações desta mudança.

As principais restrições que moldam a arquitetura são:

- múltiplas organizações compartilham uma instalação, um banco PostgreSQL e um schema de aplicação;
- dados organizacionais são segregados por `organization_id`;
- definições de entidades são globais, compartilhadas e possuem uma única versão ativa;
- cada entidade dinâmica possui tabela física própria, compartilhada entre organizações;
- autenticação é delegada ao Keycloak e autorização de negócio pertence à ENT.IA;
- o Nginx Open Source é o servidor web e proxy reverso da borda pública;
- a implantação é agnóstica de provedor, usa containers OCI e começa com Docker Compose em camadas físicas separadas;
- PostgreSQL 18.x admite operação gerenciada ou self-hosted sob o mesmo contrato técnico;
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

O núcleo será um monólito modular Java. Nginx, Keycloak, PostgreSQL e o provedor futuro de LLM são dependências externas. O orquestrador de IA terá uma fronteira que permita isolamento ou implantação separada, mas não força a decomposição prematura do restante da plataforma.

```text
+---------+       +--------------------+
| Browser |------>| Nginx              |
+---------+       | public edge        |
                  +---------+----------+
                            |
              +-------------+------------------+
              |             |                  |
              v             v                  v
        React SPA      Spring Boot          Keycloak
        static files   modular monolith     auth frontchannel
                            |                  ^
                            |                  |
                            +------------------+
                            | OIDC backchannel
                            |
              +-------------+-------------+
              |                           |
              v                           v
        PostgreSQL                  AI orchestrator
        shared                             |
                                           v
                                      LLM provider
```

Alternativa considerada: microsserviços por capacidade. Foi rejeitada para a fundação por aumentar coordenação, observabilidade, consistência distribuída e custo operacional antes de existir demanda que justifique a separação.

### 2. Stack base

- OpenJDK 25 LTS;
- Spring Boot 4.1;
- Spring Modulith para limites, eventos e testes entre módulos;
- Spring Security para BFF, OIDC e proteção das APIs;
- jOOQ para SQL estático e construção segura das operações dinâmicas;
- PostgreSQL 18.x como banco principal;
- Nginx Open Source como servidor web e proxy reverso de borda;
- containers OCI e Docker Compose para a implantação inicial;
- OpenTofu para Infrastructure as Code, com módulos específicos por ambiente;
- Liquibase para schema estático, sujeito à revisão de licença da versão adotada;
- Maven para build;
- Testcontainers para testes de integração com infraestrutura real;
- React 19, TypeScript e Vite no frontend;
- MUI Core, TanStack Table, TanStack Query e React Hook Form;
- Vitest, React Testing Library e Playwright para testes do frontend.

Alternativas consideradas: NoSQL como armazenamento primário e uma arquitetura frontend com SSR. PostgreSQL foi escolhido pela integridade relacional e capacidade de DDL, enquanto a aplicação autenticada e intensiva em interação não necessita de SSR inicialmente.

### 2.1 Topologia inicial, portabilidade e PostgreSQL

A arquitetura de implantação será agnóstica de provedor. Imagens, portas, configuração externa, health checks e contratos de rede permanecerão portáveis; recursos de infraestrutura específicos de nuvem ou on-premises serão encapsulados em módulos OpenTofu próprios. Kubernetes não será adotado inicialmente, mas os containers permanecerão compatíveis com sua adoção futura.

Em produção, borda, aplicação e dados ficarão em camadas fisicamente separadas. A implantação inicial usará projetos Docker Compose independentes por host, pois Compose não será tratado como um orquestrador multi-host:

```text
+-------------------+       +----------------------+       +-------------------+
| Edge host         |------>| Application host     |------>| Data layer        |
| Nginx             |       | Spring Boot          |       | PostgreSQL 18.x   |
| Docker Compose    |       | Keycloak             |       | managed or OCI    |
+-------------------+       | Docker Compose       |       +-------------------+
                            +----------------------+
```

O PostgreSQL seguirá uma política híbrida. A implantação de referência será self-hosted em container OCI no host de dados; ambientes que ofereçam um PostgreSQL gerenciado compatível poderão usá-lo, com preferência para produção em nuvem. A aplicação não exigirá extensões ou interfaces proprietárias de provedor.

Inicialmente haverá um único serviço ou cluster PostgreSQL 18.x com dois databases: `entia`, para a aplicação, e `keycloak`, exclusivo do provedor de identidade. Todas as organizações continuarão compartilhando o database e o schema de aplicação do `entia`, segregadas por `organization_id`. Os databases terão proprietários e credenciais independentes, sem acesso cruzado.

As credenciais do `entia` serão separadas por responsabilidade ao menos entre: proprietário técnico sem uso cotidiano, migrations estáticas do Liquibase, motor controlado de schema dinâmico, runtime da aplicação, ingestão em massa e escrita de auditoria. O Keycloak usará credencial própria limitada ao database `keycloak`. Nenhum processo normal da aplicação usará superusuário do cluster.

Alternativas consideradas: produção em host único, Kubernetes desde a fundação e dependência obrigatória de banco gerenciado. Foram rejeitadas, respectivamente, por concentrar falhas e exposição, introduzir orquestração prematura e contrariar a portabilidade on-premises.

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

Row-Level Security (RLS) será obrigatória desde a fundação em todas as tabelas persistentes organizacionais, estáticas ou dinâmicas, acessadas pelos fluxos regulares. O backend continuará incluindo o escopo organizacional nas consultas e aplicando RBAC; RLS será uma segunda barreira no banco, não um substituto desses controles. As políticas usarão o contexto confiável definido pela aplicação no início de cada transação tanto para leitura (`USING`) quanto para escrita (`WITH CHECK`). Ausência de contexto válido resultará em negação.

A role de runtime não será proprietária das tabelas, não terá `BYPASSRLS` e não usará superusuário. As tabelas organizacionais terão RLS habilitada e forçada; migrations, motor de schema e rotinas administrativas usarão credenciais separadas, sem compartilhar seu privilégio com caminhos de requisição. Jobs percorrerão organizações explicitamente e abrirão uma transação com contexto restrito para cada unidade de trabalho.

Registros dinâmicos usam UUIDv7 global, armazenado no tipo nativo `uuid`, como identificador técnico imutável. O backend ou importador confiável gera o UUIDv7 antes da persistência; `uuidv7()` do PostgreSQL 18.x pode ser mantido como default de segurança. A chave primária física será composta por `(organization_id, id)`, mesmo com a unicidade prática global do UUIDv7, para tornar o isolamento parte da integridade referencial. Identificadores de negócio continuam separados e podem possuir formato e unicidade próprios. Restrições únicas de negócio incluem `organization_id`, e relacionamentos entre registros não atravessam organizações. O formato lógico é equivalente a:

```text
PRIMARY KEY (organization_id, id)

UNIQUE (organization_id, business_key)

(organization_id, referenced_id)
    --> target (organization_id, id)
```

Definições e versões do catálogo não possuem proprietário organizacional. A relação `organization_entity` controla somente habilitação local; permissões continuam independentes.

Sequences ou colunas identity com `BIGINT` ficam reservadas a numeração local, versões e posições técnicas que não precisem ser globalmente únicas nem conhecidas antes da gravação. Não haverá, por padrão, a duplicação de uma chave `BIGINT` e outra UUID em todas as tabelas dinâmicas. Se uma numeração de negócio exigir ausência legal de lacunas, ela precisará de mecanismo transacional próprio, pois uma sequence comum pode apresentar lacunas após rollback ou falha.

Alternativas consideradas: database/schema por organização, definição personalizada por organização, UUIDv4 aleatório como padrão e duas chaves técnicas em todo registro. Foram rejeitadas, respectivamente, para reduzir custo operacional, divergência de versões, dispersão dos índices em cargas intensas e duplicação de armazenamento, índices e complexidade.

### 5. Identidade, Keycloak e sessão BFF

O Keycloak self-hosted será o provedor padrão e broker de identidades. Haverá um realm `ent-ia`, identidades globais e vínculos organizacionais mantidos pela aplicação. Cada organização poderá associar um provedor OIDC ou SAML; quem não possuir um usará o provedor padrão, incluindo login social habilitado.

O Keycloak ficará em rede interna. O Nginx publicará em `auth.ent.ia.br` somente o frontchannel necessário, incluindo `/realms/ent-ia/*`, `/resources/*` e `/.well-known/*`, após validação final das rotas. Console administrativo, Admin REST API, realm `master`, health, métricas e porta de gerenciamento permanecerão internos ou sob VPN.

O Nginx também servirá os artefatos estáticos do React, encaminhará as rotas do BFF/API ao Spring Boot, sobrescreverá de forma segura os cabeçalhos encaminhados e aplicará limites de requisições, conexões, payload e tempo. Os backends não aceitarão acesso público direto. A eventual adoção de WAF será uma camada adicional de defesa, sem substituir autenticação, autorização, validação ou isolamento organizacional na aplicação.

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

- UUIDv7 técnico;
- `organization_id`;
- versão do registro para concorrência otimista;
- informações de criação e última alteração;
- marcador de exclusão lógica;
- campos definidos no catálogo.

O catálogo armazena snapshots completos de cada versão. Deltas são derivados apenas para análise de impacto; não são a fonte de reconstrução. Entidades e campos possuem identificadores internos imutáveis, chaves lógicas estáveis e mapeamento físico independente de rótulos editáveis.

Alternativas consideradas: uma tabela JSONB única e tabela por organização. A primeira concentra volume e reduz as garantias físicas por campo; a segunda multiplica tabelas e migrations pelo número de clientes. A tabela por entidade mantém isolamento lógico e permite índices e constraints adequados ao domínio.

### 7.1 Identidades lógicas e nomes físicos

Cada entidade e campo possui três representações distintas:

- nome de exibição, livre, traduzível e alterável;
- chave lógica canônica em `camelCase`, estável após a publicação;
- nome físico em `snake_case`, gerado, registrado no catálogo e imutável após a publicação.

A mesma chave lógica será usada no catálogo, JSON, REST e OpenAPI. O OpenAPI poderá derivar nomes técnicos de schemas a partir dela, mas não introduzirá uma segunda chave canônica em `PascalCase` ou outro formato. `flatcase` não será usado, pois elimina a separação visual entre palavras.

Somente tabelas de entidades dinâmicas recebem o prefixo `en_`; tabelas globais e estáticas da plataforma não recebem esse prefixo, pois ele identifica exclusivamente entidades registradas no catálogo. O nome normal de uma tabela dinâmica segue `en_<chave_em_snake_case>`, por exemplo `en_pedido_compra`.

Campos estruturais da plataforma preservam nomes convencionais e explícitos, como `organization_id`, `id`, `record_version`, `created_at`, `updated_at` e `deleted_at`. Campos definidos pelo catálogo usam um prefixo semântico de exatamente dois caracteres, seguido do conceito em `snake_case`. O vocabulário inicial é:

- `id`: identificação;
- `dt`: data;
- `nr`: número;
- `nm`: nome;
- `ds`: descrição;
- `vl`: valor.

Termos excessivamente longos serão rejeitados na validação com sugestão de abreviação controlada. A abreviação escolhida passa a integrar o mapeamento persistido; o motor nunca trunca identificadores silenciosamente.

Constraints e índices seguem os prefixos `pk_`, `fk_`, `uq_`, `ck_` e `ix_`, respectivamente. O restante do nome identifica de forma legível a tabela e o campo, relação ou finalidade. Toda chave estrangeira dinâmica terá índice correspondente iniciado por `organization_id`.

O nome legível, sem sufixo, é sempre tentado primeiro. Se dois objetos legitimamente registrados no catálogo resultarem no mesmo identificador físico, o segundo recebe `_` e um sufixo determinístico de exatamente três caracteres alfanuméricos minúsculos em base 36, derivado do UUID interno imutável do objeto. Novas tentativas determinísticas resolvem a improvável segunda colisão. O sufixo também pode ser aplicado a constraints e índices quando necessário para unicidade, sempre dentro do limite do PostgreSQL. O mapeamento final é persistido antes da ativação; não são usados contadores como `_1` ou `_2`.

Uma tabela física existente sem o respectivo registro no catálogo caracteriza drift. A ativação é bloqueada para investigação, em vez de criar silenciosamente outro nome com sufixo.

### 7.2 Inserções em lote e importações massivas

Lotes regulares recebidos pelas APIs usarão `INSERT` multi-row ou batch JDBC, sempre sob contexto organizacional, autorização, RLS, validações, constraints e auditoria. O UUIDv7 será gerado antecipadamente pelo backend, permitindo que o resultado e os relacionamentos sejam conhecidos sem recuperar uma sequence depois de cada inserção.

Importações de alto volume usarão uma área de staging controlada. O componente de ingestão do módulo `dynamic-records` criará um `import_id`, fixará a organização a partir do contexto confiável, manterá a chave de origem de cada linha, gerará os UUIDv7 e carregará a staging com `COPY FROM STDIN`. Depois de validar tipos, duplicidades, permissões e relacionamentos, moverá os dados com operações SQL em conjunto para as tabelas finais protegidas por RLS. A chave de origem permitirá mapear referências entre registros do mesmo arquivo para os UUIDs definitivos e sustentar retries idempotentes.

O PostgreSQL 18.x não permite `COPY FROM` diretamente em tabela com RLS. Por isso, a staging transitória será uma exceção controlada à política das tabelas persistentes e não será uma rota alternativa para o runtime: ficará em schema ou estrutura isolada, sem acesso público, sob role dedicada e com escopo preso a `import_id` e `organization_id`. Somente o serviço confiável de ingestão poderá promover dados para as tabelas finais; a promoção usará uma role submetida ao RLS e um contexto organizacional fixado pelo backend. O importador não aceitará tabela-alvo, SQL ou organização arbitrários do cliente. Resultado, erros e correlação da importação serão auditados.

O tamanho dos lotes, atomicidade por arquivo ou bloco, retenção da staging e índices físicos serão definidos com benchmark representativo antes da implementação. O teste deverá cobrir entidades relacionadas, constraints, RLS, WAL, auditoria e os volumes esperados, sem assumir que UUIDv7 e `BIGINT` tenham custo equivalente.

Alternativas consideradas: usar sequence para todos os IDs, inserir linha a linha para recuperar chaves e executar `COPY` direto nas tabelas finais. Foram rejeitadas porque dificultam o mapeamento antecipado de relações, reduzem paralelismo ou contornam o caminho de segurança exigido pelo RLS.

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

O relacionamento físico fundamental é muitos-para-um (`N:1`): a entidade de origem armazena a FK composta para a entidade de destino. A FK aceita `NULL` quando a relação é opcional e recebe `NOT NULL` quando é obrigatória. Um-para-muitos (`1:N`) é somente a visão inversa dessa mesma FK, consultada e apresentada a partir dos metadados; não cria coluna, constraint ou estado duplicado na entidade de destino.

Um-para-um (`1:1`) usa a mesma estrutura de FK na entidade dependente e acrescenta unicidade sobre `(organization_id, <campo_de_referencia>)`. O catálogo exige que um lado seja definido como origem e não gera FKs obrigatórias nas duas direções.

Muitos-para-muitos (`N:N`) é representado por uma entidade associativa de primeira classe contendo duas relações `N:1`. Ela segue o mesmo ciclo de versão, tabela física, RLS, autorização, API e auditoria das demais entidades e pode receber atributos próprios, como função, período ou situação. Um assistente poderá criá-la e ocultá-la da navegação principal, mas não haverá tabela de junção sem identidade ou metadado correspondente no catálogo.

Autorreferência será aceita no MVP somente como relação opcional comum. Ela não implica hierarquia, árvore, prevenção genérica de ciclos nem consultas de ancestrais e descendentes. Uma semântica especializada de hierarquia será introduzida apenas com regras explícitas de aciclicidade, profundidade e apresentação.

O modelo de relação reserva a distinção entre `REFERENCE` e `COMPOSITION`, mas somente `REFERENCE` poderá ser ativada no MVP. Entidades relacionadas por referência têm ciclos de vida independentes. `COMPOSITION`, contagem mínima de filhos e cascata lógica automática ficam adiadas até existirem planejamento de impacto, autorização, auditoria por registro, idempotência e restauração conjunta.

O seletor de uma nova referência mostra somente alvos ativos. Uma alteração que não modifica uma referência histórica para alvo logicamente excluído continua permitida; criar ou substituir a relação exige um alvo ativo. Em relações `1:1`, a unicidade continua reservada enquanto a linha dependente estiver logicamente excluída, favorecendo restauração em vez de substituição silenciosa.

Ao habilitar uma entidade, a organização recebe a lista de dependências e confirma a habilitação conjunta; dependências não são habilitadas silenciosamente. Desabilitar uma entidade requerida é bloqueado até que as dependentes sejam desabilitadas em conjunto.

Alternativas consideradas: ativação atômica de um grafo de versões e tabela de junção oculta para `N:N`. Foram adiadas ou rejeitadas pelo custo de coordenar DDL, validação e rollback entre várias tabelas e porque a estrutura oculta exigiria caminhos especiais para metadados, API, autorização, auditoria e evolução.

### 10.1 Exclusão lógica e integridade referencial

O `DELETE` da API dinâmica realiza exclusão lógica: define `deleted_at`, incrementa `record_version` e produz um evento de auditoria com ator, organização, entidade, registro, resultado e correlação. O ator não precisa ser duplicado na linha de negócio porque pertence à trilha de auditoria. Consultas e listagens normais omitem registros excluídos; operações históricas ou administrativas explicitamente autorizadas podem incluí-los.

A exclusão lógica não aciona ações `ON DELETE` do PostgreSQL, pois a linha permanece fisicamente armazenada. Relacionamentos históricos existentes são preservados e podem resolver o registro excluído nos fluxos autorizados, mas novas referências a ele são rejeitadas. Restrições únicas continuam considerando o registro excluído; sua chave de negócio permanece reservada, e a restauração controlada é o caminho padrão para reutilizá-la.

Todas as chaves estrangeiras geradas entre entidades dinâmicas usam `ON DELETE RESTRICT` e `ON UPDATE RESTRICT`, com a organização incluída nos dois lados:

```sql
FOREIGN KEY (organization_id, id_cliente)
    REFERENCES en_cliente (organization_id, id)
    ON DELETE RESTRICT
    ON UPDATE RESTRICT
```

O catálogo não permitirá configurar `CASCADE`, `SET NULL`, `SET DEFAULT` ou `NO ACTION` por relação no MVP. Isso impede efeitos destrutivos ou alterações implícitas em grafos definidos dinamicamente. Quando a semântica de composição for introduzida, uma eventual cascata lógica será planejada, apresentada, autorizada e auditada pela aplicação, não delegada implicitamente à chave estrangeira.

Exclusão física não será exposta pela API regular, pelas ferramentas da IA nem pelo runtime comum. Um futuro processo administrativo de expurgo deverá respeitar retenção, apresentar impacto e dependências, executar remoções em ordem explícita e permanecer auditado. `ON DELETE CASCADE` poderá existir somente em tabelas estáticas e técnicas cuja dependência e ciclo de vida sejam integralmente controlados pela plataforma, após decisão explícita no schema estático. Organizações e definições são desativadas ou descontinuadas, e eventos de auditoria nunca são removidos em cascata.

Alternativas consideradas: exclusão física como operação comum, ação referencial configurável por entidade e `CASCADE` como padrão. Foram rejeitadas porque reduzem a previsibilidade, dificultam a pré-visualização e a auditoria por registro e podem ampliar silenciosamente uma exclusão em um grafo dinâmico.

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

O contrato público começa em `/api/v1` e separa as áreas `/session`, `/catalog` e `/entities`. Os registros dinâmicos são endereçados pela chave lógica estável da entidade em `camelCase` e pelo UUIDv7 do registro. As operações fundamentais seguem o formato:

```text
GET    /api/v1/entities/{entityKey}/records
POST   /api/v1/entities/{entityKey}/records
POST   /api/v1/entities/{entityKey}/records/query
GET    /api/v1/entities/{entityKey}/records/{recordId}
PATCH  /api/v1/entities/{entityKey}/records/{recordId}
DELETE /api/v1/entities/{entityKey}/records/{recordId}
```

`GET` é reservado a leituras de detalhe e coleções; `POST` cria recursos, executa a consulta composta somente leitura e inicia comandos de domínio ou jobs; `PATCH`, com `application/merge-patch+json`, altera parcialmente; e `DELETE` solicita a exclusão lógica do registro. Restauração, validação, publicação e ativação são comandos explícitos com `POST`, não alterações artificiais de campos protegidos. `HEAD` e `OPTIONS` são suporte HTTP tratado pela infraestrutura e pelo framework. Substituição integral por `PUT` não integra o MVP porque campos omitidos poderiam ser apagados durante a evolução do schema. `TRACE` será desabilitado e `CONNECT` não será exposto pela API.

O contexto normal da organização vem exclusivamente da sessão ou do token confiável. `organization_id` não é aceito como autoridade em path, payload ou cabeçalho público. Endpoints administrativos excepcionais que operem sobre outra organização terão contrato, permissão e auditoria explícitos, separados do runtime comum.

Representações de um registro não usam envelope `data`: `id`, os campos dinâmicos autorizados e o objeto reservado `meta` ficam na raiz. `meta` concentra metadados de controle, como versões, e não pode ser definido como campo de negócio. Criações usam JSON plano; alterações parciais usam JSON Merge Patch. Valores decimais são transportados como string para preservar precisão. Uma referência é representada por `{ "id": "...", "label": "...", "deleted": false }`, sendo somente `id` autoritativo; `label` é uma projeção conveniente derivada do template de exibição da entidade, e `deleted` sinaliza uma referência histórica autorizada. Relações inversas são consultadas em recursos próprios e não são embutidas como coleções duplicadas no registro.

Coleções usam `items` e `meta`, também sem envelope `data`. Erros usam `application/problem+json` e extensões estruturadas para detalhes de validação, autorização, versão de schema, versão de registro e indisponibilidade de migration.

O `GET .../records` atende a listagem padrão com os parâmetros simples e reservados `select`, `limit` e um entre `after` ou `before`. O `select` é repetível na URL. Filtros ou ordenações personalizados usam `POST .../records/query`, que é idempotente em comportamento, não cria recurso e retorna a mesma representação de coleção do `GET`. Sua DSL JSON separa projeção (`select`), filtros (`filter`), ordenação (`order`) e paginação (`page`). Não será usado corpo em requisições `GET`, nem haverá duas sintaxes concorrentes para filtros compostos.

A árvore de filtros possui grupos com `logic` igual a `and` ou `or` e uma lista recursiva de `conditions`; uma condição terminal possui `field`, `operator` e, quando aplicável, `value`. O vocabulário inicial inclui `eq`, `ne`, `in`, `notIn`, `isNull`, `isNotNull`, `gt`, `gte`, `lt`, `lte`, `contains`, `startsWith`, `like` e `ilike`, limitado por tipo de campo. Relacionamentos são comparados pelo UUID do alvo, sem travessia. A DSL aceita somente campos, operadores e combinações publicados pelo catálogo e autorizados para o consumidor; não aceita SQL, funções arbitrárias, expressões regulares, scripts nem travessia arbitrária de relacionamentos.

`like` preserva distinção entre maiúsculas e minúsculas, enquanto `ilike` faz comparação sem essa distinção conforme a semântica validada do PostgreSQL. Ambos aceitam `%` para qualquer sequência e `_` para um caractere, com regra explícita de escape para uso literal. Esses operadores ficam disponíveis apenas em campo textual habilitado para pesquisa por padrão. O padrão terá até 256 caracteres e oito curingas; um curinga inicial exigirá ao menos três caracteres literais consecutivos e índice compatível, como uma estratégia GIN/GiST com `pg_trgm`. Valores continuam parametrizados pelo jOOQ.

A consulta síncrona admite árvore com até cinco níveis, 25 condições terminais, 100 valores por `in` ou `notIn`, 50 campos em `select`, três campos explícitos em `order`, corpo de até 64 KiB e página de 50 registros por padrão, limitada a 200. O executor aplica inicialmente `statement_timeout` de cinco segundos. A validação estrutural acontece antes do acesso ao PostgreSQL; relatórios, exportações e consultas que não caibam nesse perfil serão executados futuramente como jobs assíncronos, não como consultas interativas sem limite.

Cada versão publicada da entidade possui projeções padrão distintas para lista e detalhe. Se `select` for omitido, o endpoint usa a projeção correspondente; se for informado, retorna exatamente os campos de negócio solicitados e autorizados, além de `id` e `meta`, que são sempre presentes. Campos grandes, sensíveis ou descontinuados podem ficar fora das projeções padrão e exigir seleção explícita quando sua leitura for permitida. Campo desconhecido ou explicitamente não autorizado gera erro, sem omissão silenciosa. Não há `select: ["*"]`, aliases, agregações, expressões calculadas ou seleção aninhada de relacionamentos no MVP. Campos usados em filtro ou ordenação não precisam integrar a projeção, mas continuam sujeitos às mesmas permissões.

`order` é uma lista ordenada de objetos com `field`, `direction` (`asc` ou `desc`) e `nulls` opcional (`first` ou `last`). A posição define a prioridade, `nulls` assume `last` quando omitido e não há funções, expressões nem collation escolhida por requisição. Se `order` for omitido, aplica-se a ordenação padrão versionada da entidade; sem uma definição explícita, o fallback é `id desc`. O UUIDv7 é acrescentado automaticamente como desempate final na mesma direção da última chave solicitada. Relacionamentos, textos extensos, JSON, coleções, binários e campos calculados sem persistência não são ordenáveis no MVP.

Cada campo publicado declara separadamente as capacidades `selectable`, `filterable`, `sortable` e `patternSearch`, sempre subordinadas às permissões efetivas. Toda tabela dinâmica possui um índice base para registros ativos iniciado por `(organization_id, id)`. Um campo habilitado para ordenação simples exige índice compatível; combinações de duas ou três chaves somente são aceitas quando correspondem a um perfil de consulta publicado com índice composto, iniciado por `organization_id` e terminado pelo UUIDv7. Não serão criadas automaticamente todas as combinações possíveis de índices. Perfis de ordenação e busca por padrão só entram em vigor depois que seus índices forem criados e validados pelo ciclo de ativação.

A paginação padrão de registros dinâmicos usa keyset/seek pagination encapsulada em cursor opaco stateless. A requisição informa `limit` e, de forma mutuamente exclusiva, `after` ou `before`. A resposta informa em `meta.page` os cursores `nextCursor` e `previousCursor`, além de `hasNext` e `hasPrevious` quando aplicáveis. Não há salto arbitrário para número de página, e a contagem exata total não é executada implicitamente; quando necessária e autorizada, será solicitada de forma explícita.

Internamente, o cursor contém de forma compacta a direção, expiração, valores que delimitam a ordenação e o UUIDv7 como desempate determinístico. Para a ordenação padrão, o UUIDv7 pode ser o único valor de posição. Organização, sujeito autenticado, entidade, versão publicada, projeção, filtros e ordenação não são copiados integralmente: formam um fingerprint criptográfico de 16 bytes recalculado em cada requisição. O `limit` fica fora desse fingerprint para poder variar dentro do máximo permitido. Toda nova página revalida sessão, organização e permissões, porque o cursor comunica continuação, não autoridade.

O backend não persiste uma linha ou sessão para cada cursor. As instâncias compartilham somente o material criptográfico versionado necessário para emitir e validar os tokens. O cursor usa serialização binária compacta, criptografia autenticada e Base64 URL-safe; CBOR é uma opção, sem tornar seu uso obrigatório antes da implementação. Sua meta operacional é até 512 caracteres e o limite absoluto é 1.024. Os perfis de ordenação são validados pelo pior tamanho possível de suas posições, impedindo que combinações incompatíveis sejam publicadas. Gzip não integra o formato do MVP: os dados binários remanescentes têm baixa redundância, o envelope pode aumentar tokens pequenos e a compressão acrescentaria complexidade e riscos sem ganho demonstrado.

Cada cursor vale 30 minutos de forma absoluta. Não existe endpoint de renovação, mas cada página obtida com sucesso emite novos cursores com uma nova janela de 30 minutos. A troca de sessão continua possível quando o sujeito interno permanece o mesmo, porém troca de usuário ou organização invalida o token. Perda de permissão bloqueia a continuação; uma mudança que preserve todas as permissões necessárias não invalida o cursor. A ativação de outra versão da entidade torna o cursor obsoleto e exige reiniciar a consulta. Tokens malformados ou incompatíveis, expirados e obsoletos são distinguidos em respostas `application/problem+json`.

O cliente mantém temporariamente os cursores necessários à navegação, por exemplo no estado do TanStack Query, e reenvia a consulta original em cada chamada para que o backend recalcule o fingerprint. O cursor aceitará no máximo três campos explícitos de ordenação, além do UUIDv7 acrescentado automaticamente. Textos extensos, JSON e outras estruturas que tornem o token imprevisível não serão ordenações genéricas do MVP. O `GET` será desenhado para manter a URL completa preferencialmente abaixo de 2.048 caracteres; projeções ou consultas que ultrapassem esse perfil usarão `POST .../records/query`.

Essa paginação não promete snapshot consistente enquanto registros concorrentes são alterados. Exportações e relatórios que exijam visão estável ou grande profundidade usarão futuramente job assíncrono com snapshot controlado. Cursor stateful armazenado por UUID, cursor de conexão do PostgreSQL e offset profundo não fazem parte da paginação pública padrão; podem ser avaliados apenas para fluxos internos ou especializados.

OpenAPI será derivado dos contratos estáveis e dos metadados publicados. Operações adequadas à IA recebem metadados de habilitação, risco, confirmação e limites. O conjunto apresentado à LLM é uma allowlist filtrada pelas permissões do usuário, não a especificação completa.

Mutações aceitam chave de idempotência e controle otimista. O OpenAPI usa a mesma chave lógica da entidade e dos campos, sem criar um segundo identificador canônico específico para schemas.

Alternativas consideradas: GraphQL como contrato principal, envelope universal `data`, paginação por offset, cursor exposto somente como ID, cursor stateful armazenado no servidor, cursor nativo do PostgreSQL entre requisições e compressão Gzip obrigatória. REST foi escolhido por previsibilidade operacional, OpenAPI e adequação como fronteira de ferramentas da IA. O envelope `data` foi rejeitado por não agregar valor às representações atuais. Offset e ID isolado não sustentam com segurança consultas profundas com ordenações dinâmicas; cursores stateful e de banco adicionariam estado distribuído ou dependência de conexão sem necessidade no fluxo HTTP comum. Gzip foi rejeitado porque o payload binário mínimo é pouco compressível e o custo fixo pode aumentar os tokens mais comuns.

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

- [Falha de escopo organizacional expõe dados] -> centralizar contexto confiável, exigir `organization_id`, aplicar filtro no backend e RLS obrigatória, usar role sem bypass e testar tentativas cruzadas em todos os caminhos.
- [Staging de importação pode contornar o isolamento] -> separar role e schema, prender o job a uma organização confiável, proibir SQL/alvo arbitrário e promover dados somente para tabelas finais protegidas.
- [UUID aumenta índices e FKs em relação a `BIGINT`] -> usar UUIDv7 temporal, evitar chave técnica duplicada, criar índices orientados às consultas organizacionais e validar limites com benchmark real.
- [Tabela por entidade aumenta o número de objetos no PostgreSQL] -> impor limites, monitorar catálogo e particionar apenas mediante métricas.
- [Uma versão global pode ser bloqueada pelos dados de uma organização] -> relatório por organização, backfill explícito e ativação somente após conformidade total.
- [DDL bloqueia ou degrada tabelas grandes] -> classificar operações, medir impacto, executar em fases e exigir janela quando não houver técnica online segura.
- [Motor de schema próprio é uma área de alta complexidade] -> vocabulário fechado de operações, planos imutáveis, pre/pós-condições, testes reais e diário idempotente.
- [Drift por intervenção manual quebra o catálogo] -> credenciais separadas, comparação com `pg_catalog`, bloqueio e correção administrada.
- [Schema obsoleto causa perda de atualização] -> versionar schema e registro, rejeitar com conflito e invalidar caches por outbox.
- [Um realm compartilhado amplia colisões e roteamento de identidade] -> identidade por subject estável, vínculo controlado e política de descoberta ainda a definir.
- [Exposição pública do Keycloak amplia ataque] -> Nginx com rotas mínimas, console interno, proxies confiáveis, TLS, rate limiting e BFF; avaliar WAF como defesa adicional antes da exposição pública de produção.
- [Auditoria no mesmo banco não é WORM] -> append-only, privilégios mínimos, encadeamento verificável e opção futura de âncora externa.
- [LLM pode sofrer prompt injection ou solicitar ação excessiva] -> allowlist de ferramentas, contexto fora do prompt, confirmação humana, limites e autorização repetida na API.
- [Dados sensíveis podem sair para um provedor de LLM] -> classificação por campo, minimização, mascaramento e seleção do provedor condicionada a requisitos de privacidade.
- [REST interno adiciona latência e autenticação ao orquestrador] -> cliente gerado/validado, conexão privada e fronteira que permite medir e extrair o componente.
- [Licença do Liquibase pode ser incompatível com o produto] -> revisão antes de fixar versão e possibilidade de substituição restrita ao migrador estático.

## Migration Plan

Como o projeto é novo, o plano é de implantação incremental, não de migração de um sistema legado:

1. provisionar um cluster PostgreSQL 18.x com databases `entia` e `keycloak`, ou um serviço gerenciado equivalente;
2. criar proprietários e credenciais separados para Keycloak, migrations, schema dinâmico, runtime, ingestão e auditoria;
3. aplicar o schema estático inicial por migration controlada, habilitando e forçando RLS nas tabelas organizacionais e mantendo o runtime sem propriedade ou bypass;
4. configurar realm `ent-ia`, clientes BFF/API e um provedor padrão;
5. implantar Nginx com frontchannel mínimo do Keycloak, administração interna, rate limiting e limites de tráfego;
6. disponibilizar o monólito com módulos de identidade, tenancy, autorização, catálogo e auditoria;
7. validar o motor de schema em uma entidade piloto, cobrindo publicação, RLS, falha, retry e drift;
8. habilitar runtime REST, importação e frontend dinâmico somente após testes de isolamento, concorrência e carga com UUIDv7;
9. introduzir o orquestrador de IA sem ferramentas de escrita; habilitar consultas após testes de minimização e autorização;
10. habilitar mutações pela IA somente após aprovação, idempotência e auditoria estarem verificadas;
11. expandir capacidades e limites com métricas reais.

Rollback de aplicação deve preservar compatibilidade com o schema estático já aplicado. Ativação dinâmica falha antes da troca do ponteiro e mantém a versão anterior. Estruturas e dados publicados não são removidos automaticamente; restauração de backup é último recurso operacional, não mecanismo normal de rollback.

## Open Questions

### Antes de decompor o trabalho de implementação

- Qual será o contrato específico dos jobs assíncronos de exportação e dos relatórios que exijam snapshot consistente?
- Quais limites máximos de entidades, campos e registros por organização serão adotados, e quais ajustes finos de índices serão indicados pelos benchmarks?
- Qual algoritmo e granularidade de encadeamento serão usados na auditoria, e como será tratada a retenção?
- Como a organização será descoberta antes do login e como conflitos de e-mail entre provedores serão resolvidos?
- Quais metas de tempo de resposta e concorrência complementarão os limites técnicos já definidos para as consultas síncronas do MVP?

### Definíveis durante o planejamento de implantação ou de fases posteriores

- Em qual marco de exposição e risco um WAF passa a ser obrigatório, e qual solução será adotada?
- Como os projetos Docker Compose serão distribuídos e acionados remotamente em cada host?
- Quais soluções cuidarão de segredos, observabilidade, backup, recuperação e disaster recovery?
- Quais RPO, RTO e SLOs determinarão a necessidade de réplica e failover do PostgreSQL?
- Qual versão do Liquibase é compatível com a política jurídica, de distribuição e de hospedagem da ENT.IA?
- Qual provedor e modelo de LLM atendem tool calling, privacidade, residência, latência e custo?
- O provedor de LLM será global ou poderá ser substituído por organização?
- Qual será a política de retenção, exportação e exclusão das conversas?
- Quais limites de tokens, custo, duração, chamadas de ferramentas e operações em lote serão adotados?
- Quais tamanho de bloco, atomicidade, retenção e política de retomada serão usados nas importações massivas?
- Em que fase RAG, embeddings, pgvector, MCP ou agentes especializados passam a ter benefício comprovado?
