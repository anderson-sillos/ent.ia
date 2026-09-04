# Dicionário técnico do ENT.IA

Este documento reúne as siglas, os conceitos de arquitetura e os nomes das
tecnologias citadas na proposta do ENT.IA. Ele parte do stack apresentado no
slide 14 e acrescenta os termos usados no design, nas especificações e nas
discussões de infraestrutura.

O estado indicado em cada item significa:

- **Confirmado:** decisão já adotada na proposta da plataforma;
- **Recomendado:** direção técnica proposta, ainda sujeita à confirmação;
- **Candidato:** depende de prova técnica, fornecedor, versão ou licença;
- **Em aberto:** decisão necessária antes da respectiva implementação;
- **Futuro:** arquitetura preparada para possível adoção posterior;
- **Fora do MVP:** não faz parte da primeira versão planejada;
- **Conceito:** termo utilizado na arquitetura, sem representar uma escolha de
  produto.

## Arquitetura e aplicação

| Termo | Estado | Definição e aplicação no ENT.IA |
|---|---|---|
| **API — Application Programming Interface** | Confirmado | Contrato pelo qual sistemas e componentes solicitam operações. As APIs do ENT.IA serão a fronteira comum para a interface, integrações e IA. |
| **AST — Abstract Syntax Tree / Árvore Sintática Abstrata** | Conceito | Representação hierárquica dos elementos de uma linguagem, concentrada em seu significado e não na forma textual original. No ENT.IA, a DSL de consulta será interpretada como uma árvore de condições e grupos lógicos, que poderá ser validada antes da construção segura da consulta com jOOQ. |
| **BFF — Backend for Frontend** | Confirmado | Backend dedicado às necessidades do frontend. No ENT.IA, inicia o login, mantém os tokens fora do navegador, administra a sessão e expõe ao React somente contratos apropriados. |
| **CBOR — Concise Binary Object Representation** | Candidato para serialização | Formato binário compacto para representar estruturas de dados. Pode ser usado internamente na serialização dos cursores, mas o contrato exige apenas um formato binário compacto, sem fixar CBOR antes da implementação. |
| **Collation** | Conceito | Conjunto de regras usado pelo banco para comparar e ordenar textos, incluindo idioma, caixa e acentos. O ENT.IA não permitirá que cada requisição escolha uma collation arbitrária; a semântica será definida de forma compatível com o catálogo e os índices. |
| **CRUD — Create, Read, Update, Delete** | Conceito | Conjunto básico de operações de criação, consulta, alteração e exclusão de registros. As operações disponíveis dependerão das permissões da organização, entidade e ação. |
| **Contrato** | Conceito | Definição estável das entradas, saídas e regras de uma integração. Exemplos: contrato REST, OpenAPI, JSON Schema e ferramentas oferecidas à LLM. |
| **Correlation ID** | Confirmado | Identificador propagado por requisições, jobs, auditoria e chamadas da IA para relacionar eventos pertencentes à mesma operação. |
| **Cursor opaco** | Confirmado | Token de continuação cujo conteúdo interno não integra o contrato público. No ENT.IA será stateless, criptografado, autenticado e vinculado por fingerprint ao usuário, organização, entidade, versão e consulta. |
| **Design tokens** | Confirmado para white-label | Valores declarativos de aparência, como cores, superfícies, texto, foco e arredondamento. Serão validados e convertidos em variáveis CSS e tema MUI sem aceitar código fornecido pela organização. |
| **DSL — Domain-Specific Language / Linguagem Específica de Domínio** | Confirmado | Linguagem com vocabulário restrito a uma finalidade. No ENT.IA, será uma estrutura JSON usada para declarar filtros, projeção, ordenação e paginação. O backend validará a DSL contra o catálogo, os tipos e as permissões antes de convertê-la em SQL parametrizado com jOOQ; SQL e expressões textuais livres não serão aceitos. |
| **Gzip** | Não adotado nos cursores do MVP | Formato de compactação baseado em DEFLATE; não fornece criptografia nem proteção por senha. Foi descartado para o cursor porque o payload binário mínimo possui pouca redundância e o cabeçalho pode aumentar tokens pequenos. |
| **Idempotência** | Confirmado | Garantia de que repetir uma mesma solicitação identificada não provoca a mesma alteração mais de uma vez. Será obrigatória nas mutações iniciadas pela IA. |
| **JSON Merge Patch** | Confirmado | Formato de alteração parcial definido para o método `PATCH`. Campos omitidos são preservados e valores `null` seguem a semântica explícita do contrato. |
| **Keyset/seek pagination** | Confirmado | Paginação que continua a partir dos valores de ordenação do último item, evitando percorrer linhas anteriores por offset. Será encapsulada pelos cursores opacos do ENT.IA. |
| **Monólito modular** | Confirmado | Aplicação implantada inicialmente como uma unidade, mas dividida internamente em módulos com responsabilidades e contratos claros. Permite menor complexidade operacional sem criar um código fortemente acoplado. |
| **OpenAPI** | Confirmado | Especificação legível por máquinas para descrever endpoints, parâmetros, respostas e erros de uma API HTTP. Também apoiará a seleção controlada de ferramentas oferecidas à IA. |
| **Outbox** | Confirmado | Padrão que grava um evento na mesma transação dos dados de negócio, permitindo seu processamento posterior sem perder a consistência. Será armazenado inicialmente no PostgreSQL. |
| **Paginação por cursor** | Confirmado | Navegação em que `after` ou `before` recebe um token que representa a posição, sem expor os valores internos. O cursor terá validade de 30 minutos, meta de 512 caracteres e limite absoluto de 1.024. |
| **Perfil de consulta** | Confirmado | Combinação versionada de até três campos de ordenação, com prioridade, direção e tratamento de nulos. Perfis compostos somente serão ativados depois da criação e validação de um índice compatível. |
| **Problem Details** | Confirmado | Formato padronizado `application/problem+json` para erros HTTP. Será usado em validações, conflitos, falhas de autorização e estados de cursor inválido, expirado ou obsoleto. |
| **REST — Representational State Transfer** | Confirmado | Estilo de API baseado em recursos, operações HTTP e respostas padronizadas. Será a fronteira oficial das operações dinâmicas do ENT.IA. |
| **Renderer dinâmico** | Confirmado | Componente que interpreta metadados em tempo de execução para montar formulários, detalhes, filtros e tabelas sem gerar código-fonte para cada entidade. |
| **Runtime** | Conceito | Momento em que a aplicação está sendo executada. O ENT.IA interpreta o catálogo e monta APIs e telas em runtime. |
| **Schema Version** | Confirmado | Versão do contrato de uma entidade. Uma escrita baseada em versão desatualizada deverá ser rejeitada para evitar inconsistências durante ativações. |
| **Record Version** | Confirmado | Versão de um registro individual usada para concorrência otimista e prevenção de perda de atualizações simultâneas. |
| **SLI — Service Level Indicator** | Conceito adotado | Medição observada de uma característica do serviço, como disponibilidade, latência ou taxa de erros. Latência por percentil, throughput e erros serão medidos na validação do MVP. |
| **SLO — Service Level Objective** | Futuro para produção | Meta mensurável para um SLI, por exemplo: 99,9% de disponibilidade mensal. As metas de benchmark do MVP são critérios técnicos de aceitação e não constituem ainda um SLO de produção. |
| **SLA — Service Level Agreement** | Futuro | Compromisso formal de nível de serviço, normalmente associado a consequências contratuais. Um SLA pode usar SLOs como referência, mas não é sinônimo deles. |
| **SPA — Single-Page Application** | Confirmado | Aplicação web carregada como uma página principal e atualizada dinamicamente no navegador. O frontend React do ENT.IA seguirá esse modelo. |
| **SSR — Server-Side Rendering** | Fora do MVP | Renderização do HTML no servidor antes de enviá-lo ao navegador. Não é necessária inicialmente para a aplicação autenticada e interativa do ENT.IA. |
| **Stateless** | Confirmado como princípio | Componente que não depende do disco ou da memória local para preservar o estado necessário entre requisições. Os cursores não terão sessão persistida no servidor, e esse princípio facilitará réplicas e futura execução em Kubernetes. |
| **Theme template** | Confirmado | Modelo visual versionado e declarativo que reúne tokens e variantes suportados. No MVP, novos modelos serão publicados pela administração da plataforma, enquanto organizações escolhem uma versão e ajustam somente propriedades permitidas. |
| **White-label** | Confirmado | Personalização da identidade visual da experiência ENT.IA por organização. Abrangerá entrada resolvida e aplicação autenticada no MVP, sem personalização dinâmica das páginas ou e-mails internos do Keycloak. |

