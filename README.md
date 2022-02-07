## Making Sense of Code Scanning with CodeQL

It's no secret that GitHub's code scanning tool, CodeQL, can help organizations of any size develop software faster and more securely, all without needing to leave GitHub.com, and there's no shortage of awesome content explaining what this looks like from the developer or organization owner's perspectives using the tool. However, something that I've struggled with is figuring out a simple yet thorough explanation of the mechanisms behind _how_ CodeQL delivers its code scanning alerts (something even deeper than statements like "it queries a relational database representation of the repo's code, which gets generated with every trigger event"). So that's why I'm writing this, and if it's something you'd like to learn, Keep Reading.

One final note: my focus here is on _how_ CodeQL works, not why it's valuable. To better understand the value propisition of using code scanning to help SHIFT LEFT, then you can read more about it [here](https://github.blog/2020-08-27-secure-at-every-step-putting-devsecops-into-practice-with-code-scanning/).

### The Basic Steps of CodeQL
1. Preparing the code by creating a CodeQL database
2. Running CodeQL queries against the database
3. Interpreting/presenting the query results

For my deep-dive here, I'm going to tackle each one of these in turn, starting with the first. Remember that in the context of code scanning with CodeQL, the running of this  process is defined in the workflow YAML file inside your .github/workflows directory in your repo. The setup to get started with this has been automated on GitHub and occurs when you first turn on code scanning under the "Security" tab of your repo. See here:

![alt text](Conf_CodeQL.png "Configuring CodeQL in a Repo")

Clicking on the "Configure CodeQL alerts" button will create a YAML file in your workflow directory that looks something like [this](https://github.com/nomdal/continuous-integration-circle/blob/nomdal-patch-1/.github/workflows/codeql-analysis.yml). It is fully customizable, but starts off with some defaults, like executing the scan whenever there's a pull request or push against the main branch of the repo, as well as on a scheduled timetable. Bear in mind, however, that this is only the automation to execute the code scanning, and that the process itself can also be executed manually from the CLI on a repo if you have the right packages included (read more about how to do so [here](https://codeql.github.com/docs/codeql-cli/getting-started-with-the-codeql-cli/)).

But regardless of how its setup and triggered, lets figure out how the code scanning process actually works:

### 1. Preparing the Code by Creating a CodeQL Database

The most basic explanation you'll find regarding this step is that to create a database, CodeQL first extracts a single relational representation of each source file in the codebase. But what does that entail?

#### The Extraction Process

The conversion from source code to a queryable database is done by a special program called an extractor, and to perform this process on any given language, an extractor built specifically for that language is needed. As I'll explain later, the ambiguities present in certain programming languages (such as Ruby) make it harder to write extractors, as one of the key functions of an extractor is to _parse_ the code, which is more difficult for more ambiguous languages.

Beyond which language is being used, there are two different scenarios for the extraction process, both covered below:

##### Extraction for Compiled Languages

Extraction in this case works by monitoring the normal build process. Each time a compiler is invoked to process a source file, a copy of that file is made, and all relevant information about the source code is collected. This includes syntactic data about the abstract syntax tree and semantic data about name binding and type information, which we'll cover in greater detail shortly.

##### Extraction for Non-Compiled Languages

For non-compiled languages, the extractor runs directly on the source code, resolving dependencies to give an accurate representation of the codebase.

In both scenarios, a parse tree is ultimately created to represent the source code, and this tree is then transformed into a relational database to run queries against. What exactly does that mean, though? Let's use an example from the [GitHub Blog](https://github.blog/2022-02-01-code-scanning-and-ruby-turning-source-code-into-a-queryable-database/).

Whatever program we're looking at, there must be some function that kicks the whole thing off, and it's at this location that our parse tree takes root. Let's use this basic Ruby function as a hypothetical starting point:

```puts("Hello", "Ahoy")```

The program contains three expressions: the string literals ```"Hello"``` and ```"Ahoy"``` and a call to a method named ```puts```, which prints its inputs to new lines. A minimal parse tree for the program like this:

![alt text](parse_tree_basic.png "Basic Parse Tree")

Remember, it's the role of our extractor to follow along with our code to build a representation of this parse tree. Compilers and static analysis tools like CodeQL often convert the parse tree into a simpler format known as an abstract syntax tree (AST), so-called because it abstracts away some of the syntactic details that do not affect the meaning of the program, such as comments, whitespace, and parentheses. However, for CodeQL the Ruby extractor stores the parse tree in the database, since those syntactic details can sometimes be useful. We then provide a query library to transform this into an AST, and most of our queries and query libraries build on top of that transformed AST, either directly or after applying further transformations (such as the construction of a data-flow graph).

While the above example is quite simple, you can imagine how if other functions were ultimately called by the root of the tree, they would then generate their own branches beneath them, and we'd follow it to the bottom until the entire codebase has been parsed into a single tree. It is at this point that we need to convert this parse tree object into a relational database for us to query, the main job of the extractor.

Going back to the diagram above, I can attempt to convert it to relational form. To do that, I must first define a schema for the database, which is a bit more art than science.

```
expressions(id: int, kind: int)
calls(expr_id: int, name: string)
call_arguments(call_id: int, arg_id: int, arg_index: int)
string_literals(expr_id: int, val: string)
```

The _expressions_ table has a row for each expression in the program. Each one is given a unique _id_, a primary key that can be used to reference the expression from other tables. The _kind_ column defines what kind of expression it is. Let's define it so a value of 1 means the expression is a method call and 2 means it’s a string literal, and so on.

The _calls_ table has a row for each method call, and it allows us to specify data that is specific to call expressions. The first column, _expr_id_ is a foreign key, i.e. the id of the corresponding entry in the _expressions_ table, while the second column specifies the _name_ of the method. You might have expected that we'd add columns for the call’s arguments, but calls can take any number of arguments, so we wouldn’t know how many columns to add, so instead, we put them in a separate _call_arguments_ table.

The _call_arguments_ table, therefore, has one row per argument in the program. It has three columns: 1) _call_id_, which is a reference to the call expression, 2) _arg_id_, which is a reference to an argument, and 3) _arg_index_ that specifies the index, or number, of that argument.

