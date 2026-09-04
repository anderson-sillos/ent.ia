# Handoff de contexto — ENT.IA

Última consolidação: 2026-09-04

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
- PostgreSQL Row-Level Security será obrigatório desde a fundação em todas as tabelas persistentes organizacionais, estáticas ou dinâmicas, acessadas pelos fluxos regulares.
- RLS complementará, sem substituir, os filtros por organização e a autorização RBAC aplicados pelo backend.
- A role de runtime não será proprietária dessas tabelas, não terá `BYPASSRLS` e não usará superusuário; ausência de contexto organizacional válido deverá negar o acesso.
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

### Nomenclatura lógica e física

- Nome de exibição, chave lógica e nome físico são conceitos separados.
- O nome de exibição é livre, traduzível e alterável.
- A chave lógica canônica usa `camelCase` e é compartilhada por catálogo, JSON, REST e OpenAPI; não haverá outra chave canônica específica para o OpenAPI.
- Nomes físicos usam `snake_case`; `flatcase` e `PascalCase` não serão usados como formatos alternativos da chave canônica.
- Somente tabelas das entidades dinâmicas recebem o prefixo `en_`; tabelas globais e estáticas não recebem esse prefixo.
- Campos estruturais usam nomes convencionais, como `organization_id`, `id`, `record_version`, `created_at`, `updated_at` e `deleted_at`.
- Campos do catálogo usam prefixo semântico de dois caracteres. O vocabulário inicial é `id`, `dt`, `nr`, `nm`, `ds` e `vl` para identificação, data, número, nome, descrição e valor.
- Constraints e índices usam `pk_`, `fk_`, `uq_`, `ck_` e `ix_`.
- O motor tenta primeiro o nome legível sem sufixo. Colisão legítima recebe sufixo determinístico de exatamente três caracteres em base 36, derivado do UUID interno e persistido no catálogo.
- Não serão usados sufixos sequenciais como `_1` e `_2`, nem truncamento silencioso. Termos longos exigem abreviação controlada e persistida.
- Objeto físico existente sem mapeamento no catálogo é drift e bloqueia a ativação; não justifica criar outro nome com sufixo.

### Identificadores e importação massiva

- Registros dinâmicos usarão UUIDv7 como identificador técnico imutável, armazenado no tipo nativo `uuid` do PostgreSQL.
- A chave primária das tabelas dinâmicas será composta por `(organization_id, id)`.
- O backend ou importador confiável gerará o UUIDv7 antes da persistência; `uuidv7()` do PostgreSQL 18.x poderá existir como default de segurança.
- UUIDv4 aleatório não será o padrão para registros de alto volume.
- Identificadores de negócio serão separados da chave técnica e poderão ter regras próprias por organização e entidade.
- Sequences ou identity `BIGINT` ficarão reservadas a versões, posições e numerações locais que não precisem de identidade global antecipada.
- Não serão mantidas, por padrão, chaves técnicas `BIGINT` e UUID simultaneamente em todas as tabelas dinâmicas.
- Sequence comum pode apresentar lacunas e não será usada quando houver exigência de numeração legalmente contínua.
- Lotes regulares usarão `INSERT` multi-row ou batch JDBC sob RLS e os demais controles do runtime.
- Importações de alto volume carregarão uma staging controlada com `COPY FROM STDIN`, validarão o lote e promoverão os dados em conjunto para as tabelas finais protegidas por RLS.
- A staging transitória será uma exceção controlada por causa da incompatibilidade entre `COPY FROM` e RLS: ficará isolada, presa a `import_id` e `organization_id`, acessível somente por role e serviço de ingestão dedicados e sem aceitar SQL ou tabela-alvo arbitrários.
- A promoção da staging para as tabelas persistentes finais usará contexto organizacional confiável e uma role submetida ao RLS.
- Chaves de origem serão mapeadas para UUIDv7 previamente atribuídos, permitindo relacionamentos entre linhas e retomada idempotente.
- Tamanho de bloco, atomicidade, retenção e índices serão definidos depois de benchmark representativo.

### Exclusão lógica e integridade referencial

