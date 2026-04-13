# Coding Style Guide

A comprehensive, living documentation of coding principles focused on readability, maintainability, and clear communication through code and documentation.

## 🎯 Philosophy

Code is read far more often than it is written. This style guide prioritizes:

1. **Reader empathy**: Give the reader time to understand concepts
2. **Early clarity**: State what's happening upfront, exit early from bad states  
3. **Comprehensive documentation**: Every function, constant, and module deserves an explanation of *why* it exists
4. **Composability**: Reuse documentation patterns across the codebase using MDT templates

## 📚 What's Included

### Core Principles
- **Documentation-First**: Every item explains what, why, and how
- **MDT (Markdown Template)**: Create reusable documentation patterns
- **Early Returns**: Avoid deep nesting, fail fast
- **Thoughtful Whitespace**: Group related code, add breathing room
- **Security Comments**: Explain security considerations inline

### Language-Specific Guides

| Language | Focus Areas |
|----------|-------------|
| [Rust](./languages/RUST.md) | Ownership, typed-builder, lifetime management, error types |
| [TypeScript](./languages/TYPESCRIPT.md) | Type safety, Zod validation, async patterns, strict types |
| [Python](./languages/PYTHON.md) | Dataclasses, type hints, protocols, context managers |

### Templates

MDT templates for reusable documentation:
- [`module-header.mdt`](./templates/module-header.mdt) - Module-level documentation
- [`function-doc.mdt`](./templates/function-doc.mdt) - Function documentation
- [`why-comment.mdt`](./templates/why-comment.mdt) - Reusable "why" explanations

## 🚀 Quick Start

### For New Projects

1. **Read the main guide**: [SKILL.md](./SKILL.md)
2. **Choose your language**: Rust, TypeScript, or Python
3. **Copy templates**: Use MDT templates for consistent documentation
4. **Set up formatters**: Always use automated formatters (rustfmt, prettier, black)

### For Existing Projects

1. **Review the patterns**: Look at examples in your language guide
2. **Apply incrementally**: Start with early returns and whitespace
3. **Add documentation**: Document functions as you touch them
4. **Create templates**: Extract common documentation into MDT templates

## 📖 Key Patterns

### Early Returns Example

**❌ Avoid:**
```rust
if let Some(user) = request.user {
    if user.is_active {
        if user.has_permission("write") {
            // ... happy path buried deep
        }
    }
}
```

**✅ Prefer:**
```rust
let user = request.user.ok_or(Error::NoUser)?;

if !user.is_active {
    return Err(Error::UserInactive);
}

if !user.has_permission("write") {
    return Err(Error::NoPermission);
}

// Happy path is now clear and at top level
```

See more examples in [RUST.md](./languages/RUST.md), [TYPESCRIPT.md](./languages/TYPESCRIPT.md), [PYTHON.md](./languages/PYTHON.md).

### Documentation Example

**Every function needs a "Why":**

```rust
/// Validates user credentials against the database.
///
/// # Why This Exists
/// Authentication is a security-critical operation that must be
/// consistent across all entry points (web, API, CLI). This function
/// centralizes the validation logic and audit logging.
///
/// # Security Considerations
/// - Never log passwords, even hashed ones
/// - Rate limiting should be applied at the call site
/// - Failed attempts are logged for security monitoring
fn validate_credentials(credentials: &Credentials) -> Result<User, AuthError> {
    // ...
}
```

## 🔧 Tool Integration

This style guide **complements** automated formatters:
- **Rust**: rustfmt
- **TypeScript**: prettier
- **Python**: black + isort

Always defer to formatters for:
- Indentation
- Trailing commas
- Spacing around operators
- Line length

This guide covers:
- Whitespace between logical groups
- Early return patterns
- Documentation structure
- Code organization

## 📝 Contributing

This is a living document. As you discover new patterns:

1. Add examples to the relevant language guide
2. Create MDT templates for reusable patterns
3. Update the main SKILL.md with new principles

## 📄 License

MIT - See [LICENSE](./LICENSE) for details.

---

*"Code is read far more often than it is written."* — Write for your future self and your teammates.