Finally, the _string_literals_ table lets us associate the literal text value (_val_) with the corresponding entry (_expr_id_) in the _expressions_ table.

So now that we've defined our schema, let's manually populate a matching database with rows for our simple program:

```expressions```
| id    |   kind            |
| ----- | -----------       |
| 100   | 1 (call)          |
| 101   | 2 (string literal)|
| 102   | 2 (string literal |

```calls```
| expr_id    |   name            |
| ----- | -----------       |
| 100   | "puts"         |

```call_arguments```
| call_id	    |   arg_id	            | arg_index |
| ----- | -----------       | ------ |
| 100   | 101       |  0 |
| 100   | 102       |  1 |

```string_literals```
| expr_id	    |   val	            | 
| ----- | -----------       | 
| 101   | "Hello"       | 
| 102   | "Ahoy"       |

Now, suppose this was a SQL database, and we wanted to write a query to find all the expressions that are arguments in calls to the puts method. It might look like this:

```
SELECT call_arguments.arg_id
FROM call_arguments
INNER JOIN calls ON calls.expr_id = call_arguments.call_id
WHERE calls.name = "puts";
```
In practice, however, we don’t use SQL. Instead, CodeQL queries are written in the QL language and evaluated using our custom database engine. QL is an object-oriented, declarative logic-programming language that is superficially similar to SQL but based on [Datalog](https://en.wikipedia.org/wiki/Datalog). Here’s what the same query might look like in QL:

```
from MethodCall call, Expr arg
where
  call.getMethodName() = "puts" and
  arg = call.getAnArgument()
select arg
```

```MethodCall``` and ```Expr``` are classes that wrap the database tables, providing a high-level, object-oriented interface, with helpful predicates like ```getMethodName()``` and ```getAnArgument()```.

Every language has unique quirks, so each one CodeQL supports has its own schema, perfectly tuned to match that language’s syntax. Whereas our example schema has only two kinds of expression (calls and string literals), the [JavaScript](https://github.com/github/codeql/blob/d00196f6be5282ca4fa02dadb74a8c9675d96eec/javascript/ql/lib/semmlecode.javascript.dbscheme) and [Ruby](https://github.com/github/codeql/blob/d00196f6be5282ca4fa02dadb74a8c9675d96eec/ruby/ql/lib/ruby.dbscheme) schemas that GitHub defines for CodeQL, for example, both define over 100 kinds of expression. Those schemas were written manually, refined and expanded over the years as we make improvements and add support for new language features.

How easy or difficult this translation from parse tree to relational database is depends a lot on how similar those two structures look. For some languages, we defined our database schema to closely match the structure (and naming scheme) of the tree produced by the parser. That is, there’s a high level of correspondence between the parser’s node names and the database’s table names. For those languages, an extractor’s job is fairly simple. For other languages, where we perhaps decided that the tree produced by the parser didn’t map nicely to our ideal database schema, we have to do more work to convert from one to the other.

At this point, we've created a relational database representation of our code, and thus, are done with Step 1 and can move on to Step 2.

### 2. Running CodeQL queries against the database

