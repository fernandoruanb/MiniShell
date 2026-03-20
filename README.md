
<p align="center">
  <img src="https://github.com/ayogun/42-project-badges/raw/main/covers/cover-minishell.png" alt="Minishell cover" width="100%">
</p>

<h1 align="center">
  <a href="https://github.com/fernandoruanb/minishell">
    <img src="https://github.com/ayogun/42-project-badges/raw/main/badges/minishelle.png" alt="Minishell badge" width="200">
  </a>
  <br>
  Minishell
  <br>
</h1>

<h4 align="center">
  A miniature shell built in <a href="https://www.c-language.org/" target="_blank">C</a>, focused on parsing, pipes, redirects, heredoc, signals, builtins, environment handling, and faithful Bash-like behavior.
</h4>

<p align="center">
  <img src="https://img.shields.io/badge/Final%20Score-100%2F125-00C853?style=for-the-badge" alt="Final Score 100/125">
  <img src="https://img.shields.io/badge/Language-C-00599C?style=for-the-badge&logo=c" alt="Language C">
  <img src="https://img.shields.io/badge/Category-Unix%20Shell-blueviolet?style=for-the-badge" alt="Unix Shell">
  <img src="https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge" alt="Status Completed">
</p>

<p align="center">
  <a href="#about-the-project">About</a> •
  <a href="#core-idea">Core Idea</a> •
  <a href="#parser-architecture">Parser Architecture</a> •
  <a href="#lexer">Lexer</a> •
  <a href="#syntax-check">Syntax Check</a> •
  <a href="#execution-pipes-redirects-and-heredoc">Execution, Pipes, Redirects and Heredoc</a> •
  <a href="#signals-environment-and-builtins">Signals, Environment and Builtins</a> •
  <a href="#bash-compatibility-and-edge-cases">Bash Compatibility</a> •
  <a href="#testing-philosophy">Testing Philosophy</a> •
  <a href="#how-to-use">How To Use</a> •
  <a href="#team">Team</a>
</p>

---

## About the Project

<html>
<p align="center">
  <img src="./assets/minishell_demo.gif" alt="Minishell demo">
</p>
<p align="center">
  <sub>Illustrative Minishell demo</sub>
</p>
</html>

**Minishell** was our first team project at **42 São Paulo**, and it marked a major transition from isolated programming exercises into something much closer to a real system program.

The goal of the project is not to clone the entire **Bash shell**, but to recreate its essential logic well enough for the user to feel they are interacting with a real command interpreter.

That means dealing with things such as:

- pipes
- redirects
- heredoc
- builtins
- signals
- environment inheritance
- exit status propagation
- command lookup through `PATH`
- and the interaction between multiple structures in a single command line

What makes **Minishell** special is that it is not just a process launcher.

It is, above all, a project about **understanding how a shell thinks**.

And after building it, one conclusion becomes very clear:

> **Minishell is, in practice, a giant parser.**

Before any execution can happen, the shell must understand exactly what the user meant, what each token represents, what the execution priority is, and whether the command is even valid in the first place.

That is where the true depth of the project begins.

---

## Core Idea

The core idea of **Minishell** is simple in appearance:

- display a prompt
- read the user input
- understand what the input means
- validate its syntax
- organize execution priority
- run the command or pipeline correctly
- restore the shell state
- wait for the next command

But in practice, each one of those steps contains many rules.

A command like:

