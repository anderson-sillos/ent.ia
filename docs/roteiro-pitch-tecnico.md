# Roteiro do pitch executivo-técnico — ENT.IA

Material principal: [apresentacao-pitch-tecnico.html](apresentacao-pitch-tecnico.html)

## Como apresentar

Abra o arquivo HTML em um navegador. Use as setas para navegar e `F` para
alternar a tela cheia. A impressão do navegador gera uma página 16:9 por slide,
permitindo salvar a apresentação em PDF.

O roteiro abaixo foi pensado para uma conversa de aproximadamente 15 minutos,
seguida de perguntas. Ajuste o tempo conforme o grau de familiaridade dos
gestores com a proposta.

## Narrativa sugerida

1. **ENT.IA:** apresente o significado do nome e a visão de transformar a
   modelagem de entidades em capacidade reutilizável.
2. **Resumo executivo:** antecipe a decisão desejada: financiar uma validação
   incremental da fundação, e não assumir de imediato todo o custo do produto.
3. **Problema:** destaque que o desperdício está na repetição de infraestrutura
   ao redor de cada domínio, não apenas na construção das tabelas.
4. **Proposta:** explique que um mesmo metadado publicado governa persistência,
   REST, interface, autorização e ferramentas da IA.
5. **MVP:** reforce a disciplina de escopo. O projeto precisa provar primeiro o
   núcleo dinâmico e o isolamento seguro.
6. **Diferenciais:** mostre por que a proposta não é somente um gerador de
   formulários ou um CRUD genérico.
7. **Arquitetura:** percorra o desenho da esquerda para a direita. Ressalte a
   borda controlada, os serviços internos e a ausência de acesso da LLM ao banco.
8. **Monólito modular:** explique que ele reduz o custo operacional inicial sem
   abrir mão de limites que possibilitem extração futura.
9. **Dados:** enfatize tabela física por entidade, segregação por
   `organization_id` e ativação global com recuperação previsível.
10. **Identidade:** esclareça que o Keycloak trata credenciais e federação; a
    autorização dinâmica continua sob responsabilidade do ENT.IA.
11. **Autorização e auditoria:** apresente segurança e evidência como parte do
    mesmo fluxo, explicitando a limitação inicial de não possuir WORM externo.
12. **IA:** diferencie sugestão de execução. A IA usa ferramentas tipadas, e a
    REST API continua sendo a autoridade final.
13. **Stack:** conecte as escolhas à experiência Java, ao ecossistema aberto e à
    possibilidade de substituir componentes com licença ou custo inadequados.
14. **Fases:** peça investimento por gate, sempre condicionado a uma evidência
    verificável da fase anterior.
15. **Riscos:** demonstre que os riscos centrais já foram identificados e estão
    influenciando o recorte do produto.
16. **Métricas:** proponha que os valores-alvo sejam definidos com o domínio
    piloto, sem fabricar estimativas antes dos requisitos não funcionais.
17. **Decisões:** registre responsáveis e prazo para cada decisão que sair da
    reunião.
18. **Encerramento:** solicite autorização para o Gate 0 e a indicação da
    entidade piloto e do patrocinador técnico.

## Perguntas esperadas

### Por que não usar JSONB ou NoSQL para todos os registros?

O catálogo pode usar JSONB para partes flexíveis, mas os registros de negócio
terão tabela física por entidade. Essa escolha evita concentrar todo o volume em
uma tabela, permite constraints e índices próprios e preserva melhor a
integridade relacional. O custo assumido é manter um motor seguro de evolução de
schema.

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
plenamente contra superusuário. Uma âncora ou armazenamento WORM externo é uma
evolução possível quando o requisito justificar a complexidade operacional.

### A IA poderá alterar dados sem controle?

Não. Ela não terá banco, SQL livre ou HTTP arbitrário. Leituras passam pelas
permissões do usuário; criação, alteração e exclusão exigem pré-visualização,
confirmação explícita, idempotência e nova autorização na API REST.

### Quanto tempo e investimento serão necessários?

Uma estimativa responsável depende do domínio piloto, equipe, requisitos não
funcionais, volume, topologia e decisões técnicas do Gate 0. A recomendação é
financiar primeiro esse detalhamento e um spike do núcleo dinâmico, usando seus
resultados para estimar o MVP.

## Resultado mínimo esperado da reunião

- tese e recorte do MVP aceitos, rejeitados ou ajustados;
- autorização ou não do Gate 0 e do spike técnico;
- domínio piloto e patrocinador técnico indicados;
- responsáveis pelas decisões abertas;
- critérios objetivos para autorizar o Gate 1.
