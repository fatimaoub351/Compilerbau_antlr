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


// ANTLR grammar: PrettyLang.g4

grammar PrettyLang;

prog: stmt* EOF;

stmt

    : assignStmt
    
    | ifStmt
    
    | whileStmt
    
    | ';'    
  
    ;

assignStmt: ID ':=' expr;

ifStmt: 'if' expr 'do' stmt* ( 'else' 'do' stmt* )? 'end';

whileStmt: 'while' expr 'do' stmt* 'end';

expr: expr ('*'|'/') expr

    | expr ('+'|'-') expr
    
    | expr ('<' | '>' | '<=' | '>=' | '==' | '!=') expr
    
    | INT
    
    | ID
    
    | '(' expr ')'
    
    ;


ID  : [a-zA-Z_] [a-zA-Z_0-9]* ;

INT : [0-9]+ ;


WS  : [ \t\r\n]+ -> skip ;

COMMENT: '#' ~[\r\n]* -> skip ;

// Java visitor-based pretty printer

import org.antlr.v4.runtime.*;

import org.antlr.v4.runtime.tree.*;

import java.nio.file.*nimport java.io.IOException;

public class PrettyPrinter {

    public static void main(String[] args) throws IOException {
    
        if (args.length == 0) {
        
            System.err.println("Usage: java PrettyPrinter <sourcefile>");
            
            System.exit(1);
        }
        
        String text = new String(Files.readAllBytes(Paths.get(args[0])));
        
        CharStream cs = CharStreams.fromString(text);
        
        PrettyLangLexer lexer = new PrettyLangLexer(cs);
        
        CommonTokenStream tokens = new CommonTokenStream(lexer);
        
        PrettyLangParser parser = new PrettyLangParser(tokens);
        
        ParseTree tree = parser.prog();

        PrinterVisitor pv = new PrinterVisitor();
        
        String out = pv.visit(tree);
        
        System.out.println(out);
        
    }
    
}

// Visitor implementation

import org.antlr.v4.runtime.tree.AbstractParseTreeVisitor;

import java.util.stream.Collectors;

public class PrinterVisitor extends PrettyLangBaseVisitor<String> {

    private int indent = 0;
    
    private String indentStr() {
    
        return "    ".repeat(indent); 
        
    }

    @Override
    public String visitProg(PrettyLangParser.ProgContext ctx) {
        return ctx.stmt().stream()
            .map(s -> visit(s))
            .filter(s -> !s.isEmpty())
            .collect(Collectors.joining("\n"));
    }

    @Override
    public String visitAssignStmt(PrettyLangParser.AssignStmtContext ctx) {
        String left = ctx.ID().getText();
        String right = visit(ctx.expr());
        return indentStr() + left + " := " + right;
    }

    @Override
    public String visitIfStmt(PrettyLangParser.IfStmtContext ctx) {
        StringBuilder sb = new StringBuilder();
        String cond = visit(ctx.expr());
        sb.append(indentStr()).append("if ").append(cond).append(" do\n");

        indent++;
  
        int i = 0;
    

        int totalStmts = ctx.stmt().size();
        int firstBlockCount = totalStmts;
        if (ctx.getText().contains("else")) {
            // more robust: locate the token index of 'else' by checking children
            int elseIndex = -1;
            for (int c=0;c<ctx.getChildCount();c++) {
                if (ctx.getChild(c).getText().equals("else")) { elseIndex = c; break; }
            }
            if (elseIndex != -1) {
                // count stmt children that appear before else child
                firstBlockCount = 0;
                for (ParseTree ch : ctx.children) {
                    if (ch.getText().equals("else")) break;
                    if (ch instanceof PrettyLangParser.StmtContext) firstBlockCount++;
                }
            }
        }

        for (int j = 0; j < firstBlockCount; j++) {
            String s = visit(ctx.stmt(j));
            if (!s.isEmpty()) sb.append(s).append("\n");
        }
        indent--;

        if (ctx.getText().contains("else")) {
            sb.append(indentStr()).append("else do\n");
            indent++;
            for (int j = firstBlockCount; j < totalStmts; j++) {
                String s = visit(ctx.stmt(j));
                if (!s.isEmpty()) sb.append(s).append("\n");
            }
            indent--;
        }

        sb.append(indentStr()).append("end");
        return sb.toString();
    }

    @Override
    public String visitWhileStmt(PrettyLangParser.WhileStmtContext ctx) {
        StringBuilder sb = new StringBuilder();
        String cond = visit(ctx.expr());
        sb.append(indentStr()).append("while ").append(cond).append(" do\n");
        indent++;
        for (PrettyLangParser.StmtContext sctx : ctx.stmt()) {
            String s = visit(sctx);
            if (!s.isEmpty()) sb.append(s).append("\n");
        }
        indent--;
        sb.append(indentStr()).append("end");
        return sb.toString();
    }

    @Override
    public String visitExpr(PrettyLangParser.ExprContext ctx) {
        if (ctx.INT() != null) return ctx.INT().getText();
        if (ctx.ID() != null) return ctx.ID().getText();
        if (ctx.getChildCount() == 3) {
            // binary or parenthesized
            String a = visit(ctx.expr(0));
            String b = visit(ctx.expr(1));
            String op = ctx.getChild(1).getText();
            return a + " " + op + " " + b;
        }
        if (ctx.getChildCount() == 1) return visit(ctx.getChild(0));
        return "";
    }

    
    @Override
    protected String defaultResult() { return ""; }
}

# A3.3: AST 

import java.util.stream.Collectors;

public class ASTPrettyPrinter {

    private int indent = 0;
    
    private String indentStr() { return "    ".repeat(indent); }

    public String print(AST node) {
    
        if (node instanceof Program p) {
        
            return p.statements.stream()
            
                    .map(this::print)
                    
                    .collect(Collectors.joining("\n"));
                    
        } else if (node instanceof Assign a) {
        
            return indentStr() + a.id + " := " + print(a.expr);
            
        } else if (node instanceof IfNode i) {
        
            StringBuilder sb = new StringBuilder();
            
            sb.append(indentStr()).append("if ").append(print(i.cond)).append(" do\n");
            
            indent++;
            
            for (AST s : i.thenStmts) sb.append(print(s)).append("\n");
            
            indent--;
            
            if (!i.elseStmts.isEmpty()) {
            
                sb.append(indentStr()).append("else do\n");
                
                indent++;
                
                for (AST s : i.elseStmts) sb.append(print(s)).append("\n");
                
                indent--;
                
            }
            
            sb.append(indentStr()).append("end");
            
            return sb.toString();}
            
            else if (node instanceof WhileNode w) {
        
            StringBuilder sb = new StringBuilder();
            
            sb.append(indentStr()).append("while ").append(print(w.cond)).append(" do\n");
            
            indent++;
            
            for (AST s : w.body) sb.append(print(s)).append("\n");
            
            indent--;
            
            sb.append(indentStr()).append("end");
            
            return sb.toString();
            
        } 
        
        else if (node instanceof Var v) {
        
            return v.name;
            
        } 
        
        else if (node instanceof IntLit lit) {
        
            return lit.val;
            
        } 
        
        else if (node instanceof BinOp b) {
        
            return print(b.left) + " " + b.op + " " + print(b.right);
        }
        
        return "";
    }
}

