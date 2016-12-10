# dsh Shell

This is an attempt to write a simple UNIX shell using literate programming. We'll
be using Go to write it, because I like Go.

(I intend to formalize the literate programming syntax I'm using later, but it
should be fairly straight forward if you read the markdown directly. After the
triple backticks and language declaration of a codeblock, if there's a string
surrounded by quotes, it's the name of the codeblock. It can be referenced in
other code blocks as <<<name>>>. If the declaration ends in +=, it means append
to a previous declaration. If there's a string without quotes after the language,
it's a filename to save the codeblock into. I have no idea how GitHub will
render this, but it works for writing it.)

## What does a simple shell do?

A simple shell needs to do a few things.

1. Read user input.
2. Interpret the user input.
3. Execute the user's input.

And repeat, until the user inputs something like an `exit` command. A good
shell does much more (like tab completion, file globbing, etc) but for now
we'll stick to the absolute basics.

## Main Body

We'll start with the main loop. Most Go programs have a file that looks
something like this:

```go main.go
package main

import (
	<<<main.go imports>>>
)

<<<main.go globals>>>

func main() {
	<<<mainbody>>>
}
```

Our main body is going to be a loop that repeatedly reads from `os.Stdin`.
This means that we should probably start by adding `os` to the import list.

```go "main.go imports"
"os"
```

And we'll start with a loop that just repeatedly reads a rune from the user.
`*os.File` (`os.Stdin`'s type) doesn't support the ReadRune interface, but
fortunately the `bufio` package provides a wrapper which allows us to convert
any `io.Reader` into a RuneReader, so let's import that too.

```go "main.go imports" +=
"bufio"
```

Then the basis of our main body becomes a loop that initializes and repeatedly
reads a rune from `os.Stdio`. For now, we'll just print it and see how it goes:

```go "mainbody"
r := bufio.NewReader(os.Stdin)
for {
	c, _, err := r.ReadRune()
	if err != nil {
		panic(err)
	}
	print(c)
}
```

The problem with this, is that `os.Stdin` sends things to the underlying `io.Reader`
one line at a time, and doesn't provide a way to change this, so it turns out
that there's no way in the standard library to force it to send one character
at a time. Luckily for us, the third party package `github.com/pkg/term` *does*
provide a way to manipulate the settings of POSIX ttys, which is all we need
to support. So instead, let's import that. (We'll still use bufio for the
simplicity of ReadRune.)

```go "main.go imports"
"github.com/pkg/term"
"bufio"
```

```go "mainbody"
// Initialize the terminal
t, err := term.Open("/dev/tty")
if err != nil {
	panic(err)
}
// Restore the previous terminal settings at the end of the program
defer t.Restore()

r := bufio.NewReader(t)
for {
	c, _, err := r.ReadRune()
	if err != nil {
		panic(err)
	}
	println(c)
}
```

That's a good start, but we probably want to handle the rune that's read in a
way other than just printing it. Let's keep track of what the current command
read is in a string, and when a newline is pressed, call a function to handle
the current command and reset the string. We'll declare a type for commands,
and do a little refactoring, just to be proactive.

```go "mainbody"
<<<Initialize Terminal>>>
<<<Command Loop>
```

```go "main.go globals"
type Command string
```

```go "Initialize Terminal"
// Initialize the terminal
t, err := term.Open("/dev/tty")
if err != nil {
	panic(err)
}
// Restore the previous terminal settings at the end of the program
defer t.Restore()
```

```go "Command Loop"
r := bufio.NewReader(t)
var cmd Command
for {
	c, _, err := r.ReadRune()
	if err != nil {
		panic(err)
	}
	switch c {
		case '\n':
			<<<Handle Command>>>
			cmd = ""
		default:
			cmd += Command(c)
	}
}
```

