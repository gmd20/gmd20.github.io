freeDiameter的主配置文件还有那些extension的配置文件，都是用flex 和bison来做解析的。 
估计那个作者很熟悉flex和bison吧，当然通过flex 和bision也可以做到很复杂的规则吧。
不过有杀鸡用牛刀的感觉。
      下面参考freeDiameter的extension里面的用法，写个简单的测试程序，实现字符串还有整数的解析。还有数组的元素的解析。也是熟悉一下flex和 bison的写法吧，很早之前，看thrift的代码的时候，就想试试这两个东西了。

Lex and YACC primer/HOWTO  http://www.tldp.org/HOWTO/Lex-YACC-HOWTO.html
上面一篇的中文翻译   http://www.cnblogs.com/itech/archive/2012/03/04/2375746.html
flex和 bison的官方文档
http://flex.sourceforge.net/manual/
http://www.gnu.org/software/bison/manual/html_node/index.html
一本flex & bison的图书      http://www.cin.ufpe.br/~wdbm/flex%20e%20bison.pdf
网上有人翻译的flex bison LLVM实现自定义的语言的例子。
http://coolshell.cn/articles/1547.html

下面是相关的代码：

目标是让自己的程序解析和加载下面这个配置文件 ： app_conf.txt 的内容
#log enabled
log_enabled  = true;

# l;og_level 
LOG_LEVEL = 2 ;

#aop name
app_name = "my application name ";

#数组的输入
a = [1, -2, 3,4, 5, 6,  7, 8, 90];

把它加载到一个c++的结构里面， 这个结构在 app_conf.h 文件里面定义，内容如下
#include <iostream>
#include <vector>
#include <string>

struct app_conf
{
    bool                log_enabled;
    int                 log_level;
    std::string         app_name;
    std::vector<int>    array;
    void init() {
        log_enabled  = false;
        log_level = 0;
        app_name.clear();
        array.clear();
    }
}; 

extern struct app_conf my_app_conf;



测试程序是这样写的： main.cpp 文件内容
#include "app_conf.h"
struct app_conf my_app_conf;

extern int load_conf(char * conffile);

int main ()
{
   
    if (0 == load_conf("app_conf.txt")) {
        std::cout << "log_enabled = " <<  my_app_conf.log_enabled << std::endl;
        std::cout << "log_level = " <<  my_app_conf.log_level<< std::endl;
        std::cout << "app_name = " <<  my_app_conf.app_name<< std::endl;

        std::cout << "array = ";
        for (std::vector<int>::iterator it = my_app_conf.array.begin() ; it != my_app_conf.array.end(); ++it) {
            std::cout << ' ' << *it;
        }
        std::cout << '\n';
    }
    return 0;
}



最后是实现这个功能的 flex和bison的脚本

flex的词法解析器 app_conf.l 文件内容如下：
flex编译这个 *.l 文件之后，其实就是生成一个yylex函数供bison调用。 yylex调用一次返回 bison里面定义的token的类型，比如下面的QSTRING类型，然后相关的值会保存bison里面定义的 %union结构里面。yylval->string 就是%union的一个成员。不过有的token类型是没有返回值的，还有就是忽略空格和换行的。 已经最后的错误报告错误的匹配。 其实这个文件的结构就是一个正则表达式跟这一段处理代码。 最主要的就是明白变量yytext 表示当前匹配的内容，yyleng 是当前匹配字符的长度。 其他的正则表达式的写法可以采纳靠flex的文档。 最前面的 %{ %} 里面的是直接插入到最后生成的代码里面去的。 注意各个匹配的顺序就可以了，flex应该是写在前面的匹配优先级比较高的。
%{
#include <iostream>
#include <string>
#include <vector>

// bison  不会把 c++ 的头文件加到下面这个头文件里面去，
// 所以包含这个头文件直接，只要要手工在前面加上c++的头文件

/* bision生成的头文件包含bison文件定义的token 常量， Include yacc tokens definitions */
#include "app_conf_parse.hpp"

#include <stdio.h>

/* Update the column information */
#define YY_USER_ACTION { 						\
	yylloc->first_column = yylloc->last_column + 1; 		\
	yylloc->last_column = yylloc->first_column + yyleng - 1;	\
}

%}

    /* flex的选项设置   yywrap函数在默认yyin输入文件读到结尾之后调用，用于加载多个yyin文件。一般都不会用这个
     所以下面noyywrap禁用了这个功能了。 --bison-bridge 告诉flex生成和bison兼容的api接口 所有参数说明都可以在
     这里找到
      http://flex.sourceforge.net/manual/Index-of-Scanner-Options.html#Index-of-Scanner-Options
     */