## Desempenho e capacidade

| Termo | Estado | Definição e aplicação no ENT.IA |
|---|---|---|
| **Benchmark** | Confirmado | Teste reproduzível usado para medir capacidade e desempenho em ambiente, dados e carga conhecidos. O MVP terá um perfil de referência antes que seus limites sejam ampliados. |
| **Latência** | Confirmado como métrica | Tempo decorrido para concluir uma operação. As metas do backend serão medidas sem incluir navegador, IdP externo ou provedor de LLM. |
| **p95 / p99** | Confirmado como métrica | Percentis de latência: p95 indica que 95% das medições ficaram naquele valor ou abaixo; p99 aplica o mesmo raciocínio a 99%. Evitam avaliar desempenho apenas pela média. |
| **RPS — Requests per Second** | Confirmado como métrica | Quantidade de requisições processadas por segundo. O benchmark sustentará 30 RPS por 30 minutos e validará um pico de 60 RPS por cinco minutos. |
| **Throughput** | Confirmado como métrica | Volume de trabalho concluído por unidade de tempo. Será analisado junto com latência, erros, consumo de recursos e integridade da auditoria. |
| **Usuário virtual — VU** | Confirmado como métrica | Usuário simulado por uma ferramenta de carga. O teste inicial trabalhará com 100 usuários virtuais concorrentes; isso não representa limite de sessões cadastradas. |

## Métodos HTTP da API REST

Embora sejam frequentemente chamados de “verbos REST”, tecnicamente estes são
métodos HTTP aplicados aos recursos da API.

| Método | Estado | Definição e aplicação no ENT.IA |
|---|---|---|
| **GET** | Confirmado | Método seguro e somente leitura usado para consultar um recurso ou obter uma listagem padrão. No ENT.IA aceitará apenas parâmetros simples e reservados; filtros compostos usarão `POST .../query`. |
| **POST** | Confirmado | Usado para criar recursos, enviar a DSL de consulta composta e iniciar comandos ou jobs. `POST .../records/query` será somente leitura, apesar de usar esse método. |
| **PATCH** | Confirmado | Usado para alteração parcial. O ENT.IA adotará JSON Merge Patch, preservando campos omitidos e aplicando controle otimista de concorrência. |
| **DELETE** | Confirmado | Solicita a exclusão de um recurso. Para registros dinâmicos do ENT.IA, executará exclusão lógica, com auditoria e preservação da linha no banco. |
| **PUT** | Fora do MVP | Normalmente associado à substituição integral de um recurso. Não será usado inicialmente nos registros dinâmicos para evitar que campos omitidos sejam apagados durante a evolução do schema. |
| **HEAD** | Suporte automático | Equivale ao `GET`, mas sem corpo de resposta. Poderá ser tratado automaticamente pelo Nginx ou Spring quando aplicável, sem representar uma operação de negócio. |
| **OPTIONS** | Suporte automático | Usado, entre outros fins, na negociação de CORS e indicação dos métodos permitidos. Será tratado pela infraestrutura e pelo framework, sem executar comandos de negócio. |
| **TRACE** | Desabilitado | Método de diagnóstico que devolve informações da requisição recebida. Permanecerá desabilitado no Nginx e na aplicação por não ser necessário ao contrato público. |
| **CONNECT** | Não exposto | Estabelece um túnel por meio de um proxy. Não será aceito pela API pública do ENT.IA. |

## Organizações, catálogo e entidades dinâmicas

