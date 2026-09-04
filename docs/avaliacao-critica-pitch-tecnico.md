# Avaliação crítica do pitch técnico — ENT.IA

## Objetivo

Registrar, de forma isolada, uma leitura crítica da apresentação pelo ponto de
vista de um profissional com experiência de mercado, conhecimento técnico e de
arquitetura de sistemas que precisa ser convencido de que a ideia faz sentido e
merece sua atenção e dedicação, sem considerar investimentos financeiros.

Este documento serve como insumo para uma futura revisão da proposta. Ele não
altera os artefatos OpenSpec, a apresentação vigente nem o handoff do projeto.

## Diagnóstico geral

Minha leitura honesta: o ENT.IA merece atenção técnica, mas a apresentação atual
convence mais de que houve um bom trabalho de arquitetura do que de que existe
um problema relevante o suficiente para justificar a dedicação de uma equipe.

Ela demonstra coerência, segurança e domínio técnico. O ponto ainda fraco é
ligar essa arquitetura a um primeiro problema real, explicar por que as soluções
existentes não bastam e mostrar como a principal hipótese será validada com
esforço limitado.

## O que já convence

- Existe coerência entre catálogo, persistência, API, interface, autorização,
  auditoria e IA.
- As escolhas técnicas são pragmáticas: monólito modular, PostgreSQL, Keycloak e
  containers.
- O projeto reconhece riscos e evita promessas exageradas.
- A IA não aparece como acesso privilegiado ao banco; ela atua pelas mesmas APIs
  e permissões.
- Os comparativos demonstram que a abordagem orientada a metadados é
  consolidada.
- O MVP possui limites razoáveis: sem migrations destrutivas, agentes autônomos,
  Kubernetes ou regras arbitrárias.

Isso transmite maturidade arquitetural.

## O que ainda enfraquece o convencimento

### 1. O problema ainda é genérico

O slide 3 diz que existe infraestrutura repetida ao redor das entidades, mas não
mostra:

- Quem enfrenta esse problema.
- Em qual processo ele ocorre.
- Como é resolvido atualmente.
- Com que frequência surgem novas entidades ou alterações.
- Qual resultado seria significativamente melhor com o ENT.IA.

Um profissional experiente pode concordar com o problema e ainda concluir:
“isso não acontece em intensidade suficiente para justificar uma plataforma”.

A apresentação precisa de um caso real ou plausível, apresentado como fluxo:

```text
Solicitação de novo cadastro
        |
        v
Modelagem + banco + API + tela + acesso + auditoria
        |
        v
Coordenação entre diferentes especialistas
        |
        v
Entrega específica e difícil de reutilizar
```

Depois deve mostrar o mesmo processo no ENT.IA.

### 2. Existe uma tensão importante entre “personalização” e catálogo global

A descrição fala em personalização de entidades, mas a arquitetura estabelece:

- Definições globais e compartilhadas.
- Uma única versão ativa.
- Organizações apenas habilitam entidades.
- Não existe schema diferente por organização.

A pergunta inevitável será:

> Se uma organização não pode incluir seus próprios campos, o que exatamente
> ela está personalizando?

Isso precisa ser esclarecido. Existem pelo menos três posicionamentos possíveis:

- O catálogo é administrado centralmente e atende produtos padronizados.
- Organizações solicitam extensões, mas todas passam a fazer parte do catálogo
  global.
- Futuramente haverá extensões organizacionais controladas.

A decisão atual favorece governança e simplicidade, mas reduz a promessa de
personalização por cliente. Essa é provavelmente a maior questão conceitual
ainda pouco visível no pitch.

### 3. O comparativo não responde “por que não usar um produto existente?”

Os slides 8 e 9 demonstram que Salesforce, Dataverse, ServiceNow, Frappe,
Directus, SAP, Oracle, Odoo e TOTVS usam conceitos semelhantes. Isso valida a
abordagem, mas também cria uma objeção:

> Se o conceito já está disponível, por que construir o ENT.IA?

A apresentação descreve diferenças, mas não identifica requisitos
eliminatórios. Seria melhor uma matriz baseada nas necessidades concretas:

| Requisito decisivo | Produto existente | ENT.IA |
|---|---:|---:|
| Infraestrutura agnóstica e controlada | varia | objetivo central |
| Catálogo global compartilhado | varia | sim |
| Tabela física por entidade com RLS | geralmente diferente | sim |
| IdP diferente por organização | varia | previsto |
| White-label organizacional | varia | previsto |
| IA restrita às mesmas APIs e permissões | parcial | princípio arquitetural |
| Stack Java e PostgreSQL controlável | geralmente não | sim |

O pitch também precisa reconhecer honestamente quando usar um produto existente
seria melhor. Isso aumenta a credibilidade:

> Se Directus ou Frappe atenderem integralmente ao piloto, não faz sentido
> reconstruir suas capacidades. O ENT.IA somente se justifica se seus requisitos
> combinados não forem atendidos adequadamente por essas alternativas.

