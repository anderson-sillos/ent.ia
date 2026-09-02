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

### Requirement: Entidade publicada e habilitada
A plataforma MUST permitir operações de negócio somente em entidades que estejam publicadas globalmente e habilitadas para a organização atual.

#### Scenario: Operação em entidade não habilitada
- **WHEN** um usuário solicita uma operação sobre uma entidade não habilitada em sua organização
- **THEN** a plataforma rejeita a operação

### Requirement: Integridade de relacionamentos
A plataforma MUST validar que referências entre registros sejam compatíveis com as relações publicadas e não atravessem organizações indevidamente.

#### Scenario: Referência entre organizações
- **WHEN** um registro tenta referenciar um registro pertencente a outra organização
- **THEN** a plataforma rejeita a operação
