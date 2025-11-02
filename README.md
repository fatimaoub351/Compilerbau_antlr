# Compilerbau_antlr

# A3.1: Grammatik

grammar gramma;


program
    : statement+ EOF
    ;

statement
    : assignment NEWLINE
    | whileStmt
    | ifStmt
    ;

assignment
    : IDENT ':=' expr
    ;

whileStmt
    : 'while' expr 'do' NEWLINE statement+ 'end'
    ;

ifStmt
    : 'if' expr 'do' NEWLINE statement+ ('else' 'do' NEWLINE statement+)? 'end'
    ;

expr
    : expr ('==' | '!=' | '<' | '>') expr   
    | expr ('+' | '-') expr           
    | literal                             
    | IDENT                             
    | '(' expr ')'                          
    ;

literal
    : INT
    | STRING
    ;

INT     : [0-9]+ ;
STRING  : '"' (~["\r\n])* '"' ;
IDENT   : [a-zA-Z_][a-zA-Z0-9_]* ;


# A3.2: Pretty Printer

public class PrettyPrinter extends ExampleBaseVisitor<Void> {
    private int indent = 0;

    private void print(String s) {
        System.out.print(" ".repeat(indent) + s);
    }

    @Override
    public Void visitAssignment(ExampleParser.AssignmentContext ctx) {
        print(ctx.IDENT().getText() + " := " + ctx.expr().getText());
        System.out.println();
        return null;
    }

    @Override
    public Void visitWhileStmt(ExampleParser.WhileStmtContext ctx) {
        print("while " + ctx.expr().getText() + " do\n");
        indent += 4;
        ctx.statement().forEach(this::visit);
        indent -= 4;
        print("end\n");
        return null;
    }

    @Override
    public Void visitIfStmt(ExampleParser.IfStmtContext ctx) {
        print("if " + ctx.expr().getText() + " do\n");
        indent += 4;
        ctx.statement(0).forEach(this::visit);
        indent -= 4;

        if (ctx.statement().size() > 1) {
            print("else do\n");
            indent += 4;
            ctx.statement(1).forEach(this::visit);
            indent -= 4;
        }

        print("end\n");
        return null;
    }
}


NEWLINE : [\r\n]+ ;
WS      : [ \t]+ -> skip ;
COMMENT : '#' ~[\r\n]* -> skip ;

# A3.3: AST 