Okay, so how do we handle the command? If it's the string "exit"
we probably should exit. Otherwise, we'll want to execute it using
the [`os.exec`](https://golang.org/pkg/os/exec/) package. We were
proactive about declaring cmd as a type instead of a string, so
we can just define some kind of HandleCmd() method on the type and
call that.

```go "Handle Command"
if cmd == "exit" {
	os.Exit(0)
} else {
	cmd.HandleCmd();
}
```

```go "main.go globals" +=
<<<HandleCmd Implementation>>>
```

```go "HandleCmd Implementation"
func (c Command) HandleCmd() error {
	cmd := exec.Command(string(c))
	return cmd.Run()
}
```

We'll need to add `os` and `os/exec` to our imports, while we're
at it.

```go "main.go imports" +=
"os"
"os/exec"
```

If we run it and try executing something, it doesn't seem to be working.
What's going on? Let's print the error if it happens to find out.

```go "Handle Command"
if cmd == "exit" {
	os.Exit(0)
} else {
	err := cmd.HandleCmd()
	if err != nil {
		fmt.Fprintf(os.Stderr, "%v\n", err)
	}
}
```

```go "main.go imports" +=
"fmt"
```

There's still no error (unless we just hit enter without entering anything,
in which case it complains about not being able to execute "". (We should
probably handle that as a special case, too.) But what's going on with the
lack of any output or error? It turns out if we look at the `os/exec.Command`
documentation we'll see 

```go
// Stdout and Stderr specify the process's standard output and error.
//
// If either is nil, Run connects the corresponding file descriptor
// to the null device (os.DevNull).
//
// If Stdout and Stderr are the same writer, at most one
// goroutine at a time will call Write.
Stdout io.Writer
Stderr io.Writer
```

`Stdin` makes a similar claim. So let's hook those up to os.Stdin, os.Stdout
and os.Stderr.

While we're at it, let's print
a simple prompt.

So our new command handling code is:
```go "Handle Command"
if cmd == "exit" {
	os.Exit(0)
} else if cmd == "" {
	PrintPrompt()
} else {
	err := cmd.HandleCmd()
	if err != nil {
		fmt.Fprintf(os.Stderr, "%v\n", err)
	}
	PrintPrompt()
}
```

We need to define PrintPrompt() that we just used.
```go "main.go globals" +=
func PrintPrompt() {
	fmt.Printf("\n> ")
}
```

And we'll want to print it on startup too:

```go "Initialize Terminal" +=
PrintPrompt()
```

And our new HandleCmd() implementation.

```go "HandleCmd Implementation"
func (c Command) HandleCmd() error {
	cmd := exec.Command(string(c))
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	return cmd.Run()
}
```

Now we can finally run some commands from our shell!

But wait, when we run anything with arguments, we get an
error: `exec: "ls -l": executable file not found in $PATH`.
 exec is trying to run the command named "ls -l", not the
command "ls" with the parameter "-l". 


This is the signature from the GoDoc:

`func Command(name string, arg ...string) *Cmd`

*Not*

`func Command(command string) *Cmd`

We can use `Fields` function in the standard Go [`strings`](https://golang.org/pkg/strings)
package to split a string on any whitespace, which is basically what we want
right now. There will be some problems with that (notably we won't be able 
to enclose arguments in quotation marks if they contain whitespace), but at 
least we won't have to write our own tokenizer.

While we're at it, the "$PATH" in the error message reminds me. If there *are*
any arguments that start with a "$", we should probably expand that to the OS
environment variable.

```go "HandleCmd Implementation"
func (c Command) HandleCmd() error {
	parsed := strings.Fields(string(c))
	if len(parsed) == 0 {
		// There was no command, it's not an error, the user just hit
		// enter.
		PrintPrompt()
		return nil
	}

	var args []string
	for _, val := range parsed[1:] {
		if val[0] == '$' {
			args = append(args, os.Getenv(val[1:])
		} else {
			args = append(args, val)
		}
	}
	cmd := exec.Command(parsed[0], args...)
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	return cmd.Run()
}
```

```go "main.go imports"
"strings"
```

There's one minor noticable bug. If we hit `^D` on an empty line, the shell
panics with an error of "EOF". ReadRune is returning an EOF, which we should
handle gracefully, because it's not really an error condition.


```go "Command Loop"
r := bufio.NewReader(t)
var cmd Command
for {
	c, _, err := r.ReadRune()
	switch err {
		case nil: // do nothing
		case io.EOF:
			os.Exit(0)
		default:
			panic(err)
	}

	switch c {
		case '\n':
			<<<Handle Command>>>
			cmd = ""
		default:
			cmd += Command(c)
	}
}
```

```go "main.go imports" +=
"io"
```

And now.. hooray! We have a simple shell that works! We should add tab completion,
a smarter tokenizer, and a lot of other features if we were going to use this
every day, but at least we have a proof-of-concept, and maybe you learned something
by doing it.

If there's any other features you'd like to see added, feel free to either
create a pull request telling the story of how to you'd do it, or just create
an issue on GitHub and see if someone else does. Feel free to also file bug
reports in either the code or prose.

The final result of putting this all together after running `go fmt` is in the
accompanying `main.go` file.