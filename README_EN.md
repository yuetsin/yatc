TL;DR
=====

- **YATC** (Yet Another Tiger Compiler) is a **Tiger** compiler that generates
**.Net framework** managed binaries.

- YATC was made in 2014 as part of an undergrad Computer Science project by
**Damian Valdés Santiago** and **Juan Carlos Pujol Mainegra**.

- **Tiger** is a relatively simple and imperative language that features
nested functions,  records, arrays, integral variables and strings.

- Tiger was defined by **Andrew Appel** in 1998 for *Modern Compiler
Implementation in Java* (Cambridge University Press, 1998)

- This report will explain the architecture and implementation of a Tiger
compiler using **ANTLR 3.3.4** for the grammar definition, **C\# 5.0** in .Net
Framework 4.5 for the construction of the Abstract Syntax Tree (AST) and the
Dynamic Language Runtime (DLR) for the generation of the CIL/MSIL (Common
Intermediary Language/Microsoft Intermediary Language) code.

- **YATC** se presenta *tal cual* y no es mantenido de ninguna forma por sus
autores.

Summary
-------

- **YATC** (Yet Another Tiger Compiler) is a **Tiger** compiler that generates managed binaries for **.Net framework**.
- YATC was carried out in 2014 as part of a Science undergraduate project of Computing by **Damian Valdés Santiago** and **Juan Carlos Pujol Mainegra**.
- **Tiger** is a relatively simple and imperative language with nested functions, records, arrays and integer and string variables.
- Tiger was defined by **Andrew Appel** in 1998 in *Modern Compiler Implementation in Java* (Cambridge University Press, 1998)
- This report will explain the architecture and implementation of a compiler for the Tiger language using **ANTLR 3.3.4** for the definition of the grammar, **C# 5.0** in .Net Framework 4.5 for the construction of *Abstract Syntax Tree* (AST) and *Dynamic Language Runtime* (DLR) for code generation CIL / MSIL (Common Intermediary Language / Microsoft Intermediary Language).
- **YATC** is presented *as is* and is not maintained in any way by its authors.

License
=======
Yet Another Tiger Compiler (YATC)

Copyright (C) 2014 Damian Valdés Santiago, Juan Carlos Pujol Mainegra

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

About the compilation process
===============================

YATC enrolls in the classic process to compile a program, dividing it into four phases in dependence relationship:

-   **Lexical Analysis**: identifies subsets of characters significant separating the input flow into logical structures calls *tokens*. Tokens could be identifiers, words keys, operations, etc. of the language that you want to implement. The software component that performs this operation is called *lexer*. In short, the lexer transforms a string of characters into a token sequence.
-   **Syntactic analysis**: guarantees that the token sequence has a sense according to the specifications of the language, building a derivation tree (DT), which is a representation of the sequences of grammar derivations, to obtain a program in Tiger. The software component that performs this operation is called *parser*. In summary, the parser receives as input a token sequence and issues an AST as a result of the process.
-   **Semantic Check**: verify that in the language have meaningful sentences obtaining an AST. In short, it transforms from a DT to an AST.
-   **Code Generation**: the AST becomes a tree of concrete syntax and code is generated for the instructions of the program.

When you compile a program in Tiger you try to transform it in a sequence of tokens valid for Tiger, otherwise it occurs a syntactic error.

Then, the corresponding AST is constructed according to the hierarchy of implemented classes and the *TreeAdaptor* pattern of ANTLR that connects the previous phase with the current one (semantic check). It verifies that there are no problems and it is passed to the code generation. Otherwise, semantic error is reported and the process of compilation.

Tools for implementation
===================================

In the implementation of YATC the following softwares and languages:

-   ANTLR, ANTLWorks and JRE for the definition of Tiger grammar (understand lexer and parser rules), test, process, and debug for fault detection.
-   .NET C# language in Microsoft Visual Studio 2012 for the hierarchy of nodes of the AST, the realization of the semantic check and the code generation through DLR whose choice will be justified later.
-   PeVerify supplied by the .Net SDK for the correction of assembly.
-   TigerTester 1.2 for testing.

