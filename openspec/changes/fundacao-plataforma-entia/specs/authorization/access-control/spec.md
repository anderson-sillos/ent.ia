## Purpose

Controlar o acesso por organização, entidade e operação por meio de perfis organizacionais, mantendo negação por padrão e aplicação consistente das regras no servidor.

## ADDED Requirements

### Requirement: Perfis de acesso por organização
A plataforma MUST permitir que cada organização defina perfis que agrupem permissões por entidade e operação.

#### Scenario: Configuração de perfil
- **WHEN** um administrador organizacional associa operações permitidas de uma entidade a um perfil
- **THEN** essas permissões passam a valer somente dentro da organização proprietária do perfil

### Requirement: Associação de múltiplos perfis
Um vínculo de usuário com uma organização MUST poder receber múltiplos perfis de acesso, e sua permissão efetiva MUST ser a união das permissões concedidas por esses perfis.

#### Scenario: Permissão concedida por um dos perfis
- **WHEN** ao menos um perfil associado ao usuário concede a operação solicitada para a entidade no contexto atual
- **THEN** a autorização considera a operação permitida

### Requirement: Negação por padrão
A plataforma MUST negar toda operação que não esteja explicitamente concedida por um perfil aplicável, sem permissões diretas por usuário e sem regras explícitas de negação na versão inicial.

#### Scenario: Operação não concedida
- **WHEN** nenhum perfil aplicável concede ao usuário a operação solicitada
- **THEN** a plataforma rejeita a operação

### Requirement: Autorização contextual no servidor
A plataforma MUST avaliar no servidor a organização, a entidade, a operação, o vínculo do usuário e seus perfis antes de executar qualquer operação protegida.

#### Scenario: Interface apresenta uma ação indevida
- **WHEN** um cliente solicita uma operação que sua interface não deveria ter disponibilizado
- **THEN** o servidor realiza a avaliação completa e rejeita a operação não autorizada

### Requirement: Separação entre administração global e organizacional
Somente administradores globais autorizados MUST administrar definições globais de entidades, enquanto administradores organizacionais MUST limitar sua atuação à própria organização.

#### Scenario: Administrador organizacional tenta publicar entidade
- **WHEN** um administrador organizacional solicita a criação ou publicação de uma definição global
- **THEN** a plataforma rejeita a solicitação