- A exclusão regular de registros dinâmicos será lógica, por meio de `deleted_at`, com incremento de `record_version` e evento de auditoria.
- Consultas normais omitirão registros excluídos; fluxos históricos ou administrativos explicitamente autorizados poderão incluí-los.
- Relacionamentos existentes serão preservados para o histórico, mas novas referências a registros excluídos serão rejeitadas.
- Chaves únicas de negócio permanecerão reservadas após a exclusão lógica; a restauração controlada será o caminho padrão para reutilização.
- Toda chave estrangeira dinâmica incluirá `organization_id`, apontará para `(organization_id, id)`, usará `ON DELETE RESTRICT` e `ON UPDATE RESTRICT` e terá índice correspondente iniciado por `organization_id`.
- O catálogo não permitirá escolher `CASCADE`, `SET NULL`, `SET DEFAULT` ou `NO ACTION` por relação no MVP.
- A API normal e as ferramentas da IA não oferecerão exclusão física.
- Futuro expurgo será administrativo, explícito, ordenado por dependências, sujeito à retenção e auditado.
- `ON DELETE CASCADE` somente poderá ser adotado explicitamente em tabelas estáticas e técnicas controladas pela plataforma; nunca removerá eventos da auditoria.

### Cardinalidades e semântica dos relacionamentos

- `N:1` será a relação física fundamental, com FK composta armazenada na entidade de origem; poderá ser opcional ou obrigatória.
- `1:N` será uma visão inversa derivada da mesma FK, sem coleção ou estado duplicado na entidade de destino.
- `1:1` usará FK no lado dependente e unicidade sobre `(organization_id, campo_de_referencia)`.
- `N:N` será modelada por entidade associativa de primeira classe com duas relações `N:1`, nunca por uma tabela de junção oculta sem metadados.
- Um assistente poderá criar a entidade associativa e ocultá-la da navegação principal, mas ela continuará versionada, persistida, autorizada e auditada como as demais entidades.
- Autorreferência será aceita somente como relação opcional comum no MVP; isso não oferecerá automaticamente árvore, aciclicidade, limite de profundidade ou consultas hierárquicas.
- O catálogo reservará a distinção entre `REFERENCE` e `COMPOSITION`, mas somente `REFERENCE` poderá ser ativada inicialmente.
- Composição, cascata lógica automática, contagem mínima de filhos, restauração conjunta e hierarquia especializada ficam adiadas.
- Novas referências exigirão alvo ativo. Alterações que não modifiquem uma referência histórica para alvo excluído continuarão permitidas.
- Em uma relação `1:1`, a associação continuará reservada após exclusão lógica, favorecendo restauração controlada.
- Ciclos obrigatórios permanecem bloqueados; ciclos opcionais podem existir quando todas as entidades envolvidas estiverem ativas.

### Identidade e autenticação

- A identidade do usuário será global, e um usuário poderá pertencer a múltiplas organizações.
- Cada organização poderá usar seu próprio provedor de identidade OIDC ou SAML.
- Quando a organização não possuir provedor próprio, será usado o provedor padrão da instalação.
- Keycloak self-hosted será o provedor padrão e o broker de identidade.
- Será adotado um único realm Keycloak chamado `ent-ia`.
- O caminho organizacional canônico do MVP será `/o/{organizationSlug}`; a entrada raiz será neutra e identity-first, sem listar organizações.
- Quando a descoberta depender do e-mail, uma resposta pública indistinguível e uma comprovação de posse de uso único antecederão a apresentação dos vínculos elegíveis.
- Identidades externas serão vinculadas por `(issuer, subject)`; igualdade de e-mail entre provedores não produzirá associação automática.
- Senhas e hashes de senha não serão armazenados no banco da aplicação ENT.IA.
- Recuperação de credenciais e políticas de senha ficarão sob responsabilidade do provedor de identidade.
- Os dados do Keycloak ficarão logicamente separados dos dados da aplicação.

### Exposição e integração do Keycloak