| Termo | Estado | Definição e aplicação no ENT.IA |
|---|---|---|
| **Catálogo de entidades** | Confirmado | Repositório global das definições, campos, validações, relações, versões e regras de apresentação das entidades de negócio. |
| **camelCase** | Confirmado | Convenção que inicia com letra minúscula e separa palavras seguintes com maiúsculas, como `pedidoCompra`. Será o formato canônico da chave lógica no catálogo, JSON, REST e OpenAPI. |
| **Cardinalidade** | Confirmado | Quantidade de registros que pode participar de cada lado de um relacionamento, como `1:1`, `1:N` e `N:N`. O modelo físico do ENT.IA será construído a partir de relações diretas `N:1`. |
| **Capacidades de consulta** | Confirmado | Indicadores versionados `selectable`, `filterable`, `sortable` e `patternSearch` que determinam como cada campo pode participar da DSL. Continuam sujeitos ao tipo, aos índices e às permissões efetivas. |
| **Chave lógica** | Confirmado | Identificador funcional estável de entidade ou campo, independente do nome exibido e do nome físico. Usará `camelCase` e não terá uma variante canônica exclusiva para OpenAPI. |
| **Composição / `COMPOSITION`** | Futuro | Relação na qual o filho pertence ao ciclo de vida do pai. Exigirá cascata lógica controlada, impacto, auditoria e restauração conjunta; não poderá ser ativada no MVP. |
| **Entidade de negócio** | Confirmado | Estrutura configurável que representa um conceito do domínio, como cliente, contrato, equipamento ou solicitação. |
| **Entidade associativa** | Confirmado | Entidade de primeira classe que representa uma relação `N:N` por duas relações `N:1`. Pode possuir campos próprios e segue as regras normais de versão, RLS, autorização, API e auditoria. |
| **flatcase** | Não adotado | Convenção que junta palavras em minúsculas, como `pedidocompra`. Foi descartada porque reduz a legibilidade e não será uma forma alternativa da chave lógica. |
| **Metadados** | Confirmado | Dados que descrevem outros dados. No ENT.IA, definem a estrutura e o comportamento de entidades, APIs, formulários e consultas. |
| **Metadata-driven** | Confirmado | Abordagem na qual o comportamento do sistema é determinado por metadados publicados, e não por código específico para cada entidade. |
| **Multi-organização / multitenancy** | Confirmado | Uma mesma instalação atende várias organizações, mantendo seus dados segregados. As definições de entidades serão globais e os registros serão organizacionais. |
| **Nome de exibição** | Confirmado | Rótulo livre, traduzível e alterável apresentado aos usuários. Sua alteração não muda a chave lógica nem o nome físico já publicado. |
| **`organizationSlug`** | Confirmado | Identificador público, único e estável usado no endereço organizacional `/o/{organizationSlug}`. Ajuda a resolver o contexto antes do login, mas nunca funciona como autorização. |
| **`organization_id`** | Confirmado | Identificador obrigatório que delimita a organização proprietária de cada dado organizacional. O contexto confiável virá da sessão, não de um valor livre enviado pelo cliente. |
| **Referência / `REFERENCE`** | Confirmado | Relação entre registros com ciclos de vida independentes. É a única semântica de relacionamento ativável no MVP. |
| **Relação inversa** | Confirmado | Visão consultável no sentido oposto de uma FK, como os pedidos de um cliente. É derivada da relação direta e não cria estado duplicado. |
| **snake_case** | Confirmado | Convenção em minúsculas com palavras separadas por `_`, como `pedido_compra`. Será usada nos nomes físicos de tabelas, colunas, índices e constraints. |
| **Tabela física por entidade** | Confirmado | Estratégia em que cada entidade publicada possui tabela própria, compartilhada entre organizações e contendo `organization_id`. |
| **Autorreferência** | Confirmado com escopo limitado | Relação em que uma entidade aponta para outro registro dela própria. No MVP será opcional e comum, sem garantia automática de árvore ou ausência de ciclos. |
| **Versão ativa** | Confirmado | Única versão publicada de uma entidade que governa simultaneamente persistência, API, interface, autorização e ferramentas da IA. |
| **`N:1` — muitos-para-um** | Confirmado | Relação física fundamental: muitos registros de origem podem referenciar um registro de destino por FK composta. Pode ser opcional ou obrigatória. |
| **`1:N` — um-para-muitos** | Confirmado | Direção inversa de uma relação `N:1`, obtida por consulta à FK existente sem duplicar dados no registro de destino. |
| **`1:1` — um-para-um** | Confirmado | Relação direta com FK no lado dependente e unicidade por organização, permitindo no máximo um dependente para cada alvo. |
| **`N:N` — muitos-para-muitos** | Confirmado | Relação implementada por uma entidade associativa de primeira classe; não haverá tabela de junção oculta sem representação no catálogo. |

## Identidade, autenticação e segurança

