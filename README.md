# YYACC

Yet Yet Another Compiler Compiler

---

# Construction of a Compiler Framework and an Implementation of a PL/0 Language Interpreter

## Regular expressions for tokens

- number: `[0-9][0-9]*`
- identifier: `[_a-zA-Z][_a-zA-Z0-9]*`
- assignment symbol: `\:\=`
- less than or equal to: `\<\=`
- greater than or equal to: `\>\=`
- other symbols: `[!?:;.,#<=>+-\*/\(\)]`
- comment: `\{[\f\n\r\t\v -z|~]*\}`
- whitespace characters: `[ \f\n\r\t\v][ \f\n\r\t\v]*`

## Automaton architecture

An automaton can be represented by a 5-tuple $(Q,\Sigma,\delta,q _ 0,F)$.

- $Q$ is the set of states
- $\Sigma$ is the alphabet
- $\delta(q,c)$ is the transition function
  - the set of states reached from state $q$ by the character $c$
- the initial state is $q _ 0$, and the set of final (accepting) states is $F$

For simplicity, we write $\hat\delta(q,cw)=\hat\delta(\delta(q,c),w),(w \in \Sigma^*)$.

It is easy to see that both DFA and NFA are automata, but their transition functions are defined differently (NFA is `vector<vector<set<int>>>`, while DFA is `vector<vector<int>>`).

Template classes can be used to support different inheritance types.

``` cpp
template<typename T_Delta, typename T_F, typename T_isend>
class Automachine {
public:
    int Q;         // node count (index starts from 1)
    int Sigma;     // alphabet size
    T_Delta Delta; // δ(q,w)
    int q0;        // start state (node)
    T_F F;         // final states
    T_isend isend; // set if the current state is final
    Automachine() {}
    Automachine(int Q, int Sigma, T_Delta Delta, int q0, T_F F, T_isend isend):
        Q(Q), Sigma(Sigma), Delta(Delta), q0(q0), F(F), isend(isend) {};
};
class NFA: private Automachine<NFA_Delta, NFA_F, NFA_isend>; // NFA = (Q, Σ, δ, q0, F)
class DFA: private Automachine<DFA_Delta, DFA_F, DFA_isend>; // DFA = (Q, Σ, δ, q0, F)
```

### From regular expressions to ε-NFA accepting states (Thompson construction)

Using the following transformations, a regular expression can be constructed into an equivalent ε-NFA.

- empty expression `ε`:
  ![inline](img/278px-Thompson-epsilon.svg.png)
- single symbol `a`:
  ![inline](img/Thompson-a-symbol.svg.png)
- union of expressions `(s|t)`
  ![inline](img/Thompson-or.svg.png)
- concatenation of two expressions `(st)`:
  ![inline](img/Thompson-concat.svg.png)
- Kleene closure `(s*)`:
  ![inline](img/503px-Thompson-kleene-star.svg.png)

### From ε-NFA to NFA (removing ε)

Let $\epsilon(q)=\set{p|\hat \delta(q,\varepsilon)=p}$ denote the set of states reachable from state $q$ through ε-transitions.

Therefore, $\forall c \in \Sigma,\delta_{NFA}(E(q),c)=\set{\cup_{p \in S} E(p)|S=\set{r|\delta(q,c)=r} }$.

In essence, this represents the process of simulating an ε-NFA with a simplified state set.

### From NFA to DFA (determinization of NFA)

We use the subset construction method.

Similar to the simplification step for ε-NFA, we write $\Gamma _ c(q)= \set{ p|\delta(q,c)=p }$.

Then the DFA is constructed as follows: the initial state is $\set{ q _ 0 }$, and the transition function is $\delta_{DFA}(S,c)=\cup_{p \in S} \Gamma_c(p)$.

The following points are important:

1. a hash-based method can be used to build the mapping; `map<set<int>, int>` also works.
2. NFA determinization through subset construction is based on BFS, which means using `queue<set<int>> que;` to store the state set $S$, and updating it each time by taking the first element from the queue.

### From DFA to MFA (minimizing DFA)

There are many algorithms for DFA minimization. Here, the relatively direct **table-filling algorithm** is used.

States $p$ and $q$ are defined to be equivalent if and only if $\forall w\in\Sigma^+,s.t.\hat \delta(p,w)=\hat \delta(q,w)$. In other words, if $\exists w \in \Sigma^+,s.t.\hat\delta(p,w) \ne \hat \delta(q,w)$, then states $p$ and $q$ are not equivalent.

