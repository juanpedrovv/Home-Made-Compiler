# ProyectoCompiladores
## Informe
### Integrantes:
  - Sebastián Loza
  - Luis Golac
  - Juan Pedro Vásquez
### 1.- Comentarios

Para agregar a IMP0 la la posibilidad de incluir comentarios de una sola línea en cualquier punto del programa. Tuvimos que modificar el Scanner de 		```imp_parser.cpp```:

#### **Hemos modificado el método nextToken del Scanner. Si este encuentra un '/':**

```c++
case '/': 
    {
      c = nextChar();
      if (c == '/') { //1
        string comment = "//"; //2
        c = nextChar();
        while (c != '\0' && c != '\n'){
          comment += c; //3
          c = nextChar();
        }
        cout << "Comment at line " << lineCount << ": " << comment << endl; //4
        switch(c){ //5
          case '\n': token = nextToken(); break; //5.1
          case '\0': token = new Token(Token::END); //5.2.
        }
      }
      else { //6
        rollBack(); 
        token = new Token(Token::DIV);
      }
      break;
    }
```

1. Si el siguiente carácter después de / es también /, se considera que se ha detectado el inicio de un comentario.
2. Creamos un string en el que insertaremos los caracteres de nuestro comentario para posteriormente imprimirlo
3. Se utiliza un bucle while para avanzar en la cadena de caracteres hasta encontrar el final del comentario (\n o \0). El contenido del comentario se consume, pero no se procesan, ya que los comentarios se consideran comentarios y no afectan la lógica del programa. Solo agregamos los caracteres al comentario.
4. Después de consumir el comentario, se imprime el comentario y su posición.
5. Después de consumir el comentario, se evalúa el siguiente carácter después del comentario.
- 5.1. Si el comentario termina con \n, se interpreta como el final del comentario y se procede a analizar el siguiente token.
- 5.2. Si el comentario termina con \0, se considera que el comentario es el final del programa y se asigna un token especial (Token::END).
6. Si después del carácter / no se detecta otro /, se retrocede en la lectura (rollBack()) y se asume que el carácter / es simplemente el operador de división. Se asigna un token correspondiente (Token::DIV).

#### **Hemos modificado el método nextToken del Scanner. Si este encuentra un '/':**

Para imprimir los comentarios en la posición en la que están en el código original, necesitamos mantener un seguimiento de la posición actual en el archivo de entrada mientras estás escaneando. Esto se puede hacer manteniendo un contador de líneas.

```c++
class Scanner {
public:
  Scanner(string in_s);
  Token* nextToken();
  ~Scanner();  
private:
  int lineCount; //<-------------
  string input;
  int first, current;
  unordered_map<string, Token::Type> reserved;
  char nextChar();
  void rollBack();
  void startLexema();
  string getLexema();
  Token::Type checkReserved(string);
};
```
Por ello en ```imp_parser.hh``` creamos un atributo contador de líneas en la clase Scanner.

```c++
Scanner::Scanner(string s):input(s),first(0),current(0),lineCount(0)
```
Y en ```imp_parser.cpp``` lo inicializamos con 0.

```c++
char Scanner::nextChar() {
  int c = input[current];
  current++;
  if (c == '\n' || c == '\0') {
    lineCount++;
  }
  return c;
}
```
Finalmente modificamos el método nextChar() del Scanner, así comprueba si el carácter obtenido es un salto de línea ('\n') o el final de la cadena ('\0'). Si es así, incrementa el contador de líneas (lineCount). Esto se hace para llevar un seguimiento de en qué línea del input se encuentra el scanner.


### 2.- Generación de código 1
#### Constantes booleanas, and y or
Para las constantes booleanas se implementó el AST BoolConstExp, con el siguiente codegen, además cabe resalatar que como no estámos trabajando en un ambiente de funciones, todas nuestras variables están en un mismo *environment* addr:

    codegen(addr, BoolConstExp(e)) =

        codegen(addr, e) = push e->b ? 1 : 0
En el compilador de nuestro lenguaje IMP el codegen está de la siguiente forma:

```c++
int ImpCodeGen::visit(BoolConstExp* e) {
  codegen(nolabel, "push", e->b ? 1 : 0);
  return 0;  
}
```
      
