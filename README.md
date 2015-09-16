
## lumen-notes

lumen is a lisp that runs on javascript, lua, and luajit.  See [sctb/lumen](https://github.com/sctb/lumen)

this is a collection of notes made while learning it.

### lumen

folder structure looks like

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

easier to understand by split output into categories:

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

and 

```
$ cat obj/.gitignore 
*.js
*.lua
```

so we see that lumen uses these conventions:

- a github repo is a lumen lib
- lumen code (`.l` files) go in main repo dir
- `obj/` is where lumen stores compilation output
- `bin/lumen` is how you run lumen
- `bin/` is interesting.  as you write a lumen lib, you'll sometimes want 
  direct control over js and lua.  to do this for e.g. `foo.l`, make
  `bin/foo.js` and `bin/foo.lua` and define whatever you want in them.
  then you can call those defs within `foo.l`


### motor

motor is an example of how to write a public lumen library.  See
[sctb/motor](https://github.com/sctb/motor)

- provides ffi, very easy to call into c; see [sctb/motor/pq.l](https://github.com/sctb/motor/blob/master/pq.l) and http://luajit.org/ext_ffi.html
- provides postgres bindings