| Termo | Estado | Definição e aplicação no ENT.IA |
|---|---|---|
| **AEAD — Authenticated Encryption with Associated Data** | Confirmado para cursores | Classe de algoritmos que protege simultaneamente confidencialidade e integridade. Será usada nos cursores para impedir leitura e adulteração do conteúdo interno. |
| **Autenticação** | Confirmado | Processo de verificar quem é o usuário. Será delegada ao Keycloak ou a um provedor de identidade federado. |
| **Autorização** | Confirmado | Processo de decidir o que um usuário autenticado pode fazer. Será aplicada pelo backend do ENT.IA por organização, entidade e operação. |
| **Authorization Code Flow** | Confirmado | Fluxo OAuth/OIDC no qual o navegador recebe um código temporário e o backend o troca por tokens usando um canal protegido. Será usado pelo BFF. |
| **Audience** | Confirmado | Identificação do serviço ao qual um token se destina. Tokens delegados para um futuro orquestrador separado deverão ter audience limitada à API do ENT.IA. |
| **Backchannel** | Confirmado | Comunicação direta entre componentes de backend, sem passar pelo navegador. A comunicação ENT.IA–Keycloak ocorrerá pela rede interna. |
| **Base64 URL-safe** | Confirmado para cursores | Codificação textual que usa caracteres seguros em URLs. Não é criptografia; será aplicada somente depois da proteção criptográfica do cursor. |
| **Cookie `HttpOnly`** | Confirmado | Cookie inacessível ao JavaScript do navegador, reduzindo a exposição da sessão em ataques que injetam scripts. |
| **Cookie `SameSite`** | Confirmado | Atributo que limita quando o navegador envia o cookie em requisições originadas por outros sites, ajudando a reduzir ataques CSRF. |
| **Cookie `Secure`** | Confirmado | Atributo que permite o envio do cookie somente por conexões HTTPS. |
| **CSRF — Cross-Site Request Forgery** | Conceito | Ataque que induz o navegador autenticado a enviar uma operação não pretendida. O BFF deverá aplicar proteção adequada nas mutações. |
| **Deny by default** | Confirmado | Regra segundo a qual uma operação é negada quando não existe concessão explícita de acesso. |
| **Federação de identidade** | Confirmado | Integração que permite autenticar usuários por provedores externos pertencentes às organizações. |
| **Fingerprint criptográfico** | Confirmado para cursores | Resumo compacto usado para vincular um cursor ao usuário, organização, entidade, versão e consulta sem armazenar integralmente esse contexto. Terá 16 bytes e será recalculado a cada página. |
| **Frontchannel** | Confirmado | Parte do fluxo de autenticação que passa pelo navegador. Somente as rotas indispensáveis do realm do Keycloak serão expostas externamente. |
| **IdP — Identity Provider** | Confirmado | Sistema que autentica usuários e emite identidades ou tokens, como Keycloak, Microsoft Entra ID, Google ou um provedor OIDC/SAML da organização. |
| **Identity-first login** | Confirmado | Fluxo genérico em que o usuário informa primeiro seu e-mail para descobrir com segurança a rota de autenticação aplicável. Vínculos individuais somente serão revelados após comprovação de posse do endereço. |
| **Keycloak** | Confirmado | Provedor de identidade self-hosted escolhido como padrão e broker de identidades. Centralizará login, MFA, federação, recuperação de acesso e login social. |
| **Login social** | Confirmado | Autenticação usando uma conta externa, como Google, Microsoft, Apple ou GitHub, mediada pelo Keycloak. Os provedores exatos ainda serão escolhidos. |
| **MFA — Multi-Factor Authentication** | Confirmado | Autenticação que exige mais de um fator, como senha e código temporário. Será responsabilidade do Keycloak ou do IdP federado. |
| **OIDC — OpenID Connect** | Confirmado | Protocolo de identidade construído sobre OAuth 2.0. Será o protocolo principal entre o BFF e o Keycloak e poderá integrar provedores organizacionais. |
| **OAuth 2.0** | Confirmado | Framework de autorização usado como base para emissão e uso controlado de tokens. O login utiliza OIDC sobre OAuth 2.0. |
| **Privilégio mínimo** | Confirmado | Princípio de conceder a usuários, serviços e credenciais apenas os acessos indispensáveis. |
| **Proxy confiável** | Confirmado | Proxy cujos endereços e comportamento são controlados pela instalação. O backend aceitará informações de origem somente do Nginx ou de proxies allowlisted; headers enviados livremente pelo cliente serão descartados ou substituídos. |
| **RBAC — Role-Based Access Control** | Confirmado | Controle de acesso baseado em perfis. No ENT.IA, é contextual: os perfis pertencem a uma organização e concedem operações sobre entidades específicas. |
| **Realm** | Confirmado | Domínio lógico do Keycloak que agrupa usuários, clientes, políticas e provedores de identidade. A instalação usará inicialmente um realm chamado `ent-ia`. |
| **RLS — Row-Level Security** | Confirmado | Recurso do PostgreSQL que restringe no próprio banco quais linhas uma sessão pode consultar ou alterar. Será obrigatório nas tabelas persistentes organizacionais como segunda barreira ao filtro e à autorização do backend, usando o contexto confiável de `organization_id` da transação. |
| **SAML — Security Assertion Markup Language** | Confirmado | Protocolo de federação de identidade comum em ambientes corporativos. Poderá conectar o Keycloak ao provedor de uma organização. |
| **SSO — Single Sign-On** | Confirmado | Capacidade de usar uma autenticação para acessar diferentes aplicações ou serviços integrados ao mesmo provedor. |
| **`session_trace_id`** | Confirmado | UUIDv7 interno e não secreto que correlaciona eventos de uma sessão autenticada. É diferente do cookie e dos tokens e não pode ser reutilizado para assumir a sessão. |
| **Subject / `sub`** | Confirmado | Identificador estável do usuário emitido por um provedor OIDC. Deve ser usado com o emissor para vincular corretamente identidades externas. |
| **TLS — Transport Layer Security** | Confirmado | Proteção criptográfica do tráfego HTTPS. Será obrigatória na borda pública e nas comunicações internas que exigirem proteção adicional. |
| **User-Agent** | Confirmado como evidência complementar | Texto declarado pelo cliente sobre navegador e dispositivo. Será sanitizado, limitado a 512 caracteres e nunca usado sozinho como prova de identidade ou autorização. |
| **VPN — Virtual Private Network** | Recomendado | Rede privada usada para restringir acesso administrativo, como console do Keycloak, métricas e endpoints de gerenciamento. |
| **WAF — Web Application Firewall** | Em aberto | Camada adicional que inspeciona e bloqueia tráfego HTTP malicioso. A solução e o marco de adoção ainda serão definidos conforme o risco; não substitui autenticação, autorização ou validação na aplicação. |
| **Domínio personalizado** | Futuro; requisito previsto | Endereço próprio de uma organização direcionado à ENT.IA. O resolvedor será compatível com essa evolução, mas verificação de posse, DNS e certificados não serão implementados no MVP. |

## Dados e persistência

