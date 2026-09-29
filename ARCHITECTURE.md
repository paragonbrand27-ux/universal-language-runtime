# Universal Language Runtime - Architecture Guide

## Project Overview
A unified programming language runtime that seamlessly combines the best features of Python, JavaScript, Go, Rust, Java, C++, and more through intelligent cross-language communication.

## Core Principles
1. **Positive Attributes Only** - Extract best practices, drop language-specific baggage
2. **Seamless Translation** - Automatic inter-language communication without boilerplate
3. **Permission-Based Security** - Fine-grained control over what each module can access
4. **Performance** - Compiled to native where possible, interpreted where needed

---

## Phase 1: Foundation (Weeks 1-2)

### Step 1.1: Project Structure
```
universal-language-runtime/
├── src/
│   ├── core/              # Core runtime engine
│   ├── parser/            # AST parser
│   ├── ir/                # Intermediate representation
│   ├── interpreters/      # Language-specific interpreters
│   ├── adapters/          # Language adapters (Python, JS, Rust, etc.)
│   ├── permissions/       # Capability-based security
│   └── stdlib/            # Standard library
├── tests/
├── examples/
├── docs/
└── Cargo.toml             # Using Rust as the base runtime
```

### Step 1.2: Environment Setup
- **Choose Base Language**: Rust (memory-safe, high performance, excellent FFI support)
- **Set Up Tooling**: Cargo, rustfmt, clippy
- **Version Control**: Git with semantic commits
- **CI/CD**: GitHub Actions for testing

### Step 1.3: Create Project Skeleton
```bash
cargo init --name universal-language-runtime
cargo add serde serde_json
cargo add thiserror
cargo add logos lalrpop  # Lexer/Parser generators
```

---

## Phase 2: Parser & AST (Weeks 3-4)

### Step 2.1: Define the Universal Syntax
Create a **single, unified grammar** that can represent any language construct:

```
Expression ::=
  | Literal (int, float, string, bool, null)
  | Identifier
  | BinaryOp (Expression, Operator, Expression)
  | UnaryOp (Operator, Expression)
  | FunctionCall (name, args)
  | FunctionDef (name, params, body, permissions)
  | ControlFlow (if/while/for with conditions)
  | TypeAnnotation (name: Type)
  | Assignment (target = value)
```

### Step 2.2: Create Lexer
```rust
// src/parser/lexer.rs
use logos::Logos;

#[derive(Logos, Debug, Clone, PartialEq)]
pub enum Token {
    #[regex(r"[0-9]+")]
    Int,
    
    #[regex(r"[a-zA-Z_][a-zA-Z0-9_]*")]
    Identifier,
    
    #[token("fn")]
    Fn,
    
    #[token("let")]
    Let,
    
    #[token("if")]
    If,
    
    #[token("=")]
    Assign,
    
    #[regex(r"[ \t\n\f]+", logos::skip)]
    #[error]
    Error,
}
```

### Step 2.3: Create AST Nodes
```rust
// src/parser/ast.rs
#[derive(Debug, Clone)]
pub enum AstNode {
    Literal(Value),
    Identifier(String),
    BinaryOp {
        left: Box<AstNode>,
        op: BinaryOperator,
        right: Box<AstNode>,
    },
    FunctionDef {
        name: String,
        params: Vec<Parameter>,
        body: Vec<AstNode>,
        permissions: Vec<Permission>,
    },
    FunctionCall {
        name: String,
        args: Vec<AstNode>,
    },
}

#[derive(Debug, Clone)]
pub enum Value {
    Int(i64),
    Float(f64),
    String(String),
    Bool(bool),
    Null,
}
```

---

## Phase 3: Intermediate Representation (Weeks 5-6)

### Step 3.1: Design the IR (Intermediate Language)
The IR is language-agnostic bytecode:

```rust
// src/ir/bytecode.rs
#[derive(Debug, Clone)]
pub enum Instruction {
    Push(Value),           // Push value to stack
    Pop,                   // Pop from stack
    Load(String),          // Load variable
    Store(String),         // Store variable
    BinOp(BinaryOp),       // Execute binary operation
    Call(String, usize),   // Call function with N args
    Return,                // Return from function
    JumpIfFalse(usize),    // Conditional jump
    Jump(usize),           // Unconditional jump
    CheckPermission(String), // Verify capability
}

pub struct CompiledFunction {
    pub name: String,
    pub instructions: Vec<Instruction>,
    pub permissions: Vec<Permission>,
}
```

