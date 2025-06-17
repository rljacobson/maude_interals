```ebnf
special ( <hook-clause> <hook-clause>* )

<hook-clause> ::= id-hook <id-hook-spec>
                | op-hook <op-hook-spec>

                | term-hook <term-hook-spec>


<id-hook-spec> ::= <hook-name> ( <detail>* )

<op-hook-spec> ::= <hook-name> ( <op-spec> )

<term-hook-spec> ::= <hook-name> ( <term> )


<op-spec> ::= <op-name> : [<sort> '?'?]* ~> <sort>
```
