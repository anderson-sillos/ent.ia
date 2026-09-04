## Purpose

Disponibilizar operações uniformes e seguras sobre registros de entidades publicadas, respeitando metadados, organização, permissões e integridade dos relacionamentos.

## ADDED Requirements

### Requirement: Operações orientadas pela definição ativa
A plataforma MUST executar cadastro, leitura, alteração, exclusão e consulta de registros conforme a definição global ativa da entidade.

#### Scenario: Cadastro válido
- **WHEN** um usuário autorizado envia dados compatíveis com a definição ativa de uma entidade habilitada
- **THEN** a plataforma persiste o registro no contexto da organização atual

### Requirement: Validação no servidor
A plataforma MUST validar no servidor tipos, obrigatoriedade, restrições e relacionamentos definidos no catálogo, independentemente das validações feitas pela interface.

#### Scenario: Dados incompatíveis
- **WHEN** uma operação contém dados incompatíveis com a definição ativa
- **THEN** a plataforma rejeita a operação e retorna os erros de validação aplicáveis

### Requirement: Escopo organizacional dos registros
Todo registro dinâmico MUST pertencer a uma organização, e todas as operações MUST restringir seus resultados ao contexto organizacional autorizado.

#### Scenario: Consulta de registros
- **WHEN** um usuário autorizado consulta uma entidade no contexto de sua organização
- **THEN** a plataforma retorna somente registros pertencentes àquela organização

#### Scenario: Organização declarada pelo consumidor
- **WHEN** uma requisição regular tenta escolher outra organização por path, payload ou cabeçalho público
- **THEN** a plataforma ignora a informação como fonte de autoridade ou rejeita a requisição e mantém o contexto derivado da sessão ou do token confiável

### Requirement: Identificador técnico estável
Todo registro dinâmico MUST possuir um UUIDv7 técnico, imutável e armazenado no tipo nativo `uuid`, separado de qualquer identificador de negócio configurado para a entidade. A chave primária física MUST ser composta por `(organization_id, id)`.

#### Scenario: Preparação de registros relacionados em lote
- **WHEN** uma carga válida contém registros novos relacionados entre si
- **THEN** a plataforma atribui UUIDv7 antes da persistência e mantém as referências aos identificadores definitivos durante todo o processamento

### Requirement: Recursos REST estáveis
A API dinâmica MUST publicar suas operações sob `/api/v1/entities/{entityKey}/records`, MUST usar a chave lógica `camelCase` como identificador canônico da entidade e MUST usar o UUIDv7 como identificador do registro. Sessão, catálogo e entidades MUST permanecer em áreas distintas sob `/api/v1`.

#### Scenario: Registro é endereçado pela integração
- **WHEN** uma integração autorizada consulta um registro conhecido
- **THEN** ela usa a mesma chave lógica publicada pelo catálogo e o UUIDv7 retornado pela plataforma, sem depender do nome físico da tabela

### Requirement: Métodos HTTP controlados
A API regular MUST usar `GET` para leitura, `POST` para criação, consulta composta e comandos, `PATCH` com JSON Merge Patch para alteração parcial e `DELETE` para exclusão lógica. `PUT` MUST NOT integrar o contrato do MVP; `TRACE` MUST permanecer desabilitado e `CONNECT` MUST NOT ser exposto. `HEAD` e `OPTIONS` MAY ser respondidos automaticamente pela infraestrutura ou pelo framework sem representar operações de negócio.

#### Scenario: Consulta composta usa POST
- **WHEN** um consumidor executa `POST /api/v1/entities/{entityKey}/records/query` com uma DSL válida
- **THEN** a plataforma realiza somente leitura, retorna `200 OK` e não cria recurso nem exige chave de idempotência

#### Scenario: Substituição integral é solicitada
- **WHEN** um consumidor tenta substituir um registro dinâmico com `PUT`
- **THEN** a plataforma rejeita o método sem interpretar campos omitidos como exclusão

#### Scenario: Registro é restaurado
- **WHEN** um usuário autorizado solicita a restauração de um registro logicamente excluído
- **THEN** a plataforma executa um comando explícito por `POST`, sem permitir alteração direta de `deleted_at`

### Requirement: Representação de registro sem envelope de dados
A API MUST representar `id`, campos dinâmicos autorizados e o objeto reservado `meta` na raiz do registro, sem envelope `data`. Coleções MUST usar `items` e `meta`. Campos de negócio MUST NOT usar os nomes reservados `id` ou `meta`.

