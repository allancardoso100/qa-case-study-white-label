# Decisões

[← Voltar ao início](README.md)

Toda decisão de arquitetura vira um **ADR** (registro de decisão de arquitetura): o contexto, as alternativas consideradas, a escolha, as consequências e como migrar no futuro. Em 26/11/2026 eram 37. Estas são as que mais mostram o raciocínio por trás da qualidade.

| ADR | Decisão | Por quê |
|---|---|---|
| 0007 | **Arquitetura hexagonal** em cada serviço | O domínio só conhece TypeScript puro. Regras de negócio são testadas em milissegundos, sem banco nem HTTP, e a mutação mede a força desses testes |
| 0012 | **Multi-tenant por domínio** | Trocar de marca é trocar dados, não código. O preço é o risco de vazamento entre clientes, e por isso toda tabela tem `tenant_id`, todo repositório filtra por ele e há teste que prova o isolamento |
| 0017 | **Uma ferramenta por camada da pirâmide** | Vitest e Testing Library no unitário, Testcontainers na integração, Pact no contrato, Playwright com axe no E2E e StrykerJS na mutação. Cada uma no papel em que é melhor |
| 0018 | **Qualidade e segurança estática a custo zero** | TypeScript estrito, ESLint, dependency-cruiser, gitleaks, Semgrep, Trivy e `pnpm audit`: o defeito é barrado antes de existir um teste para ele |
| 0019 | **GitHub Actions e trunk-based** | O pipeline roda lint, tipos, arquitetura, todas as camadas de teste e segurança em cada PR. Para caber nos 2.000 minutos gratuitos: só o que mudou roda, com cache e E2E apenas em PR |
| 0022 | **SeaweedFS como S3 local** | A imagem do MinIO usada antes deixou de estar disponível. O código não conhece o SeaweedFS: fala só a API S3 por uma porta da arquitetura, então trocar o armazenamento não muda nenhuma regra |
| 0023 | **TypeScript fixo na 6.0** | O ecossistema (typescript-eslint, dependency-cruiser, Next.js) ainda não suportava a 7. A decisão registra o gatilho de migração em vez de arriscar o CI quebrado |
| 0026 | **Construtora no token, editável só pelo administrador** | O próprio usuário não consegue trocar a construtora do token, e um teste de contrato contra um Keycloak real prova isso |
| 0027 | **Operador global com auditoria obrigatória e eventos com outbox** | Token com perfil de operador **e** construtora é recusado: nunca existe privilégio ambíguo. Cada ação do operador fica registrada, e o evento só sai se a transação gravou |
| 0028 | **Painel como BFF, com sessão no servidor** | O navegador recebe só um cookie HttpOnly. Os tokens ficam no servidor, com login por PKCE e proteção contra *open redirect* e CSRF |
| 0029 | **Limite de usuários conferido na hora, com falha fechada** | Se o serviço de construtoras não responder, o convite é recusado (503). O sistema nunca supõe "ilimitado". Sem vaga, a pessoa recebe a explicação em português (409) |
| 0030 | **Catálogo multissegmento com edição por versão** | Atributos próprios de cada segmento, venda e locação. Duas pessoas editando o mesmo empreendimento não se apagam: a versão antiga é recusada (409) |
| 0032 | **Personalização do site com rascunho e publicações imutáveis** | Nada vai ao ar sem publicar. Cada publicação é uma foto imutável, protegida por gatilho no banco, e restaurar uma versão cria um rascunho para revisar |
| 0033 | **Mutação incremental nos PRs, com completas** | O tempo da mutação caiu pela metade sem perder nenhum mutante. [Ver o case](cases/mutacao-no-ci.md) |
| 0034 | **Vitrine revista em todo bloco e publicada no merge** | Um teste no pipeline barra o PR com a vitrine atrasada, e a cópia pública sai automaticamente, depois do gitleaks |
| 0035 | **Imagens enviadas refeitas no servidor e privadas até publicar** | O formato é conferido pelo conteúdo, a "bomba de descompressão" é barrada antes de abrir a imagem, e cada tamanho sai como imagem nova, sem a localização GPS. A imagem só vai ao espaço público quando a construtora publica |
| 0036 | **Fotos e plantas: o espaço público espelha o que está no ar** | Envio e escolha separados, a escolha só aceita imagens do próprio empreendimento, e uma peça só copia ao publicar e retira ao despublicar: copiar antes de gravar, retirar depois |
| 0037 | **Parceiros e canais de venda: contador sem dado pessoal e sem trampolim** | O botão do parceiro passa por um contador no próprio site, que leva só ao destino gravado pela construtora: nunca a um endereço vindo da URL. A tabela de cliques não tem coluna para dado pessoal, e dois exclusivos ao mesmo tempo passam só um, por uma trava no banco testada em corrida |

## O padrão de um ADR

Cada registro segue a mesma estrutura:

1. **Contexto:** o problema e as forças em jogo.
2. **Decisão:** o que foi escolhido, de forma verificável.
3. **Alternativas consideradas:** e por que foram descartadas.
4. **Consequências:** o que melhora, o que piora e os riscos novos.
5. **Migração futura:** como sair dessa escolha se o projeto crescer ou passar a ter orçamento.
