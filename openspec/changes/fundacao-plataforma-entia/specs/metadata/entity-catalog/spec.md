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

### Requirement: Identidades de apresentação, lógica e física
Cada entidade e campo MUST manter separadamente um nome de exibição alterável, uma chave lógica canônica em `camelCase` e um nome físico persistido em `snake_case`; a mesma chave lógica MUST ser usada no catálogo, JSON, REST e OpenAPI.

#### Scenario: Nome de exibição é alterado
- **WHEN** um administrador altera somente o nome de exibição de uma entidade ou campo publicado
- **THEN** a chave lógica e o nome físico permanecem inalterados

#### Scenario: Contrato OpenAPI é derivado
- **WHEN** a plataforma publica o contrato de uma entidade
- **THEN** propriedades JSON, parâmetros REST e schemas OpenAPI derivam da mesma chave lógica sem criar uma segunda chave canônica

### Requirement: Nomenclatura física determinística
O catálogo MUST reservar o prefixo `en_` exclusivamente para tabelas de entidades dinâmicas, MUST gerar campos catalogados com prefixo semântico de dois caracteres e MUST nomear constraints e índices com os prefixos `pk_`, `fk_`, `uq_`, `ck_` e `ix_`. Nomes gerados MUST ser validados contra o limite do PostgreSQL e MUST NOT ser truncados silenciosamente.

#### Scenario: Entidade sem colisão de nome
- **WHEN** a chave lógica `pedidoCompra` é mapeada e `en_pedido_compra` está livre no catálogo e no banco
- **THEN** o catálogo persiste `en_pedido_compra` como nome físico sem acrescentar sufixo

#### Scenario: Colisão legítima entre objetos catalogados
- **WHEN** dois objetos registrados resultariam no mesmo nome físico
- **THEN** o motor acrescenta ao segundo um sufixo determinístico de três caracteres em base 36, verifica novamente a unicidade e persiste o mapeamento resultante

#### Scenario: Objeto físico não registrado
- **WHEN** o nome pretendido já existe fisicamente mas não possui mapeamento correspondente no catálogo
- **THEN** a plataforma classifica a situação como drift e bloqueia a ativação em vez de contornar o conflito com outro nome

### Requirement: Vocabulário inicial de campos catalogados
O catálogo MUST reconhecer inicialmente os prefixos semânticos de campo `id`, `dt`, `nr`, `nm`, `ds` e `vl`, correspondentes a identificação, data, número, nome, descrição e valor. Termos longos MUST passar por abreviação explícita e persistida antes da publicação.

#### Scenario: Campo de nome é publicado
- **WHEN** um campo classificado semanticamente como nome é validado para publicação
- **THEN** seu nome físico começa por `nm_` e a parte restante usa a abreviação `snake_case` registrada no catálogo

### Requirement: Cardinalidades derivadas de uma relação direta
O catálogo MUST representar `N:1` por uma relação direta armazenada na entidade de origem, MUST derivar `1:N` como sua visão inversa sem persistência duplicada e MUST representar `1:1` por uma relação direta com unicidade dentro da organização.

#### Scenario: Relação muitos-para-um obrigatória
- **WHEN** uma relação `N:1` obrigatória é ativada após a migração necessária
- **THEN** o campo de referência da origem recebe FK composta e `NOT NULL`, e a direção `1:N` fica disponível como consulta inversa

#### Scenario: Relação um-para-um
- **WHEN** uma relação é definida como `1:1`
- **THEN** o catálogo exige uma entidade de origem e o plano físico acrescenta unicidade organizacional ao campo de referência

### Requirement: Muitos-para-muitos por entidade associativa
Uma relação `N:N` MUST ser modelada por uma entidade associativa de primeira classe com duas relações `N:1`, sujeita ao mesmo ciclo de vida, persistência, isolamento, autorização, API e auditoria das demais entidades.

#### Scenario: Associação possui atributos próprios
- **WHEN** o administrador precisa registrar função, período ou situação entre duas entidades
- **THEN** esses dados são definidos como campos da entidade associativa sem criar um mecanismo especial de relacionamento