Lexical and syntactic analysis
===================================

Nested Comments
--------------------

To achieve the nesting of the comments, the lexer rules are shown below:

```antlr
fragment BEGIN_COMMENT : '/*';
fragment END_COMMENT : '*/' ;
COMMENTARY : BEGIN_COMMENT ( options {greedy=false;} : COMMENTARY | . )* 
             END_COMMENT {$channel = Hidden;};
```

This rule is misleading: when testing a code with nested comments like the one shown there is no error (which is correct), without however when creating a code with nested and unbalanced comments, no error does not occur either and this is incorrect from the point of view syntactic.

```antlr
    /* Comment1 /* Comment2 */ Comment3 */
```

The default ANTLR behavior when trying to *match* a sequence is to consume it whole. Sometimes (like the one discuss) this is not desired. This purpose is achieved with options `{greedy = false;}`. The *wildcard* ($ \ cdot $) operator representsa ny character and therefore, even if it is placed in the `options rule {greedy = false;}: COMMENTARY | $ \ cdot $ `, ANTLR is not able to differentiate that the `COMMENTARY` should not be included in the recognized by the operator *wildcard*.

After noticing these errors, the rules are redefined, forcing them to consume characters until you find a comment block followed by Any sequence of characters.
    

```antlr
COMMENTARY : BEGIN_COMMENT ( options {greedy=false;} : . )* 
             (COMMENTARY ( options {greedy=false;} : . )* )* 
             END_COMMENT {$channel = Hidden;}
```

Los programa vacíos también son programas
-----------------------------------------

Un programa en Tiger es una expresión. Un programa que no tenga código
escrito también debe reconocerse y no debe presentar ningún tipo de
error. Esto se logra permitiendo que la regla principal, que produce una
expresión, permita que esta no esté a través del operador el operador
$?$.


```antlr
public a_program : expr ? EOF;
```

`a = b = c` es ilegal
---------------------

Sea la siguiente regla del parser:

```antlr
relational_expr : arith_expr ((EQ | NOTEQ | GT | LT | GTEQ | LTEQ ) arith_expr)?; 
```

El operador `?` al final de la regla le da la no asociatividad requerida
a los operadores de comparación (notar que en `arith_expr` y
`term_expr` se usa `*` para anidar estas expresiones en una lista). Con
esta regla no se permite `a = b = c`, pero si `a = (b = c)` como está
especificado.

Ayudando al chequeo semántico del for
-------------------------------------

Según las especificaciones del lenguaje Tiger, en un for solo se pueden
evaluar condiciones que retornen valor, por ello se exige en la regla
correspondiente que la primera y segunda expresiones sean
`disjunction_expr`.

```antlr
for_expr : FORKEY ID ASSIGN disjunction_expr TOKEY disjunction_expr DOKEY expr;
```

Reescritura de reglas
---------------------

La mayoría de las reglas del parser se reescriben creando nodos
“ficticios” en la sección tokens que se usarán para generar nodos
homólogos del AST a través del TreeAdaptor. Un ejemplo representativo es
la creación de un token para el operador de negación (menos unario).
Esto permite además eliminar del DT del programa, tokens innecesarios
para la próxima fase, como los paréntesis, las palabras claves, las
comas, los punto y coma, etc.

Análisis semántico
==================

Jerarquía de nodos del AST
--------------------------

Para que el TreeAdaptor funcione correctamente, se crean las clases
correspondientes a los nodos que desean comprobarse semánticamente. Cada
nodo tiene, además de sus propiedades estructurales, dos métodos

```csharp
public abstract void CheckSemantics(TigerScope scope, Report report);
internal abstract void GenerateCode(ModuleBuilder moduleBuilder);
```

