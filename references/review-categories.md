# Review Categories — Extended Examples and Antipatterns

Reference material for interpreting and enriching Cursor's review output. Consult this when the Cursor review flags an issue and additional context or validation is needed.

## Code Correctness — Common Patterns

### Logic Errors
- Inverted boolean conditions: `if (!isValid)` when `if (isValid)` was intended
- Incorrect comparison: using `==` instead of `===` in JavaScript (type coercion)
- Missing `break` in switch statements causing fallthrough
- Incorrect ternary: `condition ? b : a` when `condition ? a : b` was intended
- Off-by-one in loop bounds: `i <= arr.length` instead of `i < arr.length`

### Async Errors
- Missing `await` on async function calls (returns Promise instead of value)
- Using `.then()` without `.catch()` (unhandled rejection)
- Race condition from parallel mutations to shared state
- Calling `setState` after component unmount in React
- Deadlocks from nested locks or incorrect lock ordering

### Type Errors
- Unsafe type assertions (`as any`, `as unknown as T`)
- Missing null checks after operations that can return null/undefined
- Incorrect generic constraints allowing invalid types
- Implicit `any` types hiding real errors

## Bugs — Red Flags

### Null/Undefined
- `.find()` result used without checking for undefined
- Optional chaining missing on potentially null object access
- `document.querySelector()` result used directly without null check
- Destructuring from potentially undefined object

### Resource Leaks
- `setInterval` / `setTimeout` without corresponding `clearInterval` / `clearTimeout`
- Event listeners added without removal in cleanup
- Database connections opened without closing in error paths
- File handles not closed in finally/dispose blocks
- WebSocket connections without close handling

### State Issues
- Stale closure capturing old value in React `useEffect` / `useCallback`
- Mutating function parameters (especially objects/arrays)
- Shared mutable state between async operations
- Redux: mutating state directly instead of returning new object

## Security — Quick Checks

### Injection
- String concatenation in SQL: `` `SELECT * FROM users WHERE id = ${userId}` ``
- `eval()` or `new Function()` with user input
- `child_process.exec()` with unsanitized input (command injection)
- Template literal injection in HTML (XSS)
- `innerHTML`, `outerHTML`, or `document.write()` with user data

### Authentication & Authorization
- JWT stored in localStorage (vulnerable to XSS)
- Missing authorization checks on API endpoints
- Hardcoded API keys, passwords, or tokens in source code
- Password comparison not using constant-time comparison
- Missing CSRF protection on state-changing endpoints
- Session tokens in URL parameters

### Data Exposure
- Logging sensitive data (passwords, tokens, PII)
- Verbose error messages in production (stack traces, internal paths)
- API responses including more data than the client needs
- Missing rate limiting on authentication endpoints
- Credentials in environment files committed to git

## Tests — Quality Indicators

### Weak Tests
- `expect(result).toBeTruthy()` instead of specific value checks
- Tests that always pass (no real assertions, or assertions after early return)
- Snapshot tests without understanding what they capture
- Testing framework behavior instead of application logic
- Only testing happy path, never error conditions

### Missing Coverage
- No tests for error/exception paths
- No boundary value tests (0, -1, MAX_INT, empty string)
- No tests for concurrent/async behavior
- No integration tests for API endpoints
- No tests for authorization/permission checks

### Brittle Tests
- Tests coupled to implementation details (internal method calls, private state)
- Tests depending on specific database IDs or timestamps
- Tests failing when run in different order or in parallel
- Hardcoded file paths or environment-specific values

## Performance — Antipatterns

### Database
- N+1 queries: querying related records inside a loop instead of batch/join
- Missing indexes on frequently queried columns
- `SELECT *` when only specific columns are needed
- Loading entire tables without pagination or limits
- Unnecessary eager loading of relations

### Memory
- Growing arrays/maps without bounds or eviction
- Creating closures that capture large objects
- String concatenation in tight loops (use StringBuilder/join)
- Caching without TTL or size limits
- Detached DOM nodes still referenced in JavaScript

### Algorithmic
- Nested loops producing O(n^2) when a Set/Map lookup gives O(n)
- Sorting when only min/max is needed (O(n log n) vs O(n))
- Recomputing derived data on every render instead of memoizing
- Using `Array.includes()` in a loop (O(n^2)) instead of Set (O(n))
- Recursive functions without memoization on overlapping subproblems

### Frontend
- Creating new objects/functions inside render (breaks reference equality)
- Missing `key` prop on list items or using array index as key
- Importing entire library when only specific functions are needed
- Missing code splitting on large route components
- Unbounded DOM list rendering without virtualization

## Error Handling — Smell Tests

### Swallowed Errors
- `catch (e) {}` — empty catch block
- `catch (e) { console.log(e) }` — logged but not handled or rethrown
- `.catch(() => null)` — silently returning null on failure
- Catching generic `Exception` when specific exceptions should be handled differently

### Missing Validation
- API endpoints accepting input without schema validation
- File uploads without type/size checking
- User input used directly in database queries
- Environment variables used without checking if defined
- Configuration parsed without schema validation

### Poor Error Messages
- "Something went wrong" — no actionable information
- Including internal error details in user-facing messages
- Error codes without documentation or mapping
- Losing original error when wrapping (no `cause` chain)

## Architecture — Warning Signs

### Structural Issues
- Files longer than 500 lines of code
- Functions longer than 50 lines
- More than 5-7 parameters on a function
- Deeply nested conditionals (3+ levels of if/else)
- Circular import dependencies between modules

### Design Issues
- God objects: one class/module handling auth, database, email, logging
- Feature envy: class methods primarily using data from another class
- Shotgun surgery: one change requires modifying 5+ files
- Business logic in controller/route handlers instead of service layer
- Direct database access from UI components
- Hardcoded configuration that should be externalized

### Dependency Issues
- High-level modules importing from low-level modules (Dependency Inversion violation)
- Tight coupling between modules that should be independent
- Missing interfaces/abstractions at module boundaries
- Concrete types used where interfaces would allow flexibility
- Utility modules with unrelated functions grouped together (low cohesion)