#### Scenario: Coleção de registros é retornada
- **WHEN** uma consulta encontra registros autorizados
- **THEN** a resposta contém a lista em `items` e os metadados da consulta em `meta`, sem envolver o resultado em `data`

#### Scenario: Valor decimal é transportado
- **WHEN** um campo decimal autorizado é retornado ou recebido em JSON
- **THEN** a API usa uma representação textual que preserve sua precisão decimal

### Requirement: Alteração parcial e erros interoperáveis
A API MUST receber criações como JSON plano, MUST aplicar alterações parciais com JSON Merge Patch e MUST representar erros HTTP como `application/problem+json`, incluindo extensões estruturadas quando houver erros de campos ou conflitos de versão.

#### Scenario: Patch omite um campo
- **WHEN** uma alteração parcial válida não inclui determinado campo de negócio
- **THEN** a plataforma preserva o valor atual desse campo

#### Scenario: Payload viola a definição ativa
- **WHEN** uma criação ou alteração contém valores inválidos
- **THEN** a plataforma responde com um Problem Details e identifica de forma estruturada os campos e violações aplicáveis

### Requirement: Representação autoritativa de referência
A API MUST representar uma referência por seu `id` e MAY acrescentar `label` derivado do template de exibição e `deleted` para sinalizar alvo historicamente excluído. Somente `id` MUST ser usado como identidade autoritativa. Relações inversas MUST ser consultadas separadamente e MUST NOT ser mantidas como coleções duplicadas no registro.

#### Scenario: Referência é apresentada em um registro
- **WHEN** o consumidor está autorizado a visualizar uma referência
- **THEN** a API retorna seu `id` e pode retornar uma legenda conveniente sem transformar essa legenda em chave da relação

#### Scenario: Alvo altera seu nome de exibição
- **WHEN** os campos usados na legenda do alvo são modificados
- **THEN** a identidade da relação permanece inalterada e a legenda pode ser recalculada

### Requirement: Consulta dinâmica estruturada
A API MUST oferecer listagem padrão por `GET /api/v1/entities/{entityKey}/records`, limitada aos parâmetros reservados `select`, `limit`, `after` e `before`, e consulta composta por `POST /api/v1/entities/{entityKey}/records/query`. A consulta composta MUST separar projeção, filtros, ordenação e paginação em uma DSL JSON estruturada e MUST NOT aceitar SQL, expressão textual livre, funções arbitrárias, scripts, expressões regulares ou travessia arbitrária de relacionamentos.

#### Scenario: Consulta composta é executada
- **WHEN** um consumidor autorizado envia campos e operadores suportados pela versão ativa da entidade
- **THEN** a plataforma valida a DSL, aplica autorização e escopo organizacional e executa somente a consulta estruturada permitida

#### Scenario: Consulta tenta injetar expressão livre
- **WHEN** um filtro ou ordenação contém SQL ou operador não publicado
- **THEN** a plataforma rejeita a consulta antes de executá-la no banco

#### Scenario: Filtro composto é enviado no GET
- **WHEN** um consumidor tenta codificar a árvore de filtros ou uma ordenação personalizada nos parâmetros da listagem `GET`
- **THEN** a plataforma rejeita os parâmetros não suportados e orienta o uso de `POST .../records/query`

### Requirement: AST de filtros tipada
A DSL MUST representar filtros como uma árvore recursiva de grupos `and` ou `or` e condições terminais formadas por `field`, `operator` e, quando aplicável, `value`. O runtime MUST validar campo, tipo, permissão, operador e valor antes de produzir condições jOOQ parametrizadas.

#### Scenario: Grupo lógico válido é recebido
- **WHEN** uma consulta combina condições compatíveis em grupos `and` e `or` dentro dos limites configurados
- **THEN** a plataforma preserva a precedência expressa pela árvore e compila as condições parametrizadas

#### Scenario: Operador incompatível com o tipo é usado
- **WHEN** uma condição aplica operador numérico a um campo que não admite comparação numérica
- **THEN** a plataforma rejeita a DSL antes de consultar o PostgreSQL

