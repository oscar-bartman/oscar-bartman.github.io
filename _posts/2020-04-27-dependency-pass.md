---
layout: post
title: "Dependency Injection vs Simple Modules in Node: a Foo Fighting Showdown"
tags: [Node, Dependencies, IoC, Modules, TypeScript]
comments: true
---

Here we look at two different styles of dependency handling, one a class-based approach with dependency injection through constructor parameters, the other a straight forward node module that simply loads its dependencies.

# Why Dependencies Matter 

Dependency handling in your codebase hits some important aspects of maintainability, like testing, readability and the ease to expand, so it deserves careful consideration. In general, the challenge is to drive down complexity and make it obvious where things should go for anyone who opens up a project. In an overly complex system, testing gets thrown by the wayside and concerns about getting features to market take over.

# Two Code Styles Enter the Dojo


## Class-based with Injection

Our first contender hails from a lineage of OOP practices and injects its dependencies into class instantiations via the constructor. Classes are modeled with the use of TypeScripts `interface`. Our foo fighting example: 

### Building it

``` typescript
interface FooFighter {
    fightFoo(id): Promise<void>;
}

class FooFighterImplementation implements FooFighter {
    private client: WeFightFooIncClient;
    
    constructor(client: WeFightFooIncClient) {
        this.client = client;
    }

    async fightFoo(id) {
        await client.updateFoo(id, { defeated: true });
    }
}
```
The main here idea is that the calling code that uses the `FooFighterImplementation` never needs to be aware of the dependency being used to do the actual `foo` handling, in our case a `WeFightFooIncClient` instantiation named `client`. The logic involved with actually using the client is contained within the class. 

### Using it

Our calling code that needs a `FooFighter` implementation would do something like the following: 

``` typescript
import { WeFightFooIncClient } from "wefightfooinc"

const client = new WeFightFooIncClient(config);
const fooFighter = new FooFighter(client);

async function receiveInsultFromFoo(fooId: int, insult: int) {
    if (insult > 3) {
        await fooFighter.fightFoo(fooId);
    }
}
```

The client is gets passed from a higher level than the implementation itself. As a result, the initialization logic of `WeFightFooIncClient` is now removed from the implementation that uses it. A developer will have to take more steps to trace the initialization when he/she is working on the code. Confused and without guidance, developers will stare at their screen blankly and ruminate on the question: 
> Where do I initialize this client? 

### Testing it

When unit testing this kind of code we can use the same mechanism to pass a mocked dependency and safely call our methods with the assurance that no actual foo shall be harmed during our tests. Our mocked dependency would have stubs or spies for the functions that our `FooFighter` implementation uses. 

``` typescript
import { FooFighterImplementation } from "../src/fighters/FooFighterImplementation";
import { MockFooManagementClient } from "./mocks/MockFooManagementClient";

const client = new MockFooManagementClient()
const fooFighter = new FooFighterImplementation(client)

describe("FooFighterImplementation", () => {
    describe(".fightFoo", () => {
        it("defeats foo", async () => {
            await fooFighter.fightFoo(1);
            expect(client.updateFoo).toBeCalledWith(1, { defeated: true });
        });
    });
});
```

Note that the unit we are testing is not a function but the class instead. That is: *the tests we can now build provide me the assurance only that the method will work within the context of this class.* 

## Node Modules with Direct Loading

Next, we look at the way this would be implemented in a more functional approach along the lines of a bare Node application. We will just rely on `require` to do our bidding when it comes to pulling in dependencies and we will use Node's [modules from directories](https://nodejs.org/dist/latest-v14.x/docs/api/modules.html#modules_folders_as_modules) to encapsulate concerns. Our foo fighting example:

### Building it

We create a module called *foo* that will export the functions we need for our foo related business. It will also contain the logic of dealing with our third-party client. 

```
foo
├── index.ts
├── config.ts
└── fight.ts
```

The functions that `index.ts` exposes make up the interface of our `foo` module and as long as we don't break that interface we do not have to worry about breaking the code that uses it. 

``` typescript
// index.ts
export * from "./fight"
```

Our `fight` implementation becomes a single exported function from a file by the same name, but could also come from a file exposing multiple functions.

``` typescript
// fight.ts
import client from "./config"

export function fight(id) {
    return client.updateFoo(id, { defeated: true });
}
```

Things inside this module that are not exported by `index.ts` can be considered `private`, or somewhat analogous to `private`. For instance: `config.ts` is where our `WeFightFooIncClient` is configured, initialized and exposed. Outside this module, there is no awareness of this dependency. 

``` typescript
// config.ts
import { WeFightFooIncClient } from "wefightfooinc";

export const client = new WeFightFooIncClient(config);
```

Whereas before we had to instantiate the client somewhere outside of the class and pass it down (either in the same file while exposing a factory or somewhere else in the project structure), now we have a much more intuitive sense of where the client should come from.

### Using it

The result is a much cleaner structure of our calling code where we take what we need from the module without having to include the entire context a function lives in. Often, we're only interested in single tasks that we want modules to handle for us. When reading the code, anything that does not relate to that goal should be considered noise.

``` typescript
// receiveinsultfromfoo.ts
import { fight } from "./foo"

async function receiveInsultFromFoo(fooId: int, insult: int) {
    if (insult > 3) {
        await fight(fooId);
    }
}
```

### Testing it

When testing this we can completely isolate the function we would like to test by importing that function directly. This way our unit level of testing becomes a function, rather than a class. 

``` typescript
jest.mock("../src/foo/config.ts", () => ({
    updateFoo: jest.fn()
}))
import { updateFoo } from "../src/foo/config";
import { fight } from "../src/foo/fight"

describe("fight", () => {
    it("defeats foo", async () => {
          await fight(1);
          expect(updateFoo).toHaveBeenCalledWith(1, { defeated: true });
    });
});

```

Instead of passing a mocked dependency, we leverage the Node module system to alter the behavior of our program. Specifically, where Node modules are loaded. Here, we use [Jest](https://jestjs.io/) to return a mocked export from `config.ts` that replaces `updateFoo` with a Jest mock function.

# Conclusion

With complexity in mind, you are likely better off keeping your implementations as clean and functional as possible. Leveraging Node modules, rather than introducing a class-based structure, allows us to isolate and test single exported functions. It makes it easier to intuit where initialization logic should be found and it makes for readable, unit-testable and functional code.

---
### Further Reading: 
- [How to Use Classes and Sleep at Night](https://medium.com/@dan_abramov/how-to-use-classes-and-sleep-at-night-9af8de78ccb4)
- [Testing with Seams](https://www.informit.com/articles/article.aspx?p=359417&seqNum=1).
- [Node Modules](https://nodejs.org/dist/latest-v14.x/docs/api/modules.html)
- [TypeScript Class Interfaces](https://www.typescriptlang.org/docs/handbook/interfaces.html#class-types)