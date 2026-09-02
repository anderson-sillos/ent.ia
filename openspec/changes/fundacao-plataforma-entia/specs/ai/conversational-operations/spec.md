## Purpose

Permitir que usuários consultem e alterem registros por linguagem natural, com uma LLM mediada pela plataforma e restrita às APIs REST, permissões e controles de segurança da ENT.IA.

## ADDED Requirements

### Requirement: Área de interação conversacional
A plataforma MUST oferecer uma área de chat na qual um usuário autenticado possa solicitar consultas e alterações de registros no contexto de sua organização ativa.

#### Scenario: Início de conversa
- **WHEN** um usuário autenticado inicia uma conversa em uma organização à qual possui vínculo ativo
- **THEN** a plataforma cria a conversa vinculada ao usuário e à organização atual

### Requirement: Provedor de LLM substituível
A capacidade conversacional MUST consumir a LLM por uma abstração que permita selecionar posteriormente o provedor e o modelo sem alterar os contratos REST das operações de negócio.

#### Scenario: Substituição do provedor
- **WHEN** a configuração seleciona outro provedor compatível com as capacidades exigidas
- **THEN** as operações conversacionais continuam utilizando os mesmos contratos e controles da ENT.IA

### Requirement: Operações exclusivamente pela API REST
A IA MUST consultar e alterar registros exclusivamente por operações autenticadas da API REST da ENT.IA e MUST NOT possuir acesso direto ao banco de dados, repositórios ou SQL livre.

#### Scenario: Consulta solicitada em linguagem natural
- **WHEN** a LLM identifica a necessidade de consultar registros
- **THEN** o orquestrador executa uma operação REST autorizada e retorna seu resultado controlado à conversa

#### Scenario: Solicitação de acesso direto
- **WHEN** uma resposta da LLM tenta fornecer SQL, uma URL arbitrária ou instruções de acesso direto ao armazenamento
- **THEN** a plataforma não executa esse conteúdo como operação de negócio

### Requirement: Ferramentas REST controladas
A plataforma MUST disponibilizar à LLM somente ferramentas derivadas de operações REST explicitamente habilitadas para IA, com nome, descrição e parâmetros estruturados e validados.

#### Scenario: Ferramenta não habilitada
- **WHEN** a LLM solicita uma ferramenta ou operação REST não habilitada para IA
- **THEN** o orquestrador rejeita a solicitação sem encaminhá-la à API de negócio

### Requirement: Execução em nome do usuário
Cada chamada REST originada pela IA MUST preservar a identidade do usuário, a identidade do componente executor e o contexto organizacional determinado pela sessão, sem aceitar que o prompt substitua esse contexto.

#### Scenario: Prompt tenta trocar de organização
- **WHEN** o prompt solicita acesso a uma organização diferente da sessão autorizada
- **THEN** a chamada permanece no contexto confiável da sessão e o acesso indevido é rejeitado

### Requirement: Autorização em todas as chamadas
Cada chamada REST originada pela IA MUST passar pelas mesmas verificações de vínculo, entidade, operação e perfil aplicadas a uma chamada convencional do usuário.

#### Scenario: Usuário solicita operação não autorizada
- **WHEN** a LLM propõe uma operação que o usuário não possui permissão para executar
- **THEN** a API rejeita a operação e a conversa informa que ela não foi realizada

### Requirement: Confirmação humana para mutações
Criações, alterações e exclusões propostas pela IA MUST exigir confirmação explícita do usuário sobre uma pré-visualização antes da chamada REST que efetiva a mutação.

#### Scenario: Usuário confirma alteração
- **WHEN** o usuário confirma uma proposta válida que apresenta alvo e alterações pretendidas
- **THEN** o orquestrador executa uma única chamada REST correspondente à proposta confirmada

#### Scenario: Usuário cancela alteração
- **WHEN** o usuário cancela ou deixa expirar uma proposta de alteração
- **THEN** nenhuma chamada REST de mutação é executada

### Requirement: Proteção contra repetição e concorrência
Chamadas REST de mutação originadas pela IA MUST possuir correlação, idempotência e verificação da versão esperada do registro.

#### Scenario: Confirmação repetida
- **WHEN** a mesma aprovação é recebida mais de uma vez
- **THEN** a plataforma evita a duplicação da mutação e retorna o resultado já conhecido

#### Scenario: Registro mudou após a proposta
- **WHEN** o registro é alterado entre a pré-visualização e a confirmação
- **THEN** a plataforma rejeita a mutação obsoleta e solicita nova avaliação ao usuário

### Requirement: Minimização de dados enviados à LLM
A plataforma MUST limitar os dados enviados à LLM aos campos e registros necessários, permitidos ao usuário e autorizados pela política de uso de IA.

#### Scenario: Campo proibido para IA
- **WHEN** o resultado REST contém um campo classificado como proibido para processamento pela IA
- **THEN** o orquestrador remove ou mascara o campo antes de enviar contexto à LLM

### Requirement: Auditoria correlacionada
A plataforma MUST auditar as chamadas REST originadas pela IA, registrando usuário, organização, conversa, ferramenta, operação, resultado e eventual confirmação, sem copiar indiscriminadamente dados sensíveis para a trilha imutável.

#### Scenario: Alteração realizada pela IA
- **WHEN** uma mutação confirmada é concluída
- **THEN** a auditoria permite correlacionar a solicitação conversacional, a aprovação e a chamada REST executada

### Requirement: Falha segura e resposta fiel
A conversa MUST distinguir propostas de ações efetivamente concluídas e MUST NOT informar sucesso antes da confirmação positiva da API REST.

#### Scenario: API REST retorna erro
- **WHEN** a API rejeita ou falha ao executar uma operação solicitada pela IA
- **THEN** a conversa informa que a operação não foi concluída e preserva os detalhes rastreáveis da falha