### Requirement: Operadores iniciais da DSL
O vocabulário inicial MUST incluir `eq`, `ne`, `in`, `notIn`, `isNull`, `isNotNull`, `gt`, `gte`, `lt`, `lte`, `contains`, `startsWith`, `like` e `ilike`, cada um restrito aos tipos compatíveis. Referências MUST ser comparadas pelo UUID do alvo e MUST NOT permitir travessia implícita.

#### Scenario: Referência é filtrada
- **WHEN** um consumidor autorizado aplica `eq` a um campo de referência usando um UUID válido
- **THEN** a plataforma compara o identificador do alvo sem carregar ou atravessar seus demais campos

#### Scenario: Operador de texto é aplicado a número
- **WHEN** uma consulta aplica `contains`, `startsWith`, `like` ou `ilike` a um campo numérico
- **THEN** a plataforma rejeita a condição como incompatível

### Requirement: Padrões textuais controlados
Os operadores `like` e `ilike` MUST aceitar `%` e `_` com escape explícito, MUST usar valores parametrizados e MUST operar somente em campos textuais habilitados no catálogo para pesquisa por padrão. Cada padrão MUST possuir no máximo 256 caracteres e oito curingas. Um curinga inicial MUST exigir ao menos três caracteres literais consecutivos e índice compatível.

#### Scenario: Padrão permitido é consultado
- **WHEN** um campo habilitado recebe um padrão válido dentro dos limites e com suporte de índice exigido
- **THEN** a plataforma executa `like` com distinção de caixa ou `ilike` sem essa distinção, conforme o operador solicitado

#### Scenario: Curinga inicial sem índice compatível
- **WHEN** uma consulta usa padrão iniciado por `%` ou `_` em campo sem o suporte de índice exigido
- **THEN** a plataforma rejeita a condição antes que ela provoque uma varredura ampla não autorizada

### Requirement: Limites da consulta síncrona
A DSL síncrona MUST aceitar no máximo cinco níveis na árvore de filtros, 25 condições terminais, 100 valores por operador `in` ou `notIn`, 50 campos em `select`, três campos explícitos em `order` e corpo de 64 KiB. `page.limit` MUST assumir 50 quando omitido e MUST NOT exceder 200. O executor MUST aplicar inicialmente limite de cinco segundos à consulta no PostgreSQL.

#### Scenario: Consulta excede limite estrutural
- **WHEN** o corpo ultrapassa qualquer limite de profundidade, condições, valores, projeção, ordenação, tamanho ou página
- **THEN** a plataforma rejeita a DSL antes de executar SQL

#### Scenario: Consulta interativa excede o tempo permitido
- **WHEN** a execução no PostgreSQL alcança o limite de cinco segundos
- **THEN** a plataforma cancela a consulta e responde com Problem Details sem continuar consumindo recursos em segundo plano

#### Scenario: Consumidor necessita processamento mais pesado
- **WHEN** uma exportação, relatório ou consulta não cabe no perfil síncrono
- **THEN** a plataforma rejeita o uso do perfil interativo e não tenta executar implicitamente uma exportação ou relatório pesado de forma síncrona

### Requirement: Exportações e relatórios assíncronos fora do MVP
O MVP MUST NOT expor operações de exportação, geração de relatórios ou jobs assíncronos de consulta com snapshot. A arquitetura MAY reservar pontos de extensão, mas o contrato, armazenamento, autorização, retenção e download desses artefatos MUST ser definidos em uma fase futura.

#### Scenario: Consumidor solicita exportação
- **WHEN** um consumidor tenta iniciar uma exportação ou relatório não oferecido pelo contrato do MVP
- **THEN** a plataforma não executa a carga como consulta síncrona alternativa e informa que a capacidade não está disponível

### Requirement: Meta inicial de capacidade sem cota transacional de registros
O MVP MUST validar uma carga com 20 organizações, 100 entidades publicadas, aproximadamente cinco milhões de registros dinâmicos no conjunto e pelo menos uma tabela com um milhão de linhas, das quais até 250 mil pertençam à mesma organização. A plataforma MUST NOT executar `COUNT` nem atualizar contador serializado em cada inserção apenas para impor uma cota síncrona de registros por organização; crescimento e eventual quota MUST ser tratados por métricas e processos próprios.

#### Scenario: Organização amplia seu volume de registros
- **WHEN** uma inserção válida aumenta o volume da organização
- **THEN** o runtime aplica autorização, validação, RLS, constraints e auditoria sem executar contagem total ou bloquear contador global de quota na transação