- A instância do Keycloak permanecerá em rede interna e não será exposta diretamente à internet.
- O Nginx Open Source será o servidor web e proxy reverso da borda pública.
- O Nginx servirá os arquivos estáticos do React, encaminhará o BFF/API ao Spring Boot e publicará somente os endpoints indispensáveis ao frontchannel de autenticação, incluindo os callbacks necessários ao login social.
- O hostname público previsto para autenticação é `auth.ent.ia.br`.
- A exposição pública deverá ser limitada a `/realms/ent-ia/*`, `/resources/*` e `/.well-known/*`, sujeita à validação final durante a implantação.
- O console administrativo, a Admin REST API, o realm `master`, métricas, health checks e a porta de gerenciamento permanecerão acessíveis somente pela rede interna ou VPN.
- O Keycloak deverá aceitar conexões somente dos proxies e serviços internos autorizados.
- A comunicação de backchannel entre o backend ENT.IA e o Keycloak utilizará a rede interna, mantendo hostname e issuer públicos consistentes com o fluxo OIDC.
- Será adotado o padrão BFF: o React iniciará o login pelo backend ENT.IA, o backend fará a troca do authorization code e manterá os tokens fora do navegador.
- O navegador receberá somente uma sessão protegida por cookie `HttpOnly`, `Secure` e com política `SameSite` apropriada.
- Não será usado fluxo de autenticação que envie usuário e senha à API da ENT.IA; autenticação, MFA e login social continuarão centralizados no Keycloak e nos provedores federados.
- TLS, sobrescrita segura de cabeçalhos encaminhados, lista de proxies confiáveis, rate limiting e limites de conexões, payload e tempo deverão ser aplicados no Nginx.
- Os backends não deverão aceitar acesso público direto.
- A solução de WAF ainda não foi escolhida. Ela será uma camada adicional de defesa e não substituirá os controles de autenticação, autorização, validação e isolamento da aplicação.

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

- Os eventos ficarão no mesmo PostgreSQL da aplicação e serão append-only para os fluxos e credenciais normais.
- Toda mutação concluída gravará seu evento pela mesma conexão e transação PostgreSQL; falha na auditoria reverterá a operação de negócio.
- A role de runtime poderá inserir eventos, mas não será proprietária da tabela nem poderá atualizar, excluir ou truncar a trilha.
- Cada evento usará UUIDv7 próprio e não dependerá de cabeça compartilhada, sequence serial da trilha, hash anterior ou checkpoint.
- Encadeamento criptográfico, WORM e âncora externa estão fora do MVP; a solução inicial não detecta alterações feitas por DBA, proprietário ou superusuário.
- A auditoria registrará somente as chaves lógicas dos campos alterados e as versões envolvidas, sem valores anteriores ou posteriores.
- O BFF criará um `session_trace_id` UUIDv7 interno e não secreto, diferente de cookies e tokens, para correlacionar os eventos da sessão.
- Cada requisição auditada registrará IP normalizado, User-Agent sanitizado e limitado, cliente, canal, método, rota normalizada, request ID e correlation ID.
- O backend confiará em informações de origem somente quando normalizadas pelo Nginx ou por proxies allowlisted; headers enviados livremente pelo cliente não serão aceitos como autoridade.
- IP e User-Agent são evidências complementares e potencialmente dados pessoais, nunca prova isolada de identidade.
- Listagens e leituras comuns não serão auditadas individualmente; consultas da auditoria, acesso a excluídos e outras leituras privilegiadas serão auditadas.
- A IA reutilizará o evento da operação REST e distinguirá usuário iniciador, sessão de confirmação e orquestrador executor.
- Importações massivas registrarão um evento resumido por bloco confirmado e manterão o detalhamento por linha nas estruturas da importação referenciadas pelo `import_id`.
- Não haverá expurgo automático de eventos no MVP; crescimento da tabela e dos índices será monitorado até uma futura política explícita de retenção.

### Backend e arquitetura

- O backend será implementado em Java, tecnologia em que o responsável pelo projeto possui maior experiência.
- A aplicação seguirá o modelo de monólito modular.
- Stack recomendada e aceita:
  - OpenJDK 25 LTS;
  - Spring Boot 4.1;
  - Spring Modulith;
  - Spring Security integrado ao Keycloak;
  - jOOQ para SQL estático e dinâmico;
  - PostgreSQL 18.x;
  - Liquibase para migrations do schema estático da plataforma, condicionado à revisão da licença da versão adotada;
  - Maven;
  - Testcontainers para testes de integração.
