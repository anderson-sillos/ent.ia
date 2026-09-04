# Roteiro da apresentação conceitual e técnica — ENT.IA

Material principal: [apresentacao-pitch-tecnico.html](apresentacao-pitch-tecnico.html)

## Como apresentar

Abra o arquivo HTML em um navegador. Use as setas para navegar e `F` para
alternar a tela cheia. A impressão do navegador gera uma página 16:9 por slide,
permitindo salvar a apresentação em PDF.

Em celulares na vertical, os slides entram em modo de leitura e permitem
rolagem. Use os botões ou deslize horizontalmente para trocar de slide. No
diagrama de arquitetura, gire o aparelho ou arraste a imagem para os lados.

O roteiro abaixo foi pensado para uma conversa de aproximadamente 18 minutos,
seguida de perguntas. Ajuste o tempo conforme o grau de familiaridade do
público técnico com a proposta.

## Narrativa sugerida

1. **ENT.IA:** apresente o significado do nome e a visão de transformar a
   modelagem de entidades em capacidade reutilizável.
2. **A ideia em uma página:** apresente o que é, exemplos de aplicação e por que
   a arquitetura proposta torna o conceito tecnicamente viável. Explicite que
   ainda não existe implementação.
3. **Problema:** destaque que o desperdício está na repetição de infraestrutura
   ao redor de cada domínio, não apenas na construção das tabelas.
4. **Proposta:** explique que um mesmo metadado publicado governa persistência,
   REST, interface, autorização e ferramentas da IA.
5. **Aplicabilidade:** percorra exemplos de bons candidatos e use a coluna da
   direita para mostrar que a proposta possui limites claros.
6. **Exemplo:** use a entidade ilustrativa “Fornecedor” para mostrar que uma
   única definição governa tabela, API, interface, acesso e IA. Leia os dois
   prompts do rodapé e ressalte que o usuário não precisa conhecer endpoints ou
   o schema físico.
7. **Arquitetura:** percorra o desenho da esquerda para a direita e depois de
   cima para baixo. Mostre o Nginx como borda confirmada, a separação física das
   camadas de aplicação/identidade e dados em produção e a ausência de acesso da
   LLM ao banco. Cite containers OCI, Docker Compose por host e OpenTofu; o
   Kubernetes foi adiado, mas a arquitetura permanece compatível.
8. **Monólito modular:** explique que ele reduz a complexidade operacional sem
   abrir mão de limites que possibilitem extração futura.
9. **Dados:** enfatize PostgreSQL 18.x, databases `entia` e `keycloak` com
   credenciais separadas, tabela física por entidade, chave
   `(organization_id, UUIDv7)`, RLS obrigatória e ativação global com recuperação
   previsível. Diferencie Liquibase, usado no schema estático, do motor próprio
   que governa as entidades dinâmicas.
10. **Identidade:** comece pela entrada genérica, que não lista organizações.
    Explique o encaminhamento direto por domínio inequívoco e a confirmação do
    e-mail antes de revelar vínculos individuais. Compare a Organização Alfa,
    com Entra ID, e a Beta, com identidade ENT.IA ou login social. Após resolver
    a organização, o portal aplica seu tema declarativo; o Keycloak autentica e
    a autorização de negócio continua na plataforma.
11. **Autorização e auditoria:** explique a legenda de RBAC e relacione identidade
    autenticada, trilha append-only e gravação na mesma transação da alteração.
    Se o evento falhar, a mutação também deve falhar. Mostre que sessão não
    secreta, IP, User-Agent e canal de origem permitem correlação, mas não são
    prova isolada de identidade. Somente as chaves dos campos alterados são
    registradas, sem valores, e não há expurgo automático no MVP. Use o termo
    não-repúdio para explicar o objetivo futuro, deixando claro que o MVP entrega
    rastreabilidade operacional, sem encadeamento ou garantia criptográfica.
12. **IA:** leia os dois exemplos. Explique que a consulta autorizada pode ser
    executada automaticamente, enquanto a alteração mostra uma pré-visualização
    e aguarda confirmação. Em ambos os casos, a REST API continua sendo a
    autoridade final e a LLM não acessa o banco.
13. **API, DSL e AST:** mostre que React, integrações e IA enviam a mesma DSL
    JSON para a API da entidade. A DSL é o vocabulário externo; a AST é a árvore
    interna que preserva a lógica `and`/`or` e permite validar campo, tipo,
    operador, permissão e limites antes de o jOOQ gerar SQL parametrizado. Use o
    exemplo de fornecedores para percorrer o fluxo até PostgreSQL, RLS e o
    retorno com `items`, `meta` e cursor.
