c4 - C in four functions
========================

An exercise in minimalism.

Try the following:

    gcc -o c4 c4.c
    ./c4 hello.c
    ./c4 -s hello.c
    
    ./c4 c4.c hello.c
    ./c4 c4.c c4.c hello.c

```C
#include <stdio.h>

int token;            // 当前词法单元
int token_val;        // 当前词法单元的值
int *code, *pc, *sp;  // 目标代码、程序计数器、堆栈指针

int *sym[256];        // 符号表，用于记录标识符的信息
int sym_size;         // 符号表大小

int base, loc;        // 栈帧基址和局部变量地址

// 词法分析：获取下一个词法单元
int next() {
    return token = getchar();
}

// 符号表处理：在符号表中查找标识符，如果不存在则添加
int *sym_lookup(int id) {
    for (int i = sym_size - 1; i >= 0; i--)
        if (sym[i][0] == id) return sym[i];
    sym[sym_size] = calloc(3, 4);  // 类型、类别、值
    sym[sym_size][0] = id;
    return sym[sym_size++];
}

// 语法分析：解析表达式
int expr(int lev) {
    int *entry, id, op;

    while (1) {
        if (token == ';') { // 分号表示语句结束
            next();
            return 0;
        }

        if (token == '+' || token == '-') {  // 加减法
            op = token;
            next();
            entry = sym_lookup(token_val);   // 查找标识符
            if (entry[1] != Loc) entry = sym_lookup(1);  // 如果不是局部变量，使用全局变量
            *++pc = op;
            *++pc = entry - base;
            next();
        } else if (token == '*' || token == '/') {  // 乘除法
            op = token;
            next();
            entry = sym_lookup(token_val);
            if (entry[1] != Loc) entry = sym_lookup(1);
            *++pc = op + 2;
            *++pc = entry - base;
            next();
        } else if (token == '(') {  // 括号
            next();
            expr(1);
            if (token == ')') next();
        } else if (token == '-') {  // 负数
            next();
            *++pc = Lit;
            *++pc = -token_val;
            next();
        } else if (token == Num) {  // 数字
            *++pc = Lit;
            *++pc = token_val;
            next();
        } else if (token == Id) {   // 标识符
            entry = sym_lookup(token_val);
            if (entry[1] == Loc) {
                *++pc = entry[1];
                *++pc = entry - base;
            } else if (entry[1] == Glo) {
                *++pc = Lit;
                *++pc = entry - base;
            } else if (entry[1] == Sys) {  // 系统库函数
                *++pc = entry[2];
            }
            next();
        } else {
            printf("%d: 不合法的词法单元 %d\n", lev, token);
            return -1;
        }
    }
}

// 目标代码生成
int gen() {
    int op, *entry, id;

    next();
    while (token > 0) {
        entry = sym_lookup(token_val);
        id = entry - base;
        if (entry[1] == Loc) *++pc = entry[1];
        else if (entry[1] == Glo) *++pc = entry[1];
        else if (entry[1] == Sys) *++pc = entry[2];
        next();
    }

    return 0;
}

// 解释执行目标代码
int run() {
    int op, *entry;

    base = sp = sym;   // 初始化栈帧基址和堆栈指针
    loc = sp = pc = code;
    while (1) {
        op = *pc++;
        if (op == Lit) *--sp = *pc++;
        else if (op == Opr) {
            switch (*pc++) {
                case 0: return 0;             // 返回
                case 1: *sp = -*sp; break;     // 取负
                case 2: sp--; break;           // 出栈
                case 3: sp--; *sp = *sp + *sp; break;  // 加法
                case 4: sp--; *sp = *sp - *sp; break;  // 减法
                case 5: sp--; *sp = *sp * *sp; break;  // 乘法
                case 6: sp--; *sp = *sp / *sp; break;  // 除法
                case 7: *sp = *sp == *sp; break;  // 等于
                case 8: *sp = *sp != *sp; break;  // 不等于
                case 9: *sp = *sp < *sp; break;   // 小于
                case 10: *sp = *sp > *sp; break;  // 大于
                case 11: *sp = *sp <= *sp; break; // 小于等于
                case 12: *sp = *sp >= *sp; break; // 大于等于
            }
        } else if (op == Lod) {
            *--sp = *(entry = base + *pc++);
            if (*entry == Loc) *sp = *sp;
        } else if (op == Sto) {
            entry = base + *pc++;
            *entry = *sp;
            *sp = *sp;
        }
    }
}

// 主函数
int main() {
    int i, entry[3];
    code = calloc(256, 4);   // 目标代码数组
    pc = code;
    next();
    while (token > 0) {
        if (token == Int) {   // 声明整型变量
            next();
            while (token == Id) {
                entry[1] = Loc;
                entry[2] = loc++;
                next();
            }
        }
        if (token == Id) {    // 处理标识符
            entry[1] = Glo;
            entry[2] = (int)pc;
            next();
        }
        if (token == '(') {    // 处理函数调用
            next();
            entry[1] = Sys;
            entry[2] = token_val;
            next();
            if (token == ')') next();
        }
        if (token == Num) {   // 处理数字
            entry[1] = Lit;
            entry[2] = token_val;
            next();
        }
        *sym_lookup(token_val) = entry;  // 将标识符信息加入符号表
        next();
    }
    base = sp = sym;
    loc = sp = pc = code;
    gen();
    run();
    return 0;
}

```