| Termo | Estado | Definição e aplicação no ENT.IA |
|---|---|---|
| **Atomicidade** | Confirmado | Propriedade pela qual um conjunto de alterações é confirmado integralmente ou totalmente desfeito. Nas mutações auditadas, a alteração de negócio e o evento de auditoria pertencem à mesma transação PostgreSQL. |
| **Backfill** | Confirmado | Preenchimento de um campo novo ou incompleto nos registros existentes antes de tornar uma regra mais restritiva. |
| **Base 36** | Confirmado para colisões de nomes | Codificação usando `0-9` e `a-z`. Um sufixo determinístico de três caracteres será acrescentado somente quando dois objetos catalogados resultarem no mesmo nome físico. |
| **Bulk insert / inserção em massa** | Confirmado como capacidade | Gravação eficiente de muitos registros em uma mesma operação ou fluxo. Lotes regulares usarão batch; volumes elevados usarão staging, validação e promoção em conjunto para as tabelas finais. |
| **Cluster PostgreSQL** | Confirmado | Serviço PostgreSQL que hospeda os databases da instalação. Inicialmente haverá um cluster com databases separados para `entia` e `keycloak`; isso não significa um database por organização. |
| **Constraint** | Confirmado | Regra aplicada pelo banco, como obrigatoriedade, unicidade ou integridade referencial. As constraints das entidades serão derivadas de um vocabulário seguro do catálogo. |
| **COPY** | Confirmado para importações massivas | Comando do PostgreSQL para transferir muitas linhas com baixo overhead. Como `COPY FROM` não grava diretamente em tabelas com RLS, será usado somente para uma staging controlada, nunca para contornar as tabelas finais protegidas. |
| **Database** | Conceito | Banco lógico dentro de um cluster PostgreSQL. `entia` e `keycloak` serão databases distintos, com proprietários e credenciais independentes. |
| **DDL — Data Definition Language** | Confirmado | Comandos SQL que criam ou alteram estruturas, como tabelas, colunas, índices e constraints. O motor próprio do ENT.IA executará somente operações permitidas e planejadas. |
| **Drift de schema** | Confirmado | Divergência entre a estrutura física do banco e o estado esperado pelo catálogo ou pelas migrations. O drift bloqueará a ativação até ser corrigido. |
| **Exclusão lógica / soft delete** | Confirmado | Exclusão que preserva fisicamente a linha e registra `deleted_at`. Consultas normais omitem o registro, o histórico pode continuar resolvendo referências e toda ação permanece auditada. |
| **FK — Foreign Key / chave estrangeira** | Confirmado | Constraint que garante que uma referência exista na tabela de destino. Nas entidades dinâmicas incluirá `organization_id`, usará `ON DELETE RESTRICT` e `ON UPDATE RESTRICT` e terá índice correspondente. |
| **GIN — Generalized Inverted Index** | Candidato para padrões textuais | Tipo de índice do PostgreSQL adequado a conjuntos de componentes, podendo apoiar buscas com `pg_trgm`. A estratégia exata será validada por benchmark. |
| **GiST — Generalized Search Tree** | Candidato para padrões textuais | Estrutura extensível de índice do PostgreSQL que também pode suportar operadores de similaridade e padrões por extensões como `pg_trgm`. Será comparada ao GIN conforme escrita, espaço e leitura. |
| **Identity column** | Uso específico | Coluna cujo valor é gerado automaticamente pelo PostgreSQL por meio de uma sequence. Será adequada para numerações e posições técnicas locais, mas não será o identificador padrão dos registros dinâmicos. |
| **`inet`** | Confirmado para auditoria | Tipo nativo do PostgreSQL para endereços IPv4 e IPv6. Armazenará o IP normalizado de origem das requisições auditadas. |
| **Índice** | Confirmado | Estrutura do banco que acelera buscas e também pode suportar regras de unicidade, com impacto adicional em armazenamento e escrita. |
| **Índice parcial** | Confirmado como estratégia | Índice que contém somente linhas que atendem a uma condição. As listagens comuns poderão usar índices com `WHERE deleted_at IS NULL` para priorizar registros ativos. |
| **JSON — JavaScript Object Notation** | Confirmado | Formato textual estruturado usado em APIs, metadados e contratos. |
| **JSONB** | Uso restrito | Tipo binário de JSON do PostgreSQL, pesquisável e indexável. Pode apoiar partes flexíveis do catálogo, mas não será a tabela única para todos os registros de negócio. |
| **JSON Schema 2020-12** | Confirmado | Padrão que descreve estrutura, tipos e validações de documentos JSON. Representará o contrato lógico das entidades e formulários dinâmicos. |
| **jOOQ** | Confirmado | Biblioteca Java que oferece SQL tipado e construção controlada de consultas. Será usada tanto no schema estático quanto nas operações dinâmicas permitidas. |
| **Liquibase** | Confirmado com ressalva | Ferramenta de migrations para o schema estático da plataforma. Não administrará as tabelas dinâmicas; a versão será fixada somente após revisão de licença. |
| **Migration** | Confirmado | Alteração versionada e reproduzível da estrutura ou dos dados de um banco. Liquibase tratará o schema estático e o motor do ENT.IA tratará entidades dinâmicas. |
| **NoSQL** | Não adotado como banco principal | Família de bancos não relacionais. Foi considerada, mas PostgreSQL foi escolhido por integridade relacional, transações, constraints e controle de DDL. |
| **`ON DELETE`** | Confirmado | Regra da FK para uma exclusão física no registro referenciado. As relações dinâmicas usarão `RESTRICT`; exclusão lógica não dispara essa regra porque mantém a linha no banco. |
| **Particionamento** | Futuro | Divisão física de uma tabela grande em partes menores administradas pelo PostgreSQL. Será introduzido somente quando métricas de volume justificarem. |
| **PK — Primary Key / chave primária** | Confirmado | Constraint que identifica unicamente uma linha. Nas tabelas dinâmicas será composta por `(organization_id, id)`. |
| **`pg_trgm`** | Candidato recomendado | Extensão do PostgreSQL que divide textos em trigramas e permite índices para `LIKE`, `ILIKE` e similaridade. É a estratégia preferencial a validar para pesquisas com curinga inicial. |
| **PostgreSQL** | Confirmado — 18.x | Banco relacional principal da plataforma, usado para dados estáticos, catálogo, auditoria, fila/outbox e tabelas físicas das entidades. A linha 18.x será mantida no minor release validado mais recente. |
| **PostgreSQL gerenciado** | Confirmado como perfil | Serviço operado por um provedor, responsável por parte das rotinas de infraestrutura do banco. Poderá ser preferido em produção na nuvem quando atender ao contrato PostgreSQL portável do ENT.IA. |
| **PostgreSQL self-hosted** | Confirmado como referência | PostgreSQL operado pela própria instalação. Será o perfil de referência em container OCI no host de dados e permitirá implantação on-premises. |
| **Role de banco** | Confirmado | Identidade PostgreSQL à qual são atribuídos privilégios. Haverá separação entre proprietário técnico, Liquibase, schema dinâmico, runtime, ingestão, leitura administrativa da auditoria e Keycloak. Para preservar a atomicidade, a role de runtime poderá inserir eventos de auditoria, mas não alterá-los, excluí-los ou truncá-los. A promoção de importações para tabelas finais continuará submetida ao RLS. |
| **`RESTRICT`** | Confirmado para FKs dinâmicas | Ação referencial que bloqueia imediatamente uma exclusão física enquanto existirem registros dependentes, sem apagá-los ou alterá-los automaticamente. |
| **Schema de banco** | Confirmado | Namespace que organiza objetos como tabelas e índices. As organizações compartilharão o banco e o schema de aplicação. Não deve ser confundido com JSON Schema. |
| **Sequence** | Uso específico | Objeto do PostgreSQL que entrega números crescentes com alta concorrência. Será usado para versões, posições ou numerações locais quando apropriado; pode conter lacunas após rollback ou falha e, por isso, não garante numeração legalmente contínua. |
| **SQL — Structured Query Language** | Confirmado | Linguagem usada para consultar e modificar dados relacionais. Usuários, administradores e LLM não poderão fornecer SQL arbitrário para execução. |
| **Staging** | Confirmado para importações massivas | Área temporária e isolada onde um lote é carregado e validado antes de chegar às tabelas definitivas. Ficará presa a um `import_id` e a uma organização confiável, sem acesso público nem escolha arbitrária de SQL ou tabela-alvo. |
| **`statement_timeout`** | Confirmado para consultas síncronas | Parâmetro do PostgreSQL que cancela uma instrução ao exceder o tempo definido. O executor da DSL começará com cinco segundos para consultas interativas. |
| **Transação** | Confirmado | Unidade de trabalho que confirma todas as operações em conjunto ou desfaz todas em caso de falha. Uma mutação e seu evento de auditoria serão confirmados pelo mesmo commit. |
| **UI Schema** | Confirmado | Metadado complementar ao JSON Schema que define layout, widget, visibilidade e apresentação dos campos no frontend. |
| **UUID — Universally Unique Identifier** | Confirmado | Família de identificadores de 128 bits com unicidade prática global. O ENT.IA os armazenará no tipo nativo `uuid` do PostgreSQL, e não como texto. |
| **UUIDv7** | Confirmado | Variante temporal de UUID escolhida como identificador técnico imutável dos registros dinâmicos e demais recursos que exijam identidade global. Será gerada preferencialmente pelo backend ou importador antes da persistência; o PostgreSQL 18.x poderá fornecer `uuidv7()` como default de segurança. |

