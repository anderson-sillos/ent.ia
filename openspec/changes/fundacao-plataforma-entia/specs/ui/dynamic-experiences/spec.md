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

### Requirement: Identidade visual organizacional
A interface MUST aplicar a identidade visual da organização resolvida antes do login e durante a sessão autenticada. Na entrada genérica, antes da resolução, MUST usar a identidade neutra da ENT.IA.

#### Scenario: Organização resolvida antes do login
- **WHEN** a plataforma resolve uma organização válida por slug, domínio inequívoco ou descoberta protegida
- **THEN** a interface carrega seu nome e tema público antes de encaminhar o usuário ao provedor de identidade

#### Scenario: Troca de organização
- **WHEN** uma sessão muda validamente de contexto organizacional
- **THEN** a interface substitui o tema pelo da nova organização sem reutilizar personalizações da anterior

### Requirement: Modelos visuais declarativos e versionados
A plataforma MUST disponibilizar modelos visuais básicos e MUST permitir a publicação de novos modelos personalizados compostos somente por tokens e variantes suportados. No MVP, a criação e publicação de modelos MUST ser restrita à administração da plataforma; administradores organizacionais MAY selecionar um modelo e alterar cores, nome e ativos de marca expressamente permitidos.

#### Scenario: Publicação de novo modelo
- **WHEN** um administrador da plataforma submete um modelo declarativo válido
- **THEN** a plataforma permite sua pré-visualização e publicação como uma versão imutável disponível ao escopo autorizado

#### Scenario: Modelo inválido ou indisponível
- **WHEN** o modelo ativo viola o schema, falha ao carregar ou deixa de estar disponível
- **THEN** a interface aplica o tema padrão seguro sem impedir o acesso à plataforma

### Requirement: Limites do white-label no MVP
O white-label do MVP MUST abranger as páginas produzidas pela ENT.IA antes e depois da autenticação, mas MUST NOT prometer personalização dinâmica das páginas internas, recuperação de acesso ou e-mails do Keycloak.

#### Scenario: Redirecionamento para autenticação
- **WHEN** o usuário deixa a experiência organizacional da ENT.IA para autenticar no Keycloak ou em um IdP externo
- **THEN** a plataforma preserva o contexto de retorno sem exigir que a página externa use o tema dinâmico da organização