El primero (fundamental en esta fase) chequea la semántica del nodo en
cuestión, siguiendo un patrón de gramática atributada: un nodo está bien
si sus hijos lo están y la información brindada por sus hijos o hermanos
en la jerarquía no produce error en su funcionamiento. Estas
informaciones se modelan a través de las clases herederas de
`TigerInfo`, que dejan información conectada para la generación de
código, p.e *bindings* de variables, información relativa a la funciones
(nombre, tipo y cantidad de parámetros), etc.

Notar que el nodo `UnknownNode` se usa para homologar tokens que, si
bien no introducen error en el chequeo semántico, ayudan en la
comprensión y organización del AST. Ejemplos de token que se mapean como
UnknownNode son `FIELDS_INST`, `PARAM_DECL`, `FUN_TYPE_WRAPPER`,
`DECL_BLOCK` y `ARGS_FIELDS`.

Scope
-----

Al usar una variable, un `type` o una función, ¿cómo se sabe si fueron
declaradas antes? Esta información se obtiene de la clase `TigerScope`
que almacena nombres, con sus respectivos objetos, al que se refieren
(variables, *types* o funciones) como una tabla de símbolos. Además,
esta clase permite, mediante un árbol de scopes, anidar funciones y
expresiones `let`, de forma que solo pueda tenerse acceso a las
variables o funciones que están declaradas en ese ámbito o en el ámbito
del nodo padre. Esta clase además se encarga del binding de las
variables. Esta estructura permite que, en caso de que en un scope
interno se cree un `type` o variable de igual nombre que un `type` o
variable de un scope ancestro del actual, entonces se reescriba el valor
antiguo del `type` o variable por el nuevo y se lanzará una advertencia.

Nil
---

En el chequeo semántico de la expresión `if cond_expr then then_expr
else else_expr` se realizan las siguientes comprobaciones. Se realiza
el chequeo semántico de `cond_expr` y luego el de `then_expr`. Si
ambas expresiones están correctas se comprueba que la `cond_expr` tenga
algún valor (no sea `void`) y luego se comprueba que este valor sea
entero. De lo contrario se lanzan errores. En el caso que exista parte
`else` el chequeo es más arduo. Primero se chequea que el nodo
`else_expr` esté correcto. Luego se verifica que el tipo de
`else_expr` sea asignable, o sea, `int`, `string`, `record`, `array`.
De lo contrario, se reporta error. En este punto es donde se chequean
los casos críticos del `nil` expuestos en las especificaciones de Tiger.

Secuencia de declaraciones de tipo
----------------------------------

La construcción y verificación de las secuencias de declaraciones de
tipos dentro de un `let` se realiza mediante tres pasadas a cada uno de
los nodos hijos de cada una de ellas, dígase a los `TypeDeclNode` que
están contenidos dentro de cada `TypeDeclSeqNode`, encargado de agrupar
las mismas. Esto se consigue mediante la regla del parser

```antlr
decl : type_decl+ | var_decl | fun_decl+;
```

Esta regla está contenida dentro de un iterador + **EBNF** (*Extended
Backus-Naur Form*), leyéndose finalmente

```antlr
let_expr : LETKEY decl+ INKEY expr_seq ENDKEY;
```

Aunque esta última pareja constituya una ambigüedad, se usa a propósito
para realizar dicha agrupación, de forma análoga a la selección de a qué
`if` pertenece un determinado `else`, esto es, al de mayor anidamiento.
Esto facilita sobremanera cualquier operación posterior que se realice
sobre los conjuntos de declaraciones, ya sean de tipos o de funciones.
Las variables se tratan como conjuntos *singleton*.

Dada una declaración de tipo

```antlr
type_decl : TYPEKEY ID EQ type; 
```

se le dice parte izquierda o cabeza al token ID, que corresponde al
nombre del tipo que se quiere declarar. La parte derecha corresponde a
la regla `type`, que puede ser `type_id`, `array_decl` o
`record_decl`.

