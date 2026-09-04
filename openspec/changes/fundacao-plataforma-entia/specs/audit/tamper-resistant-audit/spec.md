## Purpose

Registrar eventos relevantes em uma trilha append-only no PostgreSQL, com rastreabilidade e atomicidade em relação às mutações de negócio, sem introduzir encadeamento criptográfico no MVP.

## ADDED Requirements

### Requirement: Registro append-only
A plataforma MUST registrar eventos de auditoria de forma append-only e MUST NOT oferecer operações de aplicação que alterem, excluam ou trunquem eventos já gravados. As roles usadas pelos fluxos normais MUST NOT ser proprietárias da tabela nem receber privilégios de `UPDATE`, `DELETE` ou `TRUNCATE` sobre ela.

#### Scenario: Tentativa de alteração pela aplicação
- **WHEN** um usuário ou integração tenta modificar ou excluir um evento de auditoria existente
- **THEN** a plataforma rejeita a operação

#### Scenario: Tentativa direta com a role de runtime
- **WHEN** a role usada por uma operação regular tenta atualizar, excluir ou truncar eventos de auditoria
- **THEN** o PostgreSQL rejeita a instrução por falta de privilégio

### Requirement: Gravação síncrona e atômica de mutações
Toda mutação concluída sobre dados de negócio, catálogo ou administração MUST inserir seu evento de auditoria de forma síncrona, na mesma conexão e transação PostgreSQL da alteração auditada. Se a inserção do evento falhar, a mutação MUST ser revertida. Fila, outbox ou processamento assíncrono MUST NOT ser o único mecanismo de auditoria dessas mutações no MVP.

#### Scenario: Evento gravado com a mutação
- **WHEN** uma mutação e seu evento de auditoria são executados com sucesso
- **THEN** ambos são confirmados pelo mesmo commit

#### Scenario: Falha ao gravar o evento
- **WHEN** a alteração de negócio é válida, mas a inserção do evento de auditoria falha
- **THEN** a transação é revertida e a alteração de negócio não é confirmada

### Requirement: Inserções independentes no MVP
Cada evento de auditoria MUST possuir identificador UUIDv7 próprio e sua inserção MUST NOT depender da atualização de uma cabeça compartilhada, sequência serial da trilha, hash do evento anterior ou checkpoint criptográfico. Encadeamento de hashes e ancoragem externa ficam fora do MVP.

#### Scenario: Escritas concorrentes
- **WHEN** transações concorrentes registram eventos independentes
- **THEN** nenhuma delas precisa bloquear uma linha compartilhada para determinar o evento anterior da trilha

### Requirement: Conteúdo rastreável do evento
Cada evento MUST registrar em campos estruturados, quando aplicável: UUIDv7 do evento, organização, ator e tipo de ator, provedor de identidade, sessão rastreável, origem, ação, resultado, tipo e identificação do alvo, chave lógica da entidade, versões anterior e posterior, instante do servidor, identificadores de requisição e correlação e referências especializadas como conversa ou importação. Detalhes adicionais MUST ser pequenos, validados e sanitizados.

#### Scenario: Alteração de registro dinâmico concluída
- **WHEN** uma alteração de registro é concluída
- **THEN** a plataforma grava na mesma transação um evento que permite identificar quem executou a ação, em qual organização, sessão e origem, sobre qual alvo, quais chaves lógicas de campos foram alteradas e com qual resultado

#### Scenario: Exclusão lógica de registro dinâmico concluída
- **WHEN** uma exclusão lógica é concluída
- **THEN** a plataforma registra na mesma transação ator, organização, entidade, identificador do registro, resultado e correlação sem remover eventos anteriores

### Requirement: Auditoria sem valores dos campos no MVP
A auditoria de registros dinâmicos MUST registrar somente as chaves lógicas dos campos alterados e as versões anterior e posterior do registro. Ela MUST NOT armazenar valores anteriores ou posteriores, corpo completo da requisição, query string integral, headers completos, cookies, credenciais, tokens, prompts ou respostas completas da LLM.

#### Scenario: Campo sensível é alterado
- **WHEN** uma mutação modifica um campo de negócio, inclusive um campo potencialmente sensível
- **THEN** o evento identifica a chave lógica do campo sem copiar seu valor anterior ou posterior

### Requirement: Cobertura de eventos relevantes
A plataforma MUST auditar mutações de registros, catálogo e administração; início, troca de organização, encerramento e revogação de sessões; ações relevantes negadas pela autorização; consultas privilegiadas; e operações originadas pela IA. Consultas comuns de listagem ou detalhe, erros usuais de validação e falhas técnicas MUST permanecer em logs operacionais e métricas, salvo quando configurados como eventos de segurança relevantes.

#### Scenario: Publicação de entidade
- **WHEN** uma versão de entidade é publicada ou ativada
- **THEN** a plataforma registra o ator, a entidade, a versão, o instante e o resultado

#### Scenario: Consulta comum de registro
- **WHEN** um usuário autorizado lista ou visualiza normalmente registros de uma entidade
- **THEN** a plataforma não cria evento individual de auditoria apenas por essa leitura

