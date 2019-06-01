---
title: Experiments with Antlr
tags: java antlr
---
At work, I've had reasons to look into [Antlr](https://www.antlr.org/) recently and decided 
to do some experiments in a stand-alone project to get some experience with it. Antlr is
a [parser generator](https://en.wikipedia.org/wiki/Comparison_of_parser_generators) -
I played with something like that way back when in college, but that was 
[bison](https://en.wikipedia.org/wiki/GNU_Bison) and 
[yacc](https://en.wikipedia.org/wiki/Yacc).
I've also implemented some parsers for our data formats when working on games, usually with 
straight up C++ or Python code.

Anyway, Antlr is a joy to work with, especially with the 
[IntelliJ plugin](https://plugins.jetbrains.com/plugin/7358-antlr-v4-grammar-plugin).
In order to get to grips with using it, I implemented a simple expression calculator
in Java.

## AntlrCalc
As always, [the source code](https://github.com/snorristurluson/AntlrCalc) is up on GitHub.

The calculator should handle simple expressions, such as:
```
1+2
3.14*10+27.1
(4+3)*2
```
In addition, I want it to handle variables in the expressions:
```
x+y*4
```
Finally, it should handle functions:
```
3*sin(x+y)
```

### Grammar
The Antlr grammar for this is quite simple:
```antlrv4
grammar Calc;
@header {
package calc;
}

expression      : term ((PLUS | MINUS) term)* ;
term            : factor ((TIMES | DIV) factor)* ;
factor          : signed_factor | xfactor ;
signed_factor   : (PLUS | MINUS) xfactor ;
xfactor         : paren_expr | function_call | value ;
paren_expr      : '(' expression ')' ;
function_call   : function_name '(' (expression (',' expression)*)? ')' ;
function_name   : ID ;
value           : variable | number ;
variable        : ID ;
number          : NUMBER ;

PLUS            : '+';
MINUS           : '-';
TIMES           : '*';
DIV             : '/';
ID              : LETTER (LETTER|DIGIT)*;
LETTER          : ('a'..'z')|('A'..'Z')|'_';
NUMBER          : DIGIT+ ('.' DIGIT+)?;
DIGIT           : ('0'..'9');
WS              : [ \r\n\t]+ -> skip;
```
In IntelliJ, we can preview the handling of this grammar - for example, the 
expression _(1+2)*3_ produces the following parse tree:

![Parse tree](/images/parseTree_simpleExpression.png)

## Generating code
In order to do something with the expression, beyond looking at a parse tree in IntelliJ,
we need to have Antlr generate the parser code. In my project, I'm using 
[Gradle](https://gradle.org/), so it handles the details of getting the Antlr runtime. 

Here's my _build.gradle_ file:

```groovy
plugins {
    id 'java'
    id 'antlr'
}

version '1.0-SNAPSHOT'

sourceCompatibility = 1.8

repositories {
    mavenCentral()
}

task wrapper(type: Wrapper) {
    gradleVersion = '4.10'
}

generateGrammarSource {
    arguments += ["-visitor"]
}

dependencies {
    testCompile group: 'junit', name: 'junit', version: '4.12'
    antlr("org.antlr:antlr4:4.7")
}
```
I've opted for using the visitor pattern for evaluating the expressions, as opposed
to a listener pattern. Antlr supports both, but by default only generates listener
classes, hence the added _visitor_ argument.

### CalcEvaluator
To evaluate an expression, I've implemented the CalcEvaluator class, extending the
CalcBaseVisitor that Antlr generates:
```java
public class CalcEvaluator extends CalcBaseVisitor<Double> {
    private Map<String, Double> variables = new HashMap<>();

    public void set(String name, double value) {
        variables.put(name, value);
    }

    @Override
    public Double visitExpression(CalcParser.ExpressionContext ctx) {
        double accumulator = ctx.getChild(0).accept(this);
        int childCount = ctx.getChildCount();
        for (int i = 1; i < childCount; i += 2) {
            int op = ((TerminalNode) ctx.getChild(i)).getSymbol().getType();
            Double nextTermValue = ctx.getChild(i + 1).accept(this);
            if(op == CalcParser.PLUS) {
                accumulator += nextTermValue;
            } else if(op == CalcParser.MINUS) {
                accumulator -= nextTermValue;
            }
        }
        return accumulator;
    }

    @Override
    public Double visitTerm(CalcParser.TermContext ctx) {
        ParseTree child = ctx.getChild(0);
        double accumulator = child.accept(this);
        int childCount = ctx.getChildCount();
        for (int i = 1; i < childCount; i += 2) {
            int op = ((TerminalNode) ctx.getChild(i)).getSymbol().getType();
            Double nextTermValue = ctx.getChild(i + 1).accept(this);
            if(op == CalcParser.TIMES) {
                accumulator *= nextTermValue;
            } else if(op == CalcParser.DIV) {
                accumulator /= nextTermValue;
            }
        }
        return accumulator;
    }

    @Override
    public Double visitSigned_factor(CalcParser.Signed_factorContext ctx) {
        int op = ((TerminalNode) ctx.getChild(0)).getSymbol().getType();
        Double value = ctx.getChild(1).accept(this);
        if(op == CalcParser.MINUS) {
            value = -value;
        }
        return value;
    }

    @Override
    public Double visitParen_expr(CalcParser.Paren_exprContext ctx) {
        return ctx.getChild(1).accept(this);
    }

    @Override
    public Double visitNumber(CalcParser.NumberContext ctx) {
        return Double.parseDouble(ctx.getText());
    }

    @Override
    public Double visitVariable(CalcParser.VariableContext ctx) {
        return variables.get(ctx.getText());
    }
}
```
The _visitNumber_ and _visitVariable_ rules are very simple - extracting the number
from the text of the node or using the text of the node as the lookup key into the
variables map.

The _visitExpression_ and _visitTerm_  methods are quite similar to each other, using
an accumulator to accumulate the value. We know from the way the rules are set up that
every other child in the parse tree is an expression - the nodes in between are terminal
nodes for the operator.

The grammar is intentionally laid out to make the visitor methods easy to implement.
If you find that a visitor method is getting complicated, it may be worth revisiting
the grammar, adding helper nodes to get Antlr to generate more methods that will
in turn be simpler to implement.

Here is an example of how to use the evaluator:
```java
class Calc {
    private static double eval(String input) {
        CodePointCharStream stream = CharStreams.fromString(input);
        CalcLexer calcLexer = new CalcLexer(stream);
        CommonTokenStream tokenStream = new CommonTokenStream(calcLexer);
        CalcParser calcParser = new CalcParser(tokenStream);
        CalcParser.ExpressionContext expression = calcParser.expression();
        CalcEvaluator calcEvaluator = new CalcEvaluator();
        return expression.accept(calcEvaluator); 
    }
}
```
If there are variables in the expression, their values can be set on the _calcEvaluator_
before passing it to the _accept_ method on the expression. The same expression can be
evaluated multiple times without parsing it again, to calculate it multiple times with
different values for the variables.

## Using lambdas
For the use case of repeated evaluations of the same expression with different variable
values, there is another approach. We can use another visitor class to generate a
lambda function that evaluates the expression. This should be more efficient, as we're
not traversing the whole parse tree every time we want to evaluate the expression.
```java
public class CalcLambdaGenerator extends CalcBaseVisitor<CalcLambda> {
    @Override
    public CalcLambda visitExpression(CalcParser.ExpressionContext ctx) {
        CalcLambda accumulator = ctx.getChild(0).accept(this);
        int childCount = ctx.getChildCount();
        for (int i = 1; i < childCount; i += 2) {
            CalcLambda current = accumulator;
            int op = ((TerminalNode) ctx.getChild(i)).getSymbol().getType();
            CalcLambda nextTerm = ctx.getChild(i + 1).accept(this);
            if(op == CalcParser.PLUS) {
                accumulator = (ValueStore vs) -> current.evaluate(vs) + nextTerm.evaluate(vs);
            } else if(op == CalcParser.MINUS) {
                accumulator = (ValueStore vs) -> current.evaluate(vs) - nextTerm.evaluate(vs);
            }

        }
        return accumulator;
    }

    @Override
    public CalcLambda visitTerm(CalcParser.TermContext ctx) {
        CalcLambda accumulator = ctx.getChild(0).accept(this);
        int childCount = ctx.getChildCount();
        for (int i = 1; i < childCount; i += 2) {
            CalcLambda current = accumulator;
            int op = ((TerminalNode) ctx.getChild(i)).getSymbol().getType();
            CalcLambda nextTerm = ctx.getChild(i + 1).accept(this);
            if(op == CalcParser.TIMES) {
                accumulator = (ValueStore vs) -> current.evaluate(vs) * nextTerm.evaluate(vs);
            } else if(op == CalcParser.DIV) {
                accumulator = (ValueStore vs) -> current.evaluate(vs) / nextTerm.evaluate(vs);
            }

        }
        return accumulator;
    }

    @Override
    public CalcLambda visitParen_expr(CalcParser.Paren_exprContext ctx) {
        return ctx.getChild(1).accept(this);
    }

    @Override
    public CalcLambda visitSigned_factor(CalcParser.Signed_factorContext ctx) {
        int op = ((TerminalNode) ctx.getChild(0)).getSymbol().getType();
        CalcLambda f = ctx.getChild(1).accept(this);
        if(op == CalcParser.MINUS) {
            return (ValueStore vs) -> -f.evaluate(vs);
        }
        return f;
    }

    @Override
    public CalcLambda visitVariable(CalcParser.VariableContext ctx) {
        return (ValueStore vs) -> vs.get(ctx.getText());
    }

    @Override
    public CalcLambda visitNumber(CalcParser.NumberContext ctx) {
        return (ValueStore vs) -> Double.parseDouble(ctx.getText());
    }
}
```
_CalcLambda_ is a simple interface:
```java
public interface CalcLambda {
    double evaluate(ValueStore values);
}
```
The _ValueStore_ class is a simple wrapper for a Map, with _set_/_get_ methods.

Note the similarities to the _CalcEvaluator_ class - the visit methods are basically
the same, except they return a lambda that takes in a ValueStore and return a double.
The _visitExpression_ method uses a similar accumulation approach, except it's building
up a chain of lambda functions, rather than calculating the numeric value as it goes
along.

## Benchmarking
I set up a simple benchmark to compare the performance of the two approaches. The
benchmark evaluates this expression:
```
123.456+x*sin(y+3.0/4+1.2)*0.25*3
```
I set up two arrays - one for x values and one for y values, as well as an array with
the expected results from the expression. The I time two loops, one using the
_CalcEvaluator_ and one using a lambda compiled with a _CalcLambdaGenerator_. I ran
the loops 10 million times.
```java
    private static long timeCalcEvaluator(CalcParser.ExpressionContext expression, double[] xValues, double[] yValues, double[] expectedResults) {
        long startTime = System.currentTimeMillis();
        CalcEvaluator calcEvaluator = new CalcEvaluator();
        for (int i = 0; i < NUM_VALUES; i++) {
            calcEvaluator.set("x", xValues[i]);
            calcEvaluator.set("y", yValues[i]);
            double result = expression.accept(calcEvaluator);
            if(abs(expectedResults[i]-result) > 1e-6) {
                throw new RuntimeException("Bad result");
            }
        }
        long endTime = System.currentTimeMillis();
        return endTime - startTime;
    }

    private static long timeCalcLambdaGenerator(CalcParser.ExpressionContext expression, double[] xValues, double[] yValues, double[] expectedResults) {
        long startTime = System.currentTimeMillis();
        CalcLambdaGenerator lambdaGenerator = new CalcLambdaGenerator();
        CalcLambda calcLambda = expression.accept(lambdaGenerator);
        ValueStore vs = new ValueStore();
        for (int i = 0; i < NUM_VALUES; i++) {
            vs.set("x", xValues[i]);
            vs.set("y", yValues[i]);
            double result = calcLambda.evaluate(vs);
            if(abs(expectedResults[i]-result) > 1e-6) {
                throw new RuntimeException("Bad result");
            }
        }
        long endTime = System.currentTimeMillis();
        return endTime - startTime;
    }
```
The _CalcEvaluator_ came in at **10.5** seconds, whereas the lambda was at **7** seconds.

## Next steps
The lambda approach is certainly faster, but it is still pathetically slow compared to
Java - evaluating the same expression in Java came in at just under half a second.

Next up I want to take a look simplifying the expression before generating the lambda.
The lambda shouldn't be accumulating constant values at run time - that should happen
when compiling it.

I also haven't covered the function calls - my goal is to allow registration of functions
with the CalcLambdaGenerator - currently the _sin_ function is hard-coded as the only
recognized function.