## Auditoria e integridade

| Termo | Estado | Definição e aplicação no ENT.IA |
|---|---|---|
| **Append-only** | Confirmado no MVP | Modelo em que novos eventos podem ser acrescentados, mas os já registrados não são atualizados, excluídos ou truncados pelos fluxos e credenciais normais da aplicação. Não impede sozinho a ação de um DBA ou superusuário. |
| **Auditoria síncrona** | Confirmado no MVP | Gravação realizada antes de concluir a operação. Nas mutações, o evento usa a mesma conexão e transação PostgreSQL; se a auditoria falhar, a alteração de negócio também sofre rollback. |
| **DBA — Database Administrator** | Conceito | Administrador com privilégios elevados no banco. A auditoria no mesmo PostgreSQL não oferece proteção absoluta contra um DBA ou superusuário malicioso. |
| **Encadeamento de hashes** | Futuro; fora do MVP | Técnica em que cada evento registra seu hash e o hash do evento anterior para tornar alterações ou lacunas detectáveis. Foi adiada porque uma cabeça compartilhada pode serializar gravações e ampliar bloqueios nas transações de negócio. |
| **Hash criptográfico** | Conceito; uso futuro na auditoria | Resumo matemático de um conteúdo. Uma pequena alteração produz outro resultado, permitindo verificar integridade sem representar criptografia reversível. O MVP não encadeará hashes dos eventos de auditoria. |
| **Imutabilidade operacional** | Confirmado no MVP | Garantia de que os eventos não podem ser alterados ou removidos pelos caminhos e credenciais normais da aplicação. Não equivale a imutabilidade criptográfica ou física diante de um administrador privilegiado. |
| **Não-repúdio** | Objetivo futuro; evidência limitada no MVP | Capacidade de produzir evidências que dificultem negar a autoria ou ocorrência de uma ação. O MVP associa eventos à identidade autenticada e ao contexto transacional, mas não alega não-repúdio criptográfico, que exigiria controles como assinatura, carimbo de tempo confiável ou âncora externa. |
| **Retenção da auditoria** | Confirmado no MVP | Não haverá expurgo automático por idade no MVP. Eventos permanecerão armazenados e o crescimento da tabela e dos índices será monitorado até existir uma política futura explícita, autorizada e juridicamente revisada. |
| **Tamper-evident** | Futuro; fora do MVP | Propriedade de tornar adulterações detectáveis, por exemplo por encadeamento de hashes e checkpoints independentes. O append-only operacional do MVP não oferece sozinho essa garantia contra atores privilegiados. |
| **WORM — Write Once, Read Many** | Fora do MVP | Armazenamento no qual o dado é escrito uma vez e depois somente lido. Não será introduzido inicialmente; poderá fortalecer a auditoria no futuro. |

## Inteligência artificial

