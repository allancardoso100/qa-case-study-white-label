# Trabalho com IA

[← Voltar ao início](README.md)

Este projeto foi construído com o **Claude Code**, um agente de IA que lê o repositório, roda comandos, escreve código e abre pull requests. Quem escreveu o código foi o agente. O meu trabalho foi o de dono do produto e responsável pela qualidade: dizer o que construir, definir como provar que funciona e não deixar passar atalho.

## Quem faz o quê

| Eu | O agente |
|---|---|
| Decido o que o produto faz e para quem | Propõe o plano de cada bloco, com alternativas e recomendação para cada decisão |
| Aprovo o plano antes de qualquer código | Implementa, escreve os testes e a documentação |
| Defino as regras de qualidade e os gates | Roda a pirâmide inteira e corrige o que falha |
| Aprovo cada merge e reviso os relatórios | Abre o PR com relatório e espera a aprovação |

## As regras que o agente segue

Ficam num arquivo de instruções (`CLAUDE.md`) que o agente lê no início de toda sessão. As principais:

- **Arquitetura que não se quebra:** serviço não importa código de outro serviço; um banco por serviço; domínio sem framework; toda tabela de negócio com `tenant_id`.
- **Pirâmide e gates:** cobertura ≥ 90% no domínio, mutation score ≥ 70%, zero erros de lint, de tipos e de regras de arquitetura.
- **Teste instável vai para quarentena com issue aberta.** Nunca se esconde instabilidade com `retry`.
- **Isolamento entre clientes nunca entra em quarentena.** Uma falha ali bloqueia o merge, sem exceção.
- **Todo defeito corrigido ganha um teste de regressão** que falhava antes da correção.
- **Nunca pular os ganchos do Git** (`--no-verify`) e **nunca mesclar sem CI verde e a minha aprovação.**

## O ritual de cada entrega

1. Plano em linguagem de negócio, com as decisões novas separadas do que só repete padrões já aprovados.
2. Branch própria e implementação.
3. Pirâmide completa: unitário, integração com banco e fila reais, contrato, E2E com acessibilidade e mutação.
4. Relatório com o que foi instalado, os arquivos, as decisões, os testes por camada e as pendências, sem eufemismo.
5. Pull request, CI verde e merge só com a minha aprovação.

## Quando precisei corrigir o rumo

O agente erra, e parte do trabalho é perceber quando. Alguns momentos reais:

**Plano sem as decisões à vista.** O plano da Fase 2 chegou direto para aprovação, misturando o que era rotina com escolhas novas de arquitetura. Recusei. Desde então, todo plano lista as decisões novas, com alternativas e recomendação, antes de pedir o "ok".

**Uma regra que não era minha.** O código contava parceiros e corretores como vagas do pacote da construtora. O agente levou isso a mim como uma "questão de negócio" e registrou a correção como "decisão do usuário". Não era: era um defeito de implementação. Ele foi corrigido e registrado como defeito, e a regra passou a ser outra: comportamento que o plano não previa é tratado primeiro como defeito, e documento só diz "decisão do usuário" quando eu de fato decidi.

**Sem atalhos para passar.** Deixei claro que teste afrouxado, asserção ajustada para passar ou meta rebaixada não são solução. Na prática:

- quando a mutação do painel ficou em 97,2%, a saída foi escrever os casos que faltavam e simplificar o código redundante. Os poucos mutantes equivalentes ficaram marcados na linha, com o motivo;
- quando a integração estourou o tempo com três Keycloaks subindo juntos, o agente mediu (cerca de 115 s para dois, 170 s para três) e dividiu a execução em etapas. **Nenhum prazo foi aumentado.**

**Parar uma entrega na hora certa.** A mutação do painel chegou a 17 de 20 minutos no pipeline. Em vez de começar o bloco seguinte, que ia acrescentar código justamente ao painel, pedi ao agente um plano estruturado para resolver o tempo **sem abrir mão de cobertura nem de qualidade**, dentro do plano gratuito. Ele mediu as causas, propôs as opções com custo e risco, e eu escolhi. Também acrescentei uma regra: **alterou o que já existe, revisa os testes de todas as camadas**, inclusive a mutação. Detalhes no [case](cases/mutacao-no-ci.md).

**A própria vitrine ficou para trás.** Esta página ficou quatro blocos sem atualização, porque a regra "atualizar ao fechar o bloco" dependia de alguém lembrar. Percebi ao preparar uma publicação. A solução foi a mesma que uso para o produto: a revisão entrou no ritual de cada bloco, um teste no pipeline barra o pull request com a vitrine atrasada, e a cópia pública passou a sair automaticamente no merge.

**Siglas viram dúvida de custo.** Num plano, li "RS256" e achei que era um valor em reais. Desde então, os planos explicam cada sigla na primeira menção e dizem explicitamente se algo tem custo.

## O que aprendi

- **A IA acelera a escrita, não o critério.** A qualidade veio de regras explícitas, escritas antes, e de alguém verificando cada entrega.
- **Teste de mutação é a melhor defesa contra teste "para passar".** Cobertura alta não prova nada sozinha. Mutante sobrevivente mostra exatamente onde o teste não protege.
- **Documentar decisões é o que permite revisar depois.** Com ADRs e relatórios por bloco, consigo auditar o que foi feito sem reler todo o código.
- **As correções viram regra.** Cada correção de rumo ficou registrada na memória do agente e nas instruções do projeto, para não precisar ser repetida a cada sessão.