### Requirement: Critérios técnicos de desempenho do MVP
No ambiente de referência e com o perfil de dados confirmado, o backend MUST atender às seguintes metas p95: consulta por identificador em até 300 ms, listagem ou filtro simples indexado em até 500 ms, consulta composta indexada em até um segundo e criação, alteração ou exclusão lógica em até 750 ms, incluindo autenticação local da sessão, autorização, RLS e auditoria. O p99 das operações comuns MUST permanecer abaixo de dois segundos, sem substituir o `statement_timeout` de cinco segundos já aplicado às consultas síncronas.

O teste MUST sustentar 100 usuários virtuais e 30 requisições por segundo durante 30 minutos, além de um pico de 60 requisições por segundo durante cinco minutos. A taxa de erros internos MUST permanecer abaixo de 1%, excluindo rejeições esperadas de validação, autenticação e autorização. O tempo do navegador, de provedores externos de identidade e da LLM MUST NOT integrar essas medições do backend.

#### Scenario: Perfil sustentado é executado
- **WHEN** o teste representativo opera com o volume, a concorrência e a duração definidos
- **THEN** as latências por percentil e a taxa de erros atendem às metas e nenhuma verificação de isolamento ou auditoria é relaxada para alcançá-las

#### Scenario: Pico temporário é executado
- **WHEN** o benchmark aplica 60 requisições por segundo durante cinco minutos
- **THEN** o backend permanece funcional, não perde eventos de mutações confirmadas e se recupera sem intervenção após o pico

### Requirement: Projeções versionadas
Cada versão publicada da entidade MUST definir uma projeção padrão de lista e outra de detalhe. A ausência de `select` MUST aplicar a projeção correspondente; um `select` explícito MUST retornar exatamente os campos de negócio solicitados e autorizados, além de `id` e `meta`. A API MUST NOT admitir wildcard, alias, agregação, expressão calculada ou seleção aninhada no MVP.

#### Scenario: Listagem omite select
- **WHEN** um consumidor autorizado solicita a coleção sem informar `select`
- **THEN** a plataforma aplica a projeção padrão de lista da versão ativa

#### Scenario: Seleção explícita é autorizada
- **WHEN** um consumidor solicita uma lista válida de campos legíveis
- **THEN** a plataforma retorna esses campos e inclui `id` e `meta` independentemente de terem sido listados

#### Scenario: Campo não autorizado é solicitado
- **WHEN** um `select` explícito contém campo cuja leitura não é permitida
- **THEN** a plataforma rejeita a consulta sem omitir silenciosamente o campo

### Requirement: Ordenação explícita e determinística
O atributo `order` MUST ser uma lista em ordem de prioridade. Cada item MUST conter `field`, `direction` igual a `asc` ou `desc` e MAY conter `nulls` igual a `first` ou `last`; a ausência de `nulls` MUST significar `last`. A plataforma MUST NOT aceitar função, expressão ou collation escolhida pela requisição. O UUIDv7 MUST ser acrescentado automaticamente como desempate final na mesma direção da última chave explícita.

#### Scenario: Ordenação por múltiplos campos
- **WHEN** a DSL informa até três campos escalares em uma combinação publicada e autorizada
- **THEN** a plataforma preserva a prioridade da lista, aplica a posição configurada dos nulos e acrescenta o UUIDv7 ao final

#### Scenario: Ordenação é omitida
- **WHEN** uma listagem não informa `order`
- **THEN** a plataforma usa a ordenação padrão da versão publicada ou, se ausente, `id desc`

#### Scenario: Expressão de ordenação é enviada
- **WHEN** o consumidor informa função, expressão, relacionamento, texto extenso, JSON, coleção, binário ou campo calculado não persistido
- **THEN** a plataforma rejeita a ordenação como não suportada no MVP

### Requirement: Capacidades e perfis de consulta publicados
O runtime MUST aceitar projeção, filtro, ordenação e pesquisa por padrão somente conforme as capacidades `selectable`, `filterable`, `sortable` e `patternSearch` da versão publicada. Toda tabela dinâmica MUST possuir índice base para registros ativos iniciado por `(organization_id, id)`. Ordenação simples habilitada MUST possuir índice compatível; combinação de duas ou três chaves MUST corresponder a um perfil de consulta publicado e a um índice composto iniciado por `organization_id` e terminado pelo UUIDv7.