First, if $p \in F,q \not \in F$, then $p$ and $q$ are different. Second, if $\delta(p,c)=p',\delta(q,c)=q'$ and $p',q'$ are different, then $p$ and $q$ are different. At the same time, if $p$ and $q$ are different, then there exists a string $w$ such that $\hat\delta(p,w) \in F,\hat\delta(q,w)\not\in F$, or the reverse.

Therefore, BFS can be used again. Initially, all state pairs $(p,q)$ satisfying $p \in F,q \notin F$ are added to the queue. Then all inequivalent state pairs are eliminated one by one. The remaining state pairs form all equivalence classes.

After that, it is only necessary to remove all states that are unreachable from the initial state to obtain the MFA. To simplify string matching, the DFA structure can be modified slightly here by removing all states that cannot reach any accepting state.

### String matching with MFA

The main technical issues here are how to use a precompiled **DFA for string matching**, and how to **read the binding structure** in a clean way.

When scanning a string linearly, each DFA is used in turn for greedy matching.

``` cpp
void match(string str) {
    int len = str.length(), ptr = 0;
    while(ptr < len) { // analyze the raw code
        walka(&hk, hooks) {
            auto &raw = get<0>(hk);
            auto &act = get<1>(hk);
            auto &dfa = get<2>(hk);
            raw = "", dfa.resetState();
            int ptrmem = ptr, isendstate = 0;
            while(ptr < len && dfa.test(str[ptr])) {
                isendstate = dfa.feed(str[ptr]);
                raw += str[ptr ++];
            }
            if(isendstate) { // state is in `endF`, and we matched a string
                act(raw);
                goto nxt;
            } else {
                ptr = ptrmem;
            }
        }
        errorlog("error in `match`: can't match any regex, char: %c (ascii: %d)", str[ptr], (int) str[ptr]);
        nxt: ;
    }
}
```

For binding matching patterns and callback functions, the method `Lexer& f() { return *this; }` is used here to support chained calls.

``` cpp
lexer.feed("[0-9][0-9]*",
           [&](string raw) {
               bpb("[number]", raw);
               tpb(TokenType :: number, raw);
           })
    .feed("[_a-zA-Z][_a-zA-Z0-9]*", // identifiers 
          [&](string raw) {
              bpb("[ident]", raw);
              tpb(TokenType :: ident, raw);
          });
```

## Design of the LL(1) grammar

First, the PL/0 grammar in EBNF form is given below:

``` pascal
program = block "." ;
block = [ "const" ident "=" number {"," ident "=" number} ";"]
        [ "var" ident {"," ident} ";"]
        { "procedure" ident ";" block ";" } statement;
statement = [ ident ":=" expression | "call" ident 
              | "?" ident | "!" expression 
              | "begin" statement {";" statement } "end" 
              | "if" condition "then" statement 
              | "while" condition "do" statement ];
condition = "odd" expression |
            expression ("="|"#"|"<"|"<="|">"|">=") expression;
expression = [ "+"|"-"] term { ("+"|"-") term};
term = factor {("*"|"/") factor};
factor = ident | number | "(" expression ")";
```

Since the parser is implemented with RDP, it is only necessary to eliminate **left recursion** (it does not have to fully satisfy LL(1)). This only requires adding suffix parts to split left-recursive productions.

The original version of PL/0 does not handle negative numbers very well, so some definitions are rewritten here.