### Step 3.2: Compile AST to IR
```rust
// src/ir/compiler.rs
pub fn compile_ast(node: &AstNode) -> Vec<Instruction> {
    match node {
        AstNode::Literal(v) => vec![Instruction::Push(v.clone())],
        AstNode::Identifier(name) => vec![Instruction::Load(name.clone())],
        AstNode::BinaryOp { left, op, right } => {
            let mut code = compile_ast(left);
            code.extend(compile_ast(right));
            code.push(Instruction::BinOp(op.clone()));
            code
        }
        _ => vec![],
    }
}
```

---

## Phase 4: Permissions & Capabilities (Weeks 7-8)

### Step 4.1: Define Permission Model
```rust
// src/permissions/model.rs
#[derive(Debug, Clone, PartialEq)]
pub enum Permission {
    FileRead(String),      // Read specific path
    FileWrite(String),     // Write to path
    NetworkAccess(String), // Access host:port
    ProcessExecution,      // Spawn child processes
    MemoryAllocation(u64), // Max memory (bytes)
    CPUTime(u64),          // Max execution time (ms)
}

pub struct ExecutionContext {
    pub permissions: Vec<Permission>,
    pub variables: HashMap<String, Value>,
}

impl ExecutionContext {
    pub fn has_permission(&self, perm: &Permission) -> bool {
        self.permissions.contains(perm)
    }
    
    pub fn enforce_permission(&self, perm: &Permission) -> Result<(), String> {
        if !self.has_permission(perm) {
            return Err(format!("Permission denied: {:?}", perm));
        }
        Ok(())
    }
}
```

### Step 4.2: Sandbox Execution
```rust
// src/permissions/sandbox.rs
pub fn execute_with_sandbox(
    code: Vec<Instruction>,
    permissions: Vec<Permission>,
) -> Result<Value, String> {
    let mut context = ExecutionContext {
        permissions,
        variables: HashMap::new(),
    };
    
    execute(&code, &mut context)
}

fn execute(code: &[Instruction], ctx: &mut ExecutionContext) -> Result<Value, String> {
    let mut stack: Vec<Value> = vec![];
    
    for instr in code {
        match instr {
            Instruction::CheckPermission(perm_name) => {
                ctx.enforce_permission(&Permission::FileRead(perm_name.clone()))?;
            }
            // ... handle other instructions
            _ => {}
        }
    }
    
    Ok(stack.pop().unwrap_or(Value::Null))
}
```

---

## Phase 5: Language Adapters (Weeks 9-12)

### Step 5.1: Python Adapter
```rust
// src/adapters/python_adapter.rs
pub struct PythonAdapter;

impl PythonAdapter {
    pub fn parse(code: &str) -> Result<Vec<AstNode>, String> {
        // Use rustpython or call CPython FFI
        // Convert Python AST → Universal AST
        Ok(vec![])
    }
    
    pub fn from_universal(ast: &[AstNode]) -> String {
        // Convert Universal AST → Python code
        String::new()
    }
}
```

### Step 5.2: JavaScript Adapter
```rust
// src/adapters/javascript_adapter.rs
pub struct JavaScriptAdapter;

impl JavaScriptAdapter {
    pub fn parse(code: &str) -> Result<Vec<AstNode>, String> {
        // Use swc or oxc (Rust-based JS parser)
        Ok(vec![])
    }
    
    pub fn from_universal(ast: &[AstNode]) -> String {
        // Convert back to JS
        String::new()
    }
}
```

### Step 5.3: Rust Adapter
```rust
// src/adapters/rust_adapter.rs
pub struct RustAdapter;

impl RustAdapter {
    pub fn parse(code: &str) -> Result<Vec<AstNode>, String> {
        // Use syn crate for Rust parsing
        Ok(vec![])
    }
    
    pub fn from_universal(ast: &[AstNode]) -> String {
        String::new()
    }
}
```

---