#### Scenario: Combinação de ordenação não publicada
- **WHEN** uma consulta combina campos individualmente ordenáveis sem um perfil composto correspondente
- **THEN** o runtime rejeita a combinação sem criar índice automaticamente

#### Scenario: Perfil ainda não possui índice validado
- **WHEN** a estrutura física necessária a um perfil não foi criada e validada
- **THEN** a versão da entidade não disponibiliza esse perfil às consultas

### Requirement: Paginação determinística por cursor
A listagem de registros dinâmicos MUST usar keyset/seek pagination com ordenação determinística. O UUIDv7 do registro MUST ser o desempate final quando a ordenação contiver outros campos. A requisição MUST aceitar `limit` e no máximo um entre `after` e `before`; a resposta MUST expor em `meta.page` os cursores e indicadores de continuidade aplicáveis. A API MUST NOT oferecer salto por número de página nesse recurso.

#### Scenario: Próxima página é solicitada
- **WHEN** o consumidor devolve `nextCursor` como `after` sem alterar os demais parâmetros vinculados
- **THEN** a plataforma continua após a última chave de ordenação da página anterior sem percorrer as linhas anteriores por offset

#### Scenario: Valores de ordenação são iguais
- **WHEN** dois registros possuem o mesmo valor nos campos escolhidos para ordenação
- **THEN** a plataforma usa o UUIDv7 como desempate e mantém uma ordem total reproduzível para a continuação

#### Scenario: Consumidor tenta combinar direções
- **WHEN** uma requisição informa simultaneamente `after` e `before`
- **THEN** a plataforma rejeita a paginação como inválida

### Requirement: Cursor opaco e sem estado no servidor
O cursor público MUST ser um token opaco, stateless, versionado, serializado de forma binária compacta, protegido por criptografia autenticada e codificado em Base64 URL-safe. Ele SHOULD possuir até 512 caracteres e MUST NOT exceder 1.024. O backend MUST NOT persistir uma sessão ou registro por cursor e MUST revalidar autenticação, organização e autorização em cada página. O formato do MVP MUST NOT aplicar Gzip.

O cursor MUST conter somente versão, expiração, direção, valores posicionais, UUIDv7 de desempate e metadados criptográficos indispensáveis, nunca a chave secreta. Um fingerprint criptográfico de 16 bytes MUST vinculá-lo à organização, ao sujeito autenticado, à entidade, à versão publicada, à projeção, aos filtros e à ordenação sem copiar integralmente esses dados. `limit` MUST NOT integrar o fingerprint, mas MUST continuar sujeito ao máximo permitido.

#### Scenario: Cursor é reutilizado em outra consulta
- **WHEN** um consumidor apresenta o cursor com usuário, filtros, ordenação, entidade, versão ou organização incompatíveis
- **THEN** a plataforma rejeita o token sem executar a continuação

#### Scenario: Cursor é adulterado ou expirou
- **WHEN** a autenticidade ou a validade temporal do token não pode ser confirmada
- **THEN** a plataforma rejeita o cursor e não usa seus valores na consulta

#### Scenario: Permissão muda entre páginas
- **WHEN** o usuário perde uma permissão depois de receber um cursor válido
- **THEN** a próxima requisição aplica as permissões atuais e não trata o cursor como autorização

#### Scenario: Consulta mantém autorização após mudança de perfil
- **WHEN** os perfis do usuário mudam mas todas as permissões exigidas pela consulta continuam válidas
- **THEN** a plataforma permite a continuação enquanto os demais vínculos e a validade do cursor permanecerem compatíveis

### Requirement: Validade e renovação por navegação
Cada cursor MUST expirar 30 minutos após sua emissão, sem renovação direta. Cada página obtida com sucesso MUST emitir novos cursores com uma nova validade de 30 minutos. A autenticação válida MUST continuar sendo exigida independentemente da validade do cursor.

#### Scenario: Cursor expirado é apresentado
- **WHEN** transcorreram mais de 30 minutos desde a emissão do token
- **THEN** a plataforma responde com Problem Details de cursor expirado e exige reinício da consulta

#### Scenario: Versão da entidade muda entre páginas
- **WHEN** outra versão da entidade é ativada depois da emissão do cursor
- **THEN** a plataforma responde com Problem Details de cursor obsoleto e exige reinício da consulta

