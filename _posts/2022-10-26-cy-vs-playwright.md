---
layout: post
title: "Cypress vs Playwright"
tags: [Frontend, Cypress, E2E, JavaScript, Playwright, Testing]
---

The following is a short summary of findings from a shallow comparison between Cypress and Playwright. Both Cypress and Playwright are end-to-end testing tools that incorporate neatly into your codebase and work by playing out scripted scenarios on you application. Cypress v1.0.0 was released in 2017 and is currently at v10.11.0. Playwright v1.0.0 stems from 2020 and is currently at v1.27.1.

## async/await

When playing scripted scenarios on a frontend application you inevitably run into its asynchronous nature. Often you have to wait for some animation to finish or for some data to load before the elements that you want to interact with are enabled or visible on the page. Both Cypress and Playwright deal with this with by re-trying until they have a result or until a certain amount of time has passed. The difference lies in the interface exposed to the user.

Cypress "manages the Promise chain on your behalf", meaning that you set up your command script in a synchronous manner but nothing happens until all the commands have been defined.

i.e.

```js
  // cypress accepts a chain of commands that it will play off once all commands have been collected
  cy.get('.collapse-menu.landscape .edit-button')
    .click()
```

A practical way to think of Cypress' approach is as a separation between defining commands and running those commands. Cypress will run your commands only after they have all been passed to the Cypress instance. As a consequence you can not write JavaScript the way you might be used to when writing cypress tests. 

Here is an example taken from the Cypress docs: 

```js
it('does not work as we expect', () => {
  cy.visit('/my/resource/path')

  cy.get('.awesome-selector')
    .click()

  let el = Cypress.$('.new-el')

  if (el.length) {
    cy.get('.another-selector')
  } else {
    cy.get('.optional-selector')
  }
})
```
[source](https://docs.cypress.io/guides/core-concepts/introduction-to-cypress#Commands-Are-Asynchronous)

Here, the query `Cypress.$(.new-el)` is synchronous, i.e. it runs directly, while the `cy.*` commands are passed to Cypress and will run after all the commands have been collected.

This means you have to use callbacks if you want to do things in consecutive order. I.e.: 

```js
it('does not work as we expect', () => {
  cy.visit('/my/resource/path')

  cy.get('.awesome-selector')
    .click()
    // passing a callback to "then" will tell Cypress to queue these commands after the click() action has completed or timed out
    .then(() => { 
      let el = Cypress.$('.new-el')
      if (el.length) {
        cy.get('.another-selector')
      } else {
        cy.get('.optional-selector')
      }
    })
})
```
[source](https://docs.cypress.io/guides/core-concepts/introduction-to-cypress#Commands-Are-Asynchronous)

In contrast, Playwright simply exposes Promises to the user so we can `await` its retry mechanism until it succeeds or fails (times out): 

```js
  await page.getByRole('button', { name: 'New' }).click()
```

This means we can write the above logic in a more straight-forward manner: 

```js
  await page.goto('/my/resource/path')

  await page.locator('.awesome-selector').click()

  const count = await page.locator('.new-el').count()

  if (count) {
    await expect(page.locator('.another-selector')).toBeVisible()
  } else {
    await expect(page.locator('.optional-selector')).toBeVisible()
  }
```

In my opinion, Playwright's async/await interface makes for consirably more readable code and is less opaque about what is going on. Waiting for Playwright to go through its retry mechanism is simply waiting for a promise to resolve. 

## Baked-in toolsets

As nobody wants to reinvent the wheel, both Cypress and Playwright come with a collection of existing tools to achieve some of their basic functionality. Think of selecting elements on the page, writing assertions and mocking JavaScript API's. 

> Cypress bundles Chai, Chai-jQuery, and Sinon-Chai to provide built-in assertions.

[source](https://docs.cypress.io/guides/core-concepts/introduction-to-cypress#Default-Assertions)

Cypress opted for the more traditional testing suite that you may already be familiar with. It bundles JQuery to select elements and it uses [chai](https://www.chaijs.com/) to write test assertions. In case you want to stub some part of the JavaScript API it exposes a [stub()](https://docs.cypress.io/api/commands/stub) interface that returns a [Sinon.js](https://sinonjs.org/) stub. Check out some examples [here](https://github.com/cypress-io/cypress-example-recipes/blob/master/examples/stubbing-spying__navigator/cypress/e2e/spec.cy.js) and [here](https://github.com/cypress-io/cypress-example-recipes/blob/master/examples/stubbing-spying__window-fetch/cypress/e2e/spy-on-fetch-spec.cy.js). 

```js
// Cypress:

// querying a nested class jquery style
cy.get('.collapse-menu.landscape .edit-button').click()

cy.get('#addModal')
  // using chai assertions to determine if the modal is visible
  .should('be.visible')
```

One advantage is that most people know JQuery, Chai and Sinon. These tools have been around since the dawn of mankind and are likely to stay. 

> Playwright Test uses Jest expect library for test assertions.

[source](https://playwright.dev/docs/test-assertions)

When it comes to assertions, Playwright bundles Jest's [expect()](https://jestjs.io/docs/expect) API, which makes it particularly useful in projects that already make use of Jest.  

Playwright takes a different approach to selecting elements. Instead of relying on JQuery it exposes a host of [selectors](https://playwright.dev/docs/selectors) ranging from text selectors to CSS selectors to (experimental) React component selectors. There are too many to mention here so just have a look for yourself. On top of that, Playwright has [recently added](https://github.com/microsoft/playwright/releases/tag/v1.27.0) a number of conveniece APIs that all take a page out of [Testing Library](https://testing-library.com/)'s book. Methods like [getByRole](https://playwright.dev/docs/api/class-page#page-get-by-role) and [getByText](https://playwright.dev/docs/api/class-page#page-get-by-text) allow you to stay aligned with Testing Library's [guiding principles](https://testing-library.com/docs/guiding-principles), which is something to consider. 

```js
  // Playwright:

  // Using testing-library styled API to select
  await page.getByRole('button', { name: 'Edit' }).click()
  // Using a CSS selector and a Jest assertion
  await expect(page.locator('#addModal')).toBeVisible()
```

Whether you choose one or the other toolset is dependent on your personal preference and you requirements of course. If you have need of a wide range of selectors, Jest assertions and you like the idea of having Tesing Library style APIs, maybe Playwright is something to consider. If you feel like the more traditional testing suite is your cup of tea, maybe go for Cypress.
