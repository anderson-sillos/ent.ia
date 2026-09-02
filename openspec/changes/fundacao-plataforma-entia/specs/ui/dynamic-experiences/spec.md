## Purpose

Apresentar formulários, detalhes, listas, filtros e ações a partir dos metadados publicados, oferecendo uma experiência consistente sem telas específicas para cada entidade.

## ADDED Requirements

### Requirement: Formulários dinâmicos
A interface MUST montar experiências de cadastro, visualização e edição a partir da definição ativa e dos metadados de apresentação da entidade.

#### Scenario: Abertura de cadastro
- **WHEN** um usuário autorizado abre o cadastro de uma entidade habilitada
- **THEN** a interface apresenta campos, tipos, obrigatoriedade e opções correspondentes à definição ativa

### Requirement: Consultas dinâmicas
A interface MUST montar listagens e filtros de entidades a partir dos campos configurados como consultáveis na definição publicada.

#### Scenario: Consulta com filtros
- **WHEN** um usuário informa filtros válidos para uma entidade habilitada
- **THEN** a interface solicita e apresenta os registros correspondentes dentro de sua organização

### Requirement: Ações conforme autorização
A interface MUST apresentar somente ações compatíveis com as permissões efetivas do usuário no contexto da organização e da entidade.

#### Scenario: Usuário possui apenas leitura
- **WHEN** um usuário acessa uma entidade para a qual possui somente permissão de leitura
- **THEN** a interface não disponibiliza ações de criação, alteração ou exclusão

### Requirement: Feedback de validação
A interface MUST apresentar erros de validação de maneira associada aos campos ou à operação correspondente, preservando o servidor como autoridade final da validação.

#### Scenario: Servidor rejeita formulário
- **WHEN** o servidor retorna erros de validação para uma operação dinâmica
- **THEN** a interface informa os erros de forma compreensível e mantém os dados editáveis do usuário

### Requirement: Atualização da versão ativa
A interface MUST obter os metadados da versão global ativa e MUST evitar operar com uma definição obsoleta após detectar sua substituição.

#### Scenario: Versão muda durante o uso
- **WHEN** a interface detecta que a versão usada para montar uma operação não é mais a versão ativa
- **THEN** ela atualiza os metadados antes de permitir nova submissão