Los pasos son los siguientes:

1. Se agregan las cabezas de los tipos, es decir, la parte izquierda de
la declaración de cada uno de ellos, con el sólo propósito de comprobar
que no exista una redeclaración dentro de una secuencia, o de forma
local y global, agregándolo a una estructura que contiene cada
`TypeDeclNode` como llave del nombre del tipo. Este paso incluye
detectar renombramientos de los nombres de los tipos primitivos como
tipos de usuario, esto es, no cambiar a `int` o `string`, por ejemplo.
En este paso se llama al `CheckHeader(TigerScope scope, Report report,
string name)` de cada hijo.

2. Para detectar eventuales ciclos es necesario construir un grafo
dirigido $G \langle V,E \rangle$ con

    $$V = \{ \text{nodos de declaración de tipos} \}$$
  
    $$E = \{ (u,v) | ParteIzquierda(u) \and BaseType(u, v) \in { Alias V Array } \and ParteDerecha(v),  u, v \in V \}$$
  
    Dicha construcción asegura que las aristas se establezcan sólo entre
    nodos de declaraciones de tipos que estén conectados como parte
    izquierda y derecha del *statement*, que sea de base `Array` o `Alias`, es
    decir, un tipo cuyo tipo, valga la redundancia, sea alguno de los
    anteriores. Estos se agregan a una estructura
    `Dictionary<TypeDeclNode, Node<TypeDeclNode>>`
    donde `Node<T>` es una representación de nodos en un
    grafo, con soporte para realizar operaciones de orden topológico además
    de detectar ciclos y su composición. Esta estructura permite conectar
    los `TypeDeclNode` en el grafo con aquellos otros de la secuencia que se
    requiera, y evitar repetir nodos en el grafo que correspondan a una
    misma declaración. Esto podría suceder dado que un mismo tipo puede
    estar varias veces en parte derecha, y a lo sumo una vez en parte
    izquierda si se ha llegado a este paso (el anterior elimina que esté más
    de una vez). De esta forma no se tenderá una arista entre un tipo
    `aliasInt` que sea alias de `int`, dado que la parte derecha no estará
    en primer lugar en la secuencia, y por tanto no puede referenciar hacia
    adelante, de forma que no hay peligro de recursión cíclica.

3. Al grafo construido se le realiza un DFS (*Depth-First Search*) con
marcado de nodos, en los que se detecta una arista back-edge, o de
retroceso, si se en la navegación se llega a un nodo marcado como gris
\[ver *Intro To Algorithms*, Cormen et al., alrededor de la 614\]. La prueba
del Teorema 22.12 acompaña el resultado. Si existe un ciclo se detecta
sus nodos componentes y se imprime. De no existir, se recorre el orden
topológico de los nodos resultantes, cuyo valor contenido son las
declaraciones de nodos, y se llama a `CheckSemantics` de cada uno en
dicho orden. La generación de código también se realiza por dicho orden.
Esto evita tener que realizar múltiples búsquedas si para construir
finalmente un tipo, aunque se conozca que está correcto.

#### 

El último paso, detectar un ciclo, y después navegar por el orden
topológico dado, devuelto en forma de lista enlazada, es vital para la
realización de las operaciones de construcción de tipos, que se realizan
de forma transparente para los hijos, los cuales no requieren búsquedas
adicionales para la determinación completa de los mismos, como se había
dicho. Para ellos se habilita que los nodos que así lo requieran no
contengan una referencia directa al tipo, dado que en el momento de
declaración y comprobación semántica, aunque este sea correcto, puede no
estar determinado, sencillamente porque es el tipo de un record. Uno de
estos casos se analiza a continuación.

#### 

Supongamos que deseamos comprobar y construir los tipos del siguiente
programa:

```antlr
let
    type a = b
    type b = { x: c }
    type c = a
in
end
```

