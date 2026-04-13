# Coding Style Guide

This skill defines a comprehensive coding style guide focused on readability, maintainability, and clear documentation. It emphasizes early returns, thoughtful whitespace, and comprehensive documentation using MDT (Markdown Template) for reusable documentation patterns.

## Philosophy

Code is read far more often than it is written. This style guide prioritizes:

1. **Reader empathy**: Give the reader time to understand concepts
2. **Early clarity**: State what's happening upfront, exit early from bad states
3. **Comprehensive documentation**: Every function, constant, and module deserves an explanation of *why* it exists
4. **Composability**: Reuse documentation patterns across the codebase

## Core Principles

### Documentation-First

Every function, constant, and module should explain:
- **What it does**: A clear description of functionality
- **Why it exists**: The reason and use cases
- **How to use it**: Practical examples where helpful

This applies even to non-exported items - you or someone else will need to understand this code in the future.

### MDT (Markdown Template) for Documentation

Use MDT to create reusable documentation templates:
- **Module-level docs**: Share templates between file headers and README
- **Common patterns**: Document recurring use cases once, use everywhere
- **Composed documentation**: Build complex docs from simple, reusable pieces

See `templates/` directory for MDT templates.

### Code Layout and Whitespace

1. **Variables at the top**: Declare variables and constants at function/file start when possible
2. **Whitespace before control flow**: Add blank lines before `if`, `match`, `for`, `while`, etc.
3. **Group related code**: Use blank lines between logical groups
4. **Breathing room**: Long variable declarations get whitespace after them

### Early Returns Over Deep Nesting

**Orange flag**: Deeply nested code is a code smell.

Prefer:
```rust
fn process_data(data: Option<Data>) -> Result<Output, Error> {
    // Guard clauses first
    let data = data.ok_or(Error::NoData)?;
    
    if !data.is_valid() {
        return Err(Error::InvalidData);
    }
    
    // Now focus on the happy path
    perform_operation(data)
}
```

Over:
```rust
fn process_data(data: Option<Data>) -> Result<Output, Error> {
    if let Some(data) = data {
        if data.is_valid() {
            perform_operation(data)
        } else {
            Err(Error::InvalidData)
        }
    } else {
        Err(Error::NoData)
    }
}
```

### Extraction Over Nesting

When nesting becomes unavoidable, extract into small, focused functions:
- Single responsibility
- Clear naming
- Reusable across the module

## Language-Specific Guides

- [Rust Style Guide](./languages/RUST.md) - Ownership, builders, lifetime management
- [TypeScript Style Guide](./languages/TYPESCRIPT.md) - Type safety, interfaces, modern JS
- [Python Style Guide](./languages/PYTHON.md) - Type hints, docstrings, patterns

## Integration with Formatters

This style guide complements, not replaces, automated formatters:
- **Always use**: `rustfmt`, `prettier`, `black`, etc.
- **This guide covers**: Whitespace between groups, early returns, documentation patterns
- **Formatters cover**: Indentation, trailing commas, spacing around operators

## Security and "Why" Comments

When something exists for security reasons:
- Add inline comments explaining the security consideration
- Reference relevant security guidelines or CVEs
- Explain the threat model briefly

## README Standards

Every project/README should answer within 2 minutes:
1. **Why does this exist?** - The problem it solves
2. **Why should I use it?** - The value proposition
3. **What are the use cases?** - Practical scenarios

Use MDT templates to keep README and module docs in sync.

## Template Usage

```mdt
<!-- Use from templates/ -->
{{> module-header title="My Module" description="Does amazing things" }}

{{> function-doc 
   name="process_data"
   purpose="Transforms raw data into structured format"
   why="Centralizes data validation and transformation logic"
}}
```
