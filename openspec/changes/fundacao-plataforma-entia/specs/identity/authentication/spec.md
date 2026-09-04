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

### Requirement: Descoberta protegida da organização antes do login
A plataforma MUST permitir iniciar o acesso por um endereço organizacional estável ou por uma entrada genérica baseada em e-mail, sem apresentar uma listagem pública e irrestrita das organizações. Quando a descoberta depender de vínculos individuais, a plataforma MUST comprovar a posse do e-mail antes de revelar ou selecionar as organizações associadas. Um domínio público e inequivocamente associado a uma única organização MAY permitir encaminhamento direto sem consultar vínculos individuais.

#### Scenario: Acesso por endereço organizacional
- **WHEN** o usuário acessa `/o/{organizationSlug}` para uma organização ativa
- **THEN** a plataforma resolve a organização, apresenta sua experiência pública e prepara o provedor de identidade correspondente

#### Scenario: Descoberta individual por e-mail
- **WHEN** o usuário informa um e-mail cuja resolução exige consultar vínculos individuais
- **THEN** a plataforma apresenta uma resposta genérica e envia um código ou link de descoberta de uso único antes de mostrar qualquer organização associada

#### Scenario: E-mail associado a várias organizações
- **WHEN** a posse do e-mail foi comprovada e existem vários vínculos elegíveis
- **THEN** a plataforma apresenta somente essas organizações para seleção e não concede acesso antes da autenticação pelo provedor aplicável

#### Scenario: E-mail desconhecido
- **WHEN** um e-mail sem associação é informado na entrada genérica
- **THEN** a resposta pública permanece indistinguível daquela apresentada para um e-mail conhecido

### Requirement: Identidade externa independente do e-mail
A plataforma MUST identificar uma conta externa pela combinação estável entre emissor e subject e MUST NOT unir automaticamente contas de provedores diferentes apenas pela igualdade do e-mail.

#### Scenario: Mesmo e-mail em provedores diferentes
- **WHEN** duas identidades externas apresentam o mesmo endereço de e-mail
- **THEN** elas permanecem distintas até uma vinculação explícita que comprove ambas as identidades ou uma ação administrativa autorizada e auditada

### Requirement: Credenciais fora do núcleo da plataforma
O núcleo da ENT.IA MUST NOT armazenar senhas ou hashes de senhas dos usuários e MUST delegar ao provedor de identidade os fluxos de credencial e recuperação de acesso.

#### Scenario: Recuperação de acesso
- **WHEN** um usuário solicita a recuperação de acesso
- **THEN** a plataforma direciona o fluxo ao provedor de identidade responsável sem manipular a senha do usuário

### Requirement: Sessão vinculada à identidade e à organização
A plataforma MUST manter sessões autenticadas vinculadas à identidade global e a um contexto organizacional válido. O BFF MUST atribuir a cada sessão autenticada um `session_trace_id` UUIDv7 interno e não secreto, diferente do cookie, dos tokens e de qualquer valor que permita reutilizar a sessão.

#### Scenario: Sessão autenticada é criada
- **WHEN** o login é concluído e o BFF cria uma sessão
- **THEN** a plataforma associa um novo `session_trace_id` à sessão e o disponibiliza internamente para correlação de auditoria sem expô-lo como credencial

#### Scenario: Troca de organização
- **WHEN** um usuário autenticado seleciona outra organização à qual possui vínculo ativo
- **THEN** a plataforma altera o contexto organizacional e reavalia todas as autorizações aplicáveis

#### Scenario: Organização sem vínculo
- **WHEN** um usuário tenta estabelecer sessão no contexto de uma organização sem vínculo ativo
- **THEN** a plataforma rejeita o acesso

#### Scenario: Troca para organização com outro provedor obrigatório
- **WHEN** um usuário autenticado seleciona outra organização cuja política exige um provedor diferente
- **THEN** a plataforma reavalia a sessão e solicita nova autenticação quando necessário antes de ativar o novo contexto