El programa está correcto, dado que los tipos `a`, `b` y `c` están bien
determinados por un ciclo que pasa por al menos un tipo record (`b`). El
tipo básico de `a` es el mismo que el de `c`, que es alias. El orden
topológico podrá arrojar una sola forma de navegarlo para la
construcción. Sea esta $b \rightarrow a \rightarrow c$. Al comprobar
semánticamente a `b` el tipo no está bien determinado, dado que tiene un
campo `x` de un tipo desconocido `c`, pero se conoce que la declaración
esta correcta, dado que `c` existe y está correcto, por lo que `b` está
bien determinado. Su definición, no obstante, no es completa. Por lo
tanto se almacena una referencia a la referencia a la definición del
mismo, de forma que esta pueda se alterada, es decir se puede completar,
y todos los nodos que dependían de esta queden bien.

#### 

Finalmente los tipos alias no se analiza semánticamente, es decir, son
indistinguibles las definiciones originales de `a`, `b` y `c`, a efectos
del resto del programa.

Generación de código
====================

La generación de código es el proceso mediante el cual un compilador
convierte la información proveniente de las fases anteriores en código
que puede ser ejecutado por una computadora. La ejecución del código
generado la realiza, en este compilador, el *Common Language Runtime*
(CLR) de la plataforma .NET. El CLR usa una estructura basada en un
modelo de pila para modelar la ejecución de código, así como un lenguaje
ensamblador propio llamado *Intermediate Language* (IL). Nuestro
compilador utiliza una componente de la biblioteca de clase de la
plataforma .NET llamada `System.Reflection.Emit` que permite crear un
ejecutable en .NET y agregarle código según se desee; y para generar
código IL se usa *Dynamic Language Runtime* (DLR).

¿Por qué usar DLR?
------------------

El DLR proporciona servicios para implementar lenguajes dinámicos sobre
el CLR. DLR facilita el desarrollo de lenguajes para ejecutar programas
sobre .NET Framework. Los lenguajes por el tipo del objeto que los
representa en tiempo de ejecución, que se especifica en tiempo de diseño
en un lenguaje estáticamnete tipado como C\#.

DLR permite a los programadores implementar lexers, parsers,
analizadores semánticos y generadores de código y la generación de
código se realiza de forma declarativa mediante árboles de expresiones
lo que hace más fácil esta etapa.

Funciones recursivas
--------------------

En la generación de código de las secuencias de funciones se realizan
tres etapas.

1.  Se recorren todos las definiciones (`headers`) de las funciones en
    la secuencia y se introducen en una lista de `ParameterExpression`
    que después se le pasará a la segunda fase.

2.  Por cada llamado a la propia función (recursión) o a otra, se crea
    una `Expression` ficticia en el que luego se sustituirá el código
    del llamado a la función.

3.  Se recorren las declaraciones de funciones en la secuencia y se le
    asigna a la `Expression` ficticia el código del llamado a la función
    correspondiente.

Para más información
====================

-   Edwards, Stephen - *Tiger Language Reference Manual*, Columbia
    University.

-   Appel, Andrew and Ginsburg, Maia - *Modern Compiler Implementation
    in C*, Press Syndicate of the University of Cambridge, 1998.

-   Parr, Terrence - *The Definitive ANTLR Reference*, The Pragmatic
    Programmers, May 2007.

-   Parr, Terrence - *The Definitive ANTLR 4 Reference*, The Pragmatic
    Programmers, Jan 2013.

-   Parr, Terrence - *Language Implementation Patterns*, The Pragmatic
    Programmers, 2010.

-   Wu, Chaur - *Pro DLR in .NET 4*, Apress, 2010.

-   Chiles, Bill - *Expression Trees v2 Spec* (DRAFT).

-   Cormen, Thomas *et al.* - *Introduction to algorithms*, 3rd ed,
    Massachusetts Institute of Technology, 2009.

-   [https://docs.microsoft.com/](https://docs.microsoft.com/ "MSDN")