#### Scenario: Página é obtida dentro da validade
- **WHEN** uma continuação válida retorna uma nova página
- **THEN** os novos cursores dessa resposta recebem sua própria janela de 30 minutos

### Requirement: Erros distinguíveis de cursor
A plataforma MUST distinguir por tipos de `application/problem+json` ao menos cursor inválido ou incompatível, cursor expirado e cursor obsoleto por mudança de versão. Falhas de sessão e de permissão MUST continuar usando os erros normais de autenticação e autorização.

#### Scenario: Token foi adulterado
- **WHEN** a criptografia autenticada não valida o conteúdo do cursor
- **THEN** a plataforma responde como cursor inválido sem revelar detalhes criptográficos

### Requirement: Limites de ordenação e contagem
A plataforma MUST restringir a ordenação genérica a campos escalares suportados e a no máximo três chaves explícitas, acrescentando o UUIDv7 como desempate final. Textos extensos, JSON e estruturas de tamanho imprevisível MUST NOT compor o cursor genérico do MVP. A publicação do perfil MUST validar o pior tamanho possível das posições e impedir combinações que possam produzir cursor acima do limite. A plataforma MUST NOT calcular a contagem exata total implicitamente em toda página. O contrato SHOULD manter a URL completa do `GET` preferencialmente abaixo de 2.048 caracteres e MUST disponibilizar o `POST .../records/query` para projeções ou consultas que não se ajustem a esse perfil.

#### Scenario: Ordenação produziria cursor sem limite previsível
- **WHEN** uma consulta tenta ordenar por campo extenso ou por mais chaves que o limite permitido
- **THEN** a plataforma rejeita a ordenação e informa os campos ou limites suportados

#### Scenario: Listagem comum é executada
- **WHEN** um consumidor solicita uma página sem pedir contagem total explícita
- **THEN** a plataforma retorna os itens e indicadores de continuidade sem executar contagem exata de todos os registros

#### Scenario: Cursor excederia o limite
- **WHEN** os valores da ordenação não podem ser representados em cursor com até 1.024 caracteres
- **THEN** a plataforma rejeita a ordenação antes de emitir um cursor inválido ou truncado

### Requirement: Entidade publicada e habilitada
A plataforma MUST permitir operações de negócio somente em entidades que estejam publicadas globalmente e habilitadas para a organização atual.

#### Scenario: Operação em entidade não habilitada
- **WHEN** um usuário solicita uma operação sobre uma entidade não habilitada em sua organização
- **THEN** a plataforma rejeita a operação

### Requirement: Integridade de relacionamentos
A plataforma MUST validar que referências entre registros sejam compatíveis com as relações publicadas e não atravessem organizações indevidamente. Cada relacionamento físico entre entidades dinâmicas MUST referenciar `(organization_id, id)`, usar `ON DELETE RESTRICT` e `ON UPDATE RESTRICT` e possuir índice correspondente iniciado por `organization_id`.

#### Scenario: Referência entre organizações
- **WHEN** um registro tenta referenciar um registro pertencente a outra organização
- **THEN** a plataforma rejeita a operação

#### Scenario: Exclusão física de registro referenciado
- **WHEN** uma rotina privilegiada tenta excluir fisicamente um registro que ainda possui referências
- **THEN** a chave estrangeira bloqueia a exclusão sem remover ou alterar automaticamente os registros dependentes

### Requirement: Execução das cardinalidades publicadas
O runtime MUST armazenar relações `N:1` na entidade de origem, resolver `1:N` por consulta inversa, garantir `1:1` por unicidade organizacional e tratar `N:N` por meio dos registros de uma entidade associativa.

#### Scenario: Consulta da direção um-para-muitos
- **WHEN** um consumidor autorizado solicita os pedidos relacionados a um cliente
- **THEN** a plataforma consulta a FK de cliente nos pedidos sem depender de uma coleção duplicada no registro do cliente

#### Scenario: Segunda associação um-para-um
- **WHEN** uma operação tenta associar ao mesmo alvo um segundo registro em uma relação `1:1`
- **THEN** a plataforma rejeita a operação pela unicidade de `(organization_id, campo_de_referencia)`

#### Scenario: Consulta muitos-para-muitos
- **WHEN** um consumidor consulta os projetos relacionados a um usuário por uma relação `N:N`
- **THEN** a plataforma atravessa os registros autorizados da entidade associativa e aplica suas regras de organização, exclusão lógica e permissão

