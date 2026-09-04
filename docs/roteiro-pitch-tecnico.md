# Roteiro da apresentação conceitual e técnica — ENT.IA

Material principal: [apresentacao-pitch-tecnico.html](apresentacao-pitch-tecnico.html)

## Como apresentar

Abra o arquivo HTML em um navegador. Use as setas para navegar e `F` para
alternar a tela cheia. A impressão do navegador gera uma página 16:9 por slide,
permitindo salvar a apresentação em PDF.

Em celulares na vertical, os slides entram em modo de leitura e permitem
rolagem. Use os botões ou deslize horizontalmente para trocar de slide. No
diagrama de arquitetura, gire o aparelho ou arraste a imagem para os lados.

O percurso principal possui 15 slides e foi pensado para aproximadamente 18 a
20 minutos de apresentação, reservando de 5 a 7 minutos para perguntas. Depois
do encerramento, avance para acessar o apêndice técnico. A tecla `A` abre
diretamente o primeiro slide do apêndice; `Home` retorna ao início e `End` leva
ao encerramento do percurso principal.

## Narrativa sugerida

1. **Tese:** apresente o ENT.IA como a transformação de uma definição de entidade
   em persistência, telas, APIs e interação por IA, com segurança e governança
   incorporadas.
2. **Problema:** destaque que o desperdício está na infraestrutura repetida ao
   redor de cada entidade, e não somente na criação das tabelas.
3. **Proposta:** explique que um catálogo central permite modelar, validar,
   publicar, ativar e operar uma entidade a partir do mesmo contrato.
4. **Exemplo de Fornecedor:** torne o conceito concreto antes de aprofundar as
   abstrações. Mostre definição, tabela física, telas, REST e dois exemplos de
   interação por prompt.
5. **Aplicabilidade:** percorra os bons candidatos e os limites explícitos para
   deixar claro onde a proposta se encaixa.
6. **Como o modelo funciona:** mostre catálogo central, interface dinâmica e uma
   fronteira operacional única. Use os termos técnicos apenas depois da
   explicação em linguagem direta.
7. **Validação de mercado:** resuma que plataformas corporativas e ERPs já
   utilizam variações da abordagem. Deixe os comparativos completos para o
   apêndice.
8. **Diferencial:** apresente a combinação entre catálogo compartilhado, modelo
   relacional, isolamento multitenant, APIs governadas e IA sem caminho
   privilegiado até os dados.
9. **Evolução progressiva:** mostre que a fundação pode ganhar produtividade,
   processos e inteligência gradualmente, conforme valor e evidência.
10. **Arquitetura:** percorra o desenho em alto nível, enfatizando a fronteira
    REST comum, a separação das camadas e a ausência de acesso da LLM ao banco.
11. **Apostas técnicas:** diferencie componentes maduros reutilizados do núcleo
    que precisa ser construído e comprovado: catálogo, schema engine, runtime
    dinâmico, consultas governadas e orquestração da IA.
12. **IA na prática:** contraste uma consulta autorizada com uma alteração que
    exige pré-visualização e confirmação. A REST API permanece como autoridade.
13. **Piloto verificável:** percorra os quatro cenários concretos — configurar,
    isolar, evoluir e conversar — e destaque que o resultado precisa ser
    repetível e observável.
14. **Alinhamentos:** contraste a fundação consolidada com o que ainda precisa
    ser definido para a prova executável.
15. **Encerramento:** proponha a escolha da entidade piloto e dos critérios
    objetivos de sucesso.

## Apêndice técnico

O apêndice preserva os detalhes para responder perguntas sem interromper a
narrativa principal:

1. resumo executivo da ideia;
2. comparativo completo com plataformas;
3. comparativo completo com ERPs;
4. monólito modular e fronteiras internas;
5. dados e evolução do schema;
6. identidade, descoberta de organização e white-label;
7. RBAC, RLS e auditoria;
8. API de entidades, DSL e AST;
9. stack tecnológica por camada;
10. papel de cada tecnologia;
11. hipóteses críticas, provas e alternativas seguras.

## Perguntas esperadas

### Esse conceito já é usado por plataformas e ERPs de mercado?

Sim. Salesforce, Microsoft Dataverse e ServiceNow são referências de plataformas
orientadas a metadados. SAP S/4HANA Cloud, Oracle Fusion, Dynamics 365, Odoo e
TOTVS aplicam variações do conceito para criar ou ampliar objetos de negócio,
campos, formulários, serviços e regras. Nos ERPs, porém, essa capacidade costuma
estender um núcleo vertical existente. O ENT.IA parte de um motor genérico e
transforma o catálogo na fonte de verdade para persistência, interface, APIs,
acesso, auditoria e IA.

Referências oficiais: [Salesforce](https://architect.salesforce.com/docs/architect/fundamentals/guide/platform-multitenant-architecture.html),
[Microsoft Dataverse](https://learn.microsoft.com/en-us/power-apps/maker/model-driven-apps/define-data-model-driven-app),
[ServiceNow](https://www.servicenow.com/docs/r/platform-administration/table-administration-and-data-management/using-table-administration.html),
[SAP S/4HANA Cloud](https://help.sap.com/docs/SAP_S4HANA_CLOUD/0f69f8fb28ac4bf48d2b57b9637e81fa/c703c5b4e6a24d52a928ea54e8ff5e52.html),
[Oracle Fusion Cloud](https://docs.oracle.com/en/cloud/saas/applications-common/26a/oacex/overview-of-using-application-composer.html),
[Odoo](https://www.odoo.com/documentation/18.0/applications/studio/models_modules_apps.html)
e [TOTVS RM](https://tdn.totvs.com/display/LRM/Metadados%2B-%2BComo%2Bincluir%2Bum%2Bprojeto).

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