Para las operaciones AND y OR se modificó el parser, se agregaron lasn nuevas operaciones binarias AND y OR y en el codegen se modificó la sentencia de codegen para BinaryExp de la siguiente manera:

    codegen(addr, BinaryExp(e1, e2, op) =
        codegen(addr, e1) =     push e1
        codegen(addr, e2) =     push e2
        op                      and / or
Cabe resaltar que las Expresiones e1 y e2 deben terminar en constantes booleanas para que esta operación se pueda realizar.
En nuestro compilador del lenguaje IMP se vería de la siguiente forma:
```c++
int ImpCodeGen::visit(BinaryExp* e) {
  e->left->accept(this);
  e->right->accept(this);
  string op = "";
  switch(e->op) {
  case PLUS: op =  "add"; break;
  case MINUS: op = "sub"; break;
  case MULT:  op = "mul"; break;
  case DIV:  op = "div"; break;
  case LT:  op = "lt"; break;
  case LTEQ: op = "le"; break;
  case EQ:  op = "eq"; break;
  case OR : op = "or"; break;
  case AND: op = "and"; break;
  default: cout << "binop " << Exp::binopToString(e->op) << " not implemented" << endl;
  }
  codegen(nolabel, op);
  return 0;
}
```
#### Foor Loop
Para la implementación de la sentencia Foor Loop se tenía que añadir un env más para poder mapear las variables loacales a la sentencia Foor Loop, como por ejemplo el iterador, su codegen se vería de la siguiente manera:

      codegen(addr, ForStatement(id, e1, e2, body)) =
          codegen(addr, e1)
          storer addr(id)
          LENTRY = skip
          loadr addr(id)
          codegen(addr, e2)
          le
          jmpz LEND
          codegen(addr, body)
          loadr addr(id)
          codegen(addr, 1)
          add
          storer addr(id)
          goto LENTRE
          LEND: skip
          
En nuestro lenguaje IMP, en el compilador, el codegen del For Statement se vería de la siguiente manera:
```c++
int ImpCodeGen::visit(ForStatement* s) {
  string l1 = next_label();
  string l2 = next_label();
  
  direcciones.add_level();
  direcciones.add_var(s->id, siguiente_direccion++);
  s->e1->accept(this);
  codegen(nolabel,"store",direcciones.lookup(s->id));
  codegen(l1,"skip");
  codegen(nolabel,"load",direcciones.lookup(s->id));
  s->e2->accept(this);
  codegen(nolabel,"le");
  codegen(nolabel, "jmpz", l2);
  s->body->accept(this);
  codegen(nolabel,"load",direcciones.lookup(s->id));
  codegen(nolabel, "push", 1);
  codegen(nolabel, "add");
  codegen(nolabel,"store",direcciones.lookup(s->id));
  codegen(nolabel,"goto",l1);
  codegen(l2,"skip");
  direcciones.remove_level();

  return 0;
}

```
### 3.- Sentencia do-while
#### Cmabio en la gramática:
```
Stm ::= id “=” Exp |
        “print” “(“ Exp “)” |
        “if” Exp “then” Body [“else” Body] “endif” |
        “while” Exp “do” Body “endwhile” |
        “do” Body “while” Exp “endowhile” |
        “for” id “:” Exp “,” Exp “do” Body “endfor”
```
#### Cambios en el código:
Se agregó el nuevo AST que deriva de la clase virtual Statement en el archivo ```imp.hh```:
```c++
class DoWhileStatement : public Stm {
public:
  Exp* cond;
  Body* body;
  DoWhileStatement(Exp* c, Body* b);
  int accept(ImpVisitor* v);
  void accept(TypeVisitor* v);
  ~DoWhileStatement();
};
```
Esto nos permite que el Do While pueda ser reconocido como cualquier otro statement dentro de cualquier body de nuestro lenguaje.
Se implementaron sus métodos en el archivo ```imp.cpp```
```c++
DoWhileStatement::DoWhileStatement(Exp* c, Body* b):cond(c), body(b){}
DoWhileStatement::~DoWhileStatement() {delete cond; delete body; }
int DoWhileStatement::accept(ImpVisitor* v){
  return v->visit(this);
}
void DoWhileStatement::accept(TypeVisitor* v) {
  return v->visit(this);
}
```
Luego se hicieron cambios en el ```parser.cpp``` en la función ```parseStatement()```, se agregó lo siguiente: 
```c++
  else if (match(Token::DO)) {
    tb = parseLoopBody(); 
    if (!match(Token::WHILE))
      parserError("Esperaba 'while'");
    e = parseExp();
    if (!match(Token::ENDOWHILE)) 
      parserError("Esperaba endowhile");
    s = new DoWhileStatement(e, tb);
  } 
```
También se agregó el token **ENDOWHILE**

Por otro lado, en el archivo ```imp_printer.cpp``` se agregó lo siguiente:
```c++
int ImpPrinter::visit(DoWhileStatement* s) {
  cout << "do {" << endl;
  s->body->accept(this);
  cout << "} while ( ";
  s->cond->accept(this);
  cout << " )" << endl;
  cout << "endowhile";
  return 0;
}
```
En el archivo ```imp_interpreter.cpp```:
```c++
int ImpInterpreter::visit(DoWhileStatement* s) {
  do {
    s->body->accept(this);
  } while (s->cond->accept(this));
  return 0;
}
```
Par el typechecking usamos la siguiente estructura de tcheck: 
```
tcheck(env, DoWhileStm(cond, body)) ifi
	tcheck(env, body) && tcheck(env, cond) = bool
```
Esto se ve representado en nuestro compilador de lenguaje imp en e archivo ```imp_typechecker.cpp```:
```c++
void ImpTypeChecker::visit(DoWhileStatement* s) {
  s->body->accept(this);
  ImpType t1 = s->cond->accept(this);
  if (!t1.match(booltype)){
    cout << "Condición del Do While debe de ser bool" << endl;
    exit(0);
  }
  return;
}
```
Las sentencias de codegen que usamos fueron las siguientes:
```
codegen(addr, DoWhileStatement(cond, body)) =
    LENTRY: skip
    codegen(addr, body)
    codegen(addr, cond)
    jmpz LEND
    goto LENTRY
```
En el compilador de nuestro lenguaje imp se vería de la siguiente forma:
```c++
int ImpCodeGen::visit(DoWhileStatement* s) {
  string l1 = next_label();
  string l2 = next_label();

  codegen(l1,"skip");
  s->body->accept(this);
  s->cond->accept(this);
  codegen(nolabel,"jmpz",l2);
  codegen(nolabel,"goto",l1);
  codegen(l2,"skip");

  return 0;
}
```
### 4.- Sentencias break y continue
#### Cambios en la gramática:
Para este caso, para la implementación del break y continue en las sentencias de while y do while se tuvo que hacer un cambio en la graática de la siguiente manera:
```
Program ::= Body
Body ::= VarDecList StatementList
VarDecList ::= (VarDec)*
VarDec ::= "var" Type VarList ";"
Type ::= id
VarList ::= id ("," id)*

StatementList :: Stm ( ";" Stm )*

Stm ::= id "=" Exp |
        "print" "(" Exp ")" |
        "if" Exp "then" LoopBody ["else" Body] "endif" |
        "while" Exp "do" LoopBody "endwhile" |
        “do” Loop Body “while” Exp “endowhile” |
        "for" id ":" Exp "," Exp "do" LoopBody "endfor"

LoopBody ::= VarDecList LoopStatementList
LoopStatementList ::= LoopStm ( ";" LoopStm )*
LoopStm ::= break |
            continue |
            id "=" Exp |
            "print" "(" Exp ")" |
            "if" Exp "then" LoopBody ["else" Body] "endif" |
            “do” Loop Body “while” Exp “endowhile” |
            "while" Exp "do" LoopBody "endwhile" |
            "for" id ":" Exp "," Exp "do" LoopBody "endfor"
Exp ::= ...
```
Sin embargo esta modificación de la gramática es parcialmente real, ya que aunque si sigue esa estructura, no es necesario crear AST's para **LoopBody**, **LoopStatementList** y **LoopStm**.
Pero si los AST's para break y continue de la siguiente forma en el archivo ```imp.hh```:
```c++
class BreakStatement : public Stm {
public:
  BreakStatement();
  int accept(ImpVisitor* v);
  void accept(TypeVisitor* v);
  ~BreakStatement();
};

class ContinueStatement : public Stm {
public:
  ContinueStatement();
  int accept(ImpVisitor* v);
  void accept(TypeVisitor* v);
  ~ContinueStatement();
};
```
También se hicieron cambios en el ```parser.cpp```, se agregaron los tokens **BREAK** y **CONTINUE** y se hicieron los siguientes cambios en el código:
Debajo de ```parseBody()```:
```c++
Body* Parser::parseLoopBody() {
  VarDecList* vdl = parseVarDecList();
  StatementList* sl = parseLoopStatementList();
  return new Body(vdl, sl);
}

StatementList* Parser::parseLoopStatementList() {
  StatementList* p = new StatementList();
  p->add(parseLoopStatement());
  while(match(Token::SEMICOLON)) {
    p->add(parseLoopStatement());
  }
  return p;
}

Stm* Parser::parseLoopStatement() {
  Stm* s = NULL;
  Exp* e;
  Body *tb, *fb;
  if (match(Token::ID)) {
    string lex = previous->lexema;
    if (!match(Token::ASSIGN)) {
      cout << "Error: esperaba =" << endl;
      exit(0);
    }
    s = new AssignStatement(lex, parseExp());
    //memoria_update(lex, v);
  } 
  else if (match(Token::PRINT)) {
    if (!match(Token::LPAREN)) {
      cout << "Error: esperaba ( " << endl;
      exit(0);
    }
    e = parseExp();
    if (!match(Token::RPAREN)) {
      cout << "Error: esperaba )" << endl;
      exit(0);
    }
    s = new PrintStatement(e);
  } 
  else if (match(Token::IF)) {
      e = parseExp();
      if (!match(Token::THEN))
	parserError("Esperaba 'then'");
      tb = parseLoopBody();
      fb = NULL;
      if (match(Token::ELSE)) {
	fb = parseLoopBody();
      }
      if (!match(Token::ENDIF))
	parserError("Esperaba 'endif'");
      s = new IfStatement(e,tb,fb);
  } 
  else if (match(Token::WHILE)) {
    e = parseExp();
    if (!match(Token::DO))
      parserError("Esperaba 'do'");
    tb = parseLoopBody();
    if (!match(Token::ENDWHILE))
	parserError("Esperaba 'endwhile'");
    s = new WhileStatement(e,tb);
  } 
  else if (match(Token::DO)) {
    tb = parseLoopBody(); 
    if (!match(Token::WHILE))
      parserError("Esperaba 'while'");
    e = parseExp();
    if (!match(Token::ENDOWHILE)) 
      parserError("Esperaba endowhile");
    s = new DoWhileStatement(e, tb);
  } 
  else if (match(Token::FOR)) {
    string var;
    Exp* e2;
    if (!match(Token::ID)) parserError("Esperaba id en for");
    var = previous->lexema;
    if (!match(Token::COLON)) parserError("Esperaba COLON en for");
    e = parseExp();
    if (!match(Token::COMMA)) parserError("Esperaba COMMA en for");
    e2 = parseExp();
    if (!match(Token::DO)) parserError("Esperaba DO en for");
    tb = parseLoopBody();
    if (!match(Token::ENDFOR)) parserError("Esperaba ENDFOR en for");
    s = new ForStatement(var,e,e2,tb);
  }
  else if (match(Token::BREAK)){
    s = new BreakStatement();
  }
  else if (match(Token::CONTINUE)){
    s = new ContinueStatement();
  }
  else {
    cout << "No se encontro Statement" << endl;
    exit(0);
  }
  return s;
}
```
Y en el parseStatement en las sentencias que implican un bucle, en vez de llamarse al *parseBody()*, se llama al *parseLoopBody()*.
En el *imp_printer.cpp*:
```c++
int ImpPrinter::visit(BreakStatement*v) {
  cout << "break";
  return 0;
}

int ImpPrinter::visit(ContinueStatement*v) {
  cout << "continue";
  return 0;
}
```
En el ```imp_typechecker.cpp```, no se tuvo que agregar ninguna lógica de typecheck ya que estas sentencias son ajenas a algún tipo de dato, sólo se agregaron las declaraciones:
```c++
void ImpTypeChecker::visit(BreakStatement* s){
  return;
}
void ImpTypeChecker::visit(ContinueStatement* s){
  return;
}
```
En cuanto al codegen, se usó lo siguiente:
- Se implementaron dos nuevas variables para tener mapeados los labels de inicio y fin de los loops.
- Se modificaron los codegen de los statements que implicaban algún loop, para que puedan actualizar las dos variables nuevas creadas en el paso previo.
- Se implementó el codegen del break y continue statement usando esas 2 variables.
```
  codegen(addr, ContinueStatement()) =
      goto LEND

  codegen(addr, BreakStatement()) =
      goto LENTRY
```
En ```codegen.hh```:
```c++
  private:
  std::ostringstream code;
  string nolabel;
  int current_label;
  vector<string> in, fin;
  int act = -1;
  Environment<int> direcciones;
  int siguiente_direccion, mem_locals;
  void codegen(string label, string instr);
  void codegen(string label, string instr, int arg);
  void codegen(string label, string instr, string jmplabel);
  string next_label();
```
En ```codegen.cpp```:
```c++
int ImpCodeGen::visit(BreakStatement* s){
  codegen(nolabel, "goto", this->fin[act]);
  return 0;
}

int ImpCodeGen::visit(ContinueStatement* s){
  codegen(nolabel, "goto", this->in[act]);
  return 0;
}

int ImpCodeGen::visit(WhileStatement* s) {
  string l1 = next_label();
  string l2 = next_label();
  this->act++;
  this->in.push_back(l1);
  this->fin.push_back(l2);
  
  codegen(l1,"skip");
  s->cond->accept(this);
  codegen(nolabel,"jmpz",l2);
  s->body->accept(this);
  codegen(nolabel,"goto",l1);
  codegen(l2,"skip");

  in.pop_back();
  fin.pop_back();
  this->act--;
  return 0;
}

int ImpCodeGen::visit(DoWhileStatement* s) {
  string l1 = next_label();
  string l2 = next_label();
  string l3 = next_label();
  this->act++;
  this->in.push_back(l3);
  this->fin.push_back(l2);

  codegen(l1,"skip");
  s->body->accept(this);
  codegen(l3,"skip");
  s->cond->accept(this);
  codegen(nolabel,"jmpz",l2);
  codegen(nolabel,"goto",l1);
  codegen(l2,"skip");

  in.pop_back();
  fin.pop_back();
  this->act--;

  return 0;
}

```