#### Scenario: Assistente cria associação simples
- **WHEN** um assistente do catálogo cria uma relação `N:N`
- **THEN** ele produz uma entidade associativa identificável no catálogo, ainda que ela seja ocultada da navegação principal

### Requirement: Semântica de referência no MVP
O catálogo MUST distinguir conceitualmente relações `REFERENCE` de futuras relações `COMPOSITION`, mas no MVP MUST permitir ativação somente de `REFERENCE`. Contagem mínima de filhos, cascata lógica automática e restauração conjunta de composições MUST NOT ser prometidas pelo contrato inicial.

#### Scenario: Tentativa de ativar composição
- **WHEN** uma definição solicita semântica `COMPOSITION` enquanto essa capacidade não está implementada
- **THEN** a validação bloqueia sua ativação e informa que apenas `REFERENCE` está disponível

### Requirement: Autorreferência opcional
Uma entidade MUST poder possuir uma relação opcional `REFERENCE` para ela própria, sem que isso seja interpretado como hierarquia ou garantia de ausência de ciclos.

#### Scenario: Categoria referencia categoria pai
- **WHEN** uma entidade define uma autorreferência opcional
- **THEN** o catálogo a trata como referência comum e não oferece automaticamente árvore, limite de profundidade ou consultas hierárquicas

### Requirement: Capacidades de consulta por campo
Cada campo publicado MUST declarar separadamente se é `selectable`, `filterable`, `sortable` e `patternSearch`. O catálogo MUST validar essas capacidades contra o tipo, tamanho e persistência do campo e MUST NOT permitir ordenação genérica de relacionamentos, textos extensos, JSON, coleções, binários ou campos calculados sem persistência no MVP.

#### Scenario: Campo textual curto é configurado
- **WHEN** um administrador habilita capacidades compatíveis de projeção, filtro, ordenação ou pesquisa por padrão
- **THEN** a versão publicada expõe somente as capacidades validadas e o runtime ainda aplica as permissões do consumidor

#### Scenario: Campo incompatível recebe capacidade de ordenação
- **WHEN** uma definição tenta marcar como `sortable` um campo sem representação posicional compacta e determinística
- **THEN** o catálogo rejeita a publicação e identifica a incompatibilidade

### Requirement: Ordenação padrão e perfis compostos
Uma versão publicada MAY definir uma ordenação padrão formada por campos escalares suportados. Na ausência dela, o runtime MUST usar `id desc`. Uma combinação de duas ou três chaves de ordenação MUST ser registrada como perfil de consulta versionado, incluindo prioridade, direção e posição dos nulos. O catálogo MUST acrescentar o UUIDv7 como desempate final e MUST NOT gerar automaticamente todas as combinações possíveis.

#### Scenario: Perfil composto é publicado
- **WHEN** uma versão define uma combinação válida de até três campos ordenáveis
- **THEN** o catálogo registra um perfil imutável e exige índice composto compatível iniciado por `organization_id` e terminado por `id`

#### Scenario: Perfil excederia o cursor permitido
- **WHEN** o pior tamanho possível dos valores posicionais produziria cursor acima de 1.024 caracteres
- **THEN** o catálogo bloqueia a publicação do perfil antes de disponibilizá-lo ao runtime

### Requirement: Ativação condicionada aos índices de consulta
Toda tabela dinâmica MUST possuir índice base para registros ativos iniciado por `(organization_id, id)`. Um campo habilitado para ordenação simples MUST possuir índice compatível, e pesquisa com curinga inicial MUST possuir estratégia de índice compatível com padrões textuais. Uma versão MUST disponibilizar uma capacidade ou perfil somente depois que os índices exigidos forem criados e validados.

#### Scenario: Índice de consulta falha durante ativação
- **WHEN** o motor não consegue criar ou validar um índice exigido por uma capacidade ou perfil
- **THEN** a nova versão não é ativada e o runtime continua usando a versão anterior

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
