Title: Debugging Compilers with FlowStorm
Date: 2023-10-19
Tags: clojure

This is (I hope) part of a series of blog posts with some ideas on using [FlowStorm](http://www.flow-storm.org), a
Clojure omniscient  and time travel debugger, to help us reason about Clojure systems.

This first post, is going to be part a FlowStorm overview, and part a tour of the ClojureScript compiler internals,
since we are going to be using it as en example of a non trivial system we would like to explore.

# Before starting

Following this post doesn't require any particular knowledge on compilers. The ClojureScript compiler can be seen as a
system that will take a string representing a Clojure code form, read it, recursively parse it into a tree of
expressions  also known as the AST (abstract syntax tree), and then walks down the tree emitting strings containing
JavaScript  code. 

You can think of it like : 

```clojure
(-> code-string
    (read)
    (analyze-and-macroexpand)
    (emit-js))
```

which is of course an over simplificaiton, but should be enough for following post.

If you like, you can follow along this post by copying and pasting the commands into your reminal. It will not
depend on any particular IDE or pre-installed tooling, appart from the clojure cli and git.

If it is your first encounter with FlowStorm, we call it a debugger, since a lot of its features are those found on
debuggers, but its capabilities can be used for much more than chasing bugs.

FlowStorm was designed as a tool for visualizing what is happening inside our Clojure programs as they run during
development, and specially designed with interactive programming and immutability in mind.

For people new to Clojure and Lisps in general, interactive programming is about developing a program by
interacting with its running process. This is quite different with the more traditional way of modifying your files,
recompilig everything (hopefully incrementally), running the process, stopping it, then rinse and repeat. Immutability
is about our programs dealing mostly with immutable values, insted of references to mutable objects or places in memory.

Interactivity and immutability alone are already quite powerful tools when trying to understand a system, given you can
poke at it by calling different functions, inspect the data, modify and recompile specific parts, all without losing your
application state or having to deal with long compilation times.

But poking at systems this way, via manual function calling, println (or the more modern tap>), scope capturing, single stepping,
etc, works their best when you know most of the system and you are confident that looking at specific points in the execution will
be enough to reveal the answers to your questions. So on top of that FlowStorm provides an easy way of recording and visualizing our programs
execution when we need it.

Ok, enough preamble, lets jump into it.

# Setting up

For working with the ClojureScript compiler we first need its sources, so lets start by clonning the repo's master branch :

```bash
$ git clone https://github.com/clojure/clojurescript
$ cd clojurescript
```

Now we can setup FlowStorm by modifying the project `deps.edn` file like this :

```clojure
{... 
 :aliases
 {...
  :storm
  {:classpath-overrides {org.clojure/clojure nil} ;; for disabling the official compiler
   :extra-deps {com.github.flow-storm/clojure {:mvn/version "1.11.1-11"}
                com.github.flow-storm/flow-storm-dbg {:mvn/version "3.8.1"}}
   :jvm-opts ["-Dclojure.storm.instrumentEnable=true"
              "-Dclojure.storm.instrumentOnlyPrefixes=cljs"
              "-Dflowstorm.startRecording=false"
              "-Dclojure.server.repl={:port 5555 :accept clojure.core.server/repl}"]}}}

```

There is quite a lot going on there, luckly we only need to do this once.

We added a new alias `:storm` so we can easily start a repl with everything we need. 

The important parts are :

  * we disabled the official Clojure compiler, since we are going to swap it by ClojureStorm, our dev compiler
  * added FlowStorm and ClojureStorm dependencies
  * told ClojureStorm that: 
    * we want instrumentation enable at startup
    * to instrument all namespaces under `cljs.*`
    *  and that recording should be disabled at startup
  * we also start a socket repl so we can have two terminals, one with a ClojureScript repl and another with a Clojure one,
    both on the same JVM process
  
Now we can finally run a Clojure repl with the `:storm` alias :

```bash
$ clj -A:storm
ClojureStorm 1.11.1-11

Evaluate the :help keyword for more info or :tut/basics for a beginners tour.
```

As soon as the repl starts, the first thing we do is to start the FlowStorm GUI, which we do by evaluating the `:dbg`
key in this ClojureStorm repl.

```clojure
user=> :dbg
user=> (require 'cljs.main)
user=> (cljs.main/-main "--repl")

ClojureScript 0.0.249695361
cljs.user=>
```

Right after, we require the main ClojureScript compiler namespace, and then run the main function with the `--repl`
arg (for people new to Clojure, the ClojureScript compiler is just a function we can invoke from our repl), which tells
it to start a browser repl. 
This should have opened a browser window and replaced our Clojure repl with a ClojureScript one. 
Everything we type on this new repl will be read, compiled into JavaScript and sent to the browser for execution.

And last but not least we will connect another Clojure repl to the same process our compiler is running, by using telnet
to connect to the socket repl we started earlier.

```bash
$ telnet 127.0.0.1 5555

user=>
```

And that is all the setup we need, at this point we should have :
    
  * a terminal with a Clojure repl
  * a terminal with a ClojureScript repl connected to the browser
  * the FlowStorm UI
  
# FlowStorm UI basics

If you are a FlowStorm user already you may want to skip this, otherwise I'm going to give a very quick tour of the
FlowStorm main UI.

<img src="assets/compilers-with-flow-storm/flow-storm-ui.png">

When you just start FlowStorm and you don't have any recordings you see :

1. Your clean button, to discard any recordings
2. Start/Stop recording button
3. A quick jump box to quickly jump into stepping any function
4. Your main tools tabs
5. And finally how much heap you have available, so you can keep an eye on it and clear your recordings when you are
   running out of heap space

Lets click the Start recording button, you should see the icon changing to a Stop, this is how you can tell if you
are currently recording.

Now we are recording let go to our ClojureScript repl terminal and eval a simple function, like : 

```clojure
cljs.user=> (defn sum [a b] (+ a b))
```

After you hit enter it will make ClojureScript read and compile that function, which will generate a bunch of recordings.

We can safely stop recording now, since we already have the recordings for a full compilation of our simple funciton.
It is good practise to stop recording when you don't need them since some applications will have threads constantly
polling which will keep unnecesarily wasting our heap and maybe polluting what we want to look at.

As soon as we have recordings, we will see a list of threads on the left appearing.

This means that activity was recorded in those threads, so this is already interesting since we get to see that typing
that expression on the repl is firing multiple threads.

Double clicking on a thread will open the thread recordings. We are just going to focus on the `main` thread since it is
where all the compilation happens, but feel free to explore other threads activities.

<img src="assets/compilers-with-flow-storm/flow-storm-ui-after-recording.png">

As soon as you open a thread you will be faced with the call tree, an expandable tree of all the functions calls
recorded. Time flows from top to bottom here. As you can see in the picture above, just by spanding some nodes a bit we
already see some familiar functions related to reading, parsing, and emitting javascript. By clicking on the `emit-str`
function we also get to see the input of `emit-str` which is an AST node, and it's output, which is a string containing
Javascript code.

There are also 3 important tools there : 

1. The call tree (the one we just talked about)
2. The code stepping tool, where you can step over the code forwards and backwards in time.
3. The functions list, where we get to see all the functions and their calls.

Since this isn't a FlowStorm tutorial I'll not go in depth into any of this tools, you can find the user guide here 
https://flow-storm.github.io/flow-storm-debugger/user_guide.html for more info.

Now lets look at how we can use FlowStorm to try to understand the different phases of the compilation.

# Parsing

## The reader

The first step of the ClojureScript compiler in terms of compilation is reading.

Read takes a string as the input and outputs a form, which is a nested structures of
Clojure lists, vectors, maps, etc, representing just the structure on the string.

ClojureScript for this uses the `clojure.tools.reader/read` which is part of the `clojure.tools.reader` library.

<img src="assets/compilers-with-flow-storm/reader1.png"> 

If we want to take a quick look at the read code we can use the `Quick jump` tool to search for a
read function like in the picture above. After hitting enter or clicking on it, it will move the 
stepper to the first and only call to this function. You can tell it was only called once from the number right next to
the name.

Now you can step using the controls at the top or click around the highlighted expression to see on the right panel what
the value for that expression was at that point in time.

As we can see from the `read` source code, it is calling `read*`. Looking at the `read*` arguments and return value
we can see it goes from a `SourceLoggingPushbackReader` into a Clojure form.
If you want to step into `read*`, position the debugger on any expression before the call, and then step next until you
jump into its source code.

Since `read*` has been called multiple time there is a better way of looking at all the read functions.

<img src="assets/compilers-with-flow-storm/reader2.png"> 

You can move to the functions list tool and constrain the list with only read functions like in the picture above.

Double clicking on each of them will list all the calls on the right with the arguments. Clicking on a call  will
show the return value in the bottom panel, while double clicking will take you to step that call.

If you are folloiwng along with your own setup take some time to explore around this read functions.

Now lets explore the analysis of those forms.

## Analisis and macroexpansion

Analisys is a recursive process that will take forms as input and produces a tree of ast nodes, which in this case
are plain Clojure maps with an `:op` key with the type of the node and `:children` containing a vector of keys
for the sub parts of the node.

<img src="assets/compilers-with-flow-storm/analyze1.png"> 

We can take the same approach as before to get an idea of this analisis functions and go to the functions list 
and constrain by `analyzer/analyze`. Lets take a look at `analyze-seq`. This time we are going to mute arguments 1, 3
and 4 using the checkboxes at the top and just look at the second one, which contains the form to analyze.
Lets click on the `(defn sum [...] ...)` one which is the top level experssion we typed at the repl.

If you look at the one right after it you will notice it is the same expression but after a `macroexpand-1` call.
This is because the `analize-seq` will also deal with macroexpansion. You can double click any of this calls to jump 
into `analyze-seq` body to check there is a call to `macroexpand-1` there.

Now back to the analyze-seq output of our defn expression. The panel at the bottom shows a pretty print of the node, but
for this nested datastructures it is inconvenient. Lets click on the `INS` to open the inspector, which allows us to
navigate this nested datastructures back and forth.

<img src="assets/compilers-with-flow-storm/analyze1-inspect-def-op.png"> 

You can click around to navigate deeper and use the breadcrumbs at the top of the inspector to navigate backwards.

Even with the inspector, trying to have a sense of the structure of this tree is kind of hard, so lets pull another
feature.

We can take any of this values back to our repl by giving them a name. While having the inspector at the root of our
value, click on the `DEF` button and give it a name, lets say `def-op`.

Now you can go to your Clojure repl (not the ClojureScript one) and eval something like `(keys def-op)` or do anything
with it at the repl.

We are going to write a little helper function here to walk down the tree and just print the `:op` with indentation
which will help us understand more about the structure of this tree.

```clojure
(defn print-ast-node
  
  "Recursively print ast-node ops with indentation"

  ([ast-node] (print-ast-node ast-node 0))
  ([{:keys [op children] :as ast-node} indent-level]
   (println (apply str (repeat (* 2 indent-level) " "))
            (str "op: " op))
   (doseq [ch-key children]
     (let [ch-node (get ast-node ch-key)]
       (if (vector? ch-node)
         (doseq [ch ch-node]
           (print-ast-node ch (inc indent-level)))
         (print-ast-node ch-node (inc indent-level)))))))
```

Now we can call it with our `def-op` reference

```clojure
user=> (print-ast-node def-op)
 op: :def
   op: :var
   op: :fn
     op: :binding
     op: :fn-method
       op: :binding
       op: :binding
       op: :do
         op: :js
           op: :local
           op: :local
```

which will give us a much better idea of the tree structure.

Take your time exploring the analysis code for each of those operations before jumping into the emission.

Ok, before jumping to the emission lets navigate to that `:op :js` which seams interesting. Go back to the root of the
value and dig into this keys `[:init :methods 0 :body :ret]`. You should have the map with the `:op :js` focused on your
inspector right pane.

Lets say we want to examine the emission of this particular node. One trick we could use is, keep the inspector open,
goto the main window and then :

1. jump to the code stepping tool
2. step to the last trace 
3. and now back into the inspector hit "Find the prev expression that contains this value", which is the left arrow at
   the top.

What this will do is search the first usage of the current focused value from the back, which  should leave us in the
emit code for that node. 

# Emmiting JS

We should be now positioned right before calling `(emit* ast)` with ast being our node, inside the `cljs.compile/emit`
funciton. 

<img src="assets/compilers-with-flow-storm/emit-js-op.png"> 

To get some more context and see how we got here, we can look at the current stack on the bottom left panel. I looks like
we are in the path of emitting our `:def`, `:fn` and  `:do` ast nodes, which make sense.

It also looks like we are emitting some other code, something wrapped in a try, which we haven't typed on 
our repl. This is because the ClojureScript compiler is wrapping our code in some extra code that it needs
before evaluating it on our browser, but we will ignore that for now.

Now stepping forward a couple of times should bring us to the emits function, which is the function
that will finally emit javascript.

<img src="assets/compilers-with-flow-storm/emits-js-op.png"> 

Inspecting the value of `s` looks like for our node, it starts by emitting the return keyword.

We can also see here that the way the ClojureScript compiler emits Javascript is by wrinting on `*out*`.
Since this is the place that writes Javascript strings, it could be interesting to see what are all the values being
written along the entire execution, not just this time. 

For this we are going to introduce one more feature, called the `Printer`. For inspecting the value of `s` along the
execution we start by right clicking the `s` expression and then `Add to prints`. A dialog should popup asking for a
message, we can write whatever we want here, this is the kind of messages we always add to our println debugging. 
We can do this on as many expressions as we want, but this one should be enough for our purpose.

<img src="assets/compilers-with-flow-storm/emits-printed.png"> 

We can find the `Printer` tool by clicking on its tab, on the left.

At the top we should see a thread selector, and right after it all our defined printers. For running the printers we
need to select a thread, in this case `main` and then click the refresh button. We should now see all our prints for the
entire execution of the `main` thread, like in the picture above.

We can use the filter textfield to shrink our prints if we are searching for any particular emited Javascript.
If you are interested in any particular print you can double click it and it will take us to the statement that print it
at that point in time, where we can then use the rest of the tools to keep inspecting.

Go ahead and double click on any of them, so we are back on our `emits` function.

Using the printer we get to see all the single writes to `*out*` , which is also all the emited Javascript, but it could
also be interesting to see it a little nicer, so we can copy and paste it in our editor for example.

We can use the fact that `*out*` is bounded to the same mutable value, a `StringWriter`, the whole time, which means that
whatever reference we get to it contains the final state, the one it took after the execution.

Most of the time this is not what we want, and FlowStorm provides a way for dealing with mutable values, if you are
interested take a look here https://flow-storm.github.io/flow-storm-debugger/user_guide.html#_dealing_with_mutable_values

But in this case we are going to take advantage of it, and just give it a name so we can take it to our repl.

This is the same we did before for our `def-node`, we click on the `*out*` expression and then on the `DEF` button at the
right panel.

Now going back to our Clojure repl we can just print it.

```js
user=> (println (.toString out))

(function() {
    try {
        return cljs.core.pr_str.call(null, (function() {
            var ret__13392__auto__ = (function() {
                cljs.user.sum = (function cljs$user$sum(a, b) {
                    return (a + b);
                });
                return (
                    new cljs.core.Var(function() {
                        return cljs.user.sum;
                    }, new cljs.core.Symbol("cljs.user", "sum", "cljs.user/sum", 1580982348, null), cljs.core.PersistentHashMap.fromArrays([new cljs.core.Keyword(null, "ns", "ns", 441598760), new cljs.core.Keyword(null, "name", "name", 1843675177), new cljs.core.Keyword(null, "file", "file", -1269645878), new cljs.core.Keyword(null, "end-column", "end-column", 1425389514), new cljs.core.Keyword(null, "source", "source", -433931539), new cljs.core.Keyword(null, "column", "column", 2078222095), new cljs.core.Keyword(null, "line", "line", 212345235), new cljs.core.Keyword(null, "end-line", "end-line", 1837326455), new cljs.core.Keyword(null, "arglists", "arglists", 1661989754), new cljs.core.Keyword(null, "doc", "doc", 1913296891), new cljs.core.Keyword(null, "test", "test", 577538877)], [new cljs.core.Symbol(null, "cljs.user", "cljs.user", 877795071, null), new cljs.core.Symbol(null, "sum", "sum", 1777518341, null), "<cljs repl>", 10, "sum", 1, 1, 1, cljs.core.list(new cljs.core.PersistentVector(null, 2, 5, cljs.core.PersistentVector.EMPTY_NODE, [new cljs.core.Symbol(null, "a", "a", -482876059, null), new cljs.core.Symbol(null, "b", "b", -1172211299, null)], null)), null, (cljs.core.truth_(cljs.user.sum) ? cljs.user.sum.cljs$lang$test : null)])));
            })();
            (cljs.core._STAR_3 = cljs.core._STAR_2);

            (cljs.core._STAR_2 = cljs.core._STAR_1);

            (cljs.core._STAR_1 = ret__13392__auto__);

            return ret__13392__auto__;
        })());
    } catch (e24083) {
        var e__13393__auto__ = e24083;
        (cljs.core._STAR_e = e__13393__auto__);

        throw e__13393__auto__;
    }
})()
```

And there we have it, a quick way of looking at the last form entire generated Javascript string. Note that it is the
full form that is going to be sent to the browser, which contains our `sum` function definition also wrapped in some
extra code to print the result and also setup ClojureScript repl `*1`, `*2` and `*3` vars and in case of an exception also `*e`.

# Wrapping up

So wrapping up we saw a bunch of techniques we can use to debug and understand the ClojureScript compiler. If we are
developing it, this gives as a nice workflow : 

* Start recording
* On the ClojureScript repl write the expression we are interested in, so it compiles it
* Stop recording
* Look around 
* Modify and re-eval your compiler code
* Repeat

All the steps described there should be instant, since we are re-evaluating just what we changed on the ClojureScript
compiler, and also compiling and recording one form at a time.

Also all the techniques described here are by no means special to compiler development, they can be used on any Clojure
application. 

Ok, I hope you had fun, find it interesting or learn something from this. Happy hacking.

Until the next time!
