# Linha do tempo

[← Voltar ao início](README.md)

O projeto avança em **blocos**. Cada bloco segue o mesmo ritual: plano aprovado, branch, implementação, testes, relatório, pull request e merge só com o pipeline verde e a minha aprovação. Esta tabela ganha uma linha a cada bloco, e um teste no pipeline barra o pull request se ela ficar para trás.

| Bloco | Data | O que entregou | Destaque de qualidade |
|---|---|---|---|
| 00 | 16/11/2026 | Ambiente preparado: ferramentas instaladas e conferidas | Versões registradas no relatório, para reproduzir a máquina |
| 01 | 16/11/2026 | Arquitetura documentada: microsserviços, um banco por serviço, eventos | Primeiros ADRs, com alternativas e consequências |
| 02 | 16/11/2026 | Estratégia de testes | Pirâmide, quality gates e matriz de riscos definidos antes do código |
| 03 | 16/11/2026 | Monorepo, ferramentas de qualidade, infraestrutura local e pipeline | Quality gates no pipeline desde o primeiro pull request |
| 04 | 16/11/2026 | White Label ponta a ponta: duas construtoras, um código | Suíte `@isolamento`, que nunca entra em quarentena |
| 05 | 17/11/2026 | Catálogo de empreendimentos e portfólio no site | Defeito escapado da fase anterior encontrado e coberto por teste de regressão |
| 06 | 17/11/2026 | Login e permissões com Keycloak | O risco crítico da matriz (falha de permissão) ganhou um bloco só dele |
| 07A | 17/11/2026 | Fundação do cadastro: operador, usuários, auditoria e eventos | Auditoria obrigatória e eventos com outbox: o evento só sai se a transação gravou |
| 07B | 18/11/2026 | Painel com login e cadastro de construtoras | Painel como BFF, com os tokens só no servidor |
| 07C | 18/11/2026 | Equipe, empreendimentos e histórico no painel | Dois convites simultâneos ocupavam a última vaga duas vezes: pego na integração |
| VT | 18/11/2026 | Esta vitrine pública | Publicada como cópia, sem o histórico privado |
| 08A | 19/11/2026 | Catálogo para todos os segmentos, locação e edição | A mutação estourou o tempo no pipeline: um runner por pacote |
| 08B | 20/11/2026 | Busca, paginação e importação por planilha | Segredo que chegava vazio ao E2E só no pipeline virou teste-guarda |
| 09A | 21/11/2026 | Personalização do site: textos, identidade, publicação e histórico | Matriz perfil × ação × chave com 64 casos de isolamento e política de segurança de conteúdo |
| 09M | 21/11/2026 | Mutação no pipeline sob controle | De 14min29s para 7min37s, com os mesmos mutantes e 100%. [Ver o case](cases/mutacao-no-ci.md) |
| 09V | 24/11/2026 | Vitrine revista em todo bloco e publicada automaticamente | Teste-guarda da vitrine; o E2E semanal ganhou um job por navegador |
| 09B | 24/11/2026 | Envio seguro do logo, do ícone, da capa e da imagem de compartilhamento | Toda imagem refeita no servidor, sem a localização GPS; privada até publicar; ataques reais nos testes |
| 09C | 24/11/2026 | Fotos e plantas dos empreendimentos enviadas pelo painel | O espaço público espelha o que está no ar: ao despublicar, a foto sai do endereço público; teste de privacidade passou a usar imagens reais |
| 10A | 26/11/2026 | Parceiros e canais de venda: "Onde comprar ou alugar" na página do empreendimento | Os contratos entre serviços pegaram um campo que o catálogo nunca devolveu, antes de chegar à produção; contador de cliques sem dado pessoal e sem redirecionamento aberto |
