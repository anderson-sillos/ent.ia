## Purpose

Manter um catálogo global e versionado que descreva entidades, atributos, relações, validações e apresentação, servindo como contrato compartilhado para toda a plataforma.

## ADDED Requirements

### Requirement: Definições globais compartilhadas
A plataforma MUST manter uma única definição global de cada entidade de negócio, compartilhada por todas as organizações que a habilitarem.

#### Scenario: Organização habilita entidade existente
- **WHEN** uma organização habilita uma entidade publicada
- **THEN** ela passa a utilizar a mesma definição global ativa utilizada pelas demais organizações

### Requirement: Conteúdo da definição
Uma definição de entidade MUST representar ao menos sua identidade estável, atributos, tipos, obrigatoriedade, validações, relacionamentos e metadados necessários às experiências dinâmicas.

#### Scenario: Definição válida é consultada
- **WHEN** um consumidor autorizado consulta uma definição publicada
- **THEN** recebe metadados suficientes para validar operações e apresentar a entidade dinamicamente

### Requirement: Ciclo de vida de definição
A plataforma MUST distinguir definições em elaboração das versões publicadas e MUST validar uma definição antes de permitir sua publicação.

#### Scenario: Publicação de definição inválida
- **WHEN** um administrador global tenta publicar uma definição com inconsistências
- **THEN** a plataforma rejeita a publicação e informa os problemas encontrados

#### Scenario: Publicação de definição válida
- **WHEN** um administrador global publica uma definição válida
- **THEN** a plataforma cria uma versão publicada imutável dessa definição

### Requirement: Uma versão global ativa
Cada entidade MUST possuir no máximo uma versão global ativa, e todas as organizações que a utilizam MUST operar sobre essa mesma versão.

#### Scenario: Ativação de nova versão
- **WHEN** uma nova versão publicada é ativada
- **THEN** ela substitui globalmente a versão ativa anterior sem criar versões distintas por organização

### Requirement: Imutabilidade das versões publicadas
Uma versão publicada MUST NOT ser alterada; qualquer evolução da definição MUST produzir uma nova versão rastreável.

#### Scenario: Tentativa de editar versão publicada
- **WHEN** um administrador tenta alterar diretamente uma versão já publicada
- **THEN** a plataforma rejeita a alteração e preserva o conteúdo original
