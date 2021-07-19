---
title: Lex & Yacc 学习笔记
author: fangfeng
date: 2019-04-25
categories:
  - 技术
tags:
  - Lex
  - Yacc
---

高级语言相较于机器语言、汇编语言，更加符合人的思考习惯。换句话说，更偏向于自然语言的风格而更偏离指令化的描述。用高级语言编写的一行代码，最终可能需要处理器执行若干条指令。如何让机器意识到高级语言代码对应的机器指令是哪些呢？当然就需要一个优秀的翻译。

无论是编译型语言还是解释型语言，总逃脱不了这样一个流程：高级语言 ➜ 目标平台的指令。所谓编译型/解释型的区别，在于其转换流程是在线的(online)还是离线的(offline)。在线的方式无法意识到后续的代码，但胜在即时反应；离线的方式可以统揽全局，进行更多的优化，但代码文本必须完整。

高级语言 ➜ 目标平台的指令，这样的流程如何实现。一般来说，划分为四个阶段：词法分析、语法分析、语义分析、目标代码生成。

![](https://ws2.sinaimg.cn/large/006tNc79ly1g2b583io0ej30wq094aa5.jpg)

本篇的主要目的，是展示“语言翻译”的几个阶段工作，以及通过 Lex & Yacc 工具演示一门自定义语言的“翻译”。

<!--more-->

## 词法分析 <- Lex 

词法分析的核心就是识别源代码，并将它按照特定的规则划分成一系列的标记(Token)。

比如 `int value = 12 + 23 * 323` 以 C 语言的划分方式可以分成 `int`, `value`, `=`, `12`, `+`, `23`, `*`, `323` 这些标记。

如何将源代码划分为标记呢？再不济不过是逐一扫描每种标记罢了。但是，规则是变化的，其它实现字符匹配的代码都是一致的。有没有一套框架，用户提供规则表就能实现词法分析呢？当然有，Lex 以及 Flex 就是这方面的好手。

![](https://ws3.sinaimg.cn/large/006tNc79ly1g2boqbqx3oj31pi0q0q44.jpg)

**Lex 语法规范**

整个 `<name>.lex` 规则文件分为三部分，由 `%%` 进行分隔

```
definitions
%%
rules
%%
user code
```

`definitions`，定义区的所有声明可以类比 C 语言中的宏，只不过无需 `#define`。

例如 

```
DIGIT [0-9]+
NZDIGIT [1-9]{DIGIT}
%%
{NZDIGIT}   {   printf("nzdigit = %s\n", yytext);   }
{DIGIT}     {   printf("digit = %s\n", yytext); }
%%
```

等效于

```
%%
[1-9][0-9]+ {   printf("nzdigit = %s\n", yytext);   }
[0-9]+      {   printf("digit = %s\n", yytext); }
%%
```

`rule`，规则区就是定义标记识别规则的核心块。lex 将按最长匹配原则确定最终匹配上的标记(Token)，如果都没匹配上，则按原样输出。如果多个匹配上，则按最先声明的规则为准。

`user code` 用户代码区的所有代码将原样拷贝到生成的 `<name>.c` 文件中。

## 语法分析 <- Yacc

仅仅是词法分析还做不了太多的事情。之后的工作都将交给语法分析器来完成。Yacc、Bison 就是两个优秀的语法分析器。当然，它们也有着一些局限（由于只是学习使用，未对其中原理展开深入了解）。Yacc / Bison 只能解决能被 BNF(Backus-Naur Form, 巴科斯范式) 描述的语法规则，而且只支持符合 LALR(1) 规则的。

不过，这也已经足够了。在 [RFC](https://www.rfc-editor.org/) 上看到大量的规范性文档都通过 BNF 描述其语法规则，通过 Yacc/Bison 其实也能解决对这些规则的解析。

**Bison 语法规范**

```
%{
C declarations
%}

Bison declarations

%%
Grammar rules
%%
Additional C code
```

- C 声明区将定义后面需要用到的类型和变量。当然，在这个区域使用 C 语言宏也是被允许的。比如 `#define`, `#include` 等

- Bison 声明区被用来定义`终止符号(terminal symbols)`和`非终止符号(nonterminal symbols)`，当然也可以用来定义符号操作优先级和`语义值`的数据类型。

- 语法规则区将`非终止符号`是如何被组成的（在这个区域声明 BNF）

- C 代码区与 lex 的规范一致，该区所有代码将被原样拷贝到生成的解析器源文件中。 

## 示例

简单计算器语言的实现（改编自 [mfcalc](https://dinosaur.compilertools.net/bison/bison_5.html#SEC29)）：
1. 支持简单四则运算
2. 支持乘方、平方根
3. 支持三角函数
4. 支持变量声明

```sh
$ ./calc
3*10+2*(3-1)
	34
alpha=beta=5*3+4
	19
sin(PI/2)
	1
cos(PI)
	-1
ln(alpha)
	2.944438979
beta=ln(alpha)
	2.944438979
sqrt(25)
	5
```



```plain
%{
#include <math.h>
#include <stdio.h>
#include <ctype.h>
#include "calc.h"
%}
%union {
double val;
symrec *tptr;
}

%token <val> NUM
%token <tptr> VAR FNCT
%type <val> expr

%right '='
%left '+' '-'
%left '*' '/'
%left NEG
%right '^'

%%
input:
        | input line
;

line: '\n'
        | expr '\n' {   printf("\t%.10g\n", $1);    }
;

expr: NUM   { $$ = $1;  }
        | VAR   { $$ = $1->value.var;   }
        | VAR '=' expr  { $$ = $3;  $1->value.var = $3; }
        | FNCT '(' expr ')' { $$ = (*($1->value.fnctptr))($3);  }
        | expr '+' expr { $$ = $1 + $3; }
        | expr '-' expr { $$ = $1 - $3; }
        | expr '*' expr { $$ = $1 * $3; }
        | expr '/' expr { $$ = $1 / $3; }
        | expr '^' expr { $$ = pow($1, $3); }
        | '-' expr %prec NEG    { $$ = -$2; }
        | '(' expr ')'  { $$ = $2;  }
%%

int main(int argc, char **argv)
{
    init_table();
    yyparse();
    return 0;
}

int yyerror (char *s)
{
    printf("%s\n", s);
}

struct init_fnct
{
    char *fname;
    double (*fnct)();
};

struct init_fnct arith_fncts[] = {
    "sin", sin,
    "cos", cos,
    "atan", atan,
    "ln", log,
    "exp", exp,
    "sqrt", sqrt,
    "floor", floor,
    "ceil", ceil,
    "abs", fabs,
    0, 0
};

struct init_var {
    char *vname;
    double value;
};

struct init_var constant_var[] = {
    "PI", M_PI,
    "E", M_E,
    0, 0
};

symrec *sym_table = (symrec *) 0;

int init_table()
{
    int i;
    symrec *ptr;
    for (int i=0;constant_var[i].vname != 0;i++)
    {
        ptr = putsym(constant_var[i].vname, VAR);
        ptr->value.var = constant_var[i].value; 
    }
    for (int i=0;arith_fncts[i].fname != 0;i++)
    {
        ptr = putsym(arith_fncts[i].fname, FNCT);
        ptr->value.fnctptr = arith_fncts[i].fnct;
    }
}

symrec * putsym(char *sym_name, int sym_type)
{
    symrec *ptr;
    ptr = (symrec *) malloc(sizeof(symrec));
    ptr->name = (char *) malloc(strlen(sym_name) + 1);
    strcpy(ptr->name, sym_name);
    ptr->type = sym_type;
    ptr->value.var = 0;
    ptr->next = (struct symrec *) sym_table;
    sym_table = ptr;
    return ptr;
}

symrec * getsym(char *sym_name)
{
    symrec *ptr;
    for (ptr = sym_table; ptr != (symrec *) 0;ptr = (symrec *) ptr->next)
        if (strcmp(ptr->name, sym_name) == 0)
            return ptr;
    return 0;
}

int yylex()
{
    int c;
    while ((c = getchar()) == ' ' || c == '\t');

    if (c == EOF)
        return 0;
    
    if (c == '.' || isdigit(c))
    {
        ungetc(c, stdin);
        scanf("%lf", &yylval.val);
        return NUM;
    }

    if (isalpha(c))
    {
        symrec *s;
        static char *symbuf = 0;
        static int length = 0;
        int i;
        if (length == 0)
            length = 40, symbuf = (char *) malloc(length + 1);
        i = 0;
        do {
            if (i == length)
            {
                length *= 2;
                symbuf = realloc(symbuf, length + 1);
            }
            symbuf[i++] = c;
            c = getchar();
        } while (c != EOF && isalnum(c));

        ungetc(c, stdin);
        symbuf[i] = '\0';
        s = getsym(symbuf);
        if (s == 0)
            s = putsym(symbuf, VAR);
        yylval.tptr = s;
        return s->type;
    }

    return c;
}
```

```c
/* Data type for links in the chain of symbols.      */
struct symrec
{
    char *name;  /* name of symbol                     */
    int type;    /* type of symbol: either VAR or FNCT */
    union {
        double var;           /* value of a VAR          */
        double (*fnctptr)();  /* value of a FNCT         */
    } value;
    struct symrec *next;    /* link field              */
};

typedef struct symrec symrec;

/* The symbol table: a chain of `struct symrec'.     */
extern symrec *sym_table;

symrec *putsym ();
symrec *getsym ();

```

```make
calc: calc.y calc.h
	bison -o calc.c calc.y 
	gcc -o calc calc.c -w
```
```plain
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```
