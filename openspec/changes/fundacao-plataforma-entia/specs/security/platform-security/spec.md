## Purpose

Definir controles transversais que protejam identidades, organizações, metadados e operações dinâmicas contra acesso indevido, entradas maliciosas e configurações inseguras.

## ADDED Requirements

### Requirement: Contexto organizacional confiável
A plataforma MUST determinar e validar o contexto organizacional no servidor e MUST NOT confiar exclusivamente em identificadores de organização fornecidos pelo cliente.

#### Scenario: Cliente manipula a organização
- **WHEN** um cliente altera um identificador para tentar operar no contexto de outra organização
- **THEN** o servidor rejeita a operação antes de acessar os dados solicitados

### Requirement: Isolamento organizacional por RLS
Todas as tabelas persistentes que armazenem dados organizacionais e sejam acessadas pelos fluxos regulares MUST usar PostgreSQL Row-Level Security como defesa adicional aos filtros e à autorização do backend. As políticas MUST restringir leitura e escrita ao contexto organizacional confiável da transação e MUST negar acesso quando esse contexto estiver ausente ou inválido. Áreas transitórias de staging que não suportem RLS MUST permanecer isoladas e sua promoção para tabelas finais MUST ocorrer sob RLS.

#### Scenario: Operação sem contexto organizacional
- **WHEN** a role de runtime tenta consultar ou alterar dados organizacionais sem um contexto válido definido para a transação
- **THEN** o PostgreSQL não disponibiliza linhas e não aceita a escrita

#### Scenario: Escrita com organização divergente
- **WHEN** uma operação tenta persistir `organization_id` diferente do contexto confiável da transação
- **THEN** a política de RLS rejeita a escrita mesmo que exista falha no filtro da aplicação

### Requirement: Role de runtime sem bypass
A role usada pelas requisições e jobs regulares MUST NOT ser proprietária das tabelas organizacionais, MUST NOT possuir `BYPASSRLS` e MUST NOT operar como superusuário. Credenciais de migration, schema dinâmico ou ingestão MUST NOT ser reutilizadas no runtime, e a role que promover importações para tabelas finais MUST permanecer submetida ao RLS.

#### Scenario: Consulta por caminho regular da aplicação
- **WHEN** uma requisição autenticada acessa uma tabela organizacional
- **THEN** o acesso ocorre por uma role submetida às políticas de RLS e com o contexto definido somente pelo backend confiável

### Requirement: Validação de entradas dinâmicas
A plataforma MUST tratar definições de metadados, filtros, ordenações e valores de registros como entradas não confiáveis e MUST validá-los antes de qualquer uso.

#### Scenario: Identificador malicioso em metadado
- **WHEN** uma definição contém um identificador ou expressão que viola as regras permitidas
- **THEN** a plataforma rejeita a definição sem executar o conteúdo fornecido

### Requirement: Autorização independente da interface
Toda operação protegida MUST ser autorizada no servidor mesmo que a interface já tenha ocultado ou bloqueado a ação.

#### Scenario: Chamada direta à API
- **WHEN** um usuário chama diretamente uma operação para a qual não possui permissão
- **THEN** a plataforma rejeita a chamada e registra o evento relevante

### Requirement: Proteção de informações sensíveis
A plataforma MUST evitar a exposição de credenciais, segredos, tokens e detalhes internos sensíveis em respostas, logs operacionais e mensagens de erro.

#### Scenario: Falha interna
- **WHEN** ocorre uma falha durante uma operação
- **THEN** a resposta ao cliente não expõe segredos nem detalhes internos que facilitem exploração

### Requirement: Configuração segura por padrão
Novas organizações, entidades, perfis e operações MUST iniciar sem concessões implícitas de acesso além das estritamente necessárias à administração autorizada.

#### Scenario: Nova entidade é publicada
- **WHEN** uma entidade é publicada globalmente
- **THEN** nenhuma organização ou usuário obtém automaticamente permissão de negócio sobre ela
