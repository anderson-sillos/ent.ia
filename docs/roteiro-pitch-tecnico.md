# Roteiro da apresentação conceitual e técnica — ENT.IA

Material principal: [apresentacao-pitch-tecnico.html](apresentacao-pitch-tecnico.html)

## Como apresentar

Abra o arquivo HTML em um navegador. Use as setas para navegar e `F` para
alternar a tela cheia. A impressão do navegador gera uma página 16:9 por slide,
permitindo salvar a apresentação em PDF.

O roteiro abaixo foi pensado para uma conversa de aproximadamente 15 minutos,
seguida de perguntas. Ajuste o tempo conforme o grau de familiaridade do gestor
técnico com a proposta.

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
   cima para baixo. Ressalte as zonas de confiança e a ausência de acesso da LLM
   ao banco. Use o glossário no rodapé para nivelar os termos técnicos sem
   interromper a narrativa.
8. **Monólito modular:** explique que ele reduz a complexidade operacional sem
   abrir mão de limites que possibilitem extração futura.
9. **Dados:** enfatize tabela física por entidade, segregação por
   `organization_id` e ativação global com recuperação previsível.
10. **Identidade:** compare os caminhos da Organização Alfa, com Entra ID
    corporativo, e da Organização Beta, que usa a identidade fornecida pelo
    ENT.IA ou login social. O Keycloak intermedeia os fluxos; a autorização de
    negócio continua na plataforma.
11. **Autorização e auditoria:** explique a legenda de RBAC e relacione identidade
    autenticada, trilha append-only e cadeia de hashes ao não-repúdio operacional.
    Não apresente isso como não-repúdio criptográfico forte: assinatura, carimbo
    de tempo ou âncora externa continuam em discussão.
12. **IA:** leia os dois exemplos. Explique que a consulta autorizada pode ser
    executada automaticamente, enquanto a alteração mostra uma pré-visualização
    e aguarda confirmação. Em ambos os casos, a REST API continua sendo a
    autoridade final e a LLM não acessa o banco.
13. **Stack:** apresente a visão por camadas e conecte as escolhas à experiência
    Java e à preferência por um ecossistema aberto.
14. **Papel das tecnologias:** percorra as três colunas sem aprofundar APIs.
    Destaque a função de cada componente e as duas escolhas ainda condicionais:
    versão/licença do Liquibase e adoção do Spring AI.
15. **Validação técnica:** mostre como cada etapa produz uma evidência executável
    antes de ampliar o escopo.
16. **Riscos:** demonstre que os riscos centrais já foram identificados, mas
    apresente as mitigações somente como hipóteses iniciais. Use a discussão para
    acordar prioridade, responsáveis, evidências e risco residual aceitável.
17. **Viabilidade:** percorra os quatro cenários concretos: configurar, isolar,
    evoluir e conversar. A repetibilidade desses resultados é o critério de
    sucesso da prova.
18. **Alinhamentos:** direcione os pontos ao conselho técnico e explique as
    legendas de SLO e RLS antes de discutir as decisões abertas.
19. **Encerramento:** proponha a seleção da entidade piloto e de critérios
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

Ela será append-only para a aplicação e terá integridade verificável por cadeia
de hashes. No mesmo banco, isso detecta e dificulta adulteração, mas não protege
plenamente contra superusuário. A solução oferece evidência para não-repúdio
operacional, mas assinatura digital, carimbo de tempo confiável ou uma âncora
externa seriam necessários caso se exija não-repúdio criptográfico forte.

### A IA poderá alterar dados sem controle?

Não. Ela não terá banco, SQL livre ou HTTP arbitrário. Leituras passam pelas
permissões do usuário; criação, alteração e exclusão exigem pré-visualização,
confirmação explícita, idempotência e nova autorização na API REST.

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