%option bison-bridge bison-locations
%option noyywrap
%option nounput


%%

    /* 记录行数  */
\n {
       yylloc->first_line++; 
       yylloc->last_line++; 
       yylloc->last_column=0; 
   }

([[:space:]]{-}[\n])+ { /* 忽略所有的空格 如果这里使用[ \t]的话，windows平台的文件 回车\r会导致匹配失败*/ }
	/* Eat all spaces but not new lines  忽略空格的另一种写法 ([[:space:]]{-}[\n])+	;
       {-}  表示所有在前面的集合 [[:space:]] 里面又不在后面集合 [\n] 里面的字符
       {+}  类似，表示集合的并集的意思，在任何前面后面的集合里面出现 
       参见 http://flex.sourceforge.net/manual/Patterns.html#Patterns
   */

    /*忽略所有的注释*/
#.*$			;

    /*匹配数字的另一种写法 [-]?[[:digit:]]+	
      参考内置的class 的定义 http://flex.sourceforge.net/manual/Patterns.html#Patterns
     */
[-]?[0-9]+ {
               int ret=0;
               ret = sscanf(yytext, "%i", &yylval->integer);
               if (ret != 1) {return LEX_ERROR;}
               return INTEGER;
           }

	/* Recognize quoted strings -- we do not support escaped \" in the string currently. */
\"[^\"]+\" {
                /*yytext 是匹配的字符串开始的指针，yyleng是长度 */
                yylval->string = new std::string (yytext+1, yyleng - 2);
                return QSTRING;
           }

    /* ?i 忽略大小写的写法
       参考 http://flex.sourceforge.net/manual/Patterns.html#Patterns 的说明
     */
(?i:"app_name") {
                    return APP_NAME;
               }

(?i:"log_enabled") {
                        return LOG_ENABLED;
                   }
true|false {
                yylval->integer = strcmp(yytext, "true"); return  LOG_STATE;
           }


(?i:"log_level") { return LOG_LEVEL; }


[a-zA-Z][a-zA-Z0-9]* {
                        yylval->string = new std::string (yytext, yyleng);
                        return STRING;
                     }


    /* 认为是合法的单个字符 */
[=;,\[\]] {return yytext[0];}  

    /* 除了上面列出的匹配模式之外，其他的都当作错误返回
     * 如果是自己的和自己的程序集成的话，这里也可以使用自己的log函数打印错误，
     * LEX_ERROR 为自定义表示错误的token，yyerror 可以重写的吧
     */
. { 
     printf("不能识别的模式：在 %d 行 %d 列， %c\n", yylloc->first_line, yylloc->first_column, *yytext);
     return LEX_ERROR;
   }

%%



bison的文件app_conf.y  内容如下：
这里面 %union里面包含所有的节点的类型要保存的值，flex里面就是把值保存到这里面去的。注意的是 那里不要放c++对象，而是放c++对象的指针，应该直接放对象是不支持还是支持的不好？ 反正文档和图书里面都是说的指针。比如自定义一个类表示对应的抽象语法树上节点，就把那个指针放进去。 然后 返回赋值时 用new 新建对象，复制个 $$ . 至于内存的管理，什么时候释放，应该还是需要自己管理的。自己记录有哪些指针是动态分配出来的。不然应该有内存泄漏。不过一般解析一次，这个抽象语法树的节点也就创建一次吧。看书上例子也没有写相应的管理的代码。
        还有就是如果%union里面包含c++ 类的的指针话， bison生成的代码头文件里面app_conf_parse.hpp，并没有包含相应的头文件定义。 所以别的文件引用这个生成的头文件的话，自己要把其他依赖的头文件比如 # include<string> 这些加到 #include "app_conf_parse.hpp" 的前面去。保证相关的c++的类型有定义。

%token 定义了 flex返回token类型和 %union中对应的关系。
%type 定义了 语法解析结构对应的 抽象语法树的节点的返回的类型。
剩下的都是语法的定义，一个节点对应的一段action 处理的代码。在那里对应的创建 节点类型和赋值之类的就可以了。

/* For development only : */
%debug 
%error-verbose

/* The parser receives the configuration file filename as parameter */
/* 这个设置之后，yyerror等函数就会多一个参数 */
%parse-param {char * conffile}

/* Keep track of location */
%locations 
%pure-parser


