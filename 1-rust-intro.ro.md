# Laboratorul 1 DAD: Introducere in Rust

## Instalare (Linux)

- **Instrumentul:** Folosim `rustup`, multiplexorul oficial al
  toolchain-ului, pentru a instala și gestiona versiunile de Rust.

- **Comanda de instalare:** Rulează asta în terminal pentru a descărca
  și rula scriptul de instalare:

  ```
  curl --proto '=https' --tlsv1.2 -sSf [https://sh.rustup.rs](https://sh.rustup.rs) | sh
  ```

  - *Notă pentru demo: Selectează instalarea implicită (opțiunea 1) când
    ești întrebat.*

- **Aplicarea modificărilor de PATH:** Instalatorul îți modifică
  variabila `PATH`, dar trebuie să reîncarci mediul shell pentru a
  folosi instrumentele imediat:

  ```
  source $HOME/.cargo/env
  ```

- **Dependența de Linker:** Rust necesită un linker C pentru a asambla
  fișierele compilate. Dacă studenții primesc erori de linkare mai
  târziu, vor avea nevoie de un compilator C (ex:
  `sudo apt install build-essential` pe Ubuntu/Debian).

- **Verificarea instalării:** Verifică dacă compilatorul Rust (`rustc`)
  a fost instalat cu succes:

  ```
  rustc --version
  ```

  - *Formatul așteptat: `rustc x.y.z (hash date)`*

## Salut, Lume! (Compilarea cu `rustc`)

- **Configurare:** Creează un director pentru proiect și un fișier numit
  `main.rs`. Fișierele Rust se termină mereu în `.rs` (folosește
  underscore pentru numele din mai multe cuvinte).

- **Codul:** Scrie asta în `main.rs`:

  ```
  fn main() {
      println!("Hello, world!");
  }
  ```

- **Anatomia Programului:**

  - `fn main()`: Punctul de intrare (entry point). Execuția codului
    începe mereu de aici.

  - `println!`: Printează text în consolă. Semnul `!` înseamnă că
    apelezi un **macro** în loc de o funcție normală (regulile pentru
    macro-uri sunt puțin diferite).

  - `;`: Indică sfârșitul instrucțiunii (statement).

- **Compilare (Ahead-of-Time):** Rust este un limbaj compilat AOT. Poți
  compila un program și da executabilul altcuiva să îl ruleze fără ca
  acea persoană să aibă Rust instalat.

  - Comanda de compilare:

    ```
    rustc main.rs
    ```

  - Aceasta generează un fișier binar executabil numit `main` (pe
    Linux/macOS).

- **Execuție:** Rulează binarul generat:

  ```
  ./main
  ```

## Salut, Cargo! (Sistemul de build și Managerul de Pachete Rust)

- **Ce este Cargo?** Instrumentul oficial pentru a construi (build)
  codul, a descărca dependențele și a compila acele dependențe. Este
  standardul pentru aproape toate proiectele Rust.

- **Crearea unui Proiect:**

  ```
  cargo new hello_cargo
  cd hello_cargo
  ```

  - Aceasta generează un director nou, inițializează un repository Git
    și creează `Cargo.toml` împreună cu un folder `src`.