```bash
cat arquivo.txt | grep myBeautifulLine > newArquivo.txt
````

looks simple to the user, but for the shell it raises several questions immediately:

* what is the command?
* what is an argument?
* where is the pipe?
* what is a redirect?
* what file is the redirect targeting?
* which process writes?
* which process reads?
* what should be duplicated with `dup2()`?
* what exit code must be preserved at the end?

So although the shell feels interactive and fluid from the outside, internally it is a carefully controlled system of **parsing, validation, process management, file descriptor manipulation, and signal behavior**.

---

## Parser Architecture

To make that possible, our project was structured around a clear execution flow.

The program was conceptually divided into three main stages:

* `create()` → environment initialization
* `execute()` → main shell behavior
* `destroy()` → cleanup of used resources

From there, the shell follows a recurring cycle:

1. draw the prompt
2. receive the user input
3. tokenize the line
4. validate the syntax
5. organize the execution structure
6. execute commands, pipes, redirects, and heredocs
7. restore what must be restored
8. return to the prompt

This architecture matters because shells are not just about executing binaries.

They are long-running interactive programs, so every cycle must preserve stability.

A shell cannot afford to leave broken file descriptors, corrupted signal behavior, invalid states, or residual execution artifacts after each command.

That is why structure is so important here.

---

## Lexer

The **lexer** is the part responsible for recognizing **what is what** inside the user input.

This part was developed by my teammate, whose profile is here:

* <a href="https://github.com/oJonasRtz/">oJonasRtz</a>

Its role is to scan the raw line and separate it into meaningful tokens.

For example, from this input:

```bash
cat arquivo.txt | grep myBeautifulLine > newArquivo.txt
```

the lexer must identify things such as:

* command
* arguments
* pipe
* redirect operators
* redirect targets
* heredoc operators and delimiters

To solve this, the lexer was built with an **automaton-style approach**, very similar to the logic used in compilers.

That means the shell walks through the string character by character and decides, for each position, whether it is reading:

* an operator like `|`, `>`, `>>`, `<`, `<<`
* or a normal word fragment that belongs to a command, argument, filename, or delimiter

In simplified form, the lexer flow behaves like this:

```text
lexer(str)
   |
   v
check(str[i])
   |
   v
handler(&str[i])
```

The handler keeps advancing until the token ends, and then stores the extracted content into a token list.

This stage is fundamental because if tokenization is wrong, everything after it becomes unreliable.

A shell that does not understand its own input cannot execute safely.

---

## Syntax Check

After tokenization, the next crucial step is the **syntax check**, which was the part developed by me.

This stage behaves like a **central validation system**.

Its job is not merely to inspect the command superficially, but to determine whether the input is structurally acceptable before execution is allowed to proceed.

This includes detecting cases such as:

* unclosed quotes
* unfinished pipes
* invalid operator sequences
* malformed redirect usage
* incomplete heredoc structures
* unsupported logical combinations
* broken command layouts

In practice, this means the shell already knows, before execution, whether the command is valid or invalid.

If it is invalid, the syntax checker can:

* stop the execution
* emit the appropriate error message
* preserve shell stability
* return control to the prompt cleanly

This was one of the most important parts of the project because the number of edge cases is very large.

Even in a reduced shell, syntax behavior quickly grows into **hundreds of combinations**.

And that is exactly why this phase matters so much:
a shell must not execute blindly.

It must first understand whether the command is structurally sound.

---

## Execution, Pipes, Redirects and Heredoc

Once the input has been tokenized and validated, the shell can move into execution planning.

At this point, the command is no longer just text.

It becomes an execution structure where priorities must be respected.

That means dealing correctly with:

* pipes
* redirects
* heredocs
* file opening order
* command recursion or chaining
* process creation with `fork()`
* and restoration of the original shell state afterward

Pipes were one of the parts I worked on directly.

Before reaching the final **Minishell** implementation, I had already built a **Pipex**, which gave me the foundation to understand process chaining, file descriptor redirection, and the flow of data between commands.

That earlier experience was essential, but for **Minishell** I went further:
I created multiple small **laboratories** to isolate and study pipe behavior in controlled scenarios, which helped me understand and solve the execution flow of all pipe combinations required by our shell.

This step-by-step experimental approach was fundamental for making the pipeline logic reliable.

Another key tool in this process was **GDB**.

Using the debugger was critical to inspect process creation, follow descriptor duplication, observe execution order, and verify exactly how each command interacted with the others inside a pipeline.

Without that low-level debugging process, understanding and stabilizing the full pipe flow would have been much harder.

### Simple execution flow

When no pipes are present, the shell follows a simpler path:

```text
exec_single_cmd()
   |
   v
