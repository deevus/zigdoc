# AGENTS.md

## Design Priorities

Prioritize correctness, simplicity, and developer experience, in that order.
Prefer explicit control flow and the smallest abstraction that clearly models
the domain.

## Zig Development

Use `zigdoc` to discover current APIs for the Zig standard library and any
third-party dependencies before coding.

Examples:
```bash
zigdoc std.fs
zigdoc std.posix.getuid
```

For imported dependencies, use the module name exposed by `build.zig`:

```bash
zigdoc <module>.<symbol>
```

## Verification

- Run `zig build test` after code changes; it also checks formatting.
- Use `zig build fmt` when only a formatting check is needed.
- Run `zig build readme` when CLI usage or help text changes, and include the
  resulting `README.md` update.

## Zig Style

- Use `camelCase` for functions and methods.
- Use lower-case `snake_case` for variables, parameters, and constants.
- Use `PascalCase` for types, structs, and enums.
- Prefer `const foo: Type = .{ .field = value };` over
  `const foo = Type{ .field = value };`.
- Pass allocators explicitly; use `errdefer` for cleanup on error.
- Keep unit tests close to the code they cover. Ensure `zig build test`
  discovers and runs every test-bearing module.
- Avoid abbreviations unless they are established domain terms.
- Include units and qualifiers in names when values could otherwise be
  ambiguous, such as `timeout_ms`, `buffer_size`, or `offset_bytes`.
- Treat indexes, counts, and byte sizes as distinct concepts; name conversions
  between them explicitly.
- Use an options struct when positional arguments of the same type are easy to
  confuse.
- Make division semantics explicit with `@divExact`, `@divFloor`, or deliberate
  ceiling division when rounding affects correctness.

### Files and Types

- Treat every `.zig` file as a namespace. Make the file itself a type only when
  its root represents one primary stateful abstraction with fields and methods.
- Name a file-backed type `PascalCase.zig`, import it directly with
  `const Widget = @import("Widget.zig");`, and begin it with an optional `//!`
  container doc followed by `const Widget = @This();`. Prefer the concrete type
  name over `Self` so signatures remain clear out of context.
- Use lower-case `snake_case.zig` for namespace modules: related free functions,
  constants, multiple peer types, or package facades. Export named types from
  these modules and import them as `@import("widget.zig").Widget`.
- Do not put a sole `pub const Widget = struct { ... };` inside `Widget.zig`;
  the file root already provides that container. Conversely, do not create a
  file-backed type merely to enforce one-type-per-file organization.
- Preferred file start: `//!` container docs when needed, the file-backed type
  alias when applicable, imports and local aliases, then scoped logging
  declarations when needed.

### Comments and Documentation

- Use `//!` at the start of a nontrivial file to document the root namespace or
  file-backed type: its purpose, conceptual model, and major invariants. Omit it
  for trivial facades whose exports make the purpose obvious.
- Use `///` for declaration-level contracts. Document public APIs when the name
  and signature do not fully convey ownership and lifetime, allocation,
  mutation or pointer invalidation, errors or nullability, units and ranges,
  thread safety, side effects, or asserted preconditions. Simple re-exports and
  self-explanatory declarations do not need filler documentation.
- Use `//` for implementation rationale, state invariants, workarounds, and
  signposts for non-obvious algorithm phases. Do not narrate syntax or restate
  the code. Doc comments must not contain notes intended only for maintainers.
- Keep comments accurate when behavior changes. Prefer deleting a stale or
  redundant comment over expanding it.

### File Size

- Cohesion, not line count, decides when to split a file. As review triggers,
  prefer hand-written files below roughly 1,000 lines, actively look for a
  cohesive extraction once a file crosses that size, and treat files above
  roughly 2,000 lines as exceptional. These are not hard limits.
- Split when a file contains independently nameable responsibilities, disjoint
  subsystems, or helpers with their own invariants. Extract a real subordinate
  type, parser, formatter, platform backend, or pure algorithm rather than an
  arbitrary range of methods.
- Keep a large file intact when its declarations jointly implement one cohesive
  type and splitting would add forwarding APIs or make invariants harder to
  follow. Generated code, data tables, and version-specific compatibility
  snapshots are exempt from the size guidance.

## Safety

- Use assertions for programmer errors: API preconditions, state transitions,
  internal invariants, and relationships between compile-time constants.
  Handle expected operating errors through the error return path instead.
- Split independent assertion conditions so failures identify the violated
  invariant precisely.
- Add assertions at both sides of important representation or state boundaries
  when each side can independently verify the invariant.
- Test both valid and invalid inputs around important boundaries, including
  transitions from valid to invalid states.
- Keep functions small, centralize control flow, and push pure computation into
  helpers.
- Keep checks close to use and avoid duplicate aliases for mutable state.
- Make resource ownership obvious. Pair acquisition immediately with `defer` or
  use `errdefer` when ownership has not yet transferred.

## Current Zig Patterns

**ArrayList:**
```zig
var list: std.ArrayList(u32) = .empty;
defer list.deinit(allocator);
try list.append(allocator, 42);
```

**HashMap/StringHashMap (default to unmanaged):**
```zig
var map: std.StringHashMapUnmanaged(u32) = .empty;
defer map.deinit(allocator);
try map.put(allocator, "key", 42);
```

**stdout/stderr writer:**
```zig
var buf: [4096]u8 = undefined;
var writer = std.fs.File.stdout().writer(&buf);
defer writer.interface.flush() catch {};
try writer.interface.print("hello {s}\n", .{"world"});
```

**JSON writing:**
```zig
var buf: [4096]u8 = undefined;
var writer = std.fs.File.stdout().writer(&buf);
defer writer.interface.flush() catch {};

var jw: std.json.Stringify = .{
    .writer = &writer.interface,
    .options = .{ .whitespace = .indent_2 },
};
try jw.write(my_struct);
```

**Allocating writer:**
```zig
var writer: std.Io.Writer.Allocating = .init(allocator);
defer writer.deinit();
try writer.writer.print("hello {s}", .{"world"});
const output = try writer.toOwnedSlice();
```