- Não introduzir Kafka inicialmente. Caso seja necessária execução assíncrona, começar com uma fila persistida no PostgreSQL.
- O Liquibase está confirmado como a ferramenta recomendada para evoluir as tabelas estáticas mantidas pelo código, como organizações, vínculos, perfis, catálogo, auditoria e conversas de IA.
- As tabelas físicas das entidades configuradas pelos usuários serão administradas por um motor próprio de schema da ENT.IA, e não por changelogs Liquibase gerados em tempo de execução.
- Em produção, as migrations estáticas deverão ser executadas uma única vez antes da disponibilização da nova versão, usando uma credencial de banco distinta e mais privilegiada que a credencial normal da aplicação.
- Antes de fixar a versão do Liquibase, deverá ser feita revisão técnica e jurídica da licença vigente. A linha Community 5.x usa FSL com conversão futura para Apache 2.0, enquanto versões anteriores possuíam condições diferentes; não atualizar a versão principal automaticamente sem essa revisão.

### Infraestrutura, topologia e PostgreSQL

- A arquitetura de implantação será agnóstica de provedor de nuvem ou ambiente on-premises.
- Imagens, portas, configuração externa, health checks e contratos de rede permanecerão portáveis; particularidades de cada ambiente ficarão em módulos OpenTofu próprios.
- Containers compatíveis com OCI serão usados desde a fundação.
- OpenTofu será a ferramenta de Infrastructure as Code, mantendo compatibilidade conceitual com Terraform.
- Kubernetes foi adiado. Os containers deverão permanecer stateless quando aplicável, usar configuração externa, health checks, logs em `stdout` e encerramento gracioso para permitir adoção futura.
- A implantação inicial usará Docker Compose.
- Em produção haverá camadas fisicamente separadas para borda, aplicação/identidade e dados.
- Cada host terá seu próprio projeto Docker Compose; Compose não será tratado como orquestrador multi-host.
- O PostgreSQL seguirá política híbrida: self-hosted em container OCI como implantação de referência e serviço gerenciado compatível como opção preferencial em produção na nuvem.
- A aplicação não dependerá de extensões ou interfaces proprietárias de um provedor de PostgreSQL.
- Será usada a linha PostgreSQL 18.x, atualizada dentro da mesma versão principal após validação.
- Inicialmente haverá um serviço ou cluster com dois databases: `entia` e `keycloak`.
- As organizações compartilharão o database e o schema de aplicação do `entia`, segregadas por `organization_id`.
- Os databases `entia` e `keycloak` terão proprietários e credenciais independentes, sem acesso cruzado.
- No database `entia`, haverá credenciais distintas para proprietário técnico, Liquibase, motor de schema dinâmico, runtime, ingestão e leitura administrativa da auditoria. A escrita transacional será permitida à role de runtime somente por `INSERT`.
- O fluxo de importação massiva terá credencial de ingestão própria, sem reutilização pelo runtime normal e sem bypass de RLS ao promover dados para tabelas finais.
- O Keycloak terá credencial própria limitada ao database `keycloak`.
- Nenhum processo normal da aplicação utilizará superusuário do cluster.

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
- A organização poderá selecionar temas declarativos e versionados e ajustar somente cores, nome e ativos permitidos; HTML, JavaScript, CSS, fontes e layouts arbitrários não serão aceitos.
- A entrada genérica será neutra até a resolução da organização. Páginas e e-mails internos do Keycloak não receberão tema dinâmico no MVP, e domínios personalizados permanecerão uma evolução futura.
- Evitar Redux inicialmente. Estado de sessão e organização pode ser mantido em contextos pequenos.
- Testes previstos com Vitest, React Testing Library e Playwright.

### API REST dinâmica, consultas e paginação

