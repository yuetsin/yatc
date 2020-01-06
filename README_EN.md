YATC
=====

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

More Information
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