``` pascal
<program> = <block> "."
<block> = <constdef> <vardef> <procdef> <statement>
<constdef>    = "const" <ident> "=" <number> <constdefpri> ";"
              | ""
<constdefpri> = "," <ident> "=" <number> <constdefpri>
              | ""
<vardef>    = "var" <ident> <vardefpri> ";"
            | ""
<vardefpri> = "," <ident> <vardefpri>
            | ""
<procdef> = "procedure" <ident> ";" <constdef> <vardef> <statement> ";" <procdef>
          | ""
<statement>    = <ident> ":=" <expression>
               | "call" <ident> 
               | "?" <ident>
               | "!" <expression> 
               | "begin" <statement> <statementpri> "end" 
               | "if" <condition> "then" <statement>
               | "while" <condition> "do" <statement>
<statementpri> = ";" <statement> <statementpri>
               | ""
<condition> = "odd" <expression>
            | <expression> "="  <expression>
            | <expression> "#"  <expression>
            | <expression> "<"  <expression>
            | <expression> "<=" <expression>
            | <expression> ">"  <expression>
            | <expression> ">=" <expression>
<expression>    = <term> <expressionpri>
                | "+" <term> <expressionpri>
                | "-" <term> <expressionpri>
<expressionpri> = "+" <term> <expressionpri>
                | "-" <term> <expressionpri>
                | ""
<term>    = <factor> <termpri>
<termpri> = "*" <factor> <termpri>
          | "/" <factor> <termpri>
          | ""
<factor> = <ident>
         | <number>
         | "(" <expression> ")"
         | "+" "(" <expression> ")" // extra
         | "-" "(" <expression> ")" // extra
         | "+" <number> // extra
         | "-" <number> // extra
         | "+" <ident>  // extra
         | "-" <ident>  // extra
<ident>  = `IDENT`
<number> = `NUMBER`
```

The main difficulty in this part is how to handle signed expressions, such as numbers like `-114` or `+514`, variables like `-homo`, or a leading sign before an expression such as `+(1919/810)`. A practical approach is to reduce all of them to `<factor>`, that is, to add prefix-sign recognition.

## Parsing the abstract syntax tree

The **recursive descent procedure method** is used here to parse the abstract syntax tree. The basic idea is to simulate grammar matching through the recursive function stack.

The CFG is stored by constructing an abstract transition graph, and then recursive descent parsing is simulated on top of it.

Through macro definitions, the use of intermediate nodes and terminal nodes becomes easier. A `Node` returns its own address so that chained operations can be written directly.

``` cpp
#define T(val) (parser.getTerminalNode(val))      // Terminal
#define N(name) (parser.getNonterminalNode(name)) // Nonterminal
N(statement) -> add({ N(ident), T(":="), N(expression) })
    -> add({ T("call"), N(ident) })
    -> add({ T("?"), N(ident) })
    -> add({ T("!"), N(expression) })
    -> add({ T("begin"), N(statement), N(statementpri), T("end") })
    -> add({ T("if"), N(condition), T("then"), N(statement) })
    -> add({ T("while"), N(condition), T("do"), N(statement) });
```

## Design of the assembly prototype and compilation scheme

First, a basic assembly instruction set is designed. It contains a total of $16$ basic instructions, which represent stack push, popping the top stack value into a variable, conditional jump, the four arithmetic operations, conditional tests (`odd` means checking whether a value is odd, and `#` means inequality), console input (`?`), and console output (`!`).

``` text
push, pop, jff, +, -, *, /, odd, =, #, <, <=, >, >=, ?, !
```

- `push x` pushes the immediate value $x$ onto the top of the stack, and `push @x` pushes the value of variable `@x` onto the stack.
- `pop @x` pops the top element of the stack and assigns it to variable `@x`.
- `jff label` first pops the top stack element; if that value is $0$, execution jumps to `label`. 
- `+, -, *, /, =, #, <, <=, >, >=` pop the top two stack elements $x_1,x_2$, then push $x_1 \oplus x_2$ onto the stack.
- `odd` pops the top stack element $x$; if $x$ is odd, it pushes $1$, otherwise it pushes $0$.
- `?` reads an integer (type `int`) from the console and pushes it onto the stack.
- `!` pops the top stack element and outputs it.

The detailed compilation scheme is shown below.

