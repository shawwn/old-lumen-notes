
## lumen-notes

lumen is a lisp that runs on javascript, lua, and luajit.  See [sctb/lumen](https://github.com/sctb/lumen)

this is a collection of notes made while learning it.

### motor

motor is an example of how to write a lumen library.  See
[sctb/motor](https://github.com/sctb/motor)

- provides ffi, very easy to call into c; see [sctb/motor/pq.l](https://github.com/sctb/motor/blob/master/pq.l) and http://luajit.org/ext_ffi.html
- provides postgres bindings

### lumen

lumen folder structure looks like

```
$ find . -type f | grep -v ^./.git
./bin/compiler.js
./bin/compiler.lua
./bin/lumen
./bin/lumen.js
./bin/lumen.lua
./bin/reader.js
./bin/reader.lua
./compiler.l
./LICENSE
./macros.l
./main.l
./makefile
./obj/.gitignore
./reader.l
./README.md
./runtime.l
./test.l
```

easier to understand when it's split into categories:

```
bin/lumen

bin/compiler.js
bin/lumen.js
bin/reader.js

bin/compiler.lua
bin/lumen.lua
bin/reader.lua

obj/.gitignore

compiler.l
macros.l
main.l
reader.l
runtime.l
test.l

makefile

README.md
LICENSE
```

and .gitignore is

```
$ cat obj/.gitignore 
*.js
*.lua
```

lumen doesn't force you follow a naming scheme or to arrange your
folder structure, but the stdlibs seem to follow these:

- a github repo is a lumen lib

- lumen code (`.l` files) go in main repo dir

- `obj/` is where lumen stores compilation output

- `bin/lumen` is how you run lumen

- `bin/` is interesting. `{reader,compiler}.l` were compiled to
`bin/{reader,compiler}.{js,lua}`, but `bin/lumen.{js,lua}` weren't
compiled from any `lumen.l` file (it doesn't exist)

- the reason is because `lumen.{lua,js}` are one level below the lumen language itself.  The language has to start somewhere, and you'd want it to start in a .lua or .js file.  In principle you could use an existing build of lumen for compiling a lumen.l into lumen.{js,lua}, but it probably wouldn't be a good idea.  choosing the right level of abstraction is important, and lumen.{js,lua} are probably best expressed as currently written.  In fact, by not sharing the codebase, you gain a few advantages: it's possible to write optimizations for lua without being concerned how you'd affect js (except insofar as worrying your optimization might cause lumen on lua to behave differently vs lumen on js.)

```
lumen$ bin/lumen --help
usage: lumen [options] <object files>
options:
  -c <input>	Compile input file
  -o <output>	Output file
  -t <target>	Target language (default: lua)
  -e <expr>	Expression to evaluate
```

so `bin/lumen -e "(+ 1 2)"` => `3`

`$ bin/lumen -c main.l -o obj/main.lua -t lua`
eqv to `$ bin/lumen -c main.l -o obj/main.lua`

but `$ bin/lumen -c main.l -o -` doesn't print to stdout; creates
output file named ./- 

### Let's Play Lumen

I decided to write this section aking to something of a "Let's Play"
style. It's intended for intermediate-level lisp programmers who feel
comfortable with the basics but feel a bit disillusioned or lost.
I vividly remember how difficult it had been to learn lisp, so I
thought I'd try to help others avoid similar issues.  I suspect people
face different challenges; I was pretty decent at writing lisp, but
terrible at reading anyone else's.  But keep trying; determination is
all it takes.  This section is written very verbosely, perhaps overly
so, in order to show ways of making sense of lisp programs.  In
a real program you wouldn't (necessarily) do this, but on the other
hand, it wouldn't be terrible.  My old worst case scenario was that
I'd get hopelessly lost without a clue how to explore the code.  So
I'll show you some tricks for avoiding that fate.  (Maybe no one else 
deals with that, though.)

Let's go!

We saw in the last section that `bin/lumen -c foo.l -o foo.lua` is
how you compile a `foo.l` file to `foo.lua`.  But now we need some
sort of excuse to explore the codebase; wouldn't it be fun to add a
feature?  It's admittedly not very useful, but we're going to tour
so many cool things along the way!

Adding a new feature is usually the best way to learn unfamiliar
territory, regardless of the project.  It doesn't have to be useful
or even a good idea, just fun.

What we're going to do is change Lumen's behavior so that 
`bin/lumen -c foo.l -o -` will print the compiled code to stdout
rather than needing to write it to disk before seeing the result.

Ok, going to drop to a terser style now, but follow along by running
`git clone https://github.com/sctb/lumen; cd lumen`

plan:
- want to find where -o flag is implemented
- check whether val of -o flag is -
- if so, then print to stdout instead of write to disk

```
$ ag '\-o' | wc -l
      19
```

hmm, we found 19 matches when searching for -o

let's search for --help instead, because the functionality will be
right beside -o

```
$ ag '\--help' | cat
bin/lumen.js:1118:  if (hd(argv) === "-h" || hd(argv) === "--help") {
bin/lumen.lua:1000:  if hd(argv) == "-h" or hd(argv) == "--help" then
main.l:44:            (= (hd argv) "--help"))
```

bingo, pop open `main.l` and search for -o in your editor.  We end up
at main:

```scm
(define main ()
  (when (or (= (hd argv) "-h")
            (= (hd argv) "--help"))
    (usage))
  (let (pre ()
        input nil
        output nil
        target1 nil
        expr nil
        n (# argv))
    (for i n
      (let a (at argv i)
        (if (or (= a "-c") (= a "-o") (= a "-t") (= a "-e"))
            (if (= i (- n 1))
                (print (cat "missing argument for " a))
              (do (inc i)
                  (let val (at argv i)
                    (if (= a "-c") (set input val)
                        (= a "-o") (set output val)
                        (= a "-t") (set target1 val)
                        (= a "-e") (set expr val)))))
            (not (= "-" (char a 0))) (add pre a))))
    (step file pre
      ((get compiler 'run-file) file))
    (if (and input output)
        (do (if target1 (set target target1))
            (let code ((get compiler 'compile-file) input)
              (write-file output code)))
      (if expr (rep expr) (repl)))))
```

to make sense of traditional code (python, C, anything) your eyes almost
always start at the top, just like a book, and you work your way down.

but that turns out not to be very effective for lisp.  The shape of lisp
programs are qualitatively different.  (the reason for this turns out to be
the same reason that makes it powerful: code is a list.  you can do to
code all of the same things you'd expect from a list:  iterate over it,
break them apart, rearrange them, or forge new ones.)

The trick to reading lisp is to train yourself to start from the *center* and
work your way outward.  You usually have some general idea of what it is
you're looking for, and in this case we're looking to see how -o
works.  So let's start there:

```scm
(define main ()
  (when (or (= (hd argv) "-h")
            (= (hd argv) "--help"))
    (usage))
  (let (pre ()
        input nil
        output nil
        target1 nil
        expr nil
        n (# argv))
    (for i n
      (let a (at argv i)
        (if (or (= a "-c") (= a "-o") (= a "-t") (= a "-e"))
            (if (= i (- n 1))
                (print (cat "missing argument for " a))
              (do (inc i)
                  (let val (at argv i)
                    (if                           
                        (= a "-o") (set output val)
                                                    
                                                     
            (not (= "-" (char a 0))) (add pre a))))
    (step file pre
      ((get compiler 'run-file) file))
    (if (and input output)
        (do (if target1 (set target target1))
            (let code ((get compiler 'compile-file) input)
              (write-file output code)))
      (if expr (rep expr) (repl)))))
```

We erased the nearby expressions that had nothing to do with -o.
In fact, at this point the essential thing to notice is

```
                        (= a "-o") (set output val)
```

that's simple enough to break into two steps: figure out what the
first set of parens are doing, then figure out the next one.

`(= a "-o")` will either seem completely obvious or completely opaque,
but don't worry either way.  Just remember the tricks whenever you
get lost.  In this case it's just a list containing three pieces of code.




so the -o flag's value is stored in var named output

`-c <infile>` and `-o <outfile>` must both be set in order for infile
to be compiled (aha, so that's why `bin/lumen -c main.l` is eqv to
`bin/lumen`...)

to understand any lisp def, the trick is to start right in the center
of the def then start to "feel" your way outwards, understanding each
expr as you go

in this case, start by understanding `(get compiler 'run-file)`:

- we search for "compiler" in our editor, and we find `(define compiler (require 'compiler))` at top of file
- now need to understand what (require 'compiler) does

```
lumen$ ag require
 bin/compiler.js  1     var reader = require("reader");
 bin/compiler.js  1000  global.require = require;
    bin/lumen.js  656   var fs = require("fs");
    bin/lumen.js  1067  var reader = require("reader");
    bin/lumen.js  1068  var compiler = require("compiler");

bin/compiler.lua  1     local reader = require("reader")
   bin/lumen.lua  948   local reader = require("reader")
   bin/lumen.lua  949   local compiler = require("compiler")

      compiler.l  3     (define reader (require 'reader))
      compiler.l  529   (target js (set (get global 'require) require))
          main.l  3     (define reader (require 'reader))
          main.l  4     (define compiler (require 'compiler))
       runtime.l  332   (target js (define fs (require 'fs)))
          test.l  7     (define reader (require 'reader))
          test.l  8     (define compiler (require 'compiler))
```


ah, lua has require built in, but js has to do something special:

```
(target js (set (get global 'require) require))
```

so lumen `require` is literally lua `require` when using lua.  What is
it for js?  Doesn't matter; whatever it does, we know it has have the
same effect. 

in lua, `local foo = require("foo")` causes all defs in `foo.lua` to
be placed into a local variable named foo (it's a table)

ok, back to main.l:

```scm
(define main ()
  (when (or (= (hd argv) "-h")
            (= (hd argv) "--help"))
    (usage))
  (let (pre ()
        input nil
        output nil
        target1 nil
        expr nil
        n (# argv))
    (for i n
      (let a (at argv i)
        (if (or (= a "-c") (= a "-o") (= a "-t") (= a "-e"))
            (if (= i (- n 1))
                (print (cat "missing argument for " a))
              (do (inc i)
                  (let val (at argv i)
                    (if (= a "-c") (set input val)
                        (= a "-o") (set output val)
                        (= a "-t") (set target1 val)
                        (= a "-e") (set expr val)))))
            (not (= "-" (char a 0))) (add pre a))))
    (step file pre
      ((get compiler 'run-file) file))
    (if (and input output)
        (do (if target1 (set target target1))
            (let code ((get compiler 'compile-file) input)
              (write-file output code)))
      (if expr (rep expr) (repl)))))
```

now interested in that expr containing "-"

strip parts we don't care about:

```scm
(define main ()
  (when (or (= (hd argv) "-h")
            (= (hd argv) "--help"))
    (usage))
  (let (pre ()
                 
    (for i n
      (let a (at argv i)
        (if (or (= a "-c") (= a "-o") (= a "-t") (= a "-e"))
                             
            (not (= "-" (char a 0))) (add pre a))))

    (step file pre
      ((get compiler 'run-file) file))
```

so pre is a list, and any cmdline arg that doesn't start with - is added to it

`step` is lumen eqv of `each`, e.g.

```
(step x (list 1 2 3 4)
  (print x))
```

so, `(get compiler 'run-file)` is eqv to lua doing `compiler.run-file`, i.e. fetches run-file fn from compiler.l

therefore

```
    (step file pre
      ((get compiler 'run-file) file))
```

means compiler.l's run-file fn will be called on every cmdline arg that doesn't start with -

what does that actually do? let's check:

```
(define run-file (path)
  (run (read-file path)))
```

what's it do?  does it behave like arc's readfile, where `(readfile "foo.arc") yields a list of all exprs in `foo.arc`?

no, turns out to behave like arc's `(filechars "foo.arc")`, e.g. returns the file as a string.

interestingly, read-file isn't defined in compiler.l

it's in runtime.l

and the def is interesting:

```
(define-global read-file (path)
  (target
    js: ((get fs 'readFileSync) path 'utf8)
    lua: (let f ((get io 'open) path)
	   ((get f 'read) f '*a))))
```

make sense of this by focusing on middle exprs first:

```
(define-global read-file (path)
         
    js: ((get fs 'readFileSync) path 'utf8)
                                     
                            
```

so, on js we use `fs.readFileSync` to read a file

```
(define-global read-file (path)
         
                                           
    lua:        ((get io 'open) path)
                            
```

```
(define-global read-file (path)
         
                                           
    lua: (let f                      
	   ((get f 'read) f '*a))))
```

and on lua we use `io.open(path).read("*a")`

cool!  ok, almost ready to understand last bit of main.l:

```scm
(define main ()
  (when (or (= (hd argv) "-h")
            (= (hd argv) "--help"))
    (usage))
  (let (pre ()
        input nil
        output nil
        target1 nil
        expr nil
        n (# argv))
    (for i n
      (let a (at argv i)
        (if (or (= a "-c") (= a "-o") (= a "-t") (= a "-e"))
            (if (= i (- n 1))
                (print (cat "missing argument for " a))
              (do (inc i)
                  (let val (at argv i)
                    (if (= a "-c") (set input val)
                        (= a "-o") (set output val)
                        (= a "-t") (set target1 val)
                        (= a "-e") (set expr val)))))
            (not (= "-" (char a 0))) (add pre a))))
    (step file pre
      ((get compiler 'run-file) file))
    (if (and input output)
        (do (if target1 (set target target1))
            (let code ((get compiler 'compile-file) input)
              (write-file output code)))
      (if expr (rep expr) (repl)))))
```

let's understand -e first:

```
$ bin/lumen -e '(+ 1 2)'
3
```

```scm
(define main ()
            
  (let (      
                   
        expr nil
                  )
    (for i n
      (let a (at argv i)
                                                    
                        (= a "-e") (set expr val)))))
                                        
      (if expr (rep expr) (repl)))))
```

so -e is stored in a var named expr

evaluation must happen here:

```scm
      (if expr (rep expr) (repl)))))
```

but what does rep mean?

ah, readprint.

```
$ ag '\brep\b' -C1 | cat

bin/lumen.js:1087  };
bin/lumen.js:1088  var rep = function (s) {
bin/lumen.js:1089    return(eval_print(reader["read-string"](s)));
--                                                            
--                                                            
bin/lumen.js:1174      if (expr) {
bin/lumen.js:1175        return(rep(expr));
bin/lumen.js:1176      } else {


bin/lumen.lua:964  end
bin/lumen.lua:965  local function rep(s)
bin/lumen.lua:966    return(eval_print(reader["read-string"](s)))
--                                                            
--                                                            
bin/lumen.lua:1056      if expr then
bin/lumen.lua:1057        return(rep(expr))
bin/lumen.lua:1058      else


main.l:10  
main.l:11  (define rep (s)
main.l:12    (eval-print ((get reader 'read-string) s)))
--                                                            
--                                                            
main.l:71                    (write-file output code))))
main.l:72        (if expr (rep expr) (repl)))))
main.l:73 -
```

ok, so rep takes a string and gives it to read-string, which 
yields an expr that's passed to eval-print. 

(q: does read-string yield one expr, or all epxrs?)
(a: one. `$ bin/lumen -e '(+ 1 2) (- 1 2)'` => `3`)


```scm
(define main ()
  (when (or (= (hd argv) "-h")
            (= (hd argv) "--help"))
    (usage))
  (let (pre ()
        input nil
        output nil
        target1 nil
        expr nil
        n (# argv))
    (for i n
      (let a (at argv i)
        (if (or (= a "-c") (= a "-o") (= a "-t") (= a "-e"))
            (if (= i (- n 1))
                (print (cat "missing argument for " a))
              (do (inc i)
                  (let val (at argv i)
                    (if (= a "-c") (set input val)
                        (= a "-o") (set output val)
                        (= a "-t") (set target1 val)
                        (= a "-e") (set expr val)))))
            (not (= "-" (char a 0))) (add pre a))))
    (step file pre
      ((get compiler 'run-file) file))
    (if (and input output)
        (do (if target1 (set target target1))
            (let code ((get compiler 'compile-file) input)
              (write-file output code)))
      (if expr (rep expr) (repl)))))
```

ok, finish line:

```scm
(define main ()
  (let (      
        input nil
        output nil
        target1 nil
        expr nil
                  )
    (for i n
      (let a (at argv i)
        (if (or (= a "-c") (= a "-o") (= a "-t") (= a "-e"))
            (if (= i (- n 1))
                (print (cat "missing argument for " a))
              (do (inc i)
                  (let val (at argv i)
                    (if (= a "-c") (set input val)
                        (= a "-o") (set output val)
                        (= a "-t") (set target1 val)
                                                     
                                                   
                  
                                      
    (if (and input output)
        (do (if target1 (set target target1))
            (let code ((get compiler 'compile-file) input)
              (write-file output code)))
                                 )))
```

```scm
    (if (and input output)
        (do (if target1 (set target target1))
            (let code ((get compiler 'compile-file) input)
              (write-file output code)))
```


```scm
            (if target1 (set target target1))
```

```scm
            (let code ((get compiler 'compile-file) input)
              (write-file output code)))
```

ok, so for `bin/lua -c foo.l`, `input` will be `"foo.l"`.

`compiler.compile-file` reads `foo.l` from disk, then parses it, iterating over all the lisp exprs, turning each expr into lua (when `-t lua`) or js (when `-t js`)

`code` gets the result, then written to disk via `write-file`.