14. **Stack:** apresente a visão por camadas e conecte as escolhas à experiência
    Java e à preferência por um ecossistema aberto.
15. **Papel das tecnologias:** percorra as quatro colunas sem aprofundar APIs.
    Destaque Nginx, containers OCI, Docker Compose e OpenTofu como decisões já
    tomadas. As escolhas ainda condicionais são a versão/licença do Liquibase,
    a adoção do Spring AI e o provedor de LLM.
16. **Validação técnica:** mostre como cada etapa produz uma evidência executável
    antes de ampliar o escopo. Explique que 20 organizações, um milhão de linhas
    em uma tabela, 100 usuários virtuais e 30 RPS compõem o primeiro patamar
    comprovável; as metas p95 são critérios de aceitação, não SLA.
17. **Riscos:** diferencie controles arquiteturais já confirmados da eficácia que
    ainda precisa ser comprovada. Use a discussão para acordar evidências,
    responsáveis e risco residual aceitável.
18. **Viabilidade:** percorra os quatro cenários concretos: configurar, isolar,
    evoluir e conversar. A repetibilidade desses resultados é o critério de
    sucesso da prova.
19. **Alinhamentos:** contraste a fundação já consolidada, incluindo descoberta
    protegida, white-label, auditoria transacional e metas de capacidade, com os
    pontos realmente pendentes: piloto, operação, licença do Liquibase e
    provedor de LLM. Exportações e relatórios assíncronos foram deliberadamente
    deixados para uma fase futura.
    Explique as legendas de SLO e RLS.
20. **Encerramento:** proponha a seleção da entidade piloto e de critérios
    objetivos para demonstrar o conceito.

## Perguntas esperadas

### Por que não usar JSONB ou NoSQL para todos os registros?

O catálogo pode usar JSONB para partes flexíveis, mas os registros de negócio
terão tabela física por entidade. Essa escolha evita concentrar todo o volume em
uma tabela, permite constraints e índices próprios e preserva melhor a
integridade relacional. A contrapartida técnica é manter um motor seguro de
evolução de schema.

### Por que não iniciar com microsserviços?

O volume, os limites de escala e as necessidades independentes de implantação
ainda não foram comprovados. O monólito modular reduz coordenação e consistência
distribuída, enquanto interfaces e propriedade de dados por módulo mantêm um
caminho de extração quando houver evidência.

### Por que não implementar autenticação própria?

Credenciais, MFA, recuperação de acesso, login social e federação compõem uma
área de risco especializada. O Keycloak oferece essas capacidades; o ENT.IA
mantém a autorização de negócio, que depende de organizações e entidades
dinâmicas.

### A auditoria é realmente imutável?

No MVP, ela será append-only para os fluxos normais da aplicação: a role de
runtime poderá inserir, mas não atualizar, excluir ou truncar eventos. Cada
mutação e seu evento usarão a mesma conexão e transação; se a auditoria falhar,
a operação será revertida. O evento registra sessão não secreta, IP, User-Agent,
origem e somente as chaves dos campos modificados, sem seus valores. Não haverá
expurgo automático no MVP. Isso entrega rastreabilidade e imutabilidade
operacional, mas não detecta alteração feita por DBA ou superusuário. Cadeia de
hashes, assinatura, carimbo de tempo confiável e âncora externa são evoluções
futuras caso seja necessário não-repúdio criptográfico forte.

### A IA poderá alterar dados sem controle?

Não. Ela não terá banco, SQL livre ou HTTP arbitrário. Leituras passam pelas
permissões do usuário; criação, alteração e exclusão exigem pré-visualização,
confirmação explícita, idempotência e nova autorização na API REST.

### Por que criar uma DSL e uma AST em vez de permitir SQL?

A DSL oferece somente campos, operadores, ordenações e limites publicados no
catálogo. O backend a transforma em uma AST tipada, aplica organização,
permissões e limites e só então usa jOOQ para gerar SQL parametrizado. Assim,
React, integrações e IA compartilham um contrato expressivo sem receber acesso a
SQL, tabelas físicas ou funções arbitrárias.

### Como comprovar a viabilidade sem construir toda a plataforma?

A validação deve usar uma fatia vertical: autenticar um usuário, selecionar uma
organização, modelar e publicar uma entidade piloto, criar sua tabela, expor a
REST API, renderizar a interface, aplicar permissões e registrar auditoria. Esse
fluxo comprova a integração das partes de maior risco sem exigir todas as
capacidades finais.

## Resultado mínimo esperado da reunião

- conceito e aplicabilidade aceitos, rejeitados ou ajustados;
- limites da primeira versão compreendidos;
- entidade e organização piloto indicadas;
- responsáveis pelas decisões abertas;
- critérios objetivos para validar a prova técnica.