#### Scenario: Consulta privilegiada
- **WHEN** um usuário consulta a auditoria, registros logicamente excluídos ou outro recurso classificado como leitura privilegiada
- **THEN** a plataforma registra a consulta com ator, organização, sessão, origem e correlação

### Requirement: Sessão e origem rastreáveis
Após autenticação, o BFF MUST atribuir à sessão um `session_trace_id` UUIDv7 interno, não secreto e diferente do cookie, dos tokens e dos identificadores reutilizáveis como credencial. Cada evento originado por usuário MUST registrar esse identificador e o contexto organizacional efetivo. Eventos de sessão MUST permitir correlacionar início, troca de organização, encerramento e revogação sem depender da permanência da sessão ativa.

Para cada requisição auditada, a plataforma MUST registrar o endereço IP normalizado, o User-Agent sanitizado e limitado a 512 caracteres, o cliente OIDC quando aplicável, a origem funcional, o método HTTP, a rota normalizada sem parâmetros sensíveis, o identificador da requisição e a correlação. O endereço IP e o User-Agent MUST ser tratados como evidência complementar, não como prova de identidade.

#### Scenario: Mutação durante sessão autenticada
- **WHEN** um usuário autenticado realiza uma mutação auditada
- **THEN** o evento referencia o `session_trace_id` não secreto e registra a origem observada para aquela requisição

#### Scenario: Sessão expira após a operação
- **WHEN** a sessão associada a eventos anteriores expira ou é removida do armazenamento operacional
- **THEN** os eventos conservam o `session_trace_id` e continuam correlacionáveis sem armazenar a credencial da sessão

### Requirement: Iniciador e executor distinguíveis
Operações delegadas MUST distinguir o ator que iniciou ou confirmou a ação do componente que a executou. Mutações pela IA MUST reutilizar o evento da operação REST, marcá-lo com origem `AI` e registrar sessão de confirmação, executor, conversa, ferramenta e aprovação, sem duplicar o evento de negócio. Jobs e importações MUST preservar a referência ao iniciador quando existir, mesmo que sua sessão já tenha expirado.

#### Scenario: IA executa mutação confirmada
- **WHEN** o orquestrador executa uma mutação aprovada por um usuário
- **THEN** o evento identifica o usuário e sua sessão como origem, o orquestrador como executor e a conversa e aprovação como correlação

### Requirement: Granularidade de importações massivas
Uma importação massiva MUST registrar na mesma transação um evento resumido por bloco promovido, contendo `import_id`, organização, entidade, identificação do bloco, quantidades relevantes, resultado e correlação. A auditoria MUST NOT criar obrigatoriamente um evento por registro importado; o detalhamento por linha MUST permanecer nas estruturas próprias da importação e ser recuperável pelo `import_id`.

#### Scenario: Bloco de importação é promovido
- **WHEN** um bloco validado é confirmado nas tabelas finais
- **THEN** a mesma transação grava um evento resumido que referencia o detalhamento persistido da importação

### Requirement: Registro síncrono de ações rejeitadas
Ações relevantes rejeitadas antes de qualquer mutação MUST ser registradas de forma síncrona em uma transação curta própria. Quando uma transação já estiver invalidada e precisar de rollback, o MVP MUST registrar a falha em log operacional estruturado e MUST NOT alegar que um evento foi preservado dentro da transação revertida.

#### Scenario: Acesso negado antes da mutação
- **WHEN** uma ação protegida é rejeitada pela autorização antes de alterar estado de negócio
- **THEN** a tentativa é registrada sincronamente em uma transação curta e a resposta somente é concluída após o resultado dessa gravação

#### Scenario: Transação invalidada por falha interna
- **WHEN** uma falha invalida a transação de negócio antes do commit
- **THEN** as alterações e o evento transacional são revertidos e a falha permanece no log operacional estruturado do MVP

### Requirement: Consulta protegida
O acesso à auditoria MUST respeitar escopo organizacional e permissões administrativas, distinguindo eventos globais de eventos pertencentes a uma organização.

#### Scenario: Consulta organizacional
- **WHEN** um administrador organizacional autorizado consulta a auditoria
- **THEN** a plataforma retorna somente eventos visíveis no contexto de sua organização

### Requirement: Independência do ciclo de exclusão
Eventos de auditoria MUST NOT ser removidos em cascata quando usuários, organizações, entidades ou registros de negócio forem desativados, descontinuados, excluídos logicamente ou expurgados.

#### Scenario: Alvo auditado é excluído
- **WHEN** um alvo associado a eventos existentes passa por exclusão lógica ou futuro expurgo autorizado
- **THEN** os eventos permanecem disponíveis e conservam a identificação histórica necessária

### Requirement: Retenção sem expurgo automático no MVP
O MVP MUST manter os eventos de auditoria sem expurgo automático e MUST monitorar o crescimento da tabela e de seus índices. Uma futura política de retenção, exportação ou expurgo MUST ser explícita, administrativa, autorizada e independente do ciclo de exclusão dos alvos auditados.

#### Scenario: Evento envelhece no MVP
- **WHEN** um evento ultrapassa qualquer idade operacional durante o MVP
- **THEN** ele permanece armazenado e não é removido por job automático de retenção
