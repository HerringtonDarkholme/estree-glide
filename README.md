# estree-glide

A lightweight utility for traversing ESTree AST nodes with generator functions.

## Features

- Traverse ESTree-compatible AST nodes using generator functions
- Access parent nodes and path information during traversal
- Support for both entry and exit phase during node traversal
- TypeScript support

## Comparison with estree-walker

While [estree-walker](https://github.com/Rich-Harris/estree-walker) is a popular library for traversing ESTree ASTs using callback functions, estree-glide takes a different approach using generator functions:

### Why generator functions?

Generator functions in estree-glide provide several advantages:

- More intuitive control flow with `yield` statements
- Ability to return processed values from child nodes
- Natural separation of entry and exit phases
- Cleaner code without method binding or `this` context


## Explanation of the API

### 1. `enter` / `exit` code in one generator function, separated by `yield`

the generator function has two phases: the enter phase and the exit phase.

The whole execution is like this:
* The enter phase code runs before `yield`
* yield is called and the child nodes are traversed, recursively
* run the exit phase code is after `yield`.

Enter phase is executed when the node is first visited. The `yield` statement is used to pause the execution of the generator function and pass control to the child nodes. The child nodes will be traversed, and when they are done, control will return to the parent node, and the code after `yield` will be executed

This is a contrived simple example to illustrate the traversal function in estree-glide:

```ts
function* traverseAST(node: Node, accumulator: { enter: number, exit: number }) {
    // code before yield is equivalent to the enter phase in estree-walker
    accumulator.enter++

    // proceed traversal to child nodes
    yield

    // code after yield is equivalent to the exit phase in estree-walker
    accumulator.exit++
}
```

This is equivalent to estree-walker:

```TypeScript
import { walk } from 'estree-walker'
let accumulator = { enter: 0, exit: 0 }
walk(ast, {
    enter(node) {
        accumulator.enter++
    },
    leave(node) {
        accumulator.exit++
    }
})
```
Note in estree-glide we don't need to rely on the global variable `accumulator`.

### 2. argument of the traversal function

The first argument is the current node during the traversal, and the second argument is an accumulator object.

The accumulator object can be used to store information during the traversal, in this case, we are counting the number of nodes entered and exited. A more realistic example would be to collect the node definitions in a scope or collecting the usage of a variable.

To execute the traversal function

```ts
import { traverse } from 'estree-glide'
import { parse } from '@babel/parser'

const ast = parse("const a = 1", {
  sourceType: 'module',
  plugins: ['typescript', 'jsx'],
})

// initial accumulator value
const accumulator = { enter: 0, exit: 0 }
traverse(ast.program, traverseAST, accumulator)
```

### 3. change the accumulator value

Given the recursive nature of the traversal, it is quite often we need to use a new accumlator value for the child nodes. In estree-glide, we can use the `yield` statement to pass new accumulator to descendent node traversal.

For example, we can create new scope for function declarations and pass it to the child nodes.

```ts
function* collectScope(node: Node, scope: Scope) {
    if (node.type === 'VariableDeclaration') {
        // collect variable declarations in the current scope
        for (const declaration of node.declarations) {
            scope.declarations.push(declaration)
        }
    } else if (node.type === 'FunctionDeclaration') {
        const newScope = new Scope(node)
        scope.children.push(newScope)
        // pass the new scope to child nodes
        yield newScope
    }
}
```

Given this source code

```js
function foo() {
    var a = 1
    function bar() { var b = 2 }
}
```

It will create a scope tree like this:

```
Scope {
  declarations: [VariableDeclaration],
  children: [
    Scope {
      declarations: [VariableDeclaration],
      children: []
    }
  ]
}
```
