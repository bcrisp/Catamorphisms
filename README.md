# Catamorphisms in JavaScript

Inspired by [Brian McNamara's work on catamorphisms in F#](http://lorgonblog.wordpress.com/2008/04/05/catamorphisms-part-one/), this is a
 proof-of-concept implementation for a tail-recursive JavaScript function using continuation-passing style.

 Due to differences in the way [D3.js](d3js.org) handles vertices and edges for force-directed graph and tree layouts, this is actually more than strictly catamophorphic: since it can take a tree to any other structure, including another tree, code editor and D3.js visualization tool for translating JavaScript to abstract syntax trees and then to a tree. This only parses a small subset of JavaScript code, but it demonstrates catamorphisms, continuation-passing style, and utilizes tail-recursion ([when JavaScript supports it.](http://blog.mozilla.org/dherman/2011/01/30/proper-tail-calls-in-harmony/))

Although this project uses several terrific JavaScript libraries for visualization and parsing, the actual AST destruction is implemented in pure JavaScript to demonstrate its generality. No JQuery, no CoffeeScript. Just JavaScript.


## Context

In functional languages, using control structures like foreach are discouraged, and instead a Fold is used. Folds abstract out the iteration from the action. For instance, say we wanted to sum the following numbers: [1, 3, 5, 7]

Here's an imperative approach:

    int sum = 0;

    for (int i; i < numbers.length; i++)
    {
	    sum = sum + numbers[i];
    }

And a functional approach:

`fold (numbers, +)`

Fold "breaks apart" each of the elements in the structure, which in this case is an array, and applies the supplied function to each argument while holding onto the accumulated value. Fold separates iterating over the data from how you want to manipulate the data. In the imperative approach, I have to declare a variable outside the scope of the loop, specify how I want to iterate over my data structure, and then do something with each element of my list.

Fold, however, only works on "one-dimensional" structures with independent elements. While it would be grea to have the simplicity of Fold applied to higher-dimensional structures, it's not immediately apparent how to do this. For complex tree structures or graphs, iteration is more complex than a step-by-step operation; it requires knowing something about the structure of the data we're traversing. Wouldn't it be great to be able to seperate the structure of the data from the operations we perform on it? Yes it would.

## Code

An example JavaScript AST. Notice that this isn't a simple binary tree -- some elements have more than one child, some have one, and some sparse trees may have none.

    {
        "type": "Program",
        "body": [{
                "type": "VariableDeclaration",
                "declarations": [{
                    "type": "VariableDeclarator",
                    "id": {
                        "type": "Identifier",
                        "name": "fibonnaci"
                    },
                    "init": {
                        "type": "BinaryExpression",
                        "operator": "*",
                        "left": {
                            "type": "BinaryExpression",
                            "operator": "*",
                            "left": {
                                "type": "Literal",
                                "value": 1,
                                "raw": "1"
                            },
                            "right": {
                                "type": "Literal",
                                "value": 1,
                                "raw": "1"
                            }
                        },
                        "right": {
                            "type": "Literal",
                            "value": 2,
                            "raw": "2"
                        }
                    }
                }],
                "kind": "var"
            }, {
                "type": "ExpressionStatement",
                "expression": {
                    "type": "CallExpression",
                    "callee": {
                        "type": "Identifier",
                        "name": "alert"
                    },
                    "arguments": [{
                        "type": "Identifier",
                        "name": "fibonnaci"
                    }]
                }
            }

A subset of the catamophism to handle the code:

      case "Program":
      // Program :: <type> <body> <kind>
      return cont(program(expression, Loop(expression.body, 
        function(body) {
          return body;
        })));
      break;

      case "VariableDeclaration":
      // VariableDeclaration ::= <type> [declarations]
      return Loop(expression.declarations, 
        function(declarations) {
          return cont(variableDeclaration(expression, declarations))
        });
      break;

      case "VariableDeclarator":
      // VariableDeclarator ::= <Identifier> <Expression>
      return Loop(expression.id,
        function(id) {
          return Loop(expression.init,
            function(init) {
              return cont(variableDeclarator(expression, id, init))
            })
        });
      break;

      case "BinaryExpression":
      // BinaryExpression ::= <Type> <Operator> <Literal> <Literal>
      return Loop(expression.left,
        function(left) {
          return Loop(expression.right,
            function(right) {
              return cont(binaryExpression(expression, left, right))
            })
        });
      break;

 Although the full details are in the source code, here's an intuitive explanation: FoldExpr takes an expression and a continuation to be excuted when it's finished processing the current level. Each statement in the program is responsible for either continuing breaking down the tree or doing something with the current expression. Here's the key: just as in Fold, the catamorphism controls the iteration over the data structure, and the functions executed on each expression have no context. We could generate the height of the tree, or sum the nodes, or any other operation without changing the underlying catamporphism, just changing the supplied arguments.

Continuations are key to implementing this with tail-recursion. Let's recall the end goal: we're trying to traverse to the end of the tree, since that's where the specific instructions are. So we're "building up" to the root from the bottom up. So we need to keep a "thread", or a history of the nodes we've weaved through so far.
 Although it doesn't matter in JavaScript (although that's slated for the Harmony release), traversing the tree builds a call stack of the function's state each and every time the function is entered. 
 Space on the call stack is precious, so instead of remembering where we've been, we just remember where we need to go.

 Imagine a catamorphism as threading through a data structure. For collections in a single dimension, this is a step-by-step iteration with the same function applied to each element; for collections that are more complex, we can view them as heterogenous -- a tree is just a list with heterogenous data. And we're reducing this two-dimensional structure down to a single dimension.

 We can view this tree as just a list of objects. We start the catamorphism on the first node, "Program". This has a length of one, so we apply the Program function to this list with a single item. But before this application, we pass control into another instance of the catamorphism with a continuation, staying on the same level. In future JavaScript engines, our catamorhism will never exceed a call stack size of 1. While this is recursive, we're not retaining any state in the previous function, and we're not taking up space on the call stack. Instead, we're building up an increasingly complex set of contuation instructions which will be executed in a chain to destruct our tree in a well-defined sequence of actions. Since we've parameterized the catamorphism, the specfic work we want to do at each step of the iteration can be swapped for any kind of functionality we might want.


# Using

* The [Esprima library](esprima.org) parses the code into an abstract syntax tree.
2. [D3](d3js.org) is used for rendering the object tree or array.
3. Code editor and syntax visualization via [CodeMirror](codemirror.net) 
4. Presentation enhanced with [Twitter Bootstrap](http://twitter.github.io/bootstrap/)

# More Reading

* [Wikpedia: Catamorphism](http://en.wikipedia.org/wiki/Catamorphism) 
* [Brian McNamara on Catamorphisms in C#](http://lorgonblog.wordpress.com/2008/04/05/catamorphisms-part-one/)