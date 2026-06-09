# Guia de QA com Claude Code — Agents e Skills

> Para QA Juniors, Plenos e Sêniors. O foco aqui não é listar o que existe — é te ensinar a **usar, adaptar e criar** do jeito certo.

---

## Sumário

1. [Como tudo funciona junto](#1-como-tudo-funciona-junto)
2. [Como instalar](#2-como-instalar)
3. [Como usar no dia a dia](#3-como-usar-no-dia-a-dia)
4. [Boas práticas de escrita de testes — Playwright](#4-boas-práticas-de-escrita-de-testes--playwright)
5. [Boas práticas de escrita de testes — Cypress](#5-boas-práticas-de-escrita-de-testes--cypress)
6. [Por que Page Object Model não funciona no Playwright](#6-por-que-page-object-model-não-funciona-no-playwright)
7. [Playwright MCP e cy.prompt — quando usar e quando evitar](#7-playwright-mcp-e-cyprompt--quando-usar-e-quando-evitar)
8. [Como as skills funcionam e quando usá-las](#8-como-as-skills-funcionam-e-quando-usá-las)
9. [Como editar uma skill existente](#9-como-editar-uma-skill-existente)
10. [Como criar uma skill nova](#10-como-criar-uma-skill-nova)
11. [Como criar um agent novo](#11-como-criar-um-agent-novo)
12. [O que não fazer](#12-o-que-não-fazer)

---

## 1. Como tudo funciona junto

O Claude Code é um assistente generalista. Por padrão, ele não sabe nada sobre o **seu** projeto — convenções de pastas, padrões de seletor, como fazer login, o que é proibido.

**Agents** resolvem isso. Um agent é um arquivo `.md` que você coloca numa pasta específica. Quando o Claude detecta que o contexto bate com a `description` do agent, ele carrega aquele arquivo inteiro como contexto — e passa a seguir todas as regras que você definiu.

**Skills** são módulos de conhecimento que um agent consulta quando precisa de detalhe técnico. Um agent pode ter 11 skills vinculadas, mas só carrega cada uma quando o contexto exige — isso mantém o contexto enxuto.

```
Você pede: "cria teste de login com Playwright"
    ↓
Claude detecta: playwright-specialist (description combina)
    ↓
Agent carrega: todas as skills declaradas no frontmatter
    ↓
Claude escreve o teste seguindo: nomenclatura, seletores,
auth via storageState, AAA com test.step(), sem POM, sem waits hardcoded
```

Sem isso, o Claude esquece as regras toda vez. Com isso, você escreve uma vez e ele segue sempre.

---

## 2. Como instalar

A pasta `.claude` fica na **raiz do workspace** aberto no VS Code, não dentro de cada projeto.

**Passo 1** — crie a estrutura se não existir:
```
Seu workspace/
└── .claude/
    ├── agents/
    └── skills/
```

**Passo 2** — copie os agents:
```
agents/playwright-specialist.md  →  .claude/agents/
agents/cypress-specialist.md     →  .claude/agents/
```

**Passo 3** — copie as **pastas** de skill (não só o arquivo):
```
skills/playwright-auth/    →  .claude/skills/playwright-auth/
skills/playwright-patterns/ →  .claude/skills/playwright-patterns/
... (todas as pastas)
```

> A skill é a **pasta inteira**, não o arquivo `SKILL.md` avulso. O nome da pasta é o nome que o agent usa para referenciá-la.

**Passo 4** — reabra o Claude Code. Pronto.

---

## 3. Como usar no dia a dia

### Ativação automática

Só descreva o que precisa. O Claude lê a `description` de todos os agents e ativa o mais adequado:

```
Cria um teste E2E para o fluxo de checkout com Playwright
```
→ `playwright-specialist` ativa sozinho

```
Revisa os testes Cypress do módulo de login
```
→ `cypress-specialist` ativa sozinho

### Forçar um agent com `@`

Quando quiser garantir qual agent está sendo usado:
```
@playwright-specialist cria testes para o filtro de datas
```

### Quando o agent vai te fazer perguntas

Os agents são configurados para **não assumir nada**. Antes de gerar qualquer teste, eles pedem:

- Qual funcionalidade está sendo testada?
- Quais são os fluxos críticos (happy path + erros)?
- Autenticação padrão ou SSO?
- Quais `data-testid` existem nos elementos?
- Precisa de desktop, tablet, mobile?

Isso não é o Claude sendo chato — é ele evitando gerar testes com seletores frágeis ou sem cobertura de erro. Quanto mais contexto você der de cara, mais direto ele vai ser.

---

## 4. Boas práticas de escrita de testes — Playwright

### Seletores: sempre `data-testid` primeiro

O Playwright tem uma hierarquia clara de seletores. Seguir ela é o que separa um teste que dura 2 anos de um que quebra na próxima refatoração de CSS.

```ts
// ✅ 1º - data-testid: totalmente desacoplado do estilo e da estrutura
page.getByTestId('login-btn')

// ✅ 2º - placeholder: bom para inputs sem data-testid
page.getByPlaceholder('Email address')

// ✅ 3º - semântico/acessível: quando os dois acima não existem
page.getByRole('button', { name: 'Save' })
page.getByLabel('Password')

// ❌ NUNCA — classes CSS mudam com refactoring
page.locator('.btn-primary')

// ❌ NUNCA — XPath é proibido sem exceção
page.locator('//button[@class="submit"]')

// ❌ NUNCA — índice sem escopo pai é frágil
page.locator('button').nth(2)
```

Se o elemento não tem `data-testid`, **não invente um seletor frágil**. Peça para o dev adicionar o atributo antes de escrever o teste.

---

### Autenticação: nunca via UI nos testes funcionais

Fazer login pela tela em cada teste é a maior causa de suite lenta e flaky. O padrão correto é logar uma vez e reaproveitar a sessão.

```ts
// tests/setup/auth.setup.ts — roda UMA VEZ antes de tudo
setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByTestId('username').fill(process.env.E2E_USERNAME!);
  await page.getByTestId('password').fill(process.env.E2E_PASSWORD!);
  await page.getByTestId('login-btn').click();
  await page.waitForURL('**/dashboard**');
  await page.context().storageState({ path: 'playwright/.auth/user.json' });
});
```

```ts
// playwright.config.ts — todos os testes carregam a sessão salva
projects: [
  { name: 'setup', testMatch: /auth\.setup\.ts/ },
  {
    name: 'e2e',
    use: { storageState: 'playwright/.auth/user.json' },
    dependencies: ['setup'],
  },
],
```

A partir daí, nenhum teste toca a tela de login — a não ser que o teste **esteja testando o login em si**.

---

### Estrutura AAA com `test.step()`

Todo teste segue o padrão **Arrange → Act → Assert**. No Playwright, use `test.step()` para separar as fases — isso aparece no relatório HTML e facilita identificar onde o teste falhou.

```ts
test('redirects to dashboard after successful login', async ({ page }) => {
  await test.step('Arrange', async () => {
    await page.goto('/login');
    await expect(page.getByTestId('login-btn')).toBeVisible();
  });

  await test.step('Act', async () => {
    await page.getByTestId('username').fill(process.env.E2E_USERNAME!);
    await page.getByTestId('password').fill(process.env.E2E_PASSWORD!);
    await page.getByTestId('login-btn').click();
  });

  await test.step('Assert', async () => {
    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByRole('heading', { name: 'Welcome' })).toBeVisible();
  });
});
```

---

### Waits: nunca hardcoded

`page.waitForTimeout()` é proibido. Além de deixar a suite lenta, ele mascara problemas reais.

```ts
// ❌ Proibido — wait cego
await page.getByTestId('save-btn').click();
await page.waitForTimeout(3000);

// ✅ Wait por resposta de rede — sabe exatamente quando terminou
const saved = page.waitForResponse('**/api/settings');
await page.getByTestId('save-btn').click();
await saved;
await expect(page.getByTestId('success-msg')).toBeVisible();

// ✅ Wait por elemento — auto-retrying, não precisa de tempo fixo
await expect(page.getByTestId('result-table')).toBeVisible();
```

---

### Nomenclatura de testes

O nome do teste é documentação. Quem lê o nome no relatório de CI precisa entender o que falhou sem abrir o arquivo.

```ts
// ✅ Padrão: [o que faz] + [em qual condição] + [resultado esperado]
test('redirects to /dashboard after successful login')
test('displays error message when submitting invalid credentials')
test('disables submit button while required fields are empty')
test('uploads PDF and shows success feedback')

// ❌ Inúteis no relatório
test('test login')
test('login 1')
test('should work')
```

---

### Dados sensíveis

Jamais coloque credenciais ou tokens direto no código.

```ts
// ❌ Proibido
await page.getByTestId('password').fill('minha-senha-123');

// ✅ Correto — variável de ambiente
await page.getByTestId('password').fill(process.env.E2E_PASSWORD!);

// ✅ Sempre { log: false } em campos de senha
await page.getByTestId('password').fill(process.env.E2E_PASSWORD!, { timeout: 5000 });
```

Crie um `.env.example` versionado com as chaves (sem valores) e um `.env` local no `.gitignore`.

---

### Dados de teste: Faker vai em fixture, nunca no teste

`faker` é ótimo para gerar dados dinâmicos e evitar colisões entre execuções paralelas. O erro comum é chamar o faker direto dentro do teste — isso embaralha o Arrange e dificulta debugar quando algo falha.

```ts
// ❌ Errado — faker espalhado dentro do teste
test('registers a new user', async ({ page }) => {
  const name = faker.person.fullName();       // ← dados gerados aqui
  const email = faker.internet.email();       // ← e aqui
  await page.getByTestId('name').fill(name);
  await page.getByTestId('email').fill(email);
});

// ✅ Correto — builder centralizado em fixture/data
// tests/fixtures/data/user.mock.ts
export function buildUserData() {
  return {
    name: faker.person.fullName(),
    email: faker.internet.email(),
    phone: faker.phone.number('(##) 9####-####'),
  };
}

// No teste — limpo, intenção clara
test('registers a new user', async ({ page }) => {
  const user = buildUserData();   // ← tudo vem do builder
  await page.getByTestId('name').fill(user.name);
  await page.getByTestId('email').fill(user.email);
});
```

**Por que isso importa:**
- Quando o teste falha, você sabe exatamente quais dados foram usados (o builder pode logar ou receber seed fixo)
- O builder é reutilizável em múltiplos testes sem duplicar lógica
- Se o schema mudar, você corrige em um lugar só
- Faker com seed fixo permite reproduzir falhas: `faker.seed(12345)`

```ts
// Reproduzindo uma falha específica com seed fixo
export function buildUserData(seed?: number) {
  if (seed !== undefined) faker.seed(seed);
  return {
    name: faker.person.fullName(),
    email: faker.internet.email(),
  };
}

// No teste que estava falhando
const user = buildUserData(12345); // sempre gera os mesmos dados
```

---

### Dados mockados: quando usar `page.route()` e quando não usar

Mockar a API significa interceptar uma chamada e devolver uma resposta falsa em vez de bater no servidor real. É uma ferramenta poderosa — mas usada no lugar errado, ela cria uma falsa sensação de segurança.

**A pergunta certa antes de mockar:**
> "O que este teste está validando — o comportamento da UI ou a integração com a API?"

```
Validando comportamento da UI  →  Mock é adequado
Validando integração com a API →  Nunca mock — use dados reais
```

#### Quando usar mock (`page.route()`)

```ts
// ✅ Testar estado vazio — a UI exibe a mensagem certa quando não há dados
test('displays empty state when there are no records', async ({ page }) => {
  await page.route('**/api/orders', route =>
    route.fulfill({ json: [] })
  );
  await page.goto('/orders');
  await expect(page.getByTestId('empty-state')).toContainText('No orders found');
});

// ✅ Testar erro de servidor — a UI exibe o banner de erro corretamente
test('displays error banner when API returns 500', async ({ page }) => {
  await page.route('**/api/orders', route =>
    route.fulfill({ status: 500, json: { message: 'Internal Server Error' } })
  );
  await page.goto('/orders');
  await expect(page.getByTestId('error-banner')).toBeVisible();
});

// ✅ Testar feature flag — comportamento condicional baseado em config
test('shows maintenance banner when flag is active', async ({ page }) => {
  await page.route('**/api/config', route =>
    route.fulfill({ json: { maintenance: true } })
  );
  await page.goto('/');
  await expect(page.getByTestId('maintenance-banner')).toBeVisible();
});
```

#### Quando NÃO usar mock

```ts
// ❌ Errado — testar o fluxo de criação mockando a própria criação
test('creates a new order', async ({ page }) => {
  await page.route('POST **/api/orders', route =>    // ← mock da criação
    route.fulfill({ json: { id: 999, status: 'created' } })
  );
  // Este teste não prova nada — a API real pode estar quebrada
  // e o teste sempre vai passar
});

// ✅ Correto — criação real via API na fixture, UI apenas valida
test('creates a new order', async ({ page, orderForm }) => {
  await orderForm.fillAndSubmit({ product: 'Widget', qty: 3 });
  const response = page.waitForResponse('POST **/api/orders');
  await orderForm.submit();
  expect((await response).status()).toBe(201);
  await expect(page.getByTestId('order-success')).toBeVisible();
});
```

#### A regra de ouro do mock

| Situação | Usar mock? |
|---|---|
| Testar como a UI reage a uma lista vazia | ✅ Sim |
| Testar como a UI reage a um erro 500 | ✅ Sim |
| Testar feature flag / configuração remota | ✅ Sim |
| Testar o fluxo que **você está validando** | ❌ Não — use API real |
| Testar dados que precisam existir antes do teste | ❌ Não — crie via API na fixture |
| Testar formulários de criação/edição | ❌ Não — valide a resposta real |

> **Sinal de alerta:** se você está mockando a mesma rota que a ação do teste dispara, você provavelmente está testando o mock, não o sistema.

---

## 5. Boas práticas de escrita de testes — Cypress

### Seletores: mesma lógica, sintaxe diferente

```js
// ✅ 1º - data-testid
cy.get('[data-testid="login-btn"]')

// ✅ 2º - aria / semântico
cy.get('[aria-label="Fechar modal"]')
cy.contains('button', 'Salvar')

// ❌ NUNCA — classes CSS
cy.get('.btn-primary')

// ❌ NUNCA — XPath
cy.xpath('//button[@id="submit"]')

// ❌ NUNCA — índice sem escopo
cy.get('button').first()
cy.get('li').eq(3)
```

---

### Autenticação: `cy.sessionLogin()` no `beforeEach`

Assim como no Playwright, você não repete o login na UI em cada teste. O Cypress usa `cy.session()` internamente para cachear a sessão.

```js
// ✅ Correto — sessão cacheada, não refaz login a cada teste
beforeEach(() => {
  cy.sessionLogin();
  cy.visit('/dashboard');
});

// ❌ Proibido — login via UI em cada teste
beforeEach(() => {
  cy.visit('/login');
  cy.get('[data-testid="username"]').type(Cypress.env('USERNAME'));
  cy.get('[data-testid="password"]').type(Cypress.env('PASSWORD'), { log: false });
  cy.get('[data-testid="login-btn"]').click();
});
```

---

### Estrutura AAA com comentários

No Cypress, o AAA é marcado com comentários — diferente do Playwright que usa `test.step()`.

```js
it('redirects to /dashboard after successful login', () => {
  // Arrange
  cy.intercept('POST', '/api/auth/login').as('login');
  cy.visit('/login');

  // Act
  cy.get('[data-testid="username"]').type(Cypress.env('USERNAME'));
  cy.get('[data-testid="password"]').type(Cypress.env('PASSWORD'), { log: false });
  cy.get('[data-testid="login-btn"]').click();

  // Assert
  cy.wait('@login');
  cy.url().should('include', '/dashboard');
  cy.get('[data-testid="welcome-msg"]').should('be.visible');
});
```

---

### Interceptação de rede: sempre antes de assertar

Toda ação que dispara uma chamada de API deve ser interceptada. Assertar antes de confirmar que a API respondeu é receita de teste flaky.

```js
// ✅ Correto — intercept + wait antes de assertar
it('saves settings successfully', () => {
  cy.intercept('PUT', '/api/settings').as('saveSettings');
  cy.visit('/settings');

  cy.get('[data-testid="name-input"]').clear().type('New Name');
  cy.get('[data-testid="save-btn"]').click();

  cy.wait('@saveSettings');  // espera a API antes de assertar
  cy.get('[data-testid="success-msg"]').should('be.visible');
});

// ❌ Proibido — wait cego no lugar de wait por alias
cy.get('[data-testid="save-btn"]').click();
cy.wait(3000); // ❌
cy.get('[data-testid="success-msg"]').should('be.visible');
```

---

### Um único `beforeEach` por nível

Dois `beforeEach` no mesmo `describe` se executam em série e criam confusão sobre a ordem. Consolide tudo em um único.

```js
// ❌ Proibido
describe('Dashboard', () => {
  beforeEach(() => { cy.sessionLogin(); });
  beforeEach(() => { cy.visit('/dashboard'); }); // ❌ segundo hook
});

// ✅ Correto
describe('Dashboard', () => {
  beforeEach(() => {
    cy.sessionLogin();
    cy.visit('/dashboard');
    cy.get('[data-testid="dashboard-header"]').should('be.visible');
  });
});
```

---

### Dados sensíveis no Cypress

```js
// ❌ Proibido
cy.get('[data-testid="password"]').type('minha-senha');

// ✅ Correto
cy.get('[data-testid="password"]').type(Cypress.env('PASSWORD'), { log: false });
```

`cypress.env.json` nunca entra no repositório. Versione apenas o `cypress.env.example.json` com as chaves vazias.

---

### Dados de teste: Faker vai em fixture, nunca no teste

A mesma regra do Playwright se aplica aqui. Faker direto dentro do `it()` mistura Arrange com geração de dados e dificulta debugar falhas.

```js
// ❌ Errado — faker espalhado dentro do teste
it('registers a new user', () => {
  const name = faker.person.fullName();   // ← dados gerados aqui
  const email = faker.internet.email();
  cy.get('[data-testid="name"]').type(name);
  cy.get('[data-testid="email"]').type(email);
});

// ✅ Correto — builder centralizado em cypress/fixtures/
// cypress/fixtures/user.js
export function buildUserData() {
  return {
    name: faker.person.fullName(),
    email: faker.internet.email(),
    phone: faker.phone.number('(##) 9####-####'),
  };
}

// No teste — limpo e reutilizável
it('registers a new user', () => {
  const user = buildUserData();
  cy.get('[data-testid="name"]').type(user.name);
  cy.get('[data-testid="email"]').type(user.email);
});
```

**Quando o teste falhar**, você consegue reproduzir passando um seed fixo para o faker no builder — sem precisar descobrir quais dados causaram a falha na execução anterior.

---

### Dados mockados: quando usar `cy.intercept()` e quando não usar

A mesma filosofia do Playwright se aplica no Cypress: mock serve para controlar o comportamento da UI diante de respostas específicas da API — não para substituir a API real nos fluxos que você está validando.

**A pergunta certa antes de mockar:**
> "O que este teste está validando — o comportamento da UI ou a integração com a API?"

```
Validando comportamento da UI  →  cy.intercept() é adequado
Validando integração com a API →  Nunca mock — use dados reais
```

#### Quando usar mock (`cy.intercept()`)

```js
// ✅ Testar estado vazio
it('shows empty state when there are no orders', () => {
  cy.intercept('GET', '/api/orders', { body: [] }).as('getOrders');
  cy.visit('/orders');
  cy.wait('@getOrders');
  cy.get('[data-testid="empty-state"]').should('contain', 'No orders found');
});

// ✅ Testar erro de servidor
it('shows error banner when API returns 500', () => {
  cy.intercept('GET', '/api/orders', { statusCode: 500 }).as('getOrders');
  cy.visit('/orders');
  cy.wait('@getOrders');
  cy.get('[data-testid="error-banner"]').should('be.visible');
});

// ✅ Testar loading state
it('shows skeleton while loading', () => {
  cy.intercept('GET', '/api/orders', (req) => {
    req.reply((res) => {
      res.setDelay(2000);  // simula latência
    });
  }).as('getOrders');
  cy.visit('/orders');
  cy.get('[data-testid="skeleton-loader"]').should('be.visible');
  cy.wait('@getOrders');
  cy.get('[data-testid="skeleton-loader"]').should('not.exist');
});
```

#### Quando NÃO usar mock

```js
// ❌ Errado — mockando a própria criação que está sendo testada
it('creates a new order', () => {
  cy.intercept('POST', '/api/orders', { body: { id: 999, status: 'created' } }).as('createOrder');
  // Este teste não prova nada — a API real pode estar quebrada
  // e o teste sempre vai passar
  cy.get('[data-testid="submit-btn"]').click();
  cy.wait('@createOrder');
});

// ✅ Correto — deixa a API real processar, valida a resposta
it('creates a new order', () => {
  cy.intercept('POST', '/api/orders').as('createOrder');
  cy.get('[data-testid="product-input"]').type('Widget');
  cy.get('[data-testid="submit-btn"]').click();
  cy.wait('@createOrder').its('response.statusCode').should('eq', 201);
  cy.get('[data-testid="order-success"]').should('be.visible');
});
```

#### A regra de ouro do mock

| Situação | Usar mock? |
|---|---|
| Testar como a UI reage a uma lista vazia | ✅ Sim |
| Testar como a UI reage a um erro 500 | ✅ Sim |
| Testar loading state com latência simulada | ✅ Sim |
| Testar o fluxo que **você está validando** | ❌ Não — use API real |
| Testar dados que precisam existir antes do teste | ❌ Não — crie via API no `before()` |
| Testar formulários de criação/edição | ❌ Não — valide o status code real |

> **Sinal de alerta:** se você usa `cy.intercept()` para interceptar exatamente a chamada que a ação do teste dispara e preenche uma resposta de sucesso manual, você está testando o mock — não o sistema.

---

## 6. Por que Page Object Model não funciona no Playwright

Quem vem do Selenium ou Cypress mais antigo vai estranhar essa regra. Mas ela existe por razões técnicas concretas.

### O problema real do POM

```ts
// ❌ Page Object — parece organizado, mas tem três problemas graves
class OrdersPage {
  async filterByDate(start: string, end: string) {
    await this.page.getByTestId('date-start').fill(start);
    await this.page.getByTestId('date-end').fill(end);
    await this.page.getByTestId('btn-filter').click();
  }
}
```

**Problema 1 — Teardown não garantido.**
Se o teste falhar no meio, o método do POM não tem como garantir que o banco vai ser limpo. `afterEach` no Playwright pode não rodar em caso de timeout ou crash.

**Problema 2 — Esconde a intenção.**
Quando você lê `ordersPage.filterByDate()` no teste, precisa abrir outro arquivo para entender o que acontece. O teste não é mais autoexplicativo.

**Problema 3 — Acoplamento.**
Qualquer mudança no componente de filtro quebra todos os testes que usam `OrdersPage`, mesmo os que não testam filtro.

---

### A solução: Custom Fixtures

Fixtures do Playwright resolvem os três problemas ao mesmo tempo:

```ts
// tests/fixtures/orders.fixture.ts
export const ordersFixture = {
  createdOrder: async ({ request }, use) => {
    // Setup via API — não clica na UI
    const res = await request.post('/api/orders', { data: buildRandomOrderData() });
    const { id } = await res.json();

    await use(id); // ← o teste roda aqui

    // Teardown GARANTIDO — executa mesmo se o teste crashar
    await request.delete(`/api/orders/${id}`);
  },
};
```

```ts
// No teste — intenção clara, teardown garantido
test('cancels order and updates status', async ({ page, createdOrder }) => {
  await page.goto(`/orders/${createdOrder}`);
  await page.getByTestId('btn-cancel').click();
  await expect(page.getByTestId('status-badge')).toHaveText('Cancelled');
});
```

### Quando usar o quê

```
Mesmo setup em 3+ specs  →  Custom Fixture em tests/fixtures/
Setup em 1 spec só        →  beforeEach inline
Lógica de navegação compartilhada  →  Função utilitária pura (sem classe)
Pensando em criar classe *Page  →  Pare. Crie uma fixture.
```

---

## 7. Playwright MCP e cy.prompt — quando usar e quando evitar

Essas são duas ferramentas de IA para geração automática de testes que estão ganhando espaço. Entender o que cada uma faz — e os limites reais — evita que você deposite confiança em código que vai quebrar em produção.

---

### Playwright MCP

O **Playwright MCP** é um servidor MCP oficial da Microsoft que permite que um LLM controle um navegador via Playwright usando snapshots de acessibilidade (não screenshots). A ideia é que o modelo "veja" a página e tome ações automaticamente.

#### O que ele faz de fato

Em vez de você escrever `page.getByTestId('login-btn').click()`, o LLM navega pelo app, identifica os elementos pelo snapshot de acessibilidade e gera ou executa ações por conta própria.

#### Quando é útil

| Cenário | Vale usar? |
|---|---|
| Exploração inicial de um app desconhecido | ✅ Sim |
| Geração de rascunho de teste para fluxos simples | ✅ Com revisão obrigatória |
| Workflows automatizados de longa duração (automações, não testes) | ✅ Sim |
| Escrita de testes de produção para CI/CD | ❌ Não |
| Apps com Shadow DOM (Web Components, Lit, Shoelace) | ❌ Não funciona |

#### Limitações reais que você precisa conhecer

**Autenticação é um pesadelo.** O MCP re-autentica a cada execução. Com OAuth, SSO ou MFA, ele trava ou dispara alertas de segurança. A gestão de sessão continua sendo manual.

**Shadow DOM é ponto cego.** Componentes que usam Shadow DOM (muito comum em design systems modernos) ficam invisíveis para os snapshots de acessibilidade. O modelo tenta clicar em elementos que não existem para ele.

**Flakiness por seletor desatualizado.** Se a UI muda, o modelo pode continuar usando seletores que não existem mais — e o teste passa em verde testando a coisa errada.

**Concorrência limitada.** Só consegue rodar um perfil persistente por vez. Para CI com paralelismo, precisa de infraestrutura extra.

**Não vale abaixo de ~200 testes.** O overhead de manter a infraestrutura do MCP supera o benefício de não escrever os testes manualmente.

#### Regra prática

> Use o MCP para **explorar e rascunhar**. Nunca entregue o output dele direto para o CI sem revisão humana e adequação aos padrões do projeto (seletores, fixtures, storageState, nomenclatura).

---

### cy.prompt (Cypress AI)

O **cy.prompt** é um comando experimental nativo do Cypress (disponível a partir da v15.4.0) que converte linguagem natural em testes Cypress executáveis. Você escreve em inglês o que quer testar e ele gera o código — ou roda continuamente em modo "self-healing".

#### O que ele faz de fato

```js
// Você escreve isso
cy.prompt('click the login button and fill in the email and password fields')

// Cypress gera e executa algo como
cy.get('[data-testid="login-btn"]').click()
cy.get('[data-testid="email"]').type('...')
```

#### Quando é útil

| Cenário | Vale usar? |
|---|---|
| Rascunho inicial de testes para UIs estáveis | ✅ Com revisão |
| Times com membros não-técnicos escrevendo casos de teste | ✅ Sim |
| Fluxos simples de happy path | ✅ Com revisão |
| Testes de API (`cy.request`) | ❌ Não suportado |
| Iframes | ❌ Não suportado |
| Canvas | ❌ Não suportado |
| Firefox ou WebKit | ❌ Chromium apenas |
| Fluxos com mais de 50 passos | ❌ Trava no limite |
| Testes com lógica de negócio complexa | ❌ Output genérico sem contexto do app |

#### Limitações reais que você precisa conhecer

**Só inglês.** Não há suporte para outros idiomas no prompt.

**Não conhece seu app.** O cy.prompt não lê o código-fonte nem os `data-testid` existentes. Ele infere seletores pelo que vê renderizado — o que pode gerar seletores frágeis que não seguem os padrões do projeto.

**Self-healing pode testar a coisa errada.** No modo contínuo, se o seletor muda, o Cypress pode "se curar" encontrando outro elemento — que não é o que você queria testar. O teste fica verde e você não percebe.

**Requer Cypress Cloud.** Precisa de autenticação com Cypress Cloud para funcionar. Não roda offline ou em ambientes isolados.

**Limite de 50 passos por prompt.** Fluxos complexos precisam ser quebrados em múltiplos prompts, o que complica a manutenção.

#### Regra prática

> Use o cy.prompt para **acelerar a criação inicial** de testes em fluxos simples e estáveis. Sempre revise o output gerado, adapte os seletores para `data-testid` e ajuste a nomenclatura antes de commitar.

---

### Comparação direta: MCP vs cy.prompt vs escrita manual

| | Playwright MCP | cy.prompt | Escrita manual com agent |
|---|---|---|---|
| Qualidade do output | Média — precisa revisão | Média — precisa revisão | Alta — segue os padrões do projeto |
| Shadow DOM | ❌ Não funciona | ✅ Funciona | ✅ Funciona |
| SSO / Auth complexa | ❌ Problemático | ⚠️ Parcial | ✅ Funciona |
| Iframes | ⚠️ Parcial | ❌ Não suportado | ✅ Funciona |
| Segue convenções do projeto | ❌ Não sabe | ❌ Não sabe | ✅ Sim (via agent/skills) |
| Bom para rascunho inicial | ✅ Sim | ✅ Sim | ✅ Sim |
| Bom para CI/CD de produção | ❌ Não sem revisão | ❌ Não sem revisão | ✅ Sim |
| Requer infraestrutura extra | ✅ Sim (MCP server) | ✅ Sim (Cypress Cloud) | ❌ Não |

---

### Conclusão

Nenhuma das duas ferramentas substitui a escrita manual com um agent bem configurado para projetos com padrões definidos. O melhor fluxo de trabalho é:

```
1. Use MCP ou cy.prompt para explorar e gerar um rascunho rápido
2. Traga o rascunho para o Claude Code com o agent ativo
3. Peça para o agent revisar e adaptar seguindo os padrões do projeto
4. Revise o resultado antes de commitar
```

Isso combina a velocidade da geração automática com a qualidade dos padrões definidos no agent.

---

## 8. Como as skills funcionam e quando usá-las

### O sistema de 3 níveis

O Claude não carrega todas as skills de uma vez. Ele usa três camadas:

```
Nível 1 — sempre na memória
  Nome + description de cada skill (~100 palavras cada)
  "Se o assunto for autenticação, carregue playwright-auth"

Nível 2 — carregado quando a skill ativa
  Conteúdo completo do SKILL.md (até ~5.000 palavras)
  "Aqui estão as regras de storageState e SSO"

Nível 3 — carregado quando o Claude precisar
  Arquivos em references/, scripts/, assets/ (sem limite)
  "Aqui está o template completo do auth.setup.ts"
```

Isso significa que você pode ter skills grandes sem pesar no contexto — o Claude só lê o que precisa para aquela tarefa.

### Quando uma skill é ativada

Há duas formas de uma skill ser carregada:

**1. Declarada no frontmatter do agent** — carrega sempre que o agent ativa:
```markdown
---
name: playwright-specialist
skills: playwright-auth, playwright-selectors, playwright-patterns
---
```

**2. Mencionada nas instruções do agent** — o Claude carrega quando o contexto exige:
```markdown
## Quando trabalhar com upload de arquivos
Consulte a skill `playwright-interactions` antes de gerar qualquer código de upload.
```

Use o frontmatter para skills que o agent **sempre vai precisar**. Use menção nas instruções para skills **situacionais** — assim você não pesa o contexto em toda sessão.

---

## 9. Como editar uma skill existente

As skills são arquivos Markdown simples. Para editar, abra o `SKILL.md` da skill desejada e altere diretamente.

### Quando editar uma skill

- O projeto adotou uma nova convenção de seletor
- Um padrão de autenticação mudou
- Descobriram um anti-pattern que precisa ser documentado
- Exemplos de código ficaram desatualizados

### Cuidados ao editar

**1. Não duplique conteúdo entre SKILL.md e references/.**
Se você tem um guia detalhado em `references/auth-sso.md`, não repita as mesmas regras no `SKILL.md`. O SKILL.md é o resumo executivo; references é o detalhe.

**2. Mantenha o SKILL.md enxuto.**
O SKILL.md é carregado inteiro no contexto toda vez que a skill ativa. Se ele tiver 10.000 palavras, vai pesar. Mova detalhes para `references/` e mencione quando ler:

```markdown
## SSO com Microsoft

Para implementar SSO via Microsoft Entra, leia
`references/sso-microsoft.md` antes de escrever qualquer código.
```

**3. Teste depois de editar.**
Peça ao Claude algo que deveria ativar a skill e veja se ele segue as regras novas. Se não seguir, a `description` pode estar vaga ou o conteúdo ambíguo.

**4. Atualize a `description` no frontmatter se o escopo mudou.**
A description é o que o Claude lê para decidir se carrega ou não a skill. Se ela ficou desatualizada, a skill pode não ativar nos momentos certos.

---

## 10. Como criar uma skill nova

### Opção 1 — Com o Skill Creator (recomendado para iniciantes)

O Skill Creator te guia passo a passo:

```
/skill-creator
```

Ou descreva o que quer:
```
Quero criar uma skill para padronizar relatórios de teste no Jira
```

Ele vai fazer perguntas sobre casos de uso, gerar a estrutura e te entregar um rascunho para editar.

### Opção 2 — Manualmente

Crie a pasta e o arquivo:
```
.claude/skills/minha-skill/
└── SKILL.md
```

Estrutura mínima do `SKILL.md`:
```markdown
---
name: minha-skill
description: Descreva em uma linha quando esta skill deve ser carregada
             e para qual tipo de tarefa ela serve.
---

## O que esta skill cobre

[resumo em 2-3 linhas]

## Regras

[lista de regras que o agent deve seguir]

## Exemplos

[código mostrando certo e errado]
```

### Adicionando recursos à skill

```
minha-skill/
├── SKILL.md                    ← Regras e referências
├── references/
│   └── guia-detalhado.md       ← Documentação longa, exemplos extensos
├── scripts/
│   └── gerar-relatorio.py      ← Scripts que o Claude pode executar
└── assets/
    └── template-spec.ts        ← Templates prontos para copiar
```

Mencione no SKILL.md quando cada recurso deve ser lido:
```markdown
## Gerando relatório

Para gerar o relatório automaticamente, execute:
`python scripts/gerar-relatorio.py`

Para ver o template base de spec, leia `assets/template-spec.ts`.
```

### Distribuindo para o time

```bash
python scripts/package_skill.py .claude/skills/minha-skill
```

Isso valida o formato e gera um `.zip`. Se der erro de validação, corrija e rode de novo.

---

## 11. Como criar um agent novo

### Estrutura do arquivo

```markdown
---
name: nome-do-agent
description: Descreva em 2-3 frases QUANDO usar este agent e PARA QUÊ.
             Seja específico — o Claude usa este campo para decidir se ativa ou não.
skills: skill-1, skill-2, skill-3
tools: All tools
model: inherit
---

## 1. Papel

[Descreva em 1 parágrafo o que este agent faz]

## 2. Antes de gerar qualquer coisa, pergunte

[Liste o que o agent precisa saber antes de agir]

## 3. Regras obrigatórias

[Regras com exemplos de certo e errado]

## 4. Anti-patterns proibidos

[O que nunca deve aparecer no output]
```

### O campo `description` é o mais importante

É ele que o Claude lê para decidir se ativa o agent. Uma description vaga = agent que nunca ativa.

```markdown
# ❌ Ruim — nunca vai ativar no momento certo
description: Use for tests

# ✅ Bom — o Claude sabe exatamente quando usar
description: Use for E2E test automation with Playwright + TypeScript for frontend
             web projects. Activate when creating, reviewing, or auditing end-to-end
             tests, generating test reports, or implementing CI/CD test pipelines.
```

### Testando o agent

Após criar, faça pelo menos 3 pedidos diferentes:
1. Um que **deve** ativar o agent
2. Um em área diferente que **não deve** ativar
3. Um pedido específico para ver se as regras são seguidas

Se o agent não ativar quando deveria, revise a `description`. Se ativar quando não deveria, ela está genérica demais.

---

## 12. O que não fazer

### Em testes Playwright

| ❌ Nunca faça                      | ✅ Faça assim |
|---|---|
| `page.waitForTimeout(3000)`           | `page.waitForResponse('**/api/...')` |
| Login via UI em cada teste            | `storageState` no `auth.setup.ts` |
| Criar classe `*Page`                  | Custom Fixture em `tests/fixtures/` |
| `page.locator('.btn-primary')`        | `page.getByTestId('...')` |
| XPath qualquer que seja               | Proibido sem exceção |
| `.nth(3)` sem escopo pai              | Adicione `data-testid` no elemento |
| `if/else` baseado em estado da UI     | Crie dois testes separados |
| Hardcodar credenciais                 | `process.env.E2E_PASSWORD!` |
| Importar de `@playwright/test` direto | Importar de `app-fixtures.ts` |
| `afterEach` para limpar dados         | `yield` dentro da fixture |

### Em testes Cypress

| ❌ Nunca faça                  | ✅ Faça assim |
|---|---|
| `cy.wait(3000)`                 | `cy.wait('@alias')` após `cy.intercept()` |
| Login via UI em cada teste      | `cy.sessionLogin()` no `beforeEach` |
| Dois `beforeEach` no mesmo nível| Um único `beforeEach` consolidado |
| `cy.get('.btn-primary')`        | `cy.get('[data-testid="..."]')` |
| XPath                           | Proibido sem exceção |
| `testIsolation: false`          | Sempre `true` |
| Hardcodar credenciais           | `Cypress.env('PASSWORD')` + `{ log: false }` |
| `if/else` baseado em UI         | Crie casos de teste separados |

### Em skills e agents

| ❌ Nunca faça                              | ✅ Faça assim |
|---|---|
| Description genérica de 3 palavras          | Description de 2-3 frases explicando quando e para quê |
| SKILL.md com 10.000 palavras                | SKILL.md enxuto + detalhes em `references/` |
| Mesmo conteúdo no SKILL.md e no references/ | Escolha um lugar só |
| Colocar 15 skills no frontmatter            | Só as essenciais no frontmatter, situacionais nas instruções |
| Criar agent e nunca testar                  | 3 pedidos diferentes para validar comportamento |
| Distribuir sem empacotar                    | `package_skill.py` valida antes de zipar |

---

*2026-06-09*