| Termo | Estado | Definição e aplicação no ENT.IA |
|---|---|---|
| **Allowlist de ferramentas** | Confirmado | Relação explícita das operações que a LLM pode solicitar. Será filtrada pelas permissões efetivas do usuário. |
| **Agente autônomo** | Fora do MVP | Sistema de IA que planeja e executa sequências de ações com maior autonomia. O ENT.IA começará com ferramentas controladas e confirmação humana para mutações. |
| **Embeddings** | Fora do MVP | Representações numéricas usadas para comparar semanticamente textos ou outros conteúdos. Serão avaliados apenas se surgir necessidade de busca semântica ou RAG. |
| **IA — Inteligência Artificial** | Confirmado como capacidade | Área do produto que permitirá consultar e propor alterações nos registros por linguagem natural, sempre através das APIs e autorizações da plataforma. |
| **LLM — Large Language Model** | Provedor em aberto | Modelo de linguagem capaz de interpretar prompts e produzir respostas ou solicitações estruturadas de ferramentas. Provedor e modelo ainda serão escolhidos. |
| **MCP — Model Context Protocol** | Fora do MVP | Protocolo para conectar modelos e agentes a ferramentas e fontes de contexto. Um adaptador futuro poderá usar as APIs existentes, sem substituir a fronteira REST. |
| **Minimização de dados** | Confirmado | Envio à LLM somente dos campos e registros estritamente necessários e autorizados, com mascaramento ou bloqueio de dados sensíveis. |
| **Orquestrador de IA** | Confirmado | Módulo que administra conversas, valida tool calls, aplica limites e permissões, solicita confirmações e chama a API REST do ENT.IA. |
| **Prompt** | Confirmado | Instrução em linguagem natural enviada à LLM. É sempre tratado como entrada não confiável e não pode substituir o contexto de usuário ou organização. |
| **Prompt injection** | Risco reconhecido | Tentativa de inserir instruções para contornar regras ou induzir acessos indevidos. Será mitigada com ferramentas tipadas, allowlist, autorização no backend e isolamento do contexto confiável. |
| **RAG — Retrieval-Augmented Generation** | Fora do MVP | Técnica que recupera conteúdo externo e o inclui no contexto da LLM antes da resposta. Não é necessária para o primeiro fluxo de operações sobre entidades. |
| **Spring AI** | Candidato | Biblioteca do ecossistema Spring para abstrair modelos, prompts e tool calling. A adoção depende da escolha do provedor e de uma prova técnica. |
| **Token da LLM** | Conceito | Unidade usada pelo modelo para processar entrada e gerar saída. Limites de tokens ajudam a controlar tempo, volume e consumo. Não deve ser confundido com token OAuth. |
| **Tool calling** | Confirmado | Capacidade de a LLM solicitar uma operação estruturada. O orquestrador valida a solicitação antes de transformá-la em uma chamada REST. |

## Backend, build e qualidade

| Tecnologia | Estado | Papel no ENT.IA |
|---|---|---|
| **OpenJDK 25 LTS** | Confirmado | Implementação aberta e de suporte prolongado da plataforma Java usada pelo backend. |
| **LTS — Long-Term Support** | Conceito | Versão com período estendido de manutenção e correções, adequada como base de produção. |
| **Spring Boot 4.1** | Confirmado | Base para configuração, servidor HTTP, APIs REST, BFF e execução da aplicação Java. |
| **Spring Modulith** | Confirmado | Define, documenta e testa os limites entre módulos do monólito modular. |
| **Spring Security** | Confirmado | Implementa proteção de endpoints, sessão BFF, integração OIDC e controles de segurança no backend. |
| **Maven** | Confirmado | Gerencia dependências, compilação, testes e empacotamento do projeto Java. |
| **Testcontainers** | Confirmado | Inicia dependências reais em containers descartáveis durante testes, como PostgreSQL e Keycloak. |

## Frontend e testes

| Tecnologia | Estado | Papel no ENT.IA |
|---|---|---|
| **React 19** | Confirmado | Biblioteca usada para construir a SPA e os componentes do renderer dinâmico. |
| **TypeScript** | Confirmado | Extensão tipada do JavaScript que melhora contratos, refatoração e segurança do frontend. |
| **Vite** | Confirmado | Ferramenta de desenvolvimento e build do frontend. |
| **MUI Core** | Confirmado | Biblioteca de componentes visuais usada como base consistente e acessível da interface. |
| **TanStack Query** | Confirmado | Gerencia chamadas, cache e sincronização do estado proveniente das APIs. |
| **TanStack Table** | Confirmado | Base para tabelas, filtros, ordenação e paginação das consultas dinâmicas. |
| **React Hook Form** | Confirmado | Gerencia estado, submissão e validação dos formulários dinâmicos. |
| **Vitest** | Confirmado | Executor de testes unitários e de integração do frontend. |
| **RTL — React Testing Library** | Confirmado | Biblioteca para testar componentes React pelo comportamento percebido pelo usuário. |
| **Playwright** | Confirmado | Ferramenta de testes ponta a ponta executados em navegadores reais. |
| **Redux** | Fora do MVP | Biblioteca de estado global. Não será introduzida inicialmente; estado remoto ficará no TanStack Query e estados pequenos em contextos locais. |

## Containers, infraestrutura e operação

As tecnologias desta seção incluem decisões confirmadas e capacidades ainda em
discussão para a implantação.

