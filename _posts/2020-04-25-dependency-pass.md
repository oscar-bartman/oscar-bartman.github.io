---
layout: post
title: "Dependency Passing vs Direct Loading in Node/TypeScript"
tags: [Node, Dependencies, IoC, Modules, TypeScript]
comments: true
---


This post is the result of a recent conversation I had with a colleague that centered around dependencies in Node.js. I tried to limit myself to a limited set of topics as much as I could to keep this post from sprawling out of control, but given the nature of this topic, it proved to be challenging not to involve a wide range of issues. Dependency handling in your codebase hits pretty much every aspect of maintainability so it deserves careful consideration but keeping the scope limited is difficult, especially in face to face communication. This post is an attempt to organize some of what I took away.

On the table are two different styles of pulling in dependencies and encapsulating concerns, one with a direct `require` and splitting modules into directories and another where we pass a dependency to a constructor and split modules into classes. The latter can be characterized, I think, as a more OOP approach. Let's take the example of a module that uses a vendor-supplied SDK to manage foo data. The latter approach would look something like this: 

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

This style borrows from Inversion of Control in the sense that it mimics constructor injection. I say *borrows* because there is no outside framework that wires up our codebase at runtime and injects a dependency when it spots this kind of constructor parameter. We can still determine the actual dependency through static analysis, and not by opening up a config file. One could argue that there is *injection* from the perspective of our `FooFighterImplementation`, and if we decided to take our foo handling business elsewhere, we would be able to swap out the client, change the `FooFighterImplementation` to use the new client while keeping the `FooFighter` interface the same, so the calling code that uses it to fight foo never has to change, ideally. (There are [frameworks](https://www.npmjs.com/package/inversify) that provide IoC in Node, but that is outside the scope of this post).

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

As you can see, the actual client used in our `FooFighterImplementation` is getting passed from a higher level than the implementation itself. A side effect is that the initialization logic of `WeFightFooIncClient` is removed from the implementation that uses it, which forces a developer to take more steps to trace this initialization logic when he/she is working on the code that uses the client. It also produces discussions between developers around the question of *"where do I initialize this client?"*

When unit testing this kind of code we can use the same mechanism to pass a mocked dependency and safely call our methods with the assurance that no actual foo shall be harmed in our tests. Here is what a test could look like:

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

We can see that to test a function I need to do a couple of things. First, I need to build a mock `FooFighter` implementation that adheres to the interface we defined so I can pass it to the constructor of `FooFighterImplementation`. Second, I need to instantiate the `FooFighterImplementation` class so the function I want to test becomes available to me. Note that the second requirement implies that the unit we can test is no longer a single function but the class instead. The tests I can now build provide me the assurance only that the method will work within the context of this class, therefore we are taking one step away from functional JavaScript that we have learned to love Node for. 

Let's summarize some of the pros and cons here: 
+ we are programming to an interface (+)
+ the logic of using the client dependency is contained (+)
+ initialization logic is harder to trace through static analysis (-)
+ the location of initialization logic now requires a kind of policy enforcement (-)
+ our testable units of code are classes and not functions (-)

Next, we look at the way this would be implemented in a more functional approach along the lines of a bare Node application. We will just rely on `require` to do our bidding when it comes to pulling in dependencies and we will use Node's [modules from directories](https://nodejs.org/dist/latest-v14.x/docs/api/modules.html#modules_folders_as_modules) to encapsulate concerns. First, rather than defining a class to an interface, we will simply create a new module where our foo fighting logic is going to live:

```
/src
    /foo
        /index.ts
        /config.ts
        /fight.ts
```
A familiar setup to anyone who has seen a Node project. Node easily facilitates modularity using your project's directory structure so the tendency to contain bits of logic can be considerably high. It is not uncommon to find modules that export a single one-liner function or a simple object. In the structure above we have created a directory called "foo" that will export the functions we need for our foo related business, and will contain the logic of dealing with our third-party foo management client. 

The contents of the directory can look a bit like this: 

``` typescript
// index.ts
export * from "./fight"
```

``` typescript
// fight.ts
import client from "./config"

export function fight(id) {
    return client.updateFoo(id, { defeated: true });
}
```

``` typescript
// config.ts
import { WeFightFooIncClient } from "wefightfooinc";

export const client = new WeFightFooIncClient(config);
```

The functions that `index.ts` exposes make up the interface of our `foo` module and as long as we don't break that interface we won't have to worry about changing the calling code that makes use of this module. Hence, we are still programming to an interface but we are simply using our exported function signatures rather than a typescript `interface` to describe it. 

Things inside this module that are not exported by `index.ts` can be considered `private`, or somewhat analogous to `private`. For instance: `config.ts` is where our `WeFightFooIncClient` is configured, initialized and exported from. Mind that, outside the contents of this module, there is no concern or knowledge about the client dependency that our foo module uses. In addition, whereas before we had to instantiate the client somewhere outside the definition of the class and pass it down (either in the same file or anywhere else in the project structure, I have seen all variants), now we have a much more intuitive sense of where the client should come from. Any developer can open up a module and see where the dependency lives. 

The result is a much cleaner structure of our calling code where we take what we need from the module without having to include the entire context where a function was designed to live. Often, we're only interested in single tasks that we want modules to handle for us so, when reading the code, anything outside of that procedure is considered noise.

``` typescript
// receiveinsultfromfoo.ts
import { fight } from "./foo"

async function receiveInsultFromFoo(fooId: int, insult: int) {
    if (insult > 3) {
        await fight(fooId);
    }
}
```

Your unit tests should always give a good impression of the degree to which you can take apart your application into single pieces of logic (units, if you will). In this case, since we not only export functions from `index.ts` in our module but also export them all individually, we can completely isolate the functions in our test files. Since we can no longer pass a mocked dependency the way we did before we are going to leverage the Node module system instead. I am using [Jest](https://jestjs.io/) in this example because I am familiar with it but there are other ways of achieving the same thing in your Node application. 

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
Here we use `require` as a seam in our program and replace the config module with a mocked implementation in its stead.

So let's list some of the pros and cons again: 
+ we are programming to an interface (+)
+ the logic of using the client dependency is contained (+)
+ initialization logic is easy to trace through static analysis (+)
+ localizing initialization logic can be intuited from the structure of the project (+)
+ our testable units of code are functions (+)

The ultimate measure of our codebase is maintainability, which is almost a culture level concept, but it ultimately boils down to complexity. Complexity determines how fast your developers can move around and how inclined they are to write tests or look for shortcuts through the intended designs. In an overly complex system, testing gets thrown by the wayside and business concerns about getting features to market take over. Phrases like "let's just get this work for now" get tossed around and developers start updating their LinkedIn profiles. A Node project that leverages the module system can make for an elegant and straight-forward read with simple function exports that are easily isolated and tested, making it attractive for individual developers to include a lot of coverage.

## Conclusion
This has been a comparison of two different styles of dependency handling and encapsulation across some dimensions that I feel are appropriate. I tried to make clear that some of the choices we make can derive from OOP practices like Inversion of Control without actually leveraging an IoC design. Class-based encapsulation and dependency injection are often considered as obvious choices when faced with the problem of containing parts of the codebase, but Node allows us to leverage the module system as an alternative.

---
### Notes/further reading:
1. For some interesting reading about using seams in programs: check [these three pages](https://www.informit.com/articles/article.aspx?p=359417&seqNum=1).
2. The `config.ts` file in our module could have been named after the client dependency it exports as well. You could even argue that it would make the setup a bit more obvious.
3. When looking at the client export from `config.ts` one should be aware that Node uses paths constructed by `require` as cache keys, which could result in the client loading multiple times. Read more about it [here](https://nodejs.org/dist/latest-v14.x/docs/api/modules.html#modules_module_caching_caveats)
4. I felt like the obvious choice for an insult threshold from foo had to be 3. This goes without saying.