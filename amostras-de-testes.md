# Amostras de testes

[← Voltar ao início](README.md)

Um exemplo real de cada camada da pirâmide, copiado do repositório privado (alguns trechos foram encurtados). Os nomes dos testes ficam em português de propósito: eles descrevem a regra de negócio.

## 1. Unitário: objeto de valor com casos de borda

Uma cor de marca inválida não pode entrar no domínio. Onze entradas inválidas, uma por linha, cada uma por um motivo diferente.

<sub>Origem: `services/tenant-service/src/domain/hex-color.spec.ts`</sub>

```ts
describe('HexColor', () => {
  it('deve criar a cor quando estiver no formato #RRGGBB', () => {
    expect(HexColor.create('#1F4E8C').value).toBe('#1F4E8C');
  });

  it('deve normalizar para maiúsculas', () => {
    expect(HexColor.create('#1f4e8c').value).toBe('#1F4E8C');
  });

  it.each([
    '1F4E8C', '#1F4E8', '#1F4E8CA', '#GGGGGG', '#1F4E8 ', '',
    '   ', 'azul', '#FFF', 'x#1F4E8C', '##1F4E8C',
  ])('deve rejeitar "%s" por não estar no formato #RRGGBB', (invalid) => {
    expect(() => HexColor.create(invalid)).toThrow(InvalidHexColorError);
  });

  it('deve considerar iguais cores com o mesmo valor normalizado', () => {
    expect(HexColor.create('#abcdef').equals(HexColor.create('#ABCDEF'))).toBe(true);
    expect(HexColor.create('#ABCDEF').equals(HexColor.create('#ABCDEE'))).toBe(false);
  });
});
```

## 2. Unitário: a "prova de sensibilidade"

Um teste que passa sempre não prova nada. Este par mostra as duas metades: com a trava, a corrida pela última vaga é barrada; **sem a trava, a mesma corrida fura o limite**. Se o segundo teste um dia deixar de furar, o primeiro perdeu o poder de detectar a regressão.

<sub>Origem: `services/user-service/src/application/user-management.spec.ts`</sub>

```ts
it('dois convites ao mesmo tempo não ocupam a última vaga duas vezes (R41)', async () => {
  contracts.set(TENANT, 1);

  const results = await Promise.allSettled([
    users.invite(admin(), invitation({ email: 'a@construtora.example' })),
    users.invite(admin(), invitation({ email: 'b@construtora.example' })),
  ]);

  expect(results.filter((result) => result.status === 'fulfilled')).toHaveLength(1);
  const [rejected] = results.filter((result) => result.status === 'rejected');
  expect(rejected?.reason).toBeInstanceOf(UserLimitReachedError);
  expect(enabledCount()).toBe(1);
});

it('prova de sensibilidade: sem a exclusividade por construtora, a mesma corrida furaria o limite', async () => {
  // Caracteriza o defeito que a trava evita: se este teste deixar de furar, o anterior perdeu o
  // poder de detectar a regressão e precisa ser revisto.
  const semTrava: TenantStates = Object.assign(new InMemoryTenantStates(), {
    exclusively: <T>(_tenantId: string, work: () => Promise<T>): Promise<T> => work(),
  });
  const semExclusividade = new UserManagement(identity, audit, semTrava, contracts);
  contracts.set(TENANT, 1);

  await Promise.all([
    semExclusividade.invite(admin(), invitation({ email: 'a@construtora.example' })),
    semExclusividade.invite(admin(), invitation({ email: 'b@construtora.example' })),
  ]);

  expect(enabledCount()).toBe(2);
});
```

## 3. Integração: duas cópias do serviço disputando a última vaga

O unitário prova a regra; a integração prova que ela vale em produção, onde o serviço roda em mais de uma cópia. Aqui são **duas instâncias do serviço sobre o mesmo PostgreSQL real** (Testcontainers), cada uma recebendo um convite ao mesmo tempo. Também confere a tabela de saída de eventos: nenhum evento fantasma para o convite recusado.

<sub>Origem: `services/user-service/test/integration/user-limit.int-spec.ts`</sub>

```ts
describe('Corrida entre cópias do serviço (R41)', () => {
  it('dois convites ao mesmo tempo, um em cada cópia, não ocupam a última vaga duas vezes', async () => {
    tenantService.limits.set(tenantId, 1);

    const respostas = await Promise.all([invite(0), invite(1)]); // cópia 0 e cópia 1

    expect(respostas.map((resposta) => resposta.status).sort()).toEqual([201, 409]);
    expect(enabledOf(tenantId)).toBe(1);
    expect(await outboxOf(tenantId)).toBe(1);
  });

  it('construtoras diferentes convidam ao mesmo tempo sem esperar uma pela outra', async () => {
    const outra = randomUUID();
    tenantService.limits.set(tenantId, 1).set(outra, 1);

    const respostas = await Promise.all([invite(0, tenantId), invite(1, outra)]);

    expect(respostas.map((resposta) => resposta.status)).toEqual([201, 201]);
  });
});
```

O segundo teste cuida do outro lado: a trava é **por construtora**, então ela não pode virar um gargalo para a plataforma inteira.

## 4. Integração: isolamento entre construtoras (`@isolamento`)

Toda rota com dados de cliente tem este tipo de teste, e ele nunca entra em quarentena. Inclui a tentativa de acessar o empreendimento de outra construtora pelo endereço direto (IDOR), que precisa devolver o mesmo 404 neutro de um recurso inexistente.

