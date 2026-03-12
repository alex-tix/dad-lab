# DAD Lab 1: Rust Introduction

## Installation (Linux)

- **The Tool:** We use `rustup`, the official toolchain multiplexer, to
  install and manage Rust versions.

- **Install Command:** Run this in your terminal to download and run the
  setup script:

  ```
  curl --proto '=https' --tlsv1.2 -sSf [https://sh.rustup.rs](https://sh.rustup.rs) | sh
  ```

  - *Note for demo: Just select the default installation (option 1) when
    prompted.*

- **Applying PATH Changes:** The installer modifies your `PATH`, but you
  need to reload your shell environment to use the tools immediately:

  ```
  source $HOME/.cargo/env
  ```

- **Linker Dependency:** Rust requires a C linker to assemble the
  compiled outputs. If students get linking errors later, they'll need a
  C compiler (e.g., `sudo apt install build-essential` on
  Ubuntu/Debian).

- **Verify Installation:** Check that the Rust compiler (`rustc`) is
  successfully installed:

  ```
  rustc --version
  ```

  - *Expected output format: `rustc x.y.z (hash date)`*

## Hello, World! (Compiling with `rustc`)

- **Setup:** Create a directory for the project and a file named
  `main.rs`. Rust files always end in `.rs` (use underscores for
  multi-word filenames).

- **The Code:** Write this in `main.rs`:

  ```
  fn main() {
      println!("Hello, world!");
  }
  ```

- **Anatomy of the Program:**

  - `fn main()`: The entry point. Code execution always begins here.

  - `println!`: Prints text to the console. The `!` means you are
    calling a **macro** instead of a normal function (rules for macros
    are slightly different).

  - `;`: Indicates the end of the statement.

- **Compilation (Ahead-of-Time):** Rust is an AOT compiled language. You
  can compile a program and give the executable to someone else to run
  without them needing Rust installed.

  - Compile command:

    ```
    rustc main.rs
    ```

  - This outputs an executable binary file named `main` (on
    Linux/macOS).

- **Execution:** Run the generated binary:

  ```
  ./main
  ```

## Hello, Cargo! (Rust's Build System & Package Manager)

- **What is Cargo?** The official tool to build your code, download
  dependencies, and build those dependencies. It's standard for almost
  all Rust projects.

- **Creating a Project:**

  ```
  cargo new hello_cargo
  cd hello_cargo
  ```

  - This generates a new directory, initializes a Git repository, and
    creates `Cargo.toml` and a `src` folder.