check redirects
   |
   v
fork()
   |
   v
check builtin
   |
   v
execute
```

This flow is still delicate, because even a single command may include redirects, environment effects, builtins, and signal-sensitive behavior.

### Redirect logic

Our redirect flow followed a very clear structure:

```text
origin = save_origin()
   |
   v
manage_redirect()
   |
   v
restore_redirect(origin)
```

This means the shell first saves the current `stdin` and `stdout`, then applies the necessary duplications and file openings, and finally restores the original descriptors after execution.

That restoration step is essential.

Without it, one command could permanently damage the interactive shell session.

### Heredoc and execution order

One of the subtle parts of the project is that execution must follow **Bash-like priority rules**.

For example:

```bash
cat << EOF > oi > oi2 > oi3 > oi4 | cat < oi4
```

In a structure like this, the heredoc content ultimately ends up being redirected into the last output target, while the input redirect on the right side may read from a file that has already been overwritten earlier in the process.

This is exactly the kind of situation that shows why shell execution is not trivial.

The challenge is not only to support isolated features like pipes or redirects.

The real challenge is making them behave correctly **when many of them appear together**.

---

## Signals, Environment and Builtins

Another important part of **Minishell** is that it must behave like a shell not only in parsing and execution, but also in its interaction with the terminal and with the process environment.

### Signals

I had already worked on signals in **Minitalk**, so in this project I handled the signal behavior of the shell.

That includes dealing correctly with:

* `Ctrl+C` → `SIGINT`
* `Ctrl+\` → `SIGQUIT`

A shell must not react to signals in the same way under every context.

Its behavior changes depending on whether it is:

* waiting at the prompt
* executing a child process
* handling interactive input
* or cleaning up after interruption

If signal handling is wrong, the shell stops feeling like a shell immediately.

The user notices broken line behavior, bad prompt restoration, incorrect interruption semantics, or terminal corruption.

### Environment variables

My teammate handled environment logic, including:

* inherited environment variables
* shell-managed internal variables
* exported variables visible to child processes

That means the shell can distinguish between local internal values and variables meant to be inherited after `export`.

This becomes especially important when launching subprocesses or even another shell inside the shell.

### Builtins

We also implemented builtins such as:

* `cd`
* `pwd`
* `echo`
* `export`
* `unset`
* `env`
* `exit`

Builtins are especially important because they are not just commands.

They often modify the shell's own internal state.

That is why many of them must be executed in the shell context itself rather than delegated blindly to external binaries.

---

## Bash Compatibility and Edge Cases

One of the hardest parts of **Minishell** is not the happy path.

It is the huge number of strange cases that still need to behave in a way that feels correct.

For example, commands such as:

```bash
/bin/ls
/bin/echo
/b'i'n'/l's
```

must still be interpreted properly when valid, even though they are not simple plain command names.

That means path extraction and command resolution must be robust.

### PATH behavior

If the user does:

```bash
unset PATH
```

the main commands are no longer automatically found.

If the user later places an executable into a directory and sets that directory back into `PATH`, the shell must be able to discover commands again.

This is one of the places where shell behavior becomes deeply tied to environment management.

### Quote-sensitive expansion

Shell behavior also changes depending on quoting.

For example:

```bash
echo "$SHELL"
```

must expand the variable, while:

```bash
echo '$SHELL'
```

must not.

This kind of behavior is small from the user perspective, but very important for shell authenticity.

### Fragile builtins and terminal-sensitive cases

A few edge cases are especially interesting because they reveal how subtle shell logic can be.

For instance:

```bash
mkdir folder
cd folder
rm -rf ../folder
pwd
```

Many student shells break or behave strangely here.

A robust shell must deal gracefully with the fact that the current directory may no longer have a normal path representation.

Likewise:

```bash
cd .
```

in an invalidated directory context should behave carefully and may produce an error depending on the state.

Another classic case is:

```bash
exit 999999999999999999999999999999999999999999999
```

which must trigger the correct numeric error behavior instead of silently producing undefined results.

And there are even terminal-level traps.

If a command such as `su` is not handled carefully, the shell may leave the terminal without normal echo behavior, which affects how characters appear on screen.

These examples show an important truth about **Minishell**:

the project is full of small details that seem minor until they break the user experience completely.

---

## Testing Philosophy

Because shell behavior is full of edge cases, testing had to be extremely rigorous.

The goal was never just:

> “Does it run?”

The real goal was:

> “Does it still behave correctly when the input becomes weird, broken, mixed, incomplete, nested, or shell-like in unexpected ways?”

That meant testing many categories of situations, such as:

* syntax failures
* broken quotes
* malformed redirects
* bad pipes
* invalid command combinations
* environment mutations
* path resolution changes
* signal interruptions
* builtin edge cases
* descriptor restoration
* heredoc interactions
* mixed command structures

For syntax testing, one repository that helped me a lot was:

* <a href="https://github.com/DanielSurf10/minishell/blob/executor/tester/parsing/tests_list.py">DanielSurf10 minishell parsing tests</a>

Another important practical point is that **readline** has its own known memory allocations, which can appear as leaks in analysis tools.

Because of that, we used a **Valgrind suppression file** so that leak checking could focus only on what actually belonged to our project.

That was essential to avoid false positives and keep debugging meaningful.

And there is one more important note:
features like command history via the up arrow were not reimplemented by us from scratch.

That behavior already comes from **readline**, and our responsibility was to integrate it correctly and close the shell cleanly around it.

For the pipe system in particular, debugging with **GDB** was fundamental.

Because pipelines involve multiple processes, descriptor duplication, and execution order happening together, the debugger made it possible to inspect the behavior at a much deeper level than simple output testing would allow.

This was especially important in the laboratories I created to study pipe behavior in isolation before integrating everything into the final shell execution flow.

---

## How To Use

To compile the project, run:

```bash
make
```

Then execute the shell:

```bash
./minishell
```

Once inside, you can test commands such as:

```bash
echo hello
ls -la
cat infile | grep text > outfile
export NAME=Fernando
echo "$NAME"
unset NAME
```

The shell will then handle:

* prompt display
* parsing
* syntax validation
* builtins
* pipes
* redirects
* heredoc
* environment behavior
* exit status tracking
* and signal-aware execution flow

---

## Team

**Minishell** was developed as a team project at **42 São Paulo**.

### Contributors

- **Fernando Ruan** — syntax checking, signal logic, pipe flow design, pipeline laboratories, command validation, Bash-like structural analysis
* **Jonas** (<a href="https://github.com/oJonasRtz/">oJonasRtz</a>) — lexer, file descriptor handling, environment variable logic, execution support

This project was especially important because it was our **first real team system project**, where parser architecture, execution logic, debugging, and shell behavior had to come together as one coherent program.

---

## Final Note

For me, **Minishell** was much more than a shell project.

It was a project about:

* understanding that a shell is fundamentally a parser
* learning how tokenization defines execution
* validating before trusting
* respecting the priority of pipes, redirects, and heredocs
* handling signals without breaking the terminal
* preserving file descriptor integrity
* understanding environment inheritance between parent and child processes
* reproducing the feeling of Bash through careful behavior, not through brute force

What makes **Minishell** special is that it looks simple from the outside.

A prompt.
A command.
A result.

But inside, there is a dense structure of parsing, validation, process control, and shell rules working together all the time.

That is what makes it one of the most educational projects in the **42 Common Core**.

**Final result:** `minishell` — **100/125**
**Status:** Completed