%{
#include "app_conf.h"    //自定义的app_conf 结构
#include <errno.h>
#include <stdio.h>

#include <iostream>
#include <vector>
#include <string>
// bison  不会把 c++ 的头文件加到下面这个头文件里面去，
// 所以包含这个头文件直接，只要要手工在前面加上c++的头文件
#include "app_conf_parse.hpp"

/* bison的默认解析函数， Forward declaration */
int yyparse(char * conffile);

/* 提供给其他模块使用的加载设置的函数 */
int load_conf(char * conffile)
{
    // flex默认会从这个yyin文件指针读取输入
    // 如果没有修改flex生成的前缀，应该是这个名字 
    // 不过可以通过给flex指定参数，修改yy为自己想要的前缀名字
    // 除了这个yyin的用法，flex的api也有直接从内存数组里面加载要解析的内容的。
    extern FILE * yyin;
    int ret;

    yyin = fopen(conffile, "r");
	if (yyin == NULL) {   
        ret = errno;
        return ret;
    }

    my_app_conf.init(); 
    ret = yyparse(conffile);
	fclose(yyin);

    return ret;
}


/* The Lex parser prototype */
int yylex(YYSTYPE *lvalp, YYLTYPE *llocp);

/* Function to report the errors */
void yyerror (YYLTYPE *ploc, char * conffile, char const *s)
{
	
	if (ploc->first_line != ploc->last_line)
		printf("%s:%d.%d-%d.%d : %s\n", conffile, ploc->first_line, ploc->first_column, ploc->last_line, ploc->last_column, s);
	else if (ploc->first_column != ploc->last_column)
		printf("%s:%d.%d-%d : %s\n", conffile, ploc->first_line, ploc->first_column, ploc->last_column, s);
	else
		printf("%s:%d.%d : %s\n", conffile, ploc->first_line, ploc->first_column, s);
}


%}


/* Values returned by lex for token */
%union {
    int		      integer;  /* 保存整型数 */
    std::string       *string;    /* 保存字符串 */
    std::vector<int>  *array_data;  /*保存数组的内容*/
}



/* In case of error in the lexical analysis */
%token 		LEX_ERROR

/* Key words */
%token 		LOG_ENABLED
%token 		LOG_LEVEL
%token 		APP_NAME
            

/* 有返回值的，类型相关的定义 */
%token <string>	QSTRING   
%token <string>	STRING   
%token <integer> INTEGER
%token <integer> LOG_STATE 

/* 声明语法树节点的类型，对应 %unino 里面的哪一种类型 */
%type  <array_data>  array_items;


/* ----------------------- 开始语法定义--------------------------------- */

%%

conffile:		/* empty grammar is OK */
			| conffile log_enabled
			| conffile log_level
			| conffile app_name
			| conffile array
			;

log_enabled: LOG_ENABLED '=' LOG_STATE ';'
            {
                 my_app_conf.log_enabled =  (bool) $3;    
            }
            ;

log_level: LOG_LEVEL '=' INTEGER ';'
            {
                 my_app_conf.log_level = $3;    
            }
            ;

app_name: APP_NAME '=' QSTRING ';'
            {
                 my_app_conf.app_name = *$3;    
            }
            ;

array: STRING '=' '[' array_items ']'';'
            {
                 std::cout << "array name = " << *$1 << std::endl;    
                 my_app_conf.array = * $4;
            }
            ;

array_items: INTEGER
             {
                         //  $$表示当前冒号左边的节点， $1 $2 ... 表示 冒号右边的第几个节点。
                 $$ = new std::vector<int> ();
                 $$->push_back($1);     
             }
           | array_items ',' INTEGER
               {
                   $$ = new std::vector<int> ();
                   * $$ = * $1;
                   $$->push_back($3); 
               }
           ;



代码的编译和执行结果如下
[root@LINUX-01 bright]# bison -d -o app_conf_parse.cpp app_conf.y
[root@LINUX-01 bright]# ls
app_conf.l    app_conf.y          app_conf_parse.hpp
app_conf.txt  app_conf_parse.cpp  main.cpp

[root@LINUX-01 bright]# flex -o app_conf_lex.cpp app_conf.l
[root@LINUX-01 bright]# ls  
app_conf.l    app_conf.y        app_conf_parse.cpp  main.cpp
app_conf.txt  app_conf_lex.cpp  app_conf_parse.hpp

g++ -g -o my_app app_conf_parse.cpp  app_conf_lex.cpp main.cpp 

[root@LINUX-01 bright]# ./my_app 
array name = a
log_enabled = 0
log_level = 2
app_name = my application name 
array =  1 -2 3 4 5 6 7 8 90

可以看到成功解析和加载了。这个只是简单的例子，flex和bison可以用来搞非常复杂的结构的解析吧。 如果做的好的话，应该比那些xml配置文件之类的要灵活一些。 freediuameter就是这样使用的。