### Requirement: Validação do estado do alvo
Uma referência nova ou substituída MUST apontar para um registro ativo. Uma atualização que preserve uma referência histórica já existente para um alvo logicamente excluído MUST continuar possível quando todas as demais regras forem atendidas.

#### Scenario: Campo não relacionado é alterado
- **WHEN** um usuário altera outro campo de um registro cuja referência histórica aponta para alvo excluído e não modifica essa referência
- **THEN** a plataforma permite a alteração sem reativar ou substituir automaticamente o alvo

#### Scenario: Seletor de relacionamento é consultado
- **WHEN** a interface solicita candidatos para estabelecer uma relação
- **THEN** a plataforma retorna somente registros ativos da organização atual que o usuário pode consultar

### Requirement: Autorreferência sem semântica hierárquica
O runtime MUST tratar a autorreferência opcional do MVP como uma referência comum e MUST NOT inferir ordenação em árvore, aciclicidade ou consultas de ancestrais e descendentes.

#### Scenario: Grafo autorreferenciado é consultado
- **WHEN** registros possuem referências opcionais para a própria entidade
- **THEN** a plataforma retorna os vínculos autorizados sem apresentá-los como uma hierarquia garantida

### Requirement: Exclusão lógica de registros
A operação regular de exclusão MUST marcar o registro com `deleted_at`, incrementar sua versão otimista e registrar a ação na auditoria, sem remover fisicamente a linha. Criações, alterações, exclusões lógicas e restaurações MUST auditar as chaves lógicas dos campos afetados e as versões envolvidas, sem copiar valores anteriores ou posteriores. A API regular e as ferramentas da IA MUST NOT oferecer exclusão física.

#### Scenario: Usuário exclui registro
- **WHEN** um usuário autorizado confirma a exclusão de um registro ativo
- **THEN** a plataforma realiza a exclusão lógica, registra o evento e omite o registro das consultas normais

#### Scenario: Consulta histórica autorizada
- **WHEN** um consumidor autorizado solicita explicitamente um registro excluído para fins históricos ou administrativos
- **THEN** a plataforma pode retorná-lo dentro do mesmo contexto organizacional e das permissões aplicáveis

#### Scenario: Nova referência a registro excluído
- **WHEN** uma criação ou alteração tenta estabelecer uma nova referência para um registro logicamente excluído
- **THEN** a plataforma rejeita a operação

#### Scenario: Referência histórica existente
- **WHEN** um registro ativo já referencia um registro posteriormente excluído de forma lógica
- **THEN** a plataforma preserva o vínculo e permite sua resolução apenas nos fluxos históricos autorizados

#### Scenario: Reutilização de chave reservada
- **WHEN** a criação de outro registro tenta reutilizar uma chave de negócio única pertencente a registro excluído
- **THEN** a plataforma rejeita a duplicidade e orienta o fluxo de restauração controlada

### Requirement: Importação massiva controlada
A plataforma MUST processar importações de alto volume por um fluxo de staging controlado e MUST aplicar contexto organizacional, autorização, validação, integridade, RLS e auditoria antes de promover os registros para as tabelas finais. Cada bloco confirmado MUST gravar na mesma transação um evento resumido referenciado pelo `import_id`, enquanto o detalhamento por linha permanece nas estruturas próprias da importação.

#### Scenario: Promoção de lote validado
- **WHEN** todas as linhas selecionadas de um lote passam pelas validações aplicáveis
- **THEN** a plataforma as insere em conjunto nas tabelas finais sob o contexto confiável da organização e registra na mesma transação um resumo auditável do bloco

#### Scenario: Organização divergente no lote
- **WHEN** uma linha ou referência tenta usar uma organização diferente daquela fixada pelo job de importação
- **THEN** a plataforma rejeita a promoção dessa informação e registra o erro sem permitir acesso cruzado

### Requirement: Retomada idempotente de importação
A plataforma MUST conservar uma chave de origem e os UUIDv7 já atribuídos enquanto uma importação puder ser retomada, para que uma repetição identificada não crie duplicatas nem altere relacionamentos já resolvidos.

#### Scenario: Repetição de um bloco confirmado
- **WHEN** o importador repete um bloco com o mesmo identificador de importação e as mesmas chaves de origem
- **THEN** a plataforma reconhece os registros já processados e não os cria novamente