<sub>Origem: `services/catalog-service/test/integration/development-http.int-spec.ts`</sub>

```ts
it('não deve listar empreendimentos de outra construtora @isolamento', async () => {
  const [responseA, responseB] = await Promise.all([
    get(`/tenants/${TENANT_A}/developments`),
    get(`/tenants/${TENANT_B}/developments`),
  ]);

  expect(JSON.stringify(responseA.body)).not.toContain('Boreal');
  expect((responseB.body as { slug: string }[]).map((item) => item.slug)).toEqual([
    'boreal-logistico',
  ]);
});

it.each([
  ['inexistente', `/tenants/${TENANT_A}/developments/nao-existe`],
  ['em rascunho (R19)', `/tenants/${TENANT_A}/developments/projeto-sigiloso`],
  [
    'de outra construtora pelo slug (IDOR) @isolamento',
    `/tenants/${TENANT_A}/developments/boreal-logistico`,
  ],
])('deve retornar o mesmo 404 neutro para empreendimento %s', async (_description, path) => {
  const response = await get(path);

  expect(response.status).toBe(404);
  expect(response.body).toEqual({
    statusCode: 404,
    error: 'DEVELOPMENT_NOT_FOUND',
    message: 'Empreendimento não encontrado.',
  });
});
```

As três situações respondem exatamente igual. Assim, quem tenta adivinhar endereços não descobre se o empreendimento existe, se está em rascunho ou se é de outra construtora.

## 5. Contrato: o site define o que espera do serviço de construtoras (Pact)

O consumidor (site) descreve a resposta de que precisa; o provedor (serviço de construtoras) verifica o contrato no seu próprio CI. Se alguém renomear um campo, o merge é bloqueado antes de o site quebrar.

<sub>Origem: `apps/site-publico/test/contract/tenant-service.pact.spec.ts`</sub>

```ts
const { like, regex, eachLike, uuid } = MatchersV3;
const HEX_COLOR = '^#[0-9A-F]{6}$';

it('deve obter o tema da construtora pelo domínio', async () => {
  await provider
    .addInteraction()
    .given('existe a construtora com domínio construtora-a.localhost')
    .uponReceiving('uma consulta de construtora por domínio existente')
    .withRequest('GET', '/tenants/by-domain/construtora-a.localhost', (builder) => {
      builder.headers({ accept: 'application/json' });
    })
    .willRespondWith(200, (builder) => {
      builder.jsonBody({
        id: uuid('a0a0a0a0-0000-4000-8000-000000000001'),
        slug: like('construtora-aurora'),
        name: like('Construtora Aurora'),
        primaryDomain: like('construtora-a.localhost'),
        domains: eachLike('construtora-a.localhost'),
        theme: {
          primaryColor: regex(HEX_COLOR, '#1F4E8C'),
          secondaryColor: regex(HEX_COLOR, '#F2A900'),
          logoUrl: like('/logos/aurora.svg'),
        },
      });
    })
    .executeTest(async (mockServer) => {
      const tenant = await fetchTenantByDomain('construtora-a.localhost', {
        baseUrl: mockServer.url,
      });

      expect(tenant?.name).toBe('Construtora Aurora');
      expect(tenant?.theme.primaryColor).toBe('#1F4E8C');
    });
});
```

## 6. E2E: cada domínio com a sua marca, com acessibilidade

A jornada crítica do White Label, no navegador: o domínio mostra a marca certa, não mostra nada da outra construtora e não tem violações graves de acessibilidade.

<sub>Origem: `e2e/tests/white-label.e2e.ts`</sub>

```ts
for (const construtora of [AURORA, BOREAL]) {
  test.describe(construtora.name, () => {
    test(`deve exibir a marca da ${construtora.name} no domínio dela @isolamento`, async ({ page }) => {
      const resposta = await page.goto(siteUrl(construtora.host));

      expect(resposta?.status()).toBe(200);
      await expect(page).toHaveTitle(construtora.name);
      await expect(page.getByRole('heading', { level: 1, name: construtora.name })).toBeVisible();
      await expect(page.getByRole('img', { name: `Logo da ${construtora.name}` })).toBeVisible();
      await expect(page.getByRole('banner')).toHaveCSS('background-color', construtora.primaryRgb);
      await expect(page.getByText(construtora.outra)).toHaveCount(0);
    });

    test('não deve ter violações graves de acessibilidade', async ({ page }) => {
      await page.goto(siteUrl(construtora.host));

      await expectSemViolacoesGraves(page);
    });
  });
}
```

O auxiliar de acessibilidade é o gate G8 em código: WCAG 2.0 e 2.1, níveis A e AA, e a falha lista cada regra violada com os elementos afetados.

<sub>Origem: `e2e/tests/support/acessibilidade.ts`</sub>

```ts
export async function expectSemViolacoesGraves(page: Page): Promise<void> {
  const resultado = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
    .analyze();
  const graves = resultado.violations
    .filter((violacao) => violacao.impact === 'serious' || violacao.impact === 'critical')
    .map((violacao) => ({
      regra: violacao.id,
      impacto: violacao.impact,
      descricao: violacao.help,
      elementos: violacao.nodes.map((node) => node.target.join(' ')),
    }));

  expect(graves, 'violações graves de acessibilidade').toEqual([]);
}
```
