## Purpose

Estabelecer identidade global, autenticação federada e sessões seguras para que uma pessoa possa acessar uma ou mais organizações da ENT.IA sem manter credenciais no núcleo da plataforma.

## ADDED Requirements

### Requirement: Identidade global do usuário
A plataforma MUST manter uma única identidade global por usuário, permitindo que ela seja associada a uma ou mais organizações sem duplicação de credenciais.

#### Scenario: Usuário pertence a múltiplas organizações
- **WHEN** uma identidade autenticada possui vínculos ativos com mais de uma organização
- **THEN** a plataforma disponibiliza apenas as organizações vinculadas à identidade para seleção de contexto

### Requirement: Autenticação por provedor de identidade
A plataforma MUST delegar a autenticação a um provedor de identidade, aceitando um provedor configurado pela organização ou o provedor padrão da instalação quando a organização não possuir um provedor próprio.

#### Scenario: Organização possui provedor próprio
- **WHEN** o usuário inicia o acesso por uma organização com provedor de identidade configurado
- **THEN** a autenticação é encaminhada ao provedor dessa organização

#### Scenario: Organização não possui provedor próprio
- **WHEN** o usuário inicia o acesso por uma organização sem provedor de identidade configurado
- **THEN** a autenticação é encaminhada ao provedor padrão da instalação

### Requirement: Credenciais fora do núcleo da plataforma
O núcleo da ENT.IA MUST NOT armazenar senhas ou hashes de senhas dos usuários e MUST delegar ao provedor de identidade os fluxos de credencial e recuperação de acesso.

#### Scenario: Recuperação de acesso
- **WHEN** um usuário solicita a recuperação de acesso
- **THEN** a plataforma direciona o fluxo ao provedor de identidade responsável sem manipular a senha do usuário

### Requirement: Sessão vinculada à identidade e à organização
A plataforma MUST manter sessões autenticadas vinculadas à identidade global e a um contexto organizacional válido.

#### Scenario: Troca de organização
- **WHEN** um usuário autenticado seleciona outra organização à qual possui vínculo ativo
- **THEN** a plataforma altera o contexto organizacional e reavalia todas as autorizações aplicáveis

#### Scenario: Organização sem vínculo
- **WHEN** um usuário tenta estabelecer sessão no contexto de uma organização sem vínculo ativo
- **THEN** a plataforma rejeita o acesso