- **Fișierul de Configurare (`Cargo.toml`):**

  - Folosește formatul TOML (Tom's Obvious, Minimal Language).

  - Conține detalii `[package]` (nume, versiune, ediție).

  - Conține `[dependencies]` (gol deocamdată, dar aici vor fi adăugate
    pachetele externe/crate-urile).

- **Structura Proiectului:**

  - Codul sursă merge în directorul `src` (Cargo a generat `src/main.rs`
    pentru tine).

  - Directorul principal este strict pentru README-uri, informații
    despre licență și configurări.

- **Construirea (Build) Proiectului:**

  ```
  cargo build
  ```

  - Compilează proiectul și plasează binarul neoptimizat în
    `target/debug/hello_cargo`.

  - Rulează-l manual: `./target/debug/hello_cargo`.

- **Construire și Rulare (Scurtătura):**

  ```
  cargo run
  ```

  - Compilează codul (dacă s-a modificat) și rulează imediat
    executabilul rezultat. Pe asta o vei folosi cel mai des.

- **Verificarea Compilării (Rapid):**

  ```
  cargo check
  ```

  - Verifică dacă codul tău compilează fără a produce efectiv un
    executabil.

  - *Sfat pentru studenți:* Rulați asta frecvent în timp ce scrieți cod
    pentru a prinde erorile rapid, fără a aștepta un build complet.

- **Build pentru Release (Producție):**

  ```
  cargo build --release
  ```

  - Compilează cu optimizări. Durează mai mult să compileze, dar binarul
    rezultat va rula mult mai rapid.

  - Artefactele sunt plasate în `target/release/` în loc de
    `target/debug/`.

## Programarea unui Joc de Ghicit

- **Configurare:** Începe un proiect nou: `cargo new guessing_game` și
  deschide `src/main.rs`.

### Pasul 1: Preluarea Input-ului de la Utilizator

- **Codul:**

  ```
  use std::io;

  fn main() {
      println!("Guess the number!"); // Ghicește numărul!
      println!("Please input your guess."); // Te rog introdu o estimare.

      let mut guess = String::new();

      io::stdin()
          .read_line(&mut guess)
          .expect("Failed to read line"); // A eșuat citirea liniei

      println!("You guessed: {guess}"); // Ai ghicit: {guess}
  }
  ```

- **Concepte de explicat:**

  - `use std::io;`: Importă biblioteca standard pentru input/output. (În
    mod implicit, Rust aduce doar câteva elemente în scope, cunoscute
    sub numele de *prelude*).

  - `let mut guess`: Variabilele în Rust sunt **imutabile în mod
    implicit**. Folosirea `mut` le face mutabile.

  - `String::new()`: `::` indică o *funcție asociată* (similar cu o
    metodă statică) a tipului `String`.

  - `&mut guess`: Simbolul `&` denotă o **referință**, permițând mai
    multor părți din cod să acceseze datele fără a le copia.
    Referințele, la fel ca variabilele, sunt imutabile implicit, deci
    avem nevoie de `&mut`.

  - `.expect()`: `read_line` returnează un enum `Result` (fie `Ok`, fie
    `Err`). Dacă este `Err`, `expect` oprește forțat (crashes) programul
    și afișează mesajul tău. Dacă îl omiți, compilatorul te va avertiza
    despre erori netratate!

  - `println!("{guess}")`: Interpolarea stringurilor. Poți face și
    `println!("You guessed: {}", guess);`.

### Pasul 2: Generarea unui Număr Secret

- **Adăugarea unei Dependențe (`Cargo.toml`):**

  - Sub `[dependencies]`, adaugă: `rand = "0.8.5"`

  - Explică faptul că `cargo build` va descărca acum acest crate
    (pachet) de pe crates.io și va actualiza fișierul `Cargo.lock`
    pentru a "îngheța" (freeze) versiunea.

- **Codul (Actualizează `main.rs`):**

  ```
  use std::io;
  use rand::Rng; // Adaugă metode implementate de generatoarele de numere aleatoare

  fn main() {
      println!("Guess the number!");

      let secret_number = rand::thread_rng().gen_range(1..=100);
      println!("The secret number is: {secret_number}"); // (Șterge asta mai târziu)

      println!("Please input your guess.");
      // ... (păstrează codul de input anterior) ...
  }
  ```

- **Concepte de explicat:**

  - `use rand::Rng`: Un trait care trebuie să fie în scope pentru a
    putea folosi metode precum `gen_range`.

  - `1..=100`: O expresie de range (inclusivă pentru limitele inferioară
    și superioară).

### Pasul 3: Compararea Estimării

- **Codul (Actualizează `main.rs`):**

  ```
  use rand::Rng;
  use std::cmp::Ordering;
  use std::io;

  fn main() {
      // ... (păstrează codul de setup) ...

      let mut guess = String::new();
      io::stdin().read_line(&mut guess).expect("Failed to read line");

      // SHADOWING pentru a converti String în u32
      let guess: u32 = guess.trim().parse().expect("Please type a number!");

      println!("You guessed: {guess}");

      match guess.cmp(&secret_number) {
          Ordering::Less => println!("Too small!"),    // Prea mic!
          Ordering::Greater => println!("Too big!"),   // Prea mare!
          Ordering::Equal => println!("You win!"),     // Ai câștigat!
      }
  }
  ```

- **Concepte de explicat:**

  - `std::cmp::Ordering`: Un enum cu variantele `Less` (Mai mic),
    `Greater` (Mai mare) și `Equal` (Egal).

  - expresia `match`: Inima fluxului de control din Rust. Compară o
    valoare cu o serie de tipare (patterns) și execută ramura care se
    potrivește. (Gândește-te la ea ca la un `switch` cu super-puteri).

  - **Shadowing:** Redeclarăm `let guess`. Asta ne permite să refolosim
    numele variabilei în loc să fim forțați să creăm `guess_str` și
    `guess_num`.

  - `.trim().parse()`: `trim` elimină newline-ul (`\n`) adăugat când
    utilizatorul a apăsat Enter. `parse` convertește stringul într-un
    alt tip. Specificăm `: u32` pentru a-i spune lui Rust exact *în ce*
    tip să îl convertească.

### Pasul 4: Bucle și Rafinare

- **Codul Final:**

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

          // Tratarea inputului invalid cu match în loc de expect()
          let guess: u32 = match guess.trim().parse() {
              Ok(num) => num,
              Err(_) => continue, // '_' este o valoare catch-all (prinde orice)
          };

          println!("You guessed: {guess}");

          match guess.cmp(&secret_number) {
              Ordering::Less => println!("Too small!"),
              Ordering::Greater => println!("Too big!"),
              Ordering::Equal => {
                  println!("You win!");
                  break; // Iese din buclă
              }
          }
      }
  }
  ```

- **Concepte de explicat:**

  - `loop`: Creează o buclă infinită.

  - `break`: Iese din buclă când utilizatorul ghicește corect.

  - **Tratarea lui `Result` în loc de oprirea forțată (crash):**
    Schimbăm `.expect()` cu o instrucțiune `match`. Dacă `parse()`
    reușește (`Ok`), returnează numărul. Dacă eșuează (`Err`), folosim
    `continue` pentru a sări la următoarea iterație a buclei, ignorând
    crash-ul.

  - `_` (underscore): Un tipar "catch-all" care înseamnă "ignoră
    informația specifică despre eroare aici".

## Variabile și Mutabilitate

- **Imutabilitate implicită:** Implicit, odată ce o valoare este legată
  de un nume, ea nu mai poate fi schimbată. (Gândește-te la ea ca la
  `val` în Kotlin). Asta încurajează siguranța și ușurează lucrul
  concurent.

  ```
  fn main() {
      let x = 5;
      println!("The value of x is: {x}");
      x = 6; // EROARE: cannot assign twice to immutable variable
  }
  ```

- **Transformarea variabilelor în mutabile:** Adaugă `mut` pentru a
  permite reatribuirea valorii (ca `var` în Kotlin).

  ```
  fn main() {
      let mut x = 5;
      println!("The value of x is: {x}");
      x = 6; // Asta funcționează!
      println!("The value of x is: {x}");
  }
  ```

- **Constante:** Declarate cu `const`. Ele sunt *întotdeauna* imutabile
  (nu e permis `mut`), necesită adnotarea explicită a tipului, pot fi
  declarate în orice scope (inclusiv global) și trebuie să fie setate la
  expresii care pot fi calculate la momentul compilării (compile-time).

  ```
  const THREE_HOURS_IN_SECONDS: u32 = 60 * 60 * 3;
  ```

- **Shadowing (Umbrire):** Poți declara o variabilă nouă cu același nume
  ca una anterioară folosind `let`. Noua variabilă o "umbrește" pe cea
  veche.

  ```
  fn main() {
      let x = 5;
      let x = x + 1; // x este acum 6

      {
          let x = x * 2; // În acest scope interior, x este 12
          println!("Inner scope x is: {x}");
      }

      println!("Outer scope x is: {x}"); // Revine la 6 aici
  }
  ```

- **Shadowing vs. Mutabilitate:** Shadowing folosește `let`, ceea ce
  înseamnă că creezi o *variabilă complet nouă*. Asta îți permite să
  aplici transformări dar să păstrezi rezultatul imutabil și, foarte
  important, îți permite să **schimbi tipul**.

  ```
  fn main() {
      // Permis prin shadowing:
      let spaces = "   "; // Tipul este String
      let spaces = spaces.len(); // Tipul este acum usize (număr)

      // Asta ar EȘUA cu `mut` deoarece mut nu poate schimba tipul:
      // let mut spaces = "   ";
      // spaces = spaces.len(); // EROARE: expected &str, found usize
  }
  ```

## Tipuri de Date

- **Tipizate Static:** Rust trebuie să cunoască tipul tuturor
  variabilelor la momentul compilării. De obicei inferă tipul, dar
  uneori are nevoie de adnotări explicite dacă sunt posibile mai multe
  tipuri.

  ```
  // Adnotarea explicită a tipului e necesară aici, altfel compilarea eșuează
  let guess: u32 = "42".parse().expect("Not a number!");
  ```

### Tipuri Scalare (Valori Singulare)

- **Întregi (Integers):** Numere fără fracții.

  - Implicit este `i32` (semnat, 32-bit).

  - Există variante pentru numere cu semn (`i8` până la `i128`) și fără
    semn (`u8` până la `u128`).

  - `isize` și `usize` depind de arhitectură (ex: 64-bit pe o mașină
    x64) și sunt folosite în principal pentru indexarea colecțiilor.

  - Separatoarele vizuale sunt permise: `let x = 98_222;`

- **Virgulă Mobilă (Floating-Point):** Numere cu zecimale (IEEE-754).

  - `f32` (precizie simplă) și `f64` (precizie dublă).

  - Implicit este `f64` pentru că e cam la fel de rapid ca `f32` pe
    procesoarele moderne, dar mai precis.

  ```
  let x = 2.0; // f64
  let y: f32 = 3.0; // f32
  ```

- **Operații Numerice:** Operatorii standard `+`, `-`, `*`, `/`, `%`
  funcționează exact cum te-ai aștepta.

  - *Notă:* Împărțirea numerelor întregi trunchiază către zero (ex:
    `-5 / 3` este `-1`).

- **Booleene:** `true` sau `false`. Dimensiune de 1 byte.

  ```
  let t = true;
  let f: bool = false;
  ```

- **Caractere (`char`):** Reprezintă o singură valoare scalară Unicode.

  - Folosește ghilimele simple (`'z'`, `'😻'`).

  - Dimensiune de 4 bytes (spre deosebire de `char` în unele contexte
    C/C++ vechi, acesta gestionează emoticoane și scripturi specializate
    nativ).

### Tipuri Compuse (Valori Multiple)

- **Tuple:** Lungime fixă, pot conține **tipuri diferite**. (Similar cu
  tuplele din Python).

  - Accesate prin index folosind un punct (`.`), nu paranteze drepte.

  - Pot fi destructurate direct în variabile.

  ```
  let tup: (i32, f64, u8) = (500, 6.4, 1);

  // Destructurare
  let (x, y, z) = tup;
  println!("The value of y is: {y}");

  // Indexare directă
  let five_hundred = tup.0;
  ```

  - *Notă:* Un tuplu gol `()` se numește **unit type** (tip unitate) și
    reprezintă o valoare vidă / tip de return gol.

- **Array-uri:** Lungime fixă, dar trebuie să conțină **același tip**
  pentru toate elementele. Datele sunt alocate pe stivă (stack).

  - *Diferență:* Spre deosebire de `ArrayList` din Kotlin sau `list` din
    Python, array-urile din Rust nu pot crește sau scădea în dimensiune.
    (Pentru dimensiuni dinamice, Rust folosește `Vector`i, pe care îi
    vom discuta mai târziu).

  ```
  let a = [1, 2, 3, 4, 5];

  // Tip explicit: [tip; lungime]
  let b: [i32; 5] = [1, 2, 3, 4, 5];

  // Inițializare cu aceeași valoare: [valoare; lungime]
  let c = [3; 5]; // Rezultă [3, 3, 3, 3, 3]
  ```

  - **Accesare:** Folosește sintaxa standard de indexare (`a[0]`).

  - **Siguranță:** Dacă încerci să accesezi un index în afara limitelor
    (out of bounds), Rust va da panic (panic) și va opri programul *la
    runtime* (în loc să permită acces nesigur la memorie ca în C).

## Funcții

- **Sintaxa de Bază:** Definite cu cuvântul cheie `fn`. Rust folosește
  `snake_case` pentru numele de funcții și variabile.

  ```
  fn main() {
      print_hello();
  }

  fn print_hello() {
      println!("Hello, world!");
  }
  ```

  - *Notă:* Ordinea nu contează. Atâta timp cât `print_hello` este
    definită *undeva* în scope, `main` o poate apela.

- **Parametri:** Ești **obligat** să declari tipul fiecărui parametru.
  (Rust nu va infera semnăturile funcțiilor).

  ```
  fn print_labeled_measurement(value: i32, unit_label: char) {
      println!("The measurement is: {value}{unit_label}");
  }
  ```

- **Statements (Instrucțiuni) vs. Expressions (Expresii) - Concept
  Crucial:**

  - **Statements (Instrucțiuni):** Execută o acțiune și *nu* returnează
    o valoare. Se termină cu punct și virgulă `;`. (ex: `let y = 6;`).
    Nu poți face `let x = (let y = 6);`.

  - **Expressions (Expresii):** Se evaluează rezultând într-o valoare.
    Apelarea unei funcții, a unui macro, sau chiar și un nou bloc de
    scope `{}` sunt expresii.

  - *Regula de bază:* Expresiile **nu** se termină cu punct și virgulă.
    Dacă adaugi un punct și virgulă unei expresii, o transformi într-un
    statement și nu va mai returna nicio valoare.

  ```
  fn main() {
      let y = {
          let x = 3;
          x + 1 // Fără punct și virgulă aici! Acest bloc este o expresie care se evaluează la 4.
      };
      println!("The value of y is: {y}");
  }
  ```

- **Valori Returnate:** \* Declarate cu o săgeată `->` după lista de
  parametri.

  - În Rust, valoarea returnată de funcție este sinonimă cu valoarea
    **ultimei expresii** din blocul funcției. (Similar cu modul în care
    blocurile `if` sau `when` pot returna valori în Kotlin).

  ```
  fn five() -> i32 {
      5 // Returnare implicită. Fără cuvântul cheie `return`, fără punct și virgulă!
  }

  fn plus_one(x: i32) -> i32 {
      x + 1 
      // Dacă ai adăuga punct și virgulă aici (x + 1;), ar deveni un statement, 
      // ar returna `()` (tipul unitate), iar compilatorul ar da o eroare!
  }
  ```

  - Tot poți folosi cuvântul cheie `return` pentru returnări timpurii
    (early returns), dar Rust-ul idiomatic se bazează de obicei pe
    ultima expresie pentru return-ul principal.

## Fluxul de Control

### Expresii `if`

- **Sintaxa de Bază:**

  ```
  let number = 3;
  if number < 5 {
      println!("condition was true");
  } else {
      println!("condition was false");
  }
  ```

- **Booleene Stricte:** Condiția **trebuie** să fie evaluată la un
  `bool`. Rust *nu* va converti automat tipurile non-booleene în
  booleene (nu există "truthy" sau "falsy" ca în Python sau C).
  `if number { ... }` va genera o eroare de compilare.

- **`else if`:** Funcționează exact cum te-ai aștepta pentru a trata
  multiple condiții.

- **Folosirea lui `if` într-un Statement `let`:** Deoarece `if` este o
  expresie (exact ca în Kotlin), poți atribui rezultatul ei unei
  variabile.

  ```
  let condition = true;
  // Asta este identic cu: val number = if (condition) 5 else 6 din Kotlin
  let number = if condition { 5 } else { 6 };
  ```

  - *Regulă Importantă:* Tipurile de return ale tuturor ramurilor
    trebuie să se potrivească perfect. Nu poți face
    `if condition { 5 } else { "șase" }`.

### Repetiții cu Bucle (Loops)

- **`loop` (Infinită):** Execută un bloc de cod la nesfârșit, până când
  îl oprești explicit.

  ```
  loop {
      println!("again!");
      break; // Oprește bucla
  }
  ```

  - **Returnarea valorilor din bucle:** Poți returna o valoare dintr-un
    `loop` punând-o după cuvântul cheie `break`.

    ```
    let mut counter = 0;
    let result = loop {
        counter += 1;
        if counter == 10 {
            break counter * 2; // Returnează 20 în variabila 'result'
        }
    };
    ```

  - **Etichete pentru bucle (Loop Labels):** Dacă ai bucle imbricate
    (nested), `break` sau `continue` se aplică doar celei mai interioare
    bucle. Poți prefixa o buclă cu o etichetă (care începe cu o singură
    ghilimea) pentru a ieși dintr-o buclă exterioară specifică.

    ```
    'outer: loop {
        loop {
            break 'outer; // Iese din bucla exterioară, nu doar din cea interioară
        }
    }
    ```

- **Bucle `while` (Condiționale):** Rulează atâta timp cât o condiție
  este evaluată la `true`.

  ```
  let mut number = 3;
  while number != 0 {
      println!("{number}!");
      number -= 1;
  }
  println!("LIFTOFF!!!");
  ```

- **Bucle `for` (Colecții și Range-uri):** Cea mai sigură și idiomatică
  buclă din Rust. În loc să ții manual evidența unui index (ceea ce
  poate duce la panics din cauza ieșirii din limite / out-of-bounds),
  folosește `for` pentru a itera direct peste elemente. (Foarte similar
  cu `for item in list:` din Python sau `for (item in collection)` din
  Kotlin).

  - **Iterarea unui Array:**

    ```
    let a = [10, 20, 30, 40, 50];
    for element in a {
        println!("the value is: {element}");
    }
    ```

  - **Iterarea unui Range:** Pentru a face o buclă de un anumit număr de
    ori, folosește range-uri (`start..end`, exclusiv pentru end). Poți
    înlănțui metode precum `.rev()` pentru a-l inversa.

    ```
    for number in (1..4).rev() {
        println!("{number}!"); // Printează 3, 2, 1
    }
    ```

## Înțelegerea Conceptului de Ownership (Proprietate)

- **Premisa Principală:** Aceasta este caracteristica unică a lui Rust.
  Garantează siguranța memoriei *fără* a avea nevoie de un Garbage
  Collector (ca în Python/Kotlin) sau de a te forța să aloci/eliberezi
  manual memoria (ca în C/C++). Memoria este gestionată printr-un sistem
  de reguli verificat la **momentul compilării (compile time)**.

- **Stivă (Stack) vs. Heap (Mică Recapitulare):**

  - **Stack (Stiva):** Rapidă, strict LIFO (Last-In-First-Out). Toate
    datele stocate aici trebuie să aibă o dimensiune cunoscută și fixă
    (ex: `i32`, `bool`, array-uri).

  - **Heap:** Mai lent. Folosit pentru date cu o dimensiune dinamică la
    rulare. Sistemul de operare alocă spațiu și returnează un pointer
    (ex: `String`). Urmărirea pointerilor face accesul la heap mai lent
    decât cel de pe stack.

- **Cele 3 Reguli ale Ownership-ului:**

  1.  Fiecare valoare în Rust are un **proprietar (owner)**.

  2.  Poate exista doar **un singur proprietar la un moment dat**.

  3.  Când proprietarul iese din scope (domeniul de vizibilitate),
      valoarea va fi **distrusă/eliberată (dropped)** (memoria este
      eliberată).

### Scope-ul și Alocarea Memoriei

- **Scope-ul variabilei:** La fel ca în majoritatea limbajelor,
  variabilele sunt valide din punctul în care sunt declarate până la
  sfârșitul blocului curent `{}`.

- **Tipul `String`:** Pentru a ilustra lucrul cu heap-ul, folosim
  `String` (care este mutabil și are o dimensiune dinamică), în loc de
  literale string (`&str`, care sunt hardcodate).

  ```
  {
      let s = String::from("hello"); // Memoria este cerută de la OS aici
      // facem chestii cu s
  } // Scope-ul se termină. Rust apelează automat `drop` pentru a elibera memoria.
  ```

### Interacțiunea cu Datele: Mutare (Move) vs. Copiere (Copy)

- **Date de pe Stivă (Copy):** Tipurile simple de dimensiuni cunoscute
  sunt stocate în întregime pe stack.

  ```
  let x = 5;
  let y = x;
  // Atât x cât și y sunt egale cu 5. A fost făcută o copie rapidă bit-cu-bit. Ambele sunt valide.
  ```

- **Date de pe Heap (Move - Diferența crucială!):** În Python sau
  Kotlin, atribuirea unui obiect către o nouă variabilă pur și simplu
  copiază referința. Dacă Rust ar face asta, când ambele variabile ar
  ieși din scope, ar încerca să elibereze aceeași memorie de două ori (o
  eroare "double free").

  ```
  let s1 = String::from("hello");
  let s2 = s1; // s1 este MUTAT în s2.

  // println!("{}, world!", s1); // ASTA VA CAUZA O EROARE DE COMPILARE!
  ```

  - Rust consideră că `s1` este **invalid** după ce a fost atribuit lui
    `s2`. Nu îl mai poți folosi. Asta rezolvă automat problema
    eliberării duble!

- **Copiere profundă (Clone):** Dacă vrei *cu adevărat* să duplici
  datele de pe heap (o operațiune costisitoare), folosești `.clone()`.

  ```
  let s1 = String::from("hello");
  let s2 = s1.clone();
  println!("s1 = {}, s2 = {}", s1, s2); // Asta funcționează perfect!
  ```

### Ownership și Funcții

- **Transmiterea Variabilelor:** Transmiterea unei variabile într-o
  funcție funcționează exact ca atribuirea ei la o altă variabilă: fie
  va fi **mutată** (pentru tipurile de pe heap), fie **copiată** (pentru
  tipurile de pe stack).

  ```
  fn main() {
      let s = String::from("hello");
      takes_ownership(s);
      // s NU MAI ESTE VALIDĂ aici. A fost mutată în funcție.

      let x = 5;
      makes_copy(x);
      // x ESTE ÎNCĂ VALIDĂ aici deoarece i32 implementează trăsătura Copy.
  }

  fn takes_ownership(some_string: String) {
      println!("{}", some_string);
  } // some_string iese din scope și memoria este eliberată.

  fn makes_copy(some_integer: i32) {
      println!("{}", some_integer);
  }
  ```

- **Valori Returnate:** Returnarea unei valori transferă ownership-ul
  *înapoi* către cel care a apelat funcția (caller).

  ```
  fn main() {
      let s1 = gives_ownership(); // Funcția creează un string, îl mută în s1

      let s2 = String::from("hello");
      let s3 = takes_and_gives_back(s2); // s2 este mutat în funcție, apoi mutat afară în s3
  }
  // s3 și s1 sunt distruse aici. s2 fusese deja mutat, deci nu se întâmplă nimic cu el.
  ```

  *(Notă: Transmiterea constantă a ownership-ului înainte și înapoi este
  obositoare, motiv pentru care Rust introduce **Referințele**,
  explicate în capitolul următor).*

## Referințe și Împrumuturi (Borrowing)

- **Problema:** Să transmiți ownership-ul către funcții și să-l primești
  înapoi de fiecare dată e enervant.

- **Soluția (Borrowing - Împrumutul):** Putem transmite o **referință**
  către o valoare. O referință e ca un pointer în sensul că este o
  adresă pe care o putem urmări pentru a accesa datele, dar spre
  deosebire de un pointer generic, Rust garantează că o referință arată
  întotdeauna spre o valoare validă de un anumit tip.

### Referințe Imutabile (`&`)

- **Crearea unei Referințe:** Folosește ampersand-ul (`&`). Acțiunea de
  a crea o referință se numește **împrumut (borrowing)**.

  ```
  fn main() {
      let s1 = String::from("hello");
      let len = calculate_length(&s1); // Transmitem o REFERINȚĂ către s1

      // s1 ESTE ÎNCĂ VALIDĂ aici deoarece nu am renunțat la ownership!
      println!("The length of '{}' is {}.", s1, len);
  }

  // Tipul parametrului `&String` înseamnă "o referință către un String"
  fn calculate_length(s: &String) -> usize {
      s.len()
  } // s iese din scope, dar pentru că nu deține datele la care se referă, memoria nu este distrusă/eliberată.
  ```

- **Modificarea este Interzisă:** Așa cum variabilele sunt imutabile
  implicit, și referințele sunt imutabile implicit. Nu putem modifica
  ceva ce am împrumutat.

  ```
  fn change(some_string: &String) {
      // some_string.push_str(", world"); // ASTA CAUZEAZĂ O EROARE DE COMPILARE
  }
  ```

### Referințe Mutabile (`&mut`)

- **Permiterea Modificării:** Pentru a modifica o valoare împrumutată,
  avem nevoie de o referință mutabilă.

  1.  Variabila originală trebuie să fie `mut`.

  2.  Trebuie să o transmiți folosind `&mut`.

  3.  Semnătura funcției trebuie să accepte `&mut`.

  ```
  fn main() {
      let mut s = String::from("hello");
      change(&mut s);
      println!("{}", s); // Printează "hello, world"
  }

  fn change(some_string: &mut String) {
      some_string.push_str(", world");
  }
  ```

### Regulile Împrumutului (Crucial!)

Rust impune două reguli stricte la momentul compilării pentru a preveni
problemele de memorie și **data races** (condiții de cursă pe date,
motiv pentru care Rust este legendar pentru lucrul concurent fără
griji/fearless concurrency):

**Regula 1: Restricția "O Singură Referință Mutabilă"**

- La orice moment dat, poți avea **FIE**:

  - Exact *una* singură referință mutabilă (`&mut T`).

  - *Orice număr* de referințe imutabile (`&T`).

- Nu le poți avea pe ambele în același timp în același scope.

  ```
  let mut s = String::from("hello");

  let r1 = &s; // OK (Imutabil)
  let r2 = &s; // OK (Imutabil)
  println!("{} and {}", r1, r2); // Le putem citi pe amândouă

  // let r3 = &mut s; // EROARE: Nu pot împrumuta `s` ca mutabil deoarece este deja împrumutat ca imutabil!
  ```

- *De ce?* Dacă cineva deține un pointer "read-only" (doar-citire) către
  date, se așteaptă ca acele date să nu se schimbe sub el. Dacă am
  permite un pointer mutabil simultan, acel pointer mutabil ar putea
  schimba sau realoca memoria, stricând astfel pointerii read-only.

**Regula 2: Fără Referințe "Atârnate" (Dangling References)**

- Un pointer atârnat este un pointer care face referire la o memorie
  care a fost dată altcuiva sau care a fost deja eliberată.

- Compilatorul Rust garantează că acest lucru nu se va întâmpla
  **niciodată**. Se asigură că datele nu vor ieși din scope înaintea
  referinței care arată către acele date.

  ```
  fn main() {
      // let reference_to_nothing = dangle(); // EROARE
  }

  /* // ACEST COD NU VA COMPILA
  fn dangle() -> &String { 
      let s = String::from("hello"); // s este creat aici
      &s // Returnăm o referință către s
  } // s iese din scope și este eliberat. Memoria s-a dus. Pericol! 
  */
  ```

  - *Repararea:* În loc să returnezi o referință din `dangle`, pur și
    simplu returnezi `String`-ul direct pentru a muta ownership-ul afară
    din funcție.

## Definirea și Instanțierea Structurilor

- **Ce sunt Structurile?** Tipuri de date personalizate care te lasă să
  împachetezi la un loc și să numești mai multe valori relaționate.
  (Foarte similar cu `data class` din Kotlin sau `dataclass` /
  dicționarele din Python, dar puternic tipizate).

### Definirea unei Structuri

- **Sintaxa:** Folosește cuvântul cheie `struct` urmat de acolade ce
  conțin perechi `nume: tip` (câmpuri).

  ```
  struct User {
      active: bool,
      username: String,
      email: String,
      sign_in_count: u64,
  }
  ```

### Instanțierea unei Structuri

- **Crearea:** Creezi o instanță specificând numele structurii și
  oferind perechi cheie-valoare între acolade. Ordinea nu contează.

  ```
  let mut user1 = User {
      email: String::from("someone@example.com"),
      username: String::from("someusername123"),
      active: true,
      sign_in_count: 1,
  };
  ```

- **Acces și Mutabilitate:** \* Folosește notația cu punct pentru a
  accesa câmpurile: `user1.email`.

  - **Diferență Importantă față de Kotlin:** Nu poți marca câmpuri
    individuale ca mutabile sau imutabile. Mutabilitatea este o
    proprietate a *întregii instanțe*. Dacă instanța este declarată cu
    `mut`, toate câmpurile sunt mutabile. Dacă nu, niciunul nu e.

  ```
  user1.email = String::from("anotheremail@example.com"); // Merge doar pentru că user1 este `mut`
  ```

### Prescurtarea pentru Inițializarea Câmpurilor (Field Init Shorthand)

- Dacă numele variabilelor tale se potrivesc exact cu numele câmpurilor
  din structură, poți omite tastarea numelui de două ori. (Exact la fel
  ca la obiectele JavaScript moderne).

  ```
  fn build_user(email: String, username: String) -> User {
      User {
          active: true,
          username, // Prescurtare pentru username: username
          email,    // Prescurtare pentru email: email
          sign_in_count: 1,
      }
  }
  ```

### Sintaxa de Actualizare a Structurii (`..`)

- Permite crearea unei noi instanțe de structură copiind valorile
  dintr-una existentă și suprascriind doar ce specifici. (Conceptual
  identic cu `data_class.copy(field = new_value)` din Kotlin).

  ```
  let user2 = User {
      email: String::from("another@example.com"),
      ..user1 // Copiază restul câmpurilor (active, username, sign_in_count) din user1
  };
  ```

- **Capcana Ownership-ului:** Sintaxa `..user1` se comportă ca o
  atribuire (`=`). Deoarece `String` nu implementează `Copy`, câmpul
  `username` este **mutat (moved)** din `user1` în `user2`.

  - După această operațiune, `user1` este parțial invalid. Nu mai poți
    accesa `user1.username` sau să folosești `user1` ca întreg, deși
    celelalte câmpuri (`active` și `sign_in_count`, care implementează
    `Copy`) rămân accesibile individual.

### Structuri de Tip Tuplu (Tuple Structs)

- Structuri care arată ca niște tuple. Au un nume general de tip, dar
  câmpurile lor nu au nume, ci doar tipuri.

- Utile când vrei să creezi tipuri distincte pentru a beneficia de
  verificarea de tipuri a compilatorului, chiar dacă structura de date
  este identică.

  ```
  struct Color(i32, i32, i32);
  struct Point(i32, i32, i32);

  let black = Color(0, 0, 0);
  let origin = Point(0, 0, 0);
  // Color și Point sunt tipuri complet diferite. Nu poți pasa un Color unei funcții care se așteaptă la un Point!
  ```

- Accesate folosind notația cu punct și indici: `black.0`, `black.1`,
  etc.

### Structuri de tip Unit (Unit-Like Structs)

- Structuri care nu au absolut niciun câmp.

  ```
  struct AlwaysEqual;

  let subject = AlwaysEqual;
  ```

- În principal folosite când trebuie să implementezi un trait (ca o
  interfață) pe un anumit tip, dar nu ai de fapt nicio dată care trebuie
  stocată în tipul în sine.

## Definirea unui Enum

- **Ce sunt Enum-urile?** Îți permit să definești un tip prin enumerarea
  *variantelor* sale posibile.

- **Conexiunea cu Kotlin:** În Rust, enum-urile sunt în esență **clase
  `sealed` (sealed classes)** din Kotlin. Ele sunt incredibil de
  puternice pentru că fiecare variantă poate stoca diferite cantități și
  tipuri de date direct în interiorul ei.

### Enum-uri de Bază

- **Definire:** Folosește cuvântul cheie `enum`.

  ```
  enum IpAddrKind {
      V4,
      V6,
  }
  ```

- **Instanțiere:** Variantele se află în namespace-ul enum-ului și se
  folosesc cu un semn dublu colon `::`.

  ```
  let four = IpAddrKind::V4;
  let six = IpAddrKind::V6;
  ```

  *(Atât `four` cât și `six` au fix același tip: `IpAddrKind`)*.

### Enum-uri cu Date (Super-puterea)

- În loc să creezi un enum doar pentru a ține evidența "tipului" de date
  și o structură separată pentru a ține datele efectiv, poți îngloba
  datele direct în variantele enum-ului.

  ```
  enum IpAddr {
      V4(String),
      V6(String),
  }

  let home = IpAddr::V4(String::from("127.0.0.1"));
  let loopback = IpAddr::V6(String::from("::1"));
  ```

- **Structuri de Date Diferite:** Fiecare variantă poate avea tipuri și
  cantități diferite de date asociate (de asta seamănă atât de mult cu
  sealed classes din Kotlin!).

  ```
  enum Message {
      Quit,                       // Fără date deloc
      Move { x: i32, y: i32 },    // Conține o structură anonimă înăuntru
      Write(String),              // Conține un singur String
      ChangeColor(i32, i32, i32), // Conține trei valori i32
  }
  ```

  - *De ce să facem asta în loc de Structuri?* Dacă am folosi patru
    structuri diferite, nu am putea scrie ușor o singură funcție care să
    primească *oricare* dintre acele mesaje. Folosind un enum, toate
    împart tipul umbrelă `Message`.

### Metode pe Enum-uri

- La fel ca la structuri, poți defini metode pe enum-uri folosind
  `impl`.

  ```
  impl Message {
      fn call(&self) {
          // Corpul metodei ar veni aici
      }
  }

  let m = Message::Write(String::from("hello"));
  m.call();
  ```

### Enum-ul `Option` (Înlocuitorul Rust-ului pentru Null)

- **Problema cu Null:** Rust **nu** are un feature de `null`. Elementele
  nule cauzează greșeli de miliarde de dolari (NullPointerExceptions)
  atunci când încerci să folosești o valoare nulă ca fiind o valoare
  ne-nulă.

- **Soluția Rust:** Rust codifică *conceptul* de prezență sau absență a
  unei valori într-un enum special din biblioteca standard numit
  `Option<T>`.

  ```
  // Acesta este definit în biblioteca standard astfel:
  enum Option<T> {
      None,
      Some(T),
  }
  ```

  - *(`T` este un parametru de tip generic, însemnând că poate deține
    orice tip de date).*

- **Utilizare:** Este atât de comun încât `Option`, `Some` și `None`
  sunt incluse în prelude (nu trebuie să le imporți sau să le prefixezi
  cu `Option::`).

  ```
  let some_number = Some(5);
  let some_char = Some('e');

  // Dacă folosești None, TREBUIE să spui compilatorului tipul așteptat, 
  // pentru că nu poate infera din neant/nimic gol.
  let absent_number: Option<i32> = None; 
  ```

- **De ce este mai bun decât null?** `Option<i8>` și `i8` sunt **tipuri
  complet diferite** pentru compilator. Nu poți adăuga un număr întreg
  standard la un număr de tip `Option`. Trebuie **obligatoriu** să
  extragi valoarea din `Option` mai întâi. Asta te forțează să tratezi
  explicit cazul de `None` înainte de a face operațiuni, eliminând
  posibilitatea crash-urilor de la unexpected null!

## Flux de Control cu `match`

- **Ce este `match`?** Un operator puternic pentru fluxul de control
  care compară o valoare cu o serie de tipare (patterns) și execută cod.
  (Gândește-te la instrucțiunea `when` din Kotlin, dar integrată profund
  cu enum-urile din Rust).

- **Sortatorul de Monede:**

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

  - `match` este o **expresie**, adică returnează o valoare (`1`, `5`,
    etc.).

### Extragerea Valorilor Legate de Tipare (Patterns)

- **Puterea Adevărată:** `match` poate extrage datele interne din
  varianta unui enum în timp ce îi verifică tipul!

  ```
  #[derive(Debug)] // Ca să putem inspecta statul
  enum UsState {
      Alabama,
      Alaska,
      // ... etc
  }

  enum Coin {
      Penny,
      Nickel,
      Dime,
      Quarter(UsState), // Quarter acum conține date despre stat!
  }

  fn value_in_cents(coin: Coin) -> u8 {
      match coin {
          Coin::Penny => 1,
          Coin::Quarter(state) => {
              println!("State quarter from {:?}!", state);
              25
          }
          // _ (catch-all) este folosit pentru a ignora restul variantelor
          _ => 0, 
      }
  }
  ```

### Potrivirea cu `Option<T>`

- Acesta este modul standard de a extrage valori în siguranță dintr-un
  `Option` (tratând cazurile "ne-nul" vs "nul").

  ```
  fn plus_one(x: Option<i32>) -> Option<i32> {
      match x {
          None => None,          // Tratează cazul gol
          Some(i) => Some(i + 1), // Extrage valoarea `i`, folosește-o, și re-împacheteaz-o
      }
  }

  let five = Some(5);
  let six = plus_one(five);
  let none = plus_one(None);
  ```

### Exhaustivitate și Catch-Alls

- **Potrivirile (Matches) TREBUIE să fie Exhaustive:** Compilatorul va
  țipa la tine dacă nu acoperi absolut toate variantele posibile. Acest
  lucru previne stările netratate (unhandled states)!

- **Varianta Catch-All (`_`):** Dacă îți pasă doar de câteva cazuri și
  vrei să ignori restul, folosește placeholder-ul `_`.

  ```
  let dice_roll = 9;
  match dice_roll {
      3 => add_fancy_hat(),
      7 => remove_fancy_hat(),
      _ => reroll(), // Se potrivește cu orice altceva. (Folosește un nume de variabilă în loc de `_` dacă ai nevoie de valoare)
  }
  ```

  - *Notă:* Poți de asemenea să folosești `_ => ()` pentru a face
    explicit *nimic* în cazurile default.

## Flux de Control Concis cu `if let`

- **Problema:** Câteodată o expresie `match` este mult prea "verboasă"
  când vrei *doar* să verifici o singură variantă specifică și să ignori
  orice altceva.

- **Soluția:** `if let` este sintaxă de tip "syntactic sugar" (o
  prescurtare) pentru un `match` care rulează un cod doar când valoarea
  se potrivește cu un singur tipar (pattern) și le ignoră pe toate
  celelalte.

  ```
  // Modul match verbos:
  let config_max = Some(3u8);
  match config_max {
      Some(max) => println!("The maximum is configured to be {}", max),
      _ => (),
  }

  // Modul concis cu `if let`:
  let config_max = Some(3u8);
  if let Some(max) = config_max {
      println!("The maximum is configured to be {}", max);
  }
  ```

- **Dezavantaj:** Pierzi verificarea exhaustivă la care te forțează
  `match`, dar câștigi la scrierea de cod mai scurt.

- **Suport pentru `else`:** Poți atașa un bloc `else` unui `if let`,
  care se comportă exact ca ramura `_` (catch-all) dintr-un `match`.