| Termo ou tecnologia | Estado | Definição e aplicação proposta no ENT.IA |
|---|---|---|
| **Ambiente agnóstico** | Confirmado | Arquitetura implantável em nuvem ou on-premises sem dependência obrigatória de um provedor. Recursos específicos serão isolados em módulos de infraestrutura, mantendo portáveis as imagens e os contratos da aplicação. |
| **CDN — Content Delivery Network** | Futuro | Rede distribuída para entregar conteúdo estático próximo dos usuários. Pode hospedar ou acelerar os arquivos do React, mas não é requisito inicial. |
| **CI/CD — Continuous Integration / Continuous Delivery** | Recomendado | Automação de build, testes, análise e entrega. O GitHub Actions é a opção natural por o código já estar no GitHub. |
| **Container** | Confirmado | Processo isolado criado a partir de uma imagem, com a aplicação e suas dependências. Não é uma máquina virtual completa. |
| **Container OCI** | Confirmado | Container compatível com os padrões abertos da Open Container Initiative, permitindo executar a mesma imagem com Docker, Podman, containerd ou Kubernetes. Será usado desde a fundação. |
| **containerd** | Futuro | Runtime de containers compatível com Kubernetes. Pode executar as mesmas imagens OCI produzidas no processo de build. |
| **CRI-O** | Futuro | Runtime alternativo voltado à interface CRI do Kubernetes. Não há necessidade de fixá-lo enquanto o ambiente Kubernetes não for escolhido. |
| **Docker Engine** | Recomendado | Motor open source para construir e executar containers, especialmente em servidores Linux e ambientes de desenvolvimento. |
| **Docker Compose** | Confirmado | Define e executa aplicações com vários containers em um arquivo declarativo. Será usado na implantação inicial, com um projeto Compose independente por host físico, sem assumir orquestração multi-host. |
| **Docker Desktop** | Opcional; licença a revisar | Aplicação para executar Docker em estações de desenvolvimento. Possui termos comerciais diferentes do Docker Engine e pode ser substituída por alternativas. |
| **DNS — Domain Name System** | Recomendado | Resolve nomes como `ent.ia.br` e `auth.ent.ia.br` para os pontos de entrada da plataforma. |
| **Gateway API** | Futuro | Modelo moderno do Kubernetes para definir entrada e roteamento de tráfego HTTP e outros protocolos. Será preferido ao Ingress tradicional se Kubernetes for adotado. |
| **Health check** | Recomendado | Endpoint ou verificação que informa se um componente está iniciado, saudável e pronto para receber tráfego. |
| **IaC — Infrastructure as Code** | Confirmado | Prática de descrever rede, servidores, bancos, DNS e outros recursos em arquivos versionados e reproduzíveis. Será aplicada por meio do OpenTofu. |
| **Imagem OCI** | Confirmado | Pacote imutável contendo aplicação, runtime, bibliotecas e metadados necessários para criar containers. |
| **Kubernetes / K8s** | Adiado; manter compatibilidade | Orquestrador de containers para múltiplos nós, réplicas, recuperação automática e atualizações progressivas. Não será usado inicialmente, mas os containers e contratos operacionais serão preparados para adoção futura. |
| **Liveness probe** | Futuro, preparar desde a fundação | Verificação que indica se um processo travou e precisa ser reiniciado pelo orquestrador. |
| **Nginx** | Confirmado | Servidor web e proxy reverso escolhido para servir o React, terminar ou encaminhar HTTPS, rotear o BFF/API e expor somente o frontchannel necessário do Keycloak. Também aplicará limites básicos de tráfego, mas não substitui sozinho um WAF completo. |
| **OpenTofu** | Confirmado | Ferramenta open source de IaC, sob licença MPL 2.0. Será o padrão do projeto, com módulos específicos por ambiente e compatibilidade conceitual com Terraform. |
| **Pod** | Futuro | Menor unidade implantável no Kubernetes, contendo um ou mais containers executados em conjunto. |
| **Podman** | Alternativa | Motor de containers compatível com imagens OCI e operação sem daemon central, possível alternativa ao Docker Engine ou Docker Desktop. |
| **Readiness probe** | Futuro, preparar desde a fundação | Verificação que determina se uma instância está pronta para receber novas requisições. |
| **Registry de imagens** | Recomendado | Repositório para armazenar e distribuir imagens OCI versionadas. O provedor ainda não foi escolhido. |
| **Réplicas** | Futuro, preparar desde a fundação | Múltiplas instâncias equivalentes do mesmo componente para distribuir carga ou tolerar falhas. |
| **Reverse proxy / proxy reverso** | Confirmado | Serviço de borda que recebe a requisição pública e a encaminha ao componente interno correto, ocultando sua exposição direta. Essa função será exercida pelo Nginx. |
| **Terraform** | Compatibilidade conceitual | Ferramenta de IaC com conceitos e linguagem relacionados ao OpenTofu. Não será o padrão do projeto, mas a organização dos módulos evitará incompatibilidades conceituais desnecessárias. Sua licença deverá ser revista antes de eventual uso. |

## Versionamento e documentação

| Tecnologia ou termo | Estado | Papel no ENT.IA |
|---|---|---|
| **Git** | Confirmado | Sistema distribuído de controle de versão dos documentos e, futuramente, do código-fonte. |
| **GitHub** | Confirmado | Hospeda o repositório, revisões, automações e configurações de segurança do projeto. |
| **GitHub Actions** | Confirmado como infraestrutura do repositório | Executa automações do repositório. Os workflows da aplicação serão criados durante a implementação. |
| **GitHub Pages** | Confirmado | Publica a apresentação navegável do projeto. Não será a hospedagem da aplicação ENT.IA. |
| **OpenSpec** | Confirmado | Método e ferramenta usados para registrar proposta, especificações, decisões de design e tarefas antes da implementação. |
| **PR — Pull Request** | Confirmado | Fluxo de revisão e integração das mudanças na branch protegida `main`. |

## Tecnologias e abordagens deliberadamente adiadas

| Item | Situação no ENT.IA |
|---|---|
| **GraphQL** | Não será o contrato principal; REST foi escolhido por previsibilidade e integração com OpenAPI e ferramentas da IA. |
| **Kafka** | Não será introduzido inicialmente; fila persistida e outbox no PostgreSQL atenderão os primeiros trabalhos assíncronos. |
| **Microsserviços** | Não serão a arquitetura inicial; módulos poderão ser extraídos do monólito quando escala ou independência operacional justificarem. |
| **Next.js** | Não é necessário inicialmente, pois o frontend autenticado será uma SPA React sem requisito atual de SSR. |
| **Banco vetorial** | Não faz parte do MVP e só será avaliado junto com uma necessidade concreta de embeddings ou RAG. |
| **Exportações e relatórios assíncronos** | Fora do MVP; evolução prevista. O contrato de jobs, snapshot, autorização, armazenamento, retenção e download será definido em fase posterior. |

## Manutenção deste dicionário

Ao confirmar uma decisão arquitetural, o estado do termo correspondente deve ser
atualizado. Novas siglas ou tecnologias adicionadas à proposta, ao design, às
especificações ou à apresentação também devem ser registradas aqui.