``` pascal
"const" <ident> "=" <number> <constdefpri> ";"  {  }  // this part performs direct variable substitution during compilation
"var" <ident> <vardefpri> ";" { push @@__stack_top; push <vert_ident>; +; pop @@__stack_top; } // indirect filling
"procedure" <ident> ";" <constdef> <vardef> <statement> ";" <procdef>
{
    push 0;
    jff nxt;
    &<ident>:;
    push @@__stack_top;
    push @@__stack_bottom;
    push @@__stack_top;
    push 1;
    +;
    pop @@__stack_bottom;
    [<constdef>]
    <vardef>
    <statement>
    push @@__stack_bottom;
    push 1;
    -;
    pop @@__stack_top;
    pop @@__stack_bottom;
    pop @@__stack_top;
    pop @@__ptr;
    nxt:;
}
<ident> ":=" <expression>  { <expression>; pop @<ident>; }
"call" <ident>    { push @@__ptr; push 4; +; push 0; jff &<ident>; }
"?" <ident>       { ?; pop @<ident>; }  // ? reads into the stack top and reuses pop
"!" <expression>  { <expression>; !; }  // ! outputs and pops the stack top, reusing <expression>
"begin" <statement> <statementpri> "end"  { <statement> }
"if" <condition> "then" <statement>       { <condition>; jff nxt; <statement>; nxt:; } // jump if <condition> is false
"while" <condition> "do" <statement>      { beg:; <condition>; jff nxt; <statement>; push 0; jff beg; nxt:; }
"odd" <expression>                { <expression>; odd; }
<expression> "="  <expression>    { <expression1>; <expression2>; =; }
<expression> "#"  <expression>    { <expression1>; <expression2>; #; }
<expression> "<"  <expression>    { <expression1>; <expression2>; <; }
<expression> "<=" <expression>    { <expression1>; <expression2>; <=; }
<expression> ">"  <expression>    { <expression1>; <expression2>; >; }
<expression> ">=" <expression>    { <expression1>; <expression2>; >=; }
"+" <term> <expressionpri>    { <factor1>; <factor2>; +; }  // pop f1 and f2 from the stack top, then compute f1+f2 and push the result
"-" <term> <expressionpri>    { <factor1>; <factor2>; -; }  // pop f1 and f2 from the stack top, then compute f1-f2 and push the result
"*" <factor> <termpri>  { <factor1>; <factor2>; *; }  // pop f1 and f2 from the stack top, then compute f1*f2 and push the result
"/" <factor> <termpri>  { <factor1>; <factor2>; /; }  // pop f1 and f2 from the stack top, then compute f1/f2 and push the result
<ident>   { push @<ident> }
<number>  { push <number> }
"(" <expression> ")"  { <expression> } // after evaluation, <expression> leaves its value on the stack top
"+" "(" <expression> ")" { <expression> }
"-" "(" <expression> ")" { push 0; <expression>; -; }
"+" <number> { push <number>; }
"-" <number> { push 0; push <number>; -; }
"+" <ident>	 { push @<ident>; }
"-" <ident>  { push 0; push @<ident>; -; }
```

## Processing and execution of bytecode

First, line-by-line translation is performed according to the compilation scheme above to obtain assembly code; bytecode can be obtained in the same way. After compilation is completed, execution is performed through a virtual machine (implemented here as a single-stack machine).

``` cpp
if(nd -> astchild[0] -> val == "if") {
    // if <condition> do <statement>
    auto nxt = 0; // needs to be modified
    dfs(nd -> astchild[1]);
    auto idx = opcode.size();
    opcode.add(Opcode :: bc_jff(nxt)); // needs to be modified
    dfs(nd -> astchild[3]);
    opcode.modify(idx, Opcode :: bc_jff(opcode.size())); // rewrite the `nxt` address here
}
```

## Results

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="img/187109634-77818110-a7b1-4275-aaaa-cb96c8748f67.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">Lexer Output</div>
</center>

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="img/187109794-ca2ad1d8-dd0e-4fc6-8840-16a6c520b802.jpg">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">Assembly Output</div>
</center>


## References

- [1] [wikipedia-Regular language](https://en.wikipedia.org/wiki/Regular_language)
- [2] [wikipedia-Nondeterministic finite automaton](https://en.wikipedia.org/wiki/Nondeterministic_finite_automaton)
- [3] [wikipedia-Deterministic finite automaton](https://en.wikipedia.org/wiki/Deterministic_finite_automaton)
- [4] [wikipedia-Context-free grammar](https://en.wikipedia.org/wiki/Context-free_grammar)
- [5] [wikipedia-Abstract syntax tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree)
- [6] [wikipedia-LL parser](https://en.wikipedia.org/wiki/LL_parser)
- [7] [wikipedia-Recursive descent parser](https://en.wikipedia.org/wiki/Recursive_descent_parser)
- [8] [wikipedia-Powerset construction](https://en.wikipedia.org/wiki/Powerset_construction)
- [9] [wikipedia-Breadth-first search](https://en.wikipedia.org/wiki/Breadth-first_search)
- [10] [wikipedia-DFA minimization](https://en.wikipedia.org/wiki/DFA_minimization)
- [11] [wikipedia-Left recursion](https://en.wikipedia.org/wiki/Left_recursion)
