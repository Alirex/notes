# Why in search used "-1" if nothing is found? Why not "None" or exception or something else?

---

The use of `-1` to represent "not found" is largely a historical convention rooted in how computers count and how older
programming languages manage memory. While modern languages often use `None`, `null`, or `Optional` types, `-1` remains
common for specific reasons.

Here is the breakdown of why `-1` became the standard, why it wasn't `None`, and why we don't use Exceptions.

---

### 1. Zero-Based Indexing (The Logic)

In most programming languages (C, Java, Python, JavaScript), arrays and lists use **0-based indexing**. This means the
first item is at index `0`, the second at `1`, and so on.

Because `0` is a valid location for an item, the function needs to return a number that is **guaranteed** to never be a
valid index to signal "failure."

- **0 to Infinity:** Valid indices.
- **-1:** The first integer that serves as an invalid index.

If arrays started at 1 (like in Lua or older Fortran), then `0` would likely have been used as the "not found" code.

### 2\. Static Typing (Why not "None"?)

This convention comes from C and C++ (the "grandparents" of many modern languages). In these languages, functions are \*
\*statically typed\*\*.

If you write a function to find a position, you must define the return type strictly as an integer (`int`).

```c
int findIndex(char target) {
    // This function MUST return a number.
    // It cannot return 'null' or 'false' or 'None'.
}
```

Because the computer is expecting a number, you cannot return `None` (which is an object or a different data type). You
are forced to return a number that carries a special meaning. `-1` became that number.

### 3. Performance (Why not Exceptions?)

You asked why we don't raise an **Exception** (error). The answer is **speed** and **semantics**.

- **Performance:** Throwing an exception is computationally "expensive." It forces the computer to pause, unwrap the
  stack, and look for an error handler. Searching is often done inside tight loops (thousands of times per second);
  throwing errors would drastically slow down the program.
- **Semantics (Meaning):** In programming philosophy, exceptions are for _exceptional_ circumstances (things that
  shouldn't happen, like a crashed hard drive). Not finding an item in a search is a normal, expected outcome. It isn't
  an "error"; it's just a result.

### 4. Mathematical Convenience (Bitwise Operations)

In the past, developers used a bitwise trick with the Not operator (`~`).
The bitwise complement of `-1` is `0`.

- If `x` is `-1`, then `~x` is `0` (False).
- If `x` is `0` or higher, `~x` is non-zero (True).

This allowed C programmers to write very concise (though hard to read) code like:

```c
if (~indexOf(item)) {
    // Found it!
}
```

---

### Modern Approaches: Moving away from -1

Newer languages and paradigms are slowly fixing this because `-1` can be confusing (it looks like a magic number).

| Approach              | Language Example   | Description                                                                                                             |
| :-------------------- | :----------------- | :---------------------------------------------------------------------------------------------------------------------- |
| **Option / Optional** | Rust, Swift, Java  | The function returns a wrapper that says "I might have an Int, or I might be Empty." This is the safest modern way.     |
| **Union Types**       | TypeScript, Python | You can say the function returns `int \| None`. This allows returning `None` instead of -1.                             |
| **Iterators/Find**    | JavaScript         | JS `indexOf` returns `-1`, but the newer `find()` returns `undefined` if nothing is found, which is much more readable. |

### Summary

We use `-1` because:

1. **0** is taken (it's the first element).
2. **Types** required an integer return value (in older languages).
3. **Speed** prevents us from using Exceptions.