### 4. “Aplicação de negócio” pode prometer mais do que o MVP entrega

O MVP oferece principalmente:

- Entidades estruturadas.
- Validações.
- Relacionamentos.
- CRUD.
- Consultas.
- Segurança.
- Auditoria.
- Interação por prompt.

Uma aplicação de negócio real frequentemente também exige:

- Regras entre entidades.
- Campos calculados.
- Transações com múltiplos registros.
- Estados e transições.
- Aprovações.
- Workflows.
- Notificações.
- Documentos.
- Relatórios.
- Integrações com sistemas externos.

Como vários desses itens ficam para o futuro, a classificação inicial mais
segura seria:

> Plataforma para aplicações operacionais orientadas a dados estruturados.

Depois, com regras e workflows, ela pode evoluir para uma plataforma mais ampla
de aplicações de negócio.

### 5. O principal risco técnico ainda parece subestimado

O slide 22 concentra corretamente a inovação no catálogo, schema engine e
runtime dinâmico. Porém, justamente esse núcleo reúne problemas difíceis:

- DDL concorrente.
- Evolução de tabelas grandes.
- Compatibilidade entre versões.
- Backfill.
- Drift.
- Cache de metadados.
- Compatibilidade de APIs.
- Uma organização com dados incompatíveis bloqueando a versão global.
- Recuperação após falha parcial.

A apresentação deveria mostrar que o núcleo é limitado por um vocabulário
fechado de mudanças, não um mecanismo universal de banco de dados.

### 6. Falta evidência, mesmo que preliminar

O deck informa corretamente que ainda não existe implementação. Entretanto,
para alguém dedicar energia ao projeto, falta pelo menos uma destas evidências:

- Protótipo navegável da experiência.
- Experimento de criação dinâmica de tabela.
- Teste de RLS entre duas organizações.
- API dinâmica baseada em uma definição simples.
- Benchmark preliminar.
- Entrevista ou validação com potencial usuário.
- Caso real candidato ao piloto.

Atualmente, a apresentação prova que o projeto foi bem pensado, mas ainda não
prova que funciona nem que será utilizado.

### 7. A narrativa está extensa e técnica demais

São 26 slides para aproximadamente 25 minutos. Isso deixa menos de um minuto por
slide, embora vários exijam explicações detalhadas.

Os slides 6 a 11 explicam classificação, mercado, concorrentes, diferenciação e
evolução antes do primeiro exemplo concreto, que só aparece no slide 12. Para
convencimento, o exemplo deveria aparecer muito antes.

Também há detalhes que funcionariam melhor como apêndice:

- DSL e AST.
- Tecnologias individualmente.
- Detalhes de auditoria.
- Estratégia de cursor.
- Comparativos completos.
- Metas detalhadas de desempenho.
- Ciclo de schema completo.

## Narrativa recomendada

Eu reduziria o fluxo principal para aproximadamente 15 slides:

1. **Tese:** o que é o ENT.IA em uma frase.
2. **Problema real:** quem sofre e em qual situação.
3. **Antes e depois:** como uma nova entidade é entregue hoje e com o ENT.IA.
4. **Exemplo de Fornecedor:** experiência ponta a ponta.
5. **Quem usa:** administrador da plataforma, administrador organizacional e
   usuário.
6. **Escopo:** o que o MVP faz e o que não faz.
7. **Validação de mercado:** conceito já utilizado.
8. **Por que não usar uma solução existente:** requisitos eliminatórios.
9. **Diferenciais propostos:** combinação específica do ENT.IA.
10. **Arquitetura:** visão de alto nível.
11. **Apostas técnicas:** catálogo, schema engine, isolamento e runtime.
12. **IA na prática:** valor e controles.
13. **Piloto verificável:** cenário, métricas e critérios de interrupção.
14. **Evolução progressiva:** capacidades futuras sem prometer um ERP completo.
15. **Decisão solicitada:** piloto, responsáveis e próximo marco.

Os demais slides seriam mantidos como apêndice para perguntas técnicas.

## Melhorias prioritárias

Antes de apresentar, eu consideraria indispensável:

1. Definir uma entidade e uma organização piloto reais.
2. Explicar quem administra o catálogo global.
3. Delimitar o significado de “personalização”.
4. Explicar por que Directus, Frappe ou Dataverse não atendem ao caso.
5. Definir quais regras de negócio o MVP suporta.
6. Mostrar como o ENT.IA coexistirá com ERPs e sistemas legados.
7. Criar um desenho visual da experiência do usuário, não apenas da
   arquitetura.
8. Transformar o piloto em uma hipótese refutável, com critérios de continuidade
   e interrupção.
9. Apresentar uma decomposição inicial de trabalho e competências necessárias.
10. Terminar com uma solicitação objetiva de dedicação técnica.

O último ponto é relevante porque atualmente o encerramento pede apenas a
seleção da entidade piloto. Para obter comprometimento, deveria pedir algo
semelhante a:

> Validar uma prova técnica limitada, indicar um responsável de domínio, um
> revisor de arquitetura e os critérios de continuidade.

## Perguntas provavelmente feitas e ainda não respondidas

### Problema e público

- Quem é o primeiro usuário real do ENT.IA?
- Qual problema concreto ele possui hoje?
- Com que frequência novas entidades ou campos são necessários?
- O ENT.IA será um produto independente, uma extensão de ERP ou uma plataforma
  interna?
- Qual resultado do piloto demonstrará utilidade, e não apenas funcionamento
  técnico?

### Personalização e governança

- Quem pode criar, alterar, publicar e descontinuar entidades?
- O administrador de uma organização pode adicionar campos?
- Se as definições são globais, como atender necessidades específicas de uma
  organização?
- O que acontece quando duas organizações querem evoluções incompatíveis?
- Como uma alteração global será comunicada, testada e ativada?
- Uma organização pode permanecer temporariamente em uma versão anterior?

### Alternativas existentes

- Por que não construir sobre Frappe ou Directus?
- Por que não utilizar Dataverse, ServiceNow ou Odoo?
- Quais requisitos concretos eliminam essas opções?
- Qual parte do ENT.IA será difícil de reproduzir por um concorrente?
- A diferenciação está na tecnologia, na experiência, no catálogo de entidades
  ou no modelo operacional?

### Limites funcionais

- Quais tipos de campo, validações e relações serão suportados no MVP?
- Haverá regras que dependem de vários campos ou entidades?
- Como tratar transações envolvendo múltiplos registros?
- Como representar estados, aprovações e workflows?
- Como criar uma tela específica quando o formulário dinâmico não for
  suficiente?
- Arquivos, anexos e documentos fazem parte do modelo?
- Sem relatórios e exportações, o piloto será realmente utilizável?

### Integrações

- O ENT.IA será fonte oficial dos dados ou uma camada complementar?
- Como sincronizar registros com ERP, CRM e sistemas legados?
- Haverá webhooks, eventos ou somente REST?
- Como importar dados existentes para iniciar uma entidade?
- Como um cliente recupera ou exporta seus dados?
- Como evitar ciclos e conflitos de atualização entre sistemas?

### Arquitetura e dados

- Quantas entidades físicas uma instalação deverá suportar?
- Como o PostgreSQL se comportará com centenas ou milhares de tabelas?
- Como executar DDL sem bloquear organizações em operação?
- O que acontece se os dados de uma organização forem incompatíveis com uma
  versão global?
- Como será feito o backfill de um novo campo obrigatório?
- Como preservar compatibilidade de APIs após renomear ou remover campos?
- Como o contexto do RLS será aplicado com pool de conexões?
- Como caches de metadados serão invalidados entre várias instâncias?
- Qual é o caminho de escala do banco antes de particionamento ou separação
  física?

### Operação e segurança

- Como serão administrados segredos, certificados e rotação de credenciais?
- Qual será a estratégia de backup e restauração?
- Como recuperar somente os dados de uma organização?
- Quais métricas e alertas definem a saúde do schema engine?
- Quem poderá utilizar a credencial de DDL?
- Como alterações privilegiadas no banco serão detectadas?
- Qual ameaça concreta justifica cada controle previsto no MVP?

### Inteligência artificial

- O produto continua útil quando a LLM está indisponível?
- Que dados podem sair para o provedor?
- Como campos sensíveis são classificados e mascarados?
- O provedor poderá variar por organização?
- Como garantir que a confirmação corresponde exatamente à alteração executada?
- Como tratar uma alteração concorrente entre pré-visualização e confirmação?
- Qual benefício da IA não seria atendido por filtros e formulários
  convencionais?
- A IA também ajudará a modelar entidades ou apenas operar registros?

### Execução do projeto

- Qual é a menor prova técnica capaz de invalidar a tese?
- Quanto trabalho é necessário para chegar a essa prova?
- Quais competências precisam estar disponíveis?
- Qual é a ordem dos experimentos?
- Quais critérios levam a continuar, restringir o escopo ou interromper?
- Quem mantém o schema engine e responde por incidentes?
- Qual parte pode ser substituída caso a abordagem original não funcione?

## Conclusão

O ponto mais importante é este:

> A apresentação precisa deixar de ser principalmente uma explicação da
> arquitetura e passar a ser uma demonstração de que existe um problema real,
> que as alternativas disponíveis não atendem aos requisitos decisivos e que a
> principal hipótese pode ser testada com esforço controlado.

A arquitetura está suficientemente consistente para justificar uma prova
técnica. O que ainda falta para conquistar comprometimento é um caso real, uma
justificativa clara de construção versus adoção e um plano de validação limitado
e refutável.

O artefato de arquitetura já possui várias respostas técnicas que não aparecem
no pitch atual. Porém, a decomposição de implementação `tasks.md` ainda não
existe. Antes da apresentação definitiva, ela será necessária para transformar a
ideia em uma solicitação realista de tempo, responsabilidades e entregas.