- A API pública usa `/api/v1`, separando `/session`, `/catalog` e `/entities`.
- Registros dinâmicos ficam em `/api/v1/entities/{entityKey}/records`; a chave lógica `camelCase` identifica a entidade e o UUIDv7 identifica o registro.
- `GET` atende leituras, `POST` cria recursos, executa consultas compostas e inicia comandos ou jobs, `PATCH` aplica JSON Merge Patch e `DELETE` executa exclusão lógica. `PUT` não integra o MVP; `TRACE` fica desabilitado e `CONNECT` não é exposto.
- Registros retornam `id`, campos dinâmicos autorizados e `meta` na raiz, sem envelope `data`; coleções usam `items` e `meta`; erros usam `application/problem+json`.
- A listagem padrão por `GET` aceita somente `select`, `limit` e um entre `after` ou `before`. Filtros e ordenações personalizados usam `POST .../records/query` com DSL JSON estruturada.
- A DSL separa `select`, `filter`, `order` e `page`. Filtros usam AST com grupos `and`/`or` e condições tipadas; SQL, scripts, regex, funções arbitrárias e travessia implícita de relacionamentos são proibidos.
- Operadores iniciais: `eq`, `ne`, `in`, `notIn`, `isNull`, `isNotNull`, `gt`, `gte`, `lt`, `lte`, `contains`, `startsWith`, `like` e `ilike`.
- Cada campo publicado declara `selectable`, `filterable`, `sortable` e `patternSearch`, sempre subordinados ao tipo, à versão e às permissões efetivas.
- Cada versão possui projeções padrão de lista e detalhe. `select` explícito retorna os campos autorizados solicitados mais `id` e `meta`; não há wildcard, alias, agregação, expressão calculada ou seleção aninhada no MVP.
- `order` é uma lista prioritária com `field`, `direction` (`asc`/`desc`) e `nulls` (`first`/`last`, padrão `last`). Se omitido, usa a ordenação versionada ou `id desc`; o UUIDv7 é acrescentado na mesma direção da última chave.
- Ordenação genérica aceita somente campos escalares compactos, no máximo três campos explícitos e nenhuma collation ou função escolhida pela requisição. Relacionamentos, textos extensos, JSON, coleções, binários e cálculos não persistidos ficam fora do MVP.
- Limites síncronos confirmados: cinco níveis da AST, 25 condições, 100 itens por `in`/`notIn`, 50 campos em `select`, três em `order`, corpo de 64 KiB, página padrão 50 e máxima 200, padrão textual de 256 caracteres e oito curingas, curinga inicial com três caracteres literais consecutivos e índice compatível, além de `statement_timeout` inicial de cinco segundos.
- Toda tabela dinâmica terá índice base para registros ativos iniciado por `(organization_id, id)`. Ordenação simples exige índice compatível; combinações de duas ou três chaves exigem perfil de consulta publicado e índice composto iniciado por `organization_id` e terminado por `id`. Não serão criadas todas as combinações automaticamente.
- A paginação usa keyset/seek com cursor opaco stateless. Não há offset profundo, salto por página nem contagem total implícita.
- O cursor usa serialização binária compacta, AEAD e Base64 URL-safe, com meta de 512 e máximo de 1.024 caracteres. CBOR é candidato, não obrigação. Gzip foi descartado no MVP.
- Organização, sujeito, entidade, versão, projeção, filtro e ordem são representados por fingerprint criptográfico de 16 bytes; o token armazena apenas estado posicional mínimo. `limit` pode variar dentro do máximo.
- O cursor vale 30 minutos. Cada página bem-sucedida emite novos cursores por mais 30 minutos; não existe renovação direta. Mudança de usuário, organização ou versão da entidade invalida a continuação; permissões são reavaliadas em toda página.
- Perfis de ordenação são validados pelo pior tamanho possível de suas posições antes da publicação.
- Exportações, relatórios e consultas assíncronas com snapshot estão fora do MVP. Pontos de extensão serão preservados, mas o contrato de jobs, autorização, armazenamento, retenção e download será definido futuramente.

### Limites e metas técnicas do MVP