## Phase 6: Inter-Language Communication (Weeks 13-14)

### Step 6.1: Type Mapping Bridge
```rust
// src/bridge/type_bridge.rs
pub struct TypeBridge;

impl TypeBridge {
    // Python int → Universal Int
    // JavaScript number → Universal Int/Float
    // Rust i64 → Universal Int
    
    pub fn map_type(source_type: &str, source_lang: &str) -> UniversalType {
        match (source_lang, source_type) {
            ("python", "int") => UniversalType::Int,
            ("javascript", "number") => UniversalType::Number,
            ("rust", "i64") => UniversalType::Int,
            _ => UniversalType::Any,
        }
    }
}
```

### Step 6.2: Cross-Language Function Calls
```rust
// src/bridge/call_bridge.rs
pub fn call_across_languages(
    from: &str,          // "python"
    to: &str,            // "rust"
    function: &str,      // function name
    args: Vec<Value>,
) -> Result<Value, String> {
    // 1. Convert args from 'from' language to Universal
    let universal_args = convert_to_universal(from, args);
    
    // 2. Execute function
    let result = execute_function(function, universal_args)?;
    
    // 3. Convert result back to 'to' language
    let converted_result = convert_from_universal(to, result);
    
    Ok(converted_result)
}
```

---

## Phase 7: Standard Library (Weeks 15-16)

### Step 7.1: Core Functions
```rust
// src/stdlib/core.rs
pub fn builtin_print(args: Vec<Value>) -> Value {
    for arg in args {
        println!("{}", format_value(&arg));
    }
    Value::Null
}

pub fn builtin_add(a: Value, b: Value) -> Result<Value, String> {
    match (a, b) {
        (Value::Int(x), Value::Int(y)) => Ok(Value::Int(x + y)),
        (Value::Float(x), Value::Float(y)) => Ok(Value::Float(x + y)),
        (Value::String(x), Value::String(y)) => Ok(Value::String(x + &y)),
        _ => Err("Type mismatch".to_string()),
    }
}

pub fn builtin_len(v: Value) -> Result<Value, String> {
    match v {
        Value::String(s) => Ok(Value::Int(s.len() as i64)),
        _ => Err("len() requires string".to_string()),
    }
}
```

### Step 7.2: Data Structure Library
```rust
// src/stdlib/collections.rs
pub struct List(Vec<Value>);
pub struct Dict(HashMap<String, Value>);
pub struct Set(HashSet<String>);
```

---

## Phase 8: Testing & Examples (Weeks 17-18)

### Step 8.1: Unit Tests
```rust
// tests/parser_test.rs
#[test]
fn test_parse_function() {
    let code = "fn add(a, b) { return a + b }";
    let ast = parse(code).unwrap();
    assert_eq!(ast.len(), 1);
}

#[test]
fn test_ir_compilation() {
    let ast = AstNode::Literal(Value::Int(42));
    let ir = compile_ast(&ast);
    assert_eq!(ir[0], Instruction::Push(Value::Int(42)));
}
```

### Step 8.2: Integration Tests
```rust
// tests/integration_test.rs
#[test]
fn test_python_to_rust_bridge() {
    let result = call_across_languages(
        "python",
        "rust",
        "add",
        vec![Value::Int(5), Value::Int(3)],
    );
    assert_eq!(result.unwrap(), Value::Int(8));
}
```

### Step 8.3: Example Programs
```
// examples/hello_world.univ
fn main() {
    print("Hello from Universal Language!")
}

// examples/cross_lang.univ
// Call Python function from main Rust code
fn main() {
    result = call_python("calculate", [10, 20])
    print(result)
}
```

---

## Development Checklist

- [ ] Phase 1: Project setup & dependencies
- [ ] Phase 2: Lexer & parser working
- [ ] Phase 3: IR compilation functional
- [ ] Phase 4: Permission model implemented
- [ ] Phase 5: At least 3 language adapters complete
- [ ] Phase 6: Cross-language calls working
- [ ] Phase 7: Core stdlib functions
- [ ] Phase 8: Comprehensive test coverage
- [ ] Documentation complete
- [ ] Example programs working

---

## Next Steps
Start with Phase 1 and commit each phase to the repository with clear commit messages.
