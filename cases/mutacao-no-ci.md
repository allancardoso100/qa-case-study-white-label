# Case: paramos uma entrega antes que o pipeline parasse

[← Voltar ao início](../README.md) · [Linha do tempo](../linha-do-tempo.md)

> Bloco 9M, 21/11/2026. Decisão registrada no ADR 0033.

## O problema

O teste de mutação (StrykerJS) altera o código de propósito, um "mutante" por vez, e confere se algum teste falha. É a medida mais honesta da força de uma suíte, e também a mais cara.

A mutação do painel levava **de 13 a 17 minutos** no pipeline, num limite de **20 minutos por job**, e crescia a cada entrega. Mais dois ou três blocos e o pipeline começaria a falhar por tempo, não por defeito.

Havia também o orçamento. O repositório é privado e usa o plano gratuito do GitHub Actions: **2.000 minutos por mês**. Cada push num pull request gastava cerca de 64 minutos, metade só em mutação. **Custo zero era requisito.**

## A decisão de parar

O bloco seguinte, de envio de imagens, ia acrescentar código e testes justamente ao painel. Em vez de começá-lo e descobrir o limite no meio, **paramos a próxima entrega por um bloco curto**, dedicado só a isso.

## Medir antes de mexer

O diagnóstico usou os próprios relatórios do pipeline:

| Causa | Peso |
|---|---|
| **Mutantes estáticos:** código que roda ao carregar o arquivo (tabelas, listas, expressões no topo). Para cada um deles, a suíte inteira precisa rodar | 463 mutantes do painel, **23%** do total, consumiam **91%** do tempo |
| **Navegador simulado (jsdom) em todos os testes** do painel, embora só os testes de tela precisem dele | **85%** do tempo de cada rodada; só 15 dos 481 testes são de tela |

## O que fizemos

| Mudança | Efeito |
|---|---|
| Cada teste no ambiente de que precisa: navegador simulado só nos testes de tela | Suíte do painel de 10,3 s para 3,9 s |
| Mutação **incremental** nos pull requests: só o que mudou é testado de novo | O tempo deixa de crescer junto com o projeto |
| Mutação **completa** depois de cada merge, toda semana e sob pedido | A régua inteira continua sendo aplicada |
| Regra minha: **alterou o que já existe, revisa os testes de todas as camadas**, e a mutação do pacote roda completa no pull request | Mudança em código antigo nunca passa só com o incremental |
| Aviso no pipeline a partir de 12 minutos | Folga para agir antes do limite de 20 |

A regra de quando a mutação é completa ou incremental virou código com 17 testes, e um teste-guarda confere que o pipeline continua aplicando essa regra.

## O que ficou de fora, de propósito

Cada uma destas opções resolveria o tempo, e todas cobrariam em qualidade ou em orçamento:

- ignorar os mutantes estáticos (`ignoreStatic`), que incluem textos e tabelas que já pegaram defeitos reais;
- tirar arquivos da mutação;
- baixar o corte de 70% ou a meta de 100%;
- só aumentar o limite de tempo, adiando o problema;
- pagar por máquinas maiores.

## Resultado

| Medida | Antes | Depois |
|---|---|---|
| Mutação do painel no pipeline | 14min29s | **7min37s** |
| Mutantes do painel | 1.999 | 1.999, os mesmos |
| Mutation score | 100% | **100%**, conferido mutante a mutante |
| Custo extra | | **R$ 0** |

## A lição

Qualidade também é saber a hora de parar. Agir com o problema ainda pequeno custou um bloco curto. Esperar estourar custaria entregas travadas e a tentação de cortar testes para o pipeline caber no tempo.

*Eu defini as regras e aprovei o plano, as decisões e o merge. O código foi escrito pelo agente de IA (Claude Code).*
