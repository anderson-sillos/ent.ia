## Purpose

Definir o funcionamento multi-organização da instalação e garantir que usuários, configurações e dados organizacionais permaneçam corretamente segregados em todos os fluxos.

## ADDED Requirements

### Requirement: Múltiplas organizações na mesma instalação
A plataforma MUST permitir o cadastro e a operação de múltiplas organizações independentes em uma mesma instalação.

#### Scenario: Cadastro de organização
- **WHEN** um administrador global cadastra uma nova organização válida
- **THEN** a plataforma cria um contexto organizacional independente e apto a receber usuários e configurações

### Requirement: Endereço organizacional estável
Cada organização MUST possuir um `organizationSlug` público, único e estável, adequado para compor seu endereço de acesso. A arquitetura MUST permitir que endereços adicionais sejam resolvidos para o mesmo contexto sem mudar o contrato de sessão; subdomínios e domínios personalizados MUST NOT ser provisionados no MVP.

#### Scenario: Resolução pelo slug
- **WHEN** uma requisição usa um `organizationSlug` pertencente a uma organização ativa
- **THEN** a plataforma resolve internamente a organização sem tratar o slug como autorização de acesso aos seus dados

### Requirement: Identidade visual por organização
Cada organização MUST poder selecionar um modelo visual publicado e configurar somente as personalizações declarativas permitidas, sem alterar a identidade visual das demais organizações.

#### Scenario: Organização seleciona um tema
- **WHEN** um administrador organizacional autorizado ativa um modelo visual válido
- **THEN** a configuração passa a ser usada somente nas experiências pertencentes àquela organização

### Requirement: Isolamento organizacional
Todo dado pertencente a uma organização MUST ser identificado por seu contexto organizacional, e a plataforma MUST impedir leitura ou alteração a partir de outra organização.

#### Scenario: Tentativa de acesso cruzado
- **WHEN** uma operação executada no contexto de uma organização referencia dados pertencentes a outra
- **THEN** a plataforma rejeita a operação sem revelar o conteúdo protegido

### Requirement: Vínculos entre usuários e organizações
A plataforma MUST administrar vínculos independentes entre identidades globais e organizações, incluindo o estado ativo ou inativo de cada vínculo.

#### Scenario: Desativação de vínculo
- **WHEN** um administrador autorizado desativa o vínculo de um usuário com uma organização
- **THEN** o usuário deixa de acessar aquela organização sem perder seus vínculos com outras organizações

### Requirement: Ativação de entidades por organização
Cada organização MUST poder habilitar ou desabilitar para seu próprio uso as entidades globais publicadas, sem alterar a definição compartilhada dessas entidades.

#### Scenario: Entidade habilitada
- **WHEN** um administrador organizacional habilita uma entidade global publicada
- **THEN** a entidade se torna disponível na organização conforme as permissões de seus usuários

#### Scenario: Entidade desabilitada
- **WHEN** uma entidade está desabilitada para uma organização
- **THEN** a plataforma impede novas operações organizacionais sobre essa entidade
