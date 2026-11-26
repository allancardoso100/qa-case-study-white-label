# Qualidade

[← Voltar ao início](README.md)

A estratégia parte de um princípio simples: **prevenir antes de detectar**, e testar cada comportamento no nível mais baixo possível. O risco decide onde investir mais: o que pode causar dano a um cliente é testado em mais de um nível e protegido por gate bloqueante.

## Princípios

| Princípio | Na prática |
|---|---|
| **Shift-left** | Tipos estritos, lint, regras de arquitetura e objetos de valor barram defeitos antes de qualquer teste rodar |
| **Pirâmide** | Muitos testes unitários, menos de integração, poucos E2E |
| **Risco guia o esforço** | Áreas críticas da matriz de riscos têm testes em dois ou mais níveis |
| **Comportamento, não implementação** | Refatorar sem mudar o comportamento não pode quebrar teste |
| **Determinismo** | Nada de `sleep`, relógio real ou aleatoriedade sem semente |
| **Infraestrutura real na integração** | Postgres, RabbitMQ e Keycloak de verdade (Testcontainers). Mock de banco esconde os defeitos que mais importam |
| **Testes como documentação** | Os nomes dos testes, em português, descrevem as regras de negócio |
| **Regressão antes da correção** | Todo defeito ganha um teste que falha antes da correção e passa depois |

## A pirâmide hoje

| Camada | Ferramentas | Testes | Parcela |
|---|---|---:|---:|
Números de 24/11/2026 (bloco 09C):

| Camada | Ferramentas | Testes | Parcela |
|---|---|---:|---:|
| Unitário | Vitest, Testing Library | 2.853 | 79% |
| Integração | Testcontainers, Supertest | 633 | 18% |
| Contrato | Pact | 57 | 3% (com E2E) |
| E2E com acessibilidade | Playwright + axe-core | 54 | |
| **Total** | | **3.597** | |

A mutação (StrykerJS) roda sobre o domínio e os casos de uso: **100%** de mutation score nos seis pacotes com mutação (painel, biblioteca de imagens e serviços de catálogo, construtoras, usuários e auditoria), com 6.007 mutantes mortos.

### Como a mutação cabe no orçamento

- Nos pull requests, a mutação é **incremental**: só o que mudou é testado de novo.
- Depois de cada merge, toda semana e sob pedido, ela roda **completa**.
- **Alterou o que já existe, revisa os testes:** quando um pull request muda código que já existia, os testes de todas as camadas são revistos e a mutação do pacote roda completa.
- O pipeline avisa quando uma rodada passa de 12 minutos, com folga antes do limite de 20.

A história de como chegamos aqui está no [case da mutação no pipeline](cases/mutacao-no-ci.md).

### Guardas do repositório

Alguns testes não testam o produto: testam o próprio projeto. Cada um nasceu de um problema real:

| Guarda | O que impede |
|---|---|
| Matriz de mutação | Um pacote novo com mutação ficar fora do pipeline |
| Navegadores do E2E | Um navegador novo ficar fora da rodada semanal, ou os navegadores voltarem a dividir um job só |
| Variáveis do Turborepo | Uma variável lida pelo E2E chegar vazia só no pipeline |
| Saídas da geração de código | O cache do Turborepo pular uma geração necessária |
| Rótulos do histórico | Uma ação nova aparecer no painel com o código técnico em vez do texto em português |
| Caracteres invisíveis | Caracteres de controle e de direção de texto escondidos no código (Trojan Source) |
| Vitrine em dia | Esta vitrine ficar para trás, apontar para fora de si mesma ou citar algo privado |

## Quality gates

Todos verificados automaticamente no CI. Sem eles verdes, não há merge.

| # | Critério | Meta |
|---|---|---|
| G1 | Tipos, lint e formatação | 0 erros |
| G2 | Regras de arquitetura (dependency-cruiser) | 0 violações |
| G3 | Testes de todas as camadas | 100% passando |
| G4 | Cobertura de `domain/` e `application/` | ≥ 90% |
| G5 | Cobertura geral (unitário + integração) | ≥ 80% |
| G6 | Mutation score no domínio | ≥ 70% |
| G7 | Contratos verificados pelo provedor | Todos |
| G8 | Acessibilidade (WCAG 2.1 A e AA) | 0 violações graves ou críticas |
| G9 | Segredos no repositório (gitleaks) | 0 |
| G10 | Vulnerabilidades (pnpm audit, Trivy, Semgrep) | 0 altas ou críticas sem justificativa |
| G11 | Suíte de isolamento entre clientes | 100%, nunca em quarentena |

> **Cobertura não é meta, é alarme.** Cobertura alta com teste fraco é falsa segurança. Por isso a mutação pesa tanto quanto a cobertura.

## Isolamento entre clientes

O vazamento de dados entre construtoras é o risco nº 1. Ele tem uma suíte própria, marcada com `@isolamento`, em três níveis:

| Nível | O que verifica |
|---|---|
| Integração (repositório) | Dados criados nas construtoras A e B; toda consulta no contexto de A devolve só dados de A |
| Integração (HTTP) | Requisição de A nunca devolve recurso de B, nem pelo identificador direto (proteção contra IDOR) |
| E2E | Cada domínio mostra só a marca e o conteúdo da sua construtora; domínio desconhecido mostra uma página neutra |

Depois vieram as matrizes de **perfil × endpoint** (autorização), de eventos e auditoria, de sessões do painel e de limite de usuários.

## Matriz de riscos

Cada risco recebe uma nota de probabilidade e de impacto (1 a 5). A exposição (P × I) define o tratamento:

| Exposição | Classe | Tratamento |
|---|---|---|
| 15–25 | Crítico | Testes em 2 ou mais níveis, gate bloqueante e revisão específica no PR |
| 8–14 | Alto | Pelo menos um nível dedicado e gate |
| 4–7 | Médio | Cobertura pelos testes regulares da pirâmide |
| 1–3 | Baixo | Monitorar |

Exemplos:

| Risco | Exposição | Como é mitigado |
|---|---:|---|
| Vazamento de dados entre construtoras | 15 | Suíte `@isolamento` na integração e no E2E, checklist no PR |
| Marca errada exibida (cache contaminado entre domínios) | 12 | Unitário na resolução do domínio, contrato com o serviço de construtoras, E2E com duas marcas |
| Quebra de contrato entre serviços | 12 | Pact bloqueando o merge |
| Entrada inválida corrompendo o domínio | 9 | Objetos de valor com casos de borda, validação retornando 400, mutação |
| Segredo exposto no repositório | 10 | gitleaks no pre-commit e no CI |

Cada risco aponta para os testes que o cobrem, e a matriz é revisada a cada bloco.

## Testes instáveis e defeitos

- **Teste instável vai para quarentena com issue aberta**: 5 dias úteis para corrigir, no máximo 3 em quarentena ao mesmo tempo. O CI roda com `retries: 0`; `retry` para esconder instabilidade não é aceito.
- **Defeito escapado** é analisado pelo nível em que deveria ter sido pego, e o teste é reforçado ali.
- **Cada relatório de bloco** traz os defeitos encontrados, quem os pegou (unitário, integração, contrato, E2E ou mutação) e a ação tomada.
