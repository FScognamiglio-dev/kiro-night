# Code Design Principles

> Apply these principles whenever writing or modifying code (any language). Skip for prose-only responses, explanations, and documentation.

## Core Philosophy

Write code that is **abstract, reusable, and composable**. Prefer generic solutions over concrete one-off implementations. Every new piece of code must justify why it cannot reuse or extend something that already exists.

## 1. Abstraction First

- Prefer generics and type parameters over concrete types. If a class or function can work with multiple types, make it generic.
- Extract base logic into abstract classes, superclasses, or interfaces before writing concrete implementations.
- Use design patterns (Strategy, Template Method, Factory, Adapter, Observer, Composite, Decorator) when they reduce coupling and improve extensibility. Don't force a pattern where it adds complexity without benefit.
- Separate concerns: business logic, data access, presentation, and orchestration belong in distinct layers.

## 2. Reuse Before Reimplementation

Before writing any new class, function, component, or module:

1. Search the existing codebase for similar abstractions, utilities, or base classes.
2. If something similar exists, extend, compose, or adapt it rather than reimplementing from scratch.
3. If existing code is close but not quite right, refactor it to be more generic so both old and new use cases benefit.
4. Only create something new when nothing in the project can reasonably be extended.

When reusing existing code, state explicitly: "Found existing [class/function/component] at [path] — extending/composing it instead of reimplementing."

## 3. Frontend: Separate Logic from UI

- Extract business logic into generic, framework-agnostic classes or functions with type parameters.
- Components are thin wrappers that delegate to logic classes/hooks/composables.
- Use generic base classes or utility types for shared behavior (e.g., `BaseListManager<T>`, `FormHandler<TModel, TValidation>`).
- Prefer composition (hooks, composables, services) over inheritance when the framework idiom supports it.

## 4. Backend: Layered Generics

- Define generic interfaces and abstract classes for repositories, services, and controllers (e.g., `CrudService<T, ID>`, `BaseRepository<T>`).
- Concrete implementations add only domain-specific behavior — never redefine CRUD or common patterns.
- Use Template Method for operations that share a skeleton but vary in specific steps.
- Use generics in DTOs, mappers, and validators to avoid boilerplate.

## 5. Language-Specific Guidance

### TypeScript / JavaScript
- Use generic types (`<T>`, `<T extends Base>`) in utility functions, hooks, services, and data structures.
- Prefer `interface` for contracts, `abstract class` for shared implementation.
- Use discriminated unions and mapped types to reduce repetition.

### Java
- Use bounded generics (`<T extends Entity>`) in service and repository layers.
- Define abstract base classes with template methods for common workflows.
- Use interfaces with default methods for cross-cutting behavior.

### Python
- Use `typing.Generic[T]`, `Protocol`, and `ABC` for abstract base classes.
- Define generic base classes for services, repositories, and handlers.
- Use `TypeVar` with bounds to constrain generic types meaningfully.

## 6. When NOT to Abstract

- Don't abstract prematurely: if there's genuinely only one use case and no foreseeable reuse, a concrete implementation is fine. State this reasoning explicitly.
- Don't over-engineer: a simple utility function doesn't need a Strategy pattern.
- Rule of three: same pattern appearing a third time → extract an abstraction.

## 7. Performance and Algorithmic Complexity

- Avoid O(n²) in data processing, collection manipulation, or rendering logic. Prefer O(n) or O(n log n).
- Use Maps, Sets, HashMaps for lookups instead of nested loops or repeated `.find()`/`.filter()` on arrays.
- Consider dataset growth: if it can grow, the algorithm must scale.
- Prefer database-level operations (filtering, aggregation, joins) over in-memory processing for large datasets.
- Flag any nested loop over the same or related collections as a potential performance issue.

## 8. Maintainability

- Readability over cleverness. Prefer clear, explicit code over compact one-liners that require mental parsing.
- Self-documenting code, minimal comments. Comments only when:
  - The code implements a non-obvious algorithm and the reasoning behind the approach must be preserved.
  - There is a workaround for a known bug or limitation that would otherwise look wrong.
  - Never comment what the code does — if you must, refactor instead.
- Single responsibility: each function, class, or module has one reason to change.
- Testability by design: isolate side effects, inject dependencies, keep pure logic separate from I/O.
- Consistent structure: follow existing project patterns. Don't introduce a new pattern without justifying why the existing one is insufficient.
- Minimize coupling: depend on abstractions (interfaces, protocols), not concrete implementations.