- **The Configuration File (`Cargo.toml`):**

  - Uses TOML (Tom's Obvious, Minimal Language) format.

  - Contains `[package]` details (name, version, edition).

  - Contains `[dependencies]` (empty for now, but where external
    packages/crates will go).

- **Project Structure:**

  - Source code goes inside the `src` directory (Cargo generated
    `src/main.rs` for you).

  - The top-level directory is strictly for READMEs, license info, and
    configurations.

- **Building the Project:**

  ```
  cargo build
  ```

  - Compiles the project and places the unoptimized binary in
    `target/debug/hello_cargo`.

  - Run it manually: `./target/debug/hello_cargo`.

- **Building and Running (The Shortcut):**

  ```
  cargo run
  ```

  - Compiles the code (if it has changed) and immediately runs the
    resulting executable. This is what you'll use most of the time.

- **Checking Compilation (Fast):**

  ```
  cargo check
  ```

  - Verifies if your code compiles without actually producing an
    executable.

  - *Tip for students:* Run this frequently while writing code to catch
    errors quickly without waiting for a full build.

- **Building for Release:**

  ```
  cargo build --release
  ```

  - Compiles with optimizations. Takes longer to compile, but the
    resulting binary will run much faster.

  - Artifacts are placed in `target/release/` instead of
    `target/debug/`.

## Programming a Guessing Game

- **Setup:** Start a new project: `cargo new guessing_game` and open
  `src/main.rs`.

### Step 1: Getting User Input

- **The Code:**

  ```
  use std::io;

  fn main() {
      println!("Guess the number!");
      println!("Please input your guess.");

      let mut guess = String::new();

      io::stdin()
          .read_line(&mut guess)
          .expect("Failed to read line");

      println!("You guessed: {guess}");
  }
  ```

- **Concepts to explain:**

  - `use std::io;`: Importing the standard input/output library. (By
    default, Rust only brings a few items into scope, known as the
    *prelude*).

  - `let mut guess`: Variables in Rust are **immutable by default**.
    Using `mut` makes them mutable.

  - `String::new()`: `::` indicates an *associated function* (like a
    static method) of the `String` type.

  - `&mut guess`: The `&` denotes a **reference**, allowing multiple
    parts of code to access data without copying. References, like
    variables, are immutable by default, so we need `&mut`.

  - `.expect()`: `read_line` returns a `Result` enum (either `Ok` or
    `Err`). If it's `Err`, `expect` crashes the program and shows your
    message. If you omit it, the compiler warns you about unhandled
    errors!

  - `println!("{guess}")`: String interpolation. You can also do
    `println!("You guessed: {}", guess);`.

### Step 2: Generating a Secret Number

- **Adding a Dependency (`Cargo.toml`):**

  - Under `[dependencies]`, add: `rand = "0.8.5"`

  - Explain that `cargo build` will now fetch this crate (package) from
    crates.io and update the `Cargo.lock` file to freeze the version.

- **The Code (Update `main.rs`):**

  ```
  use std::io;
  use rand::Rng; // Adds methods that random number generators implement

  fn main() {
      println!("Guess the number!");

      let secret_number = rand::thread_rng().gen_range(1..=100);
      println!("The secret number is: {secret_number}"); // (Remove this later)

      println!("Please input your guess.");
      // ... (keep previous input code) ...
  }
  ```

- **Concepts to explain:**

  - `use rand::Rng`: A trait that must be in scope to use methods like
    `gen_range`.

  - `1..=100`: A range expression (inclusive on upper and lower bounds).

### Step 3: Comparing the Guess

- **The Code (Update `main.rs`):**

  ```
  use rand::Rng;
  use std::cmp::Ordering;
  use std::io;

  fn main() {
      // ... (keep setup code) ...

      let mut guess = String::new();
      io::stdin().read_line(&mut guess).expect("Failed to read line");

      // SHADOWING to convert String to u32
      let guess: u32 = guess.trim().parse().expect("Please type a number!");

      println!("You guessed: {guess}");

      match guess.cmp(&secret_number) {
          Ordering::Less => println!("Too small!"),
          Ordering::Greater => println!("Too big!"),
          Ordering::Equal => println!("You win!"),
      }
  }
  ```

- **Concepts to explain:**

  - `std::cmp::Ordering`: An enum with variants `Less`, `Greater`, and
    `Equal`.

  - `match` expression: The heart of Rust's control flow. It compares a
    value against a series of patterns and executes the matching arm.
    (Think of it as a super-powered `switch`).

  - **Shadowing:** We re-declare `let guess`. This lets us reuse the
    variable name rather than forcing us to create `guess_str` and
    `guess_num`.

  - `.trim().parse()`: `trim` removes the newline (`\n`) added when the
    user pressed Enter. `parse` converts the string to another type. We
    specify `: u32` to tell Rust exactly *what* type to parse it into.

### Step 4: Looping and Refinement

- **The Final Code:**

  ```
  use rand::Rng;
  use std::cmp::Ordering;
  use std::io;

  fn main() {
      println!("Guess the number!");
      let secret_number = rand::thread_rng().gen_range(1..=100);

      loop {
          println!("Please input your guess.");
          let mut guess = String::new();

          io::stdin()
              .read_line(&mut guess)
              .expect("Failed to read line");

          // Handling invalid input with match instead of expect()
          let guess: u32 = match guess.trim().parse() {
              Ok(num) => num,
              Err(_) => continue, // '_' is a catch-all value
          };

          println!("You guessed: {guess}");

          match guess.cmp(&secret_number) {
              Ordering::Less => println!("Too small!"),
              Ordering::Greater => println!("Too big!"),
              Ordering::Equal => {
                  println!("You win!");
                  break; // Exit the loop
              }
          }
      }
  }
  ```

- **Concepts to explain:**

  - `loop`: Creates an infinite loop.

  - `break`: Exits the loop when the user guesses correctly.

  - **Handling `Result` instead of crashing:** We swap `.expect()` for a
    `match` statement. If `parse()` succeeds (`Ok`), it returns the
    number. If it fails (`Err`), we use `continue` to jump to the next
    iteration of the loop, ignoring the crash.

  - `_` (underscore): A catch-all pattern meaning "ignore the specific
    error information here".

## Variables and Mutability

- **Immutability by Default:** By default, once a value is bound to a
  name, it can't be changed. (Think of it like `val` in Kotlin). This
  encourages safety and easy concurrency.

  ```
  fn main() {
      let x = 5;
      println!("The value of x is: {x}");
      x = 6; // ERROR: cannot assign twice to immutable variable
  }
  ```

- **Making Variables Mutable:** Add `mut` to allow the value to be
  reassigned (like `var` in Kotlin).

  ```
  fn main() {
      let mut x = 5;
      println!("The value of x is: {x}");
      x = 6; // This works!
      println!("The value of x is: {x}");
  }
  ```

- **Constants:** Declared with `const`. They are *always* immutable (no
  `mut` allowed), require explicit type annotation, can be declared in
  any scope (including global), and must be set to expressions that can
  be computed at compile-time.

  ```
  const THREE_HOURS_IN_SECONDS: u32 = 60 * 60 * 3;
  ```

- **Shadowing:** You can declare a new variable with the same name as a
  previous one using `let`. The new variable "shadows" the old one.

  ```
  fn main() {
      let x = 5;
      let x = x + 1; // x is now 6

      {
          let x = x * 2; // In this inner scope, x is 12
          println!("Inner scope x is: {x}");
      }

      println!("Outer scope x is: {x}"); // Back to 6 here
  }
  ```

- **Shadowing vs. Mutability:** Shadowing uses `let`, meaning you are
  creating a *brand new variable*. This allows you to perform
  transformations but keep the result immutable, and crucially, it
  allows you to **change the type**.

  ```
  fn main() {
      // Allowed via shadowing:
      let spaces = "   "; // Type is String
      let spaces = spaces.len(); // Type is now usize (number)

      // This would FAIL with `mut` because mut cannot change types:
      // let mut spaces = "   ";
      // spaces = spaces.len(); // ERROR: expected &str, found usize
  }
  ```

## Data Types

- **Statically Typed:** Rust must know the type of all variables at
  compile time. It usually infers the type, but sometimes needs explicit
  annotations if multiple types are possible.

  ```
  // Explicit type annotation is required here, otherwise compilation fails
  let guess: u32 = "42".parse().expect("Not a number!");
  ```

### Scalar Types (Single Values)

- **Integers:** Numbers without fractions.

  - Defaults to `i32` (signed 32-bit).

  - Variants exist for signed (`i8` to `i128`) and unsigned (`u8` to
    `u128`).

  - `isize` and `usize` depend on the architecture (e.g., 64-bit on an
    x64 machine) and are mainly used for indexing collections.

  - Visual separators are allowed: `let x = 98_222;`

- **Floating-Point:** Numbers with decimals (IEEE-754).

  - `f32` (single precision) and `f64` (double precision).

  - Defaults to `f64` because it's roughly as fast as `f32` on modern
    CPUs but more precise.

  ```
  let x = 2.0; // f64
  let y: f32 = 3.0; // f32
  ```

- **Numeric Operations:** The standard operators `+`, `-`, `*`, `/`, `%`
  all work exactly as you'd expect.

  - *Note:* Integer division truncates towards zero (e.g., `-5 / 3` is
    `-1`).

- **Booleans:** `true` or `false`. Sized at 1 byte.

  ```
  let t = true;
  let f: bool = false;
  ```

- **Characters (`char`):** Represents a single Unicode Scalar Value.

  - Uses single quotes (`'z'`, `'😻'`).

  - Sized at 4 bytes (unlike `char` in some older C/C++ contexts, this
    handles emojis and specialized scripts natively).

### Compound Types (Multiple Values)

- **Tuples:** Fixed length, can contain **different types**. (Similar to
  Python tuples).

  - Accessed by index using a dot (`.`), not brackets.

  - Can be destructured directly into variables.

  ```
  let tup: (i32, f64, u8) = (500, 6.4, 1);

  // Destructuring
  let (x, y, z) = tup;
  println!("The value of y is: {y}");

  // Direct indexing
  let five_hundred = tup.0;
  ```

  - *Note:* An empty tuple `()` is called the **unit type** and
    represents an empty value/return type.

- **Arrays:** Fixed length, but must contain the **same type** for all
  elements. Data is allocated on the stack.

  - *Contrast:* Unlike Kotlin's `ArrayList` or Python's `list`, Rust
    arrays cannot grow or shrink. (For dynamic sizing, Rust uses
    `Vector`s, which come later).

  ```
  let a = [1, 2, 3, 4, 5];

  // Explicit type: [type; length]
  let b: [i32; 5] = [1, 2, 3, 4, 5];

  // Initialize with the same value: [value; length]
  let c = [3; 5]; // Gives [3, 3, 3, 3, 3]
  ```

  - **Access:** Uses standard indexing syntax (`a[0]`).

  - **Safety:** If you try to access an index out of bounds, Rust will
    panic and crash the program *at runtime* (rather than allowing
    unsafe memory access like C).

## Functions

- **Basic Syntax:** Defined with the `fn` keyword. Rust uses
  `snake_case` for function and variable names.

  ```
  fn main() {
      print_hello();
  }

  fn print_hello() {
      println!("Hello, world!");
  }
  ```

  - *Note:* Order doesn't matter. As long as `print_hello` is defined
    *somewhere* in scope, `main` can call it.

- **Parameters:** You **must** declare the type of each parameter. (Rust
  won't infer function signatures).

  ```
  fn print_labeled_measurement(value: i32, unit_label: char) {
      println!("The measurement is: {value}{unit_label}");
  }
  ```

- **Statements vs. Expressions (Crucial Concept):**

  - **Statements:** Instructions that perform some action and do *not*
    return a value. They end with a semicolon `;`. (e.g., `let y = 6;`).
    You cannot do `let x = (let y = 6);`.

  - **Expressions:** Evaluate to a resulting value. Calling a function,
    calling a macro, or even a new scope block `{}` are expressions.

  - *Rule of thumb:* Expressions do **not** end with semicolons. If you
    add a semicolon to an expression, you turn it into a statement, and
    it stops returning a value.

  ```
  fn main() {
      let y = {
          let x = 3;
          x + 1 // No semicolon here! This block is an expression that evaluates to 4.
      };
      println!("The value of y is: {y}");
  }
  ```

- **Return Values:** \* Declared with an arrow `->` after the parameter
  list.

  - In Rust, the return value of the function is synonymous with the
    value of the **final expression** in the block of the function body.
    (Similar to how `if` or `when` blocks can return values in Kotlin).

  ```
  fn five() -> i32 {
      5 // Implicit return. No `return` keyword, no semicolon!
  }

  fn plus_one(x: i32) -> i32 {
      x + 1 
      // If you added a semicolon here (x + 1;), it becomes a statement, 
      // returns `()` (the unit type), and the compiler will throw an error!
  }
  ```

  - You *can* still use the `return` keyword for early returns, but
    idiomatic Rust usually just relies on the final expression for the
    main return.

## Control Flow

### `if` Expressions

- **Basic Syntax:**

  ```
  let number = 3;
  if number < 5 {
      println!("condition was true");
  } else {
      println!("condition was false");
  }
  ```

- **Strict Booleans:** The condition **must** evaluate to a `bool`. Rust
  will *not* automatically convert non-boolean types to booleans (there
  is no "truthy" or "falsy" like in Python or C). `if number { ... }`
  will throw a compiler error.

- **`else if`:** Works exactly as you expect for handling multiple
  conditions.

- **Using `if` in a `let` Statement:** Because `if` is an expression
  (just like in Kotlin), you can assign its result to a variable.

  ```
  let condition = true;
  // This is identical to: val number = if (condition) 5 else 6 in Kotlin
  let number = if condition { 5 } else { 6 };
  ```

  - *Important Rule:* The return types of all branches must perfectly
    match. You cannot do `if condition { 5 } else { "six" }`.

### Repetition with Loops

- **`loop` (Infinite):** Executes a block of code forever until you
  explicitly stop it.

  ```
  loop {
      println!("again!");
      break; // Stops the loop
  }
  ```

  - **Returning values from loops:** You can return a value out of a
    `loop` by putting it after the `break` keyword.

    ```
    let mut counter = 0;
    let result = loop {
        counter += 1;
        if counter == 10 {
            break counter * 2; // Returns 20 into the 'result' variable
        }
    };
    ```

  - **Loop Labels:** If you have nested loops, `break` or `continue`
    only apply to the innermost loop. You can prefix a loop with a label
    (starting with a single quote) to break out of a specific outer
    loop.

    ```
    'outer: loop {
        loop {
            break 'outer; // Breaks the outer loop, not just the inner one
        }
    }
    ```

- **`while` Loops (Conditional):** Runs as long as a condition evaluates
  to `true`.

  ```
  let mut number = 3;
  while number != 0 {
      println!("{number}!");
      number -= 1;
  }
  println!("LIFTOFF!!!");
  ```

- **`for` Loops (Collections & Ranges):** The safest and most idiomatic
  loop in Rust. Rather than manually tracking an index (which can cause
  index-out-of-bounds panics), use `for` to iterate directly over items.
  (Very similar to Python's `for item in list:` or Kotlin's
  `for (item in collection)`).

  - **Iterating an Array:**

    ```
    let a = [10, 20, 30, 40, 50];
    for element in a {
        println!("the value is: {element}");
    }
    ```

  - **Iterating a Range:** To loop a specific number of times, use
    ranges (`start..end`, exclusive of end). You can chain methods like
    `.rev()` to reverse it.

    ```
    for number in (1..4).rev() {
        println!("{number}!"); // Prints 3, 2, 1
    }
    ```

## Understanding Ownership

- **The Core Premise:** This is Rust's most unique feature. It
  guarantees memory safety *without* needing a garbage collector (like
  Python/Kotlin) or forcing you to manually allocate/free memory (like
  C/C++). Memory is managed through a system of rules verified at
  **compile time**.

- **Stack vs. Heap (Quick Refresher):**

  - **Stack:** Fast, strictly LIFO. All data stored here must have a
    known, fixed size (e.g., `i32`, `bool`, arrays).

  - **Heap:** Slower. Used for data with a dynamic size at runtime. The
    OS allocates space and returns a pointer (e.g., `String`). Following
    pointers makes heap access slower than stack access.

- **The 3 Rules of Ownership:**

  1.  Each value in Rust has an **owner**.

  2.  There can only be **one owner at a time**.

  3.  When the owner goes out of scope, the value will be **dropped**
      (memory is freed).

### Scope and Memory Allocation

- **Variable Scope:** Like most languages, variables are valid from the
  point they are declared until the end of the current block `{}`.

- **The `String` Type:** To illustrate the heap, we use `String` (which
  is mutable and dynamically sized), rather than string literals
  (`&str`, which are hardcoded).

  ```
  {
      let s = String::from("hello"); // Memory requested from the OS here
      // do stuff with s
  } // Scope ends. Rust automatically calls `drop` to return the memory.
  ```

### Data Interaction: Move vs. Copy

- **Stack Data (Copy):** Simple types of known size are entirely on the
  stack.

  ```
  let x = 5;
  let y = x;
  // Both x and y equal 5. A fast bitwise copy was made. Both are valid.
  ```

- **Heap Data (Move - The crucial difference!):** In Python or Kotlin,
  assigning an object to a new variable just copies the reference. If
  Rust did this, when both went out of scope, it would try to free the
  same memory twice (a "double free" error).

  ```
  let s1 = String::from("hello");
  let s2 = s1; // s1 is MOVED to s2.

  // println!("{}, world!", s1); // THIS WILL CAUSE A COMPILE ERROR!
  ```

  - Rust considers `s1` **invalid** after it is assigned to `s2`. You
    can no longer use it. This solves the double-free problem
    automatically!

- **Deep Copying (Clone):** If you *actually* want to duplicate the heap
  data (an expensive operation), you use `.clone()`.

  ```
  let s1 = String::from("hello");
  let s2 = s1.clone();
  println!("s1 = {}, s2 = {}", s1, s2); // This works fine!
  ```

### Ownership and Functions

- **Passing Variables:** Passing a variable into a function works
  exactly like assigning it to another variable: it will either **move**
  (for heap types) or **copy** (for stack types).

  ```
  fn main() {
      let s = String::from("hello");
      takes_ownership(s);
      // s is NO LONGER VALID here. It was moved into the function.

      let x = 5;
      makes_copy(x);
      // x IS STILL VALID here because i32 is Copy.
  }

  fn takes_ownership(some_string: String) {
      println!("{}", some_string);
  } // some_string goes out of scope and memory is dropped.

  fn makes_copy(some_integer: i32) {
      println!("{}", some_integer);
  }
  ```

- **Return Values:** Returning a value transfers ownership *back* to the
  caller.

  ```
  fn main() {
      let s1 = gives_ownership(); // Function creates string, moves it to s1

      let s2 = String::from("hello");
      let s3 = takes_and_gives_back(s2); // s2 moved in, then moved out to s3
  }
  // s3 and s1 are dropped here. s2 was already moved, so nothing happens to it.
  ```

  *(Note: Constantly passing ownership back and forth is tedious, which
  is why Rust introduces **References**, coming in the next chapter).*

## References and Borrowing

- **The Problem:** Passing ownership into functions and returning it
  back every time is annoying.

- **The Solution (Borrowing):** We can pass a **reference** to a value
  instead. A reference is like a pointer in that it's an address we can
  follow to access the data, but unlike a generic pointer, Rust
  guarantees that a reference points to a valid value of a particular
  type.

### Immutable References (`&`)

- **Creating a Reference:** Use the ampersand (`&`). The action of
  creating a reference is called **borrowing**.

  ```
  fn main() {
      let s1 = String::from("hello");
      let len = calculate_length(&s1); // We pass a REFERENCE to s1

      // s1 is STILL VALID here because we didn't give away ownership!
      println!("The length of '{}' is {}.", s1, len);
  }

  // The parameter type `&String` means "a reference to a String"
  fn calculate_length(s: &String) -> usize {
      s.len()
  } // s goes out of scope, but because it doesn't own the data it refers to, it's not dropped.
  ```

- **Modification is Forbidden:** Just like variables are immutable by
  default, references are also immutable by default. We cannot modify
  something we are borrowing.

  ```
  fn change(some_string: &String) {
      // some_string.push_str(", world"); // THIS CAUSES A COMPILE ERROR
  }
  ```

### Mutable References (`&mut`)

- **Allowing Modification:** To modify a borrowed value, we need a
  mutable reference.

  1.  The original variable must be `mut`.

  2.  You must pass it using `&mut`.

  3.  The function signature must accept `&mut`.

  ```
  fn main() {
      let mut s = String::from("hello");
      change(&mut s);
      println!("{}", s); // Prints "hello, world"
  }

  fn change(some_string: &mut String) {
      some_string.push_str(", world");
  }
  ```

### The Rules of Borrowing (Crucial!)

Rust enforces two strict rules at compile time to prevent memory issues
and **data races** (which is why Rust is legendary for fearless
concurrency):

**Rule 1: The "One Mutable Reference" Restriction**

- At any given time, you can have **EITHER**:

  - Exactly *one* mutable reference (`&mut T`).

  - *Any number* of immutable references (`&T`).

- You cannot have both at the same time in the same scope.

  ```
  let mut s = String::from("hello");

  let r1 = &s; // OK (Immutable)
  let r2 = &s; // OK (Immutable)
  println!("{} and {}", r1, r2); // We can read them both

  // let r3 = &mut s; // ERROR: Cannot borrow `s` as mutable because it is also borrowed as immutable!
  ```

- *Why?* If someone is holding a read-only pointer to data, they expect
  that data not to change underneath them. If we allowed a mutable
  pointer simultaneously, the mutable pointer could change or reallocate
  the memory, breaking the read-only pointers.

**Rule 2: No Dangling References**

- A dangling pointer is a pointer that references memory that has been
  given to someone else or freed.

- Rust's compiler guarantees this will **never** happen. It ensures data
  will not go out of scope before the reference to the data does.

  ```
  fn main() {
      // let reference_to_nothing = dangle(); // ERROR
  }

  /* // THIS CODE WILL NOT COMPILE
  fn dangle() -> &String { 
      let s = String::from("hello"); // s is created here
      &s // We return a reference to s
  } // s goes out of scope and is dropped. The memory is gone. Danger! 
  */
  ```

  - *The Fix:* Instead of returning a reference in `dangle`, you just
    return the `String` directly to move ownership out of the function.

## Defining and Instantiating Structs

- **What are Structs?** Custom data types that let you package together
  and name multiple related values. (Very similar to Kotlin's
  `data class` or Python's `dataclass` / dictionaries, but strongly
  typed).

### Defining a Struct

- **Syntax:** Use the `struct` keyword followed by curly braces holding
  `name: type` pairs (fields).

  ```
  struct User {
      active: bool,
      username: String,
      email: String,
      sign_in_count: u64,
  }
  ```

### Instantiating a Struct

- **Creation:** Create an instance by specifying the struct name and
  providing key-value pairs inside curly braces. Order does not matter.

  ```
  let mut user1 = User {
      email: String::from("someone@example.com"),
      username: String::from("someusername123"),
      active: true,
      sign_in_count: 1,
  };
  ```

- **Access and Mutability:** \* Use dot notation to access fields:
  `user1.email`.

  - **Important Difference from Kotlin:** You cannot mark individual
    fields as mutable or immutable. Mutability is a property of the
    *entire instance binding*. If the instance is declared with `mut`,
    all fields are mutable. If it's not, none are.

  ```
  user1.email = String::from("anotheremail@example.com"); // Only works because user1 is `mut`
  ```

### Field Init Shorthand

- If your variable names exactly match the struct's field names, you can
  skip typing the names twice. (Just like modern JavaScript object
  literals).

  ```
  fn build_user(email: String, username: String) -> User {
      User {
          active: true,
          username, // Shorthand for username: username
          email,    // Shorthand for email: email
          sign_in_count: 1,
      }
  }
  ```

### Struct Update Syntax (`..`)

- Allows creating a new struct instance by copying values from an
  existing one, overriding only what you specify. (Conceptually
  identical to Kotlin's `data_class.copy(field = new_value)`).

  ```
  let user2 = User {
      email: String::from("another@example.com"),
      ..user1 // Copies the remaining fields (active, username, sign_in_count) from user1
  };
  ```

- **The Ownership Trap:** The `..user1` syntax acts like an assignment
  (`=`). Because `String` does not implement `Copy`, the `username`
  field is **moved** from `user1` to `user2`.

  - After this operation, `user1` is partially invalid. You can no
    longer access `user1.username` or use `user1` as a whole, though its
    other fields (`active` and `sign_in_count`, which implement `Copy`)
    remain accessible individually.

### Tuple Structs

- Structs that look like tuples. They have an overall type name, but
  their fields don't have names, only types.

- Useful when you want to create distinct types to leverage the
  compiler's type checking, even if the data structure is identical.

  ```
  struct Color(i32, i32, i32);
  struct Point(i32, i32, i32);

  let black = Color(0, 0, 0);
  let origin = Point(0, 0, 0);
  // Color and Point are totally different types. You cannot pass a Color to a function expecting a Point!
  ```

- Accessed using dot notation and indices: `black.0`, `black.1`, etc.

### Unit-Like Structs

- Structs that have absolutely no fields.

  ```
  struct AlwaysEqual;

  let subject = AlwaysEqual;
  ```

- Mostly used when you need to implement a trait (like an interface) on
  some type, but you don't actually have any data that needs to be
  stored in the type itself.

## Defining an Enum

- **What are Enums?** They allow you to define a type by enumerating its
  possible *variants*.

- **The Kotlin Connection:** In Rust, enums are essentially Kotlin's
  **`sealed class`es**. They are incredibly powerful because each
  variant can store different amounts and types of data directly inside
  it.

### Basic Enums

- **Defining:** Use the `enum` keyword.

  ```
  enum IpAddrKind {
      V4,
      V6,
  }
  ```

- **Instantiating:** Variants are namespaced under their enum identifier
  using a double colon `::`.

  ```
  let four = IpAddrKind::V4;
  let six = IpAddrKind::V6;
  ```

  *(Both `four` and `six` are of the exact same type: `IpAddrKind`)*.

### Enums with Data (The Superpower)

- Instead of creating an enum just to track the "kind" of data and a
  separate struct to hold the actual data, you can embed the data
  directly into the enum variants.

  ```
  enum IpAddr {
      V4(String),
      V6(String),
  }

  let home = IpAddr::V4(String::from("127.0.0.1"));
  let loopback = IpAddr::V6(String::from("::1"));
  ```

- **Different Data Structures:** Each variant can have different types
  and amounts of associated data (this is why they are like Kotlin's
  sealed classes!).

  ```
  enum Message {
      Quit,                       // No data at all
      Move { x: i32, y: i32 },    // Has an anonymous struct inside
      Write(String),              // Contains a single String
      ChangeColor(i32, i32, i32), // Contains three i32 values
  }
  ```

  - *Why do this over Structs?* If we used four different structs, we
    couldn't easily write a single function that takes *any* of those
    messages. By using an enum, they all share the overarching `Message`
    type.

### Methods on Enums

- Just like structs, you can define methods on enums using `impl`.

  ```
  impl Message {
      fn call(&self) {
          // Method body would go here
      }
  }

  let m = Message::Write(String::from("hello"));
  m.call();
  ```

### The `Option` Enum (Rust's Null Replacement)

- **The Problem with Null:** Rust does **not** have a `null` feature.
  Nulls cause billion-dollar mistakes (NullPointerExceptions) when you
  try to use a null value as a not-null value.

- **The Rust Solution:** It encodes the *concept* of a value being
  present or absent into a special standard library enum called
  `Option<T>`.

  ```
  // This is defined in the standard library as:
  enum Option<T> {
      None,
      Some(T),
  }
  ```

  - *(`T` is a generic type parameter, meaning it can hold any type of
    data).*

- **Usage:** It's so common that `Option`, `Some`, and `None` are
  included in the prelude (you don't have to import them or prefix them
  with `Option::`).

  ```
  let some_number = Some(5);
  let some_char = Some('e');

  // If you use None, you MUST tell the compiler the expected type, 
  // because it can't infer it from empty nothingness.
  let absent_number: Option<i32> = None; 
  ```

- **Why is this better than null?** `Option<i8>` and `i8` are
  **completely different types** to the compiler. You cannot add a
  standard integer to an `Option` integer. You *must* extract the value
  out of the `Option` first. This forces you to handle the `None` case
  explicitly before doing operations, eliminating the possibility of
  unexpected null crashes!

## Control Flow with `match`

- **What is `match`?** A powerful control flow operator that compares a
  value against a series of patterns and executes code. (Think of it as
  Kotlin's `when` statement, but deeply integrated with Rust's enums).

- **The Coin Sorter:**

  ```
  enum Coin {
      Penny,
      Nickel,
      Dime,
      Quarter,
  }

  fn value_in_cents(coin: Coin) -> u8 {
      match coin {
          Coin::Penny => 1,
          Coin::Nickel => 5,
          Coin::Dime => 10,
          Coin::Quarter => 25,
      }
  }
  ```

  - `match` is an **expression**, meaning it returns a value (the `1`,
    `5`, etc.).

### Extracting Values Bound to Patterns

- **The True Power:** `match` can extract the inner data from an enum
  variant while checking its type!

  ```
  #[derive(Debug)] // So we can inspect the state
  enum UsState {
      Alabama,
      Alaska,
      // ... etc
  }

  enum Coin {
      Penny,
      Nickel,
      Dime,
      Quarter(UsState), // Quarter now holds state data!
  }

  fn value_in_cents(coin: Coin) -> u8 {
      match coin {
          Coin::Penny => 1,
          Coin::Quarter(state) => {
              println!("State quarter from {:?}!", state);
              25
          }
          // _ (catch-all) is used to ignore the rest of the variants
          _ => 0, 
      }
  }
  ```

### Matching with `Option<T>`

- This is the standard way to safely extract values from an `Option`
  (handling the "not null" vs "null" cases).

  ```
  fn plus_one(x: Option<i32>) -> Option<i32> {
      match x {
          None => None,          // Handle the empty case
          Some(i) => Some(i + 1), // Extract the value `i`, use it, and re-wrap it
      }
  }

  let five = Some(5);
  let six = plus_one(five);
  let none = plus_one(None);
  ```

### Exhaustiveness and Catch-Alls

- **Matches MUST be Exhaustive:** The compiler will yell at you if you
  don't cover every single possible variant. This prevents unhandled
  states!

- **The Catch-All (`_`):** If you only care about a few cases and want
  to ignore the rest, use the `_` placeholder.

  ```
  let dice_roll = 9;
  match dice_roll {
      3 => add_fancy_hat(),
      7 => remove_fancy_hat(),
      _ => reroll(), // Matches anything else. (Use a variable name instead of `_` if you need the value)
  }
  ```

  - *Note:* You can also use `_ => ()` to explicitly do *nothing* for
    the default cases.

## Concise Control Flow with `if let`

- **The Problem:** Sometimes a `match` expression is too verbose when
  you *only* care about matching one specific variant and want to ignore
  everything else.

- **The Solution:** `if let` is syntactic sugar for a `match` that runs
  code when the value matches one pattern and then ignores all other
  values.

  ```
  // Verbose match way:
  let config_max = Some(3u8);
  match config_max {
      Some(max) => println!("The maximum is configured to be {}", max),
      _ => (),
  }

  // Concise `if let` way:
  let config_max = Some(3u8);
  if let Some(max) = config_max {
      println!("The maximum is configured to be {}", max);
  }
  ```

- **Trade-off:** You lose the exhaustive checking that `match` forces
  upon you, but you gain brevity.

- **`else` support:** You can attach an `else` block to an `if let`,
  which acts exactly like the `_` (catch-all) arm in a `match`.