- Guardrails iniciais: 200 entidades publicadas por instalação, 100 habilitadas por organização, 100 campos e 20 relacionamentos por entidade, dez perfis compostos de ordenação e cinco campos com `patternSearch` por entidade.
- Uma instalação poderá reduzir esses valores; qualquer elevação exigirá configuração administrativa e benchmark prévio.
- Não haverá cota transacional de registros baseada em `COUNT` ou contador global. O volume será acompanhado por métricas e eventual quota será projetada sem contenção no caminho de escrita.
- O perfil de capacidade terá 20 organizações, 100 entidades, aproximadamente cinco milhões de registros, uma tabela com um milhão de linhas e até 250 mil linhas para uma organização.
- Ambiente de referência: Nginx com 2 vCPU/2 GiB, aplicação com 4 vCPU/8 GiB e PostgreSQL com 4 vCPU/16 GiB e SSD, em camadas separadas.
- Metas p95 do backend: consulta por ID até 300 ms, filtro simples indexado até 500 ms, consulta composta indexada até um segundo e mutação com auditoria até 750 ms; p99 das operações comuns abaixo de dois segundos.
- O teste sustentará 100 usuários virtuais e 30 RPS durante 30 minutos, com pico de 60 RPS durante cinco minutos e erros internos abaixo de 1%.
- Não será permitido alcançar as metas relaxando isolamento ou auditoria: nenhuma mutação confirmada pode ficar sem evento e nenhum dado pode atravessar organizações.
- Esses números são critérios técnicos de aceitação do MVP, não SLO de produção, SLA ou capacidade máxima do PostgreSQL.

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
- O conteúdo integral das conversas não deverá ser copiado para a auditoria append-only; mensagens terão armazenamento e retenção próprios.
- O núcleo da ENT.IA continuará como monólito modular. O orquestrador será um cliente REST interno e deverá possuir uma fronteira que permita isolamento ou implantação separada quando necessário.
- Banco vetorial, RAG, MCP e agentes autônomos não são requisitos da primeira versão dessa capacidade.

## Estado do OpenSpec

- OpenSpec instalado: versão `1.11.0`.
- Mudança ativa: `fundacao-plataforma-entia`.
- Schema do workflow: `spec-driven`.
- Diretório: `openspec/changes/fundacao-plataforma-entia/`.
- `proposal.md`: concluído.
- `specs`: concluídas para as nove capacidades propostas.
- `design.md`: concluído e atualizado com as decisões arquiteturais confirmadas.
- `tasks.md`: próximo artefato, pronto para criação após o encerramento das discussões arquiteturais pendentes.
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

- critérios futuros para adotar encadeamento, checkpoints, WORM ou âncora externa na auditoria;
- contrato futuro dos jobs assíncronos de exportação e relatórios com snapshot consistente, fora do MVP;
- forma de distribuir e acionar remotamente os projetos Docker Compose em cada host;
- gestão de segredos, certificados, backups, recuperação e observabilidade;
- RPO, RTO, SLOs e marco para introduzir réplica e failover do PostgreSQL;
- versão exata do Liquibase e compatibilidade de sua licença com distribuição, hospedagem e modelo comercial da ENT.IA;
- provedor e modelo de LLM, incluindo hospedagem, residência dos dados e possibilidade de configuração por organização;
- política de retenção, exportação e exclusão das conversas;
- limites de volume, custo e duração das interações com a LLM;
- escopo inicial de operações em lote e regras adicionais de confirmação;
- parâmetros das importações massivas: tamanho de bloco, atomicidade, retenção da staging, retomada e metas de desempenho;
- necessidade futura de RAG, embeddings, banco vetorial, MCP ou agentes especializados.
- marco de adoção e solução de WAF para a exposição pública de produção;

## Próximos passos recomendados

1. Encerrar ou classificar os pontos arquiteturais que precisam ser decididos antes da implementação.
2. Criar o `tasks.md`, dividindo a implementação em incrementos verificáveis.
3. Validar os artefatos OpenSpec contra a proposta e as nove especificações existentes.
4. Somente depois iniciar o scaffold e a implementação da aplicação.

Comandos úteis para a retomada:

```bash
openspec status --change "fundacao-plataforma-entia"
openspec instructions tasks --change "fundacao-plataforma-entia" --json
openspec validate fundacao-plataforma-entia --strict
```

Ao continuar, usar a habilidade `openspec-continue-change` e respeitar a criação de um artefato por vez.
