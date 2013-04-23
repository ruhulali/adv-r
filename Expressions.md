# Walking the code tree

<!-- 
library(pryr)
library(stringr)
find_funs("package:base", fun_calls, fixed("substitute"))
find_funs("package:base", fun_calls, fixed("eval"))
find_funs("package:base", fun_calls, fixed("match.call"))
find_funs("package:base", fun_calls, fixed("quote"))
 -->

''Flexibility in syntax, if it does not lead to ambiguity, would seem a reasonable thing to ask of an interactive programming language.'' --- Kent Pitman, http://www.nhplace.com/kent/Papers/Special-Forms.html

It's generally a bad idea to create code by operating on its string representation: there is no guarantee that you'll create valid code.  Don't get me wrong: pasting strings together will often allow you to solve your problem in the least amount of time, but it may create subtle bugs that will take your users hours to track down. Learning more about the structure of the R language and the tools that allow you to modify it is an investment that will pay off by allowing you to make more robust code.

Differences with macros: occurs at runtime, not compile time (which doesn't have any meaning in R). Returns results, not expression.  More like `fexprs`. a fexpr is like a function where the arguments aren't evaluated by default; or a macro where the result is a value, not code. 

Thoroughout this chapter we're going to use tools from the `pryr` package to help see what's going on.  If you don't already have it, install it by running `devtools::install_github("pryr")`

Why?

* to create tools that inspect functions and warn you of common problems

Manipulating calls and functions

* Calls in more detail

* Implement a function by calling another function (bad idea): `write.csv()`, `xtabs()`

* Parsing and evaluating text: `source`

* Walk the code tree: `pryr:call_tree()`, `codetools::findGlobals`, find assignments, find calls

* Modifying calls: `partial_eval`

* Create functions by hand as an alternative to closures: `pryr::make_function`, `pryr::unenclose`, `pryr::partial`, `memoise`


Because code is a tree, we're going to need recursive functions to work with it. You now have basic understanding of how code in R is put together internally, and so we can now start to write some useful functions. In this section we'll write functions to:

* List all assignments in a function
* Replace `T` with `TRUE` and `F` with `FALSE`

The `codetools` package, included in the base distribution, provides some other built-in tools based for automated code inspection:

* `findGlobals`: locates all global variables used by a function.
  This can be useful if you want to check that your functions don't inadvertently rely on variables defined in their parent 
  environment.

* `checkUsage`: checks for a range of common problems including 
  unused local variables, unused parameters and use of partial 
  argument matching.

* `showTree`: works in a similar way to the `drawTree` used above, 
  but presents the parse tree in a more compact format based on 
  Lisp's S-expressions

## Basics of R code

To compute on the language, we first need to be understand the structure of the language. That's going to require some new vocabulary, some new tools and some new ways of thinking about R code. The first thing you need to understand is the distinction between an operation and its result:

```R
x <- 4
y <- x * 10
```

We want to distinguish between the operation of multiplying x by 10 and assinging the result to `y` compared to the actual result (40).  In R, we can capture the operation with `quote()`:

```R
z <- quote(y <- x * 10)
z
```

`quote()` gives us back a quoted call: it's like a regular function call that hasn't happened yet. A quoted call is often called an expression, but we'll avoid that term because R already has expression objects which are lists of calls. `is.call()` checks if you have a quoted call:

```R
is.call(z)
```

A quoted call is also called an abstract syntax tree (AST) because it represents the abstruct structure of the code in a tree form. We can use `pryr::call_tree()` to see the hierarchy more clearly:

```R
library(pryr)
call_tree(z)
```

There are three basic things you'll commonly see in a call tree: constants, names and calls.

* __constants__ are atomic vectors, like `"a"` or `10`. These appear as is. 

    ```R
    call_tree(quote("a"))
    call_tree(quote(1))
    call_tree(quote(1L))
    call_tree(quote(TRUE))
    ```

* __names__ which represent the name of a variable, not its value. (Names are also sometimes called symbols). These are prefixed with `'`

    ```R
    call_tree(quote(x))
    call_tree(quote(mean))
    call_tree(quote(`an unusual name`))
    ```

* __calls__ represent the action of calling a function, not its result. These are suffixed with `()`.  The arguments to the function are listed below it, and can be constants, names or other calls.

    ```R
    call_tree(quote(f()))
    call_tree(quote(f(1, 2)))
    call_tree(quote(f(a, b)))
    call_tree(quote(f(g(), h(1, a))))
    ```

    As mentioned in [[Functions]], even things that don't look like function calls still follow this same hierarchical structre:

    ```R
    call_tree(quote(a + b))
    call_tree(quote(if (x > 1) x else 1/x))
    call_tree(quote(function(x, y) {x * y}))
    ```

(In general, it's possible for any type of R object to live in a call tree, but these are the only three types you'll get from parsing R code. It's possible to put anything else inside an expression using the tools described below, but while technically correct, support is often patchy.)

Note that `str()` is somewhat inconsistent with respect to this naming convention, describing calls as a language objects:

```R
str(quote(a + b))
```

Constants, names and calls define the structure of all R code. 

Computing on the language involves manipulating quoted calls in different ways.  You can:

* evaluate them, to execute the operations that they represent
* convert back and forth to text
* modify them

### Constants

Note that `quote()` is idempotent when you give it a single value:

```R
identical(1, quote(1))
identical("test", quote("test"))
```

But not when you give it multiple values, because you always create a vector of multiple values a some call:

```R
identical(1:3, quote(1:3))
identical(c(FALSE, TRUE), quote(c(FALSE, TRUE)))
```

### Names

Another way to create names is to convert strings to names:

```R
as.name("name")
identical(quote(name), as.name("name"))
```

#### The missing argument

If you experiment with the structure function arguments, you might come across a rather puzzling beast: the "missing" object:

    formals(plot)$x
    x <- formals(plot)$x
    x
    # Error: argument "x" is missing, with no default

It is basically an empty name, but you can't create it directly:

    is.name(formals(plot)$x)
    # [1] TRUE
    deparse(formals(plot)$x)
    # [1] ""
    as.name("")
    # Error in as.name("") : attempt to use zero-length variable name
    
You can either capture it from a missing argument of the formals of a function, as above, or create with `substitute()` or `bquote()`.

### Calls

You can also create calls by hand using `as.call()` or `call()`.  `call()` takes a string giving a function name, and additional arguments should be other expressions. `as.call()` takes a list where the first argument is the _name_ of a function (not a string), and the subsequent values are the arguments. 

```R
x_call <- call(":", 1, 10)
mean_call <- as.call(list(quote(mean), x_call))

identical(mean_call, quote(mean(1:10)))
```

Note that the following two calls look the same, but are actually different:

```R
(a <- call("mean", 1:10))
(b <- call("mean", quote(1:10)))
identical(a, b)
call_tree(a)
call_tree(b)
```

In the `a`, the first argument is a integer vector containing the numbers 1 to 10, and in `b` the first argument is a call to `:`.  You can put any R object into a expression, but the printing of expression objects will not always show the difference.  

The key difference is where/when evaluation occurs:

```R
a <- call("print", Sys.time())
b <- call("print", quote(Sys.time()))
eval(a); Sys.sleep(1); eval(a)
eval(b); Sys.sleep(1); eval(b)
```

The behaviour of a call is almost identical to that of a list: a call has `length`, `'[[` and `[` methods. The length of a call minus 1 gives the number of arguments:

```R
x <- quote(read.csv(x, "important.csv", row.names = FALSE))
length(x) - 1
# [1] 3
```

The first element of the call is the _name_ of the function:

```R
x[[1]]
# read.csv

is.name(x[[1]])
# [1] TRUE
```

Or a function that produces a function:

```R
(function(x) x + 1)(10))
add <- function(y) function(x) x + y
add(1)(10)

# Note that you can't create this sort of call with call
call("add(1)", 10)
# But you can with as.call
as.call(list(quote(add(1)), 10))
```

The remaining elements are the arguments to the function, which can be extracted by name or by position.

```R
x$row.names
# FALSE
x[[3]]
# [1] "important.csv"

names(x)
```

Generally, extracting arguments by position is dangerous, because R's function calling semantics are so flexible. It's better to match by name, but all arguments might not be named. The solution to this problem is to use `match.call` which takes a function and a call as arguments:

```R
match.call(read.csv, x)
# Or if you don't know in advance what the function is
match.call(eval(x[[1]]), x)
# read.csv(file = x, header = "important.csv", row.names = FALSE)
```

You can modify (or add) elements of the call with replacement operators:

```R
x$col.names <- FALSE
x
# read.csv(x, "important.csv", row.names = FALSE, col.names = FALSE)

x[[5]] <- NULL
x[[3]] <- "less-imporant.csv"
x
# read.csv(x, "less-imporant.csv", row.names = FALSE)
```

You can insert other quoted calls:

```R
x <- quote(read.csv(x, "important.csv", row.names = FALSE))
y <- match.call(read.csv, x)
y$file <- quote(paste0(filename, ".csv"))
```

Calls also support the `[` method, but use it with care: it produces a call object, and it's easy to produce invalid calls. If you want to get a list of the arguments, explicitly convert to a list.

```R
x[-3] # remove the second argument

x[-1] # remove the function name - but it's still a call!
x

# A list of the unevaluated arguments
as.list(x[-1])
```

### A cautionary tale: `write.csv`

`write.csv` is a base R function where call manipulation is used inappropriately:

```R
write.csv <- function (...) {
  Call <- match.call(expand.dots = TRUE)
  for (argname in c("append", "col.names", "sep", "dec", "qmethod")) {
    if (!is.null(Call[[argname]])) {
      warning(gettextf("attempt to set '%s' ignored", argname), domain = NA)
    }
  }
  rn <- eval.parent(Call$row.names)
  Call$append <- NULL
  Call$col.names <- if (is.logical(rn) && !rn) TRUE else NA
  Call$sep <- ","
  Call$dec <- "."
  Call$qmethod <- "double"
  Call[[1L]] <- as.name("write.table")
  eval.parent(Call)
}
```

We could write a function that behaves identically using regular function call semantics:

```R
write.csv <- function(x, file = "", sep = ",", qmethod = "double", ...) {
  write.table(x = x, file = file, sep = sep, qmethod = qmethod, ...)
}
```

This makes the function much much easier to understand - it's just calling `write.table` with different defaults.  This also fixes a subtle bug in the original `write.csv` - `write.csv(mtcars, row = FALSE)` raises an error, but `write.csv(mtcars, row.names = FALSE)` does not. Generally, you always want to use the simplest tool that will solve a problem - that makes it more likely that others will understand your code.

## Parsing and deparsing

You can convert quoted calls back and forth between text with `parse()` and `deparse()`.  `parse()` takes a character vector and returns a list of calls (an expression object). `deparse()` takes a quoted call and returns a character vector.

Note that because the primary use of `parse()` is parsing files of code on disk, the first argument is a file path, and if you have the code in a character vector, you need to use the `text` argument.

```R
z <- quote(y <- x * 10)
deparse(z)

parse(text = deparse(z))
```

`deparse()` returns a character vector with an entry for each line, and by default it will try to make lines that are around 60 characters long. If you want a single string be sure to `paste()` it back together. 

`parse()` can't return just a regular quoted call, because there might be many quoted calls in an file. You should never need to create expression objects yourself, and all you need to about them is that they're a list of calls:

```R
exp <- parse(text = c("x <- 4", "y <- x * 10"))
length(exp)
exp[[1]]
is.call(exp[[1]])
call_tree(exp)
```

`parse()` and `deparse()` are not completely symmetric.

With `parse()` and `eval()` you can write your own simple version of `source()`. We read in the file on disk, `parse()` it and then `eval()` each component in the specified environment. This version defaults to a new environment, so it doesn't affect your existing code. `source()` invisibly returns the result of the last expression and `simple_source()` does the same.

```R
simple_source <- function(file, envir = new.env()) {
  stopifnot(file.exists(file))
  stopifnot(is.environment(envir))

  lines <- readLines(file, warn = FALSE)
  exprs <- parse(text = lines, n = -1)

  n <- length(exprs)
  if (n == 0L) return(invisible())

  for (i in seq_len(n - 1)) {
    eval(exprs[i], envir)
  }
  invisible(eval(exprs[n], envir))
}
```

The real `source()` is considerably more complicated because it will also `echo` input and output, and has many additional settings to control behaviour.


## Capturing the current call

You may want to capture the expression that caused the current function to be run.  There are two ways to do this: `sys.call()` and `match.call()`.  `sys.call()` captures exactly what the user typed, where `match.call()` uses R's regular argument matching rules and converts everything to full name matching.  This is usually easier to work with because you know that the call will always have the same structure.

```R
f <- function(abc = 1, def = 2, ghi = 3, ...) {
  list(sys = sys.call(), match = match.call())
}
f(d = 2, 2)
```

If you provide `call` and `definition` arguments to `match.call()` you can use it as a general tool for standardising function calls:

```R
call <- quote(mean(n = 5, x = 1:10))
match.call(call = call, def = mean)
match.call(call = call, def = mean.default)
```

This will be an important tool when we start manipulating existing function calls. If we don't use `match.call` we'll need a lot of extra code to deal with all the possible ways to call a function.

We can wrap this up into a function. To figure out the definition of the associated function we evaluate the first component of the call, the name of the function.  We need to specify an environment here, because the function might be different in different places. Whenever we provide an environment parameter, `parent.frame()` is usually a good default. Note the check for primitive functions: they don't have `formals()` and handle argument matching specially, so there's nothing we can do.

```R
standardise_call <- function(call, env = parent.frame()) {
  stopifnot(is.call(call))
  f <- eval(call[[1]], env)
  if (is.primitive(f)) return(call)

  match.call(f, call)
}
standardise_call(call)

standardise_call(quote(f(d = 2, 2)))
```

### `quote()` and `eval()`

`quote()` and `eval()` are basically opposites. In the example below, each `eval()` peels off one layer of quoting.

```R
quote(2 + 2)
eval(quote(2 + 2))

quote(quote(2 + 2))
eval(quote(quote(2 + 2)))
eval(eval(quote(quote(2 + 2))))
```

What will this code return?

```R
eval(quote(eval(quote(eval(quote(2 + 2))))))
```

### Flexible quoting and unquoting

`bquote()` is a slightly more general form of quote: it allows you to optionally quote and unquote some parts of an expression (it's similar to the backtick operator in lisp).  Everything is quoted, _unless_ it's encapsulated in `.()` in which case it's evaluated and the result is inserted.

```R
a <- 1
b <- 3
bquote(a + b)
bquote(a + .(b))
bquote(.(a) + .(b))
bquote(.(a + b))
```

This provides a fairly easy way to control what gets evaluated when you call `bquote()`, and what gets evaluated when the expression is evaluated.

### Getting the name of an argument


```R
fname <- function(call) {
  f <- eval(call, parent.frame())
  if (is.character(f)) {
    fname <- f
    f <- match.fun(f)
  } else if (is.function(f)) {
    fname <- if (is.name(call)) as.character(call) else "<anonymous>"
  }
  list(f, fname)
}
f <- function(f) {
  fname(substitute(f))
}
f("mean")
f(mean)
f(function(x) mean(x))
```


## Find assignment

The key to any function that works with the parse tree right is getting the recursion right, which means making sure that you know what the base case is (the leaves of the tree) and figuring out how to combine the results from the recursive case. 

The nodes of the tree can be any recursive data structure that can appear in a quoted object: pairlists, calls, and expressions. The leaves can be 0-argument calls (e.g. `f()`), atomic vectors, names or `NULL`. The easiest way to tell the difference is the `is.recursive` function.

In this section, we will write a function that figures out all variables that are created by assignment in an expression. We'll start simply, and make the function progressively more rigorous. One reason to start with this function is because the recursion is a little bit simpler - we never need to go all the way down to the leaves because we are looking for assignment, a call to `<-`.  

This means that our base case is simple: if we're at a leaf, we've gone too far and can immediately return. We have two other cases: we have hit a call, in which case we should check if it's `<-`, otherwise it's some other recursive structure and we should call the function recursively on each element. Note the use of identical to compare the call to the name of the assignment function, and recall that the second element of a call object is the first argument, which for `<-` is the left hand side: the object being assigned to.

```R
is_call_to <- function(x, name) {
  is.call(x) && identical(x[[1]], as.name(name))
}

find_assign <- function(obj) {
  # Base case
  if (!is.recursive(obj)) return(character())

  if (is_call_to(obj, "<-")) {
    obj[[2]]
  } else {
    lapply(obj, find_assign)
  }
}
find_assign(quote(a <- 1))
find_assign(quote({
  a <- 1
  b <- 2
}))
```

This function seems to work for these simple cases, but it's a bit verbose.  Instead of returning a list, let's keep it simple and stick with a character vector. We'll also test it with two slightly more complicated examples:

```R
find_assign <- function(obj) {
  # Base case
  if (!is.recursive(obj)) return(character())

  if (is_call_to(obj, "<-")) {
    as.character(obj[[2]])
  } else {
    unlist(lapply(obj, find_assign))
  }
}
find_assign(quote({
  a <- 1
  b <- 2
  a <- 3
}))
# [1] "a" "b" "a"
find_assign(quote({
  system.time(x <- print(y <- 5))
}))
# [1] "x"
```

This is better, but we have two problems: repeated names, and we miss assignments inside function calls. The fix for the first problem is easy: we need to wrap `unique` around the recursive case to remove duplicate assignments. The second problem is a bit more subtle: it's possible to do assignment within the arguments to a call, but we're failing to recurse down in to this case. 

```R
find_assign <- function(obj) {
  # Base case
  if (!is.recursive(obj)) return(character())

  if (is_call_to(obj, "<-")) {
    call <- as.character(obj[[2]])
    c(call, unlist(lapply(obj[[3]], find_assign)))
  } else {
    unique(unlist(lapply(obj, find_assign)))
  }
}
find_assign(quote({
  a <- 1
  b <- 2
  a <- 3
}))
# [1] "a" "b"
find_assign(quote({
  system.time(x <- print(y <- 5))
}))
# [1] "x" "y"
```

There's one more case we need to test:

```R
find_assign(quote({
  ls <- list()
  ls$a <- 5
  names(ls) <- "b"
}))
# [1] "ls"        "ls$a"      "names(ls)"
draw_tree(quote({
  ls <- list()
  ls$a <- 5
  names(ls) <- "b"
}))
# \- {()
#    \- <-()
#       \- ls
#       \- list()
#    \- <-()
#       \- $()
#          \- ls
#          \- a
#       \- 5
#    \- <-()
#       \- names()
#          \- ls
#       \- "b"
```

This behaviour might be ok, but we probably just want assignment into whole objects, not assignment that modifies some property of the object. Drawing the tree for that quoted object helps us see what condition we should test for - we want the object on the left hand side of assignment to be a name.  This gives the final version of the `find_assign` function.

```R
find_assign <- function(obj) {
  # Base case
  if (!is.recursive(obj)) return(character())

  if (is_call_to(obj, "<-")) {
    call <- if (is.name(obj[[2]])) as.character(obj[[2]])
    c(call, unlist(lapply(obj[[3]], find_assign)))
  } else {
    unique(unlist(lapply(obj, find_assign)))
  }
}
find_assign(quote({
  ls <- list()
  ls$a <- 5
  names(ls) <- "b"
}))
[1] "ls"
```

Making this function work absolutely correct requires quite a lot more work, because we need to figure out all the other ways that assignment might happen: with `=`, `assign`, or `delayedAssign`.

## Modifying the call tree

A more useful function might solve a common problem encountered when running `R CMD check`: you've used `T` and `F` instead of `TRUE` and `FALSE`.  Again, we'll pursue the same strategy of starting simple and then gradually building up our function to deal with more cases.

First we'll start just by locating logical abbreviations. A logical abbreviation is the name `T` or `F`. We need to recursively breakup any recursive objects, anything else is not a short cut.

    find_logical_abbr <- function(obj) {
      if (!is.recursive(obj)) {
        identical(obj, as.name("T")) || identical(obj, as.name("F"))
      } else {
        any(vapply(obj, find_logical_abbr, logical(1)))
      }
    }

    
    f <- function(x = T) 1
    g <- function(x = 1) 2
    h <- function(x = 3) T
    
    find_logical_abbr(body(f))
    find_logical_abbr(formals(f))
    find_logical_abbr(body(g))
    find_logical_abbr(formals(g))
    find_logical_abbr(body(h))
    find_logical_abbr(formals(h))
  
To replace `T` with `TRUE` and `F` with `FALSE`, we need to make our recursive function return either the original object or the modified object, making sure to put calls back together appropriately. The main difference here is that we need to be much more careful with recursive objects, because each type needs to be treated differently:

* pairlists, which are used for function formals, can be processed as lists,
  but need to be turned back into pairlists with `as.pairlist`

* expressions, again can be processed like lists, but need to be turned back
  into expressions objects with `as.expression`. You won't find these inside a
  quoted object, but this might be the quoted object that you're passing in.

* calls require the most special treatment: don't process first element
  (that's the name of the function), and turn back into a call with `as.call`

We'll wrap this behaviour up into a function

```R
capply <- function(X, FUN, ...) {
  if (is.call(X)) {
    args <- lapply(X[-1], FUN, ...)
    as.call(c(list(X[[1]]), args))
  } else {
    out <- lapply(X, FUN, ...)

    if (is.function(X)) {
      out <- as.function(out, environment(X))
    } else {
      mode(out) <- mode(X)
    }
    out
  }
}
capply(expression(1, 2, 3), identity)
capply(quote(f(1, 2, 3)), identity)
capply(formals(data.frame), identity)
capply(quote(function(x) x + 1), identity)
```

This leads to:

    fix_logical_abbr <- function(obj) {
      # Base case
      if (!is.recursive(obj)) {
        if (identical(obj, as.name("T"))) return(TRUE)
        if (identical(obj, as.name("F"))) return(FALSE)
        return(obj)
      }

      capply(obj, fix_logical_abbr)
    }

However, this function is still not that useful because it loses all non-code structure:

    g <- parse(text = "f <- function(x = T) {
      # This is a comment
      if (x)                  return(4)
      if (emergency_status()) return(T)
    }")
    f <- function(x = T) {
      # This is a comment
      if (x)                  return(4)
      if (emergency_status()) return(T)
    }
    fix_logical_abbr(g)
    fix_logical_abbr(f)

It is possible to work around this problem using `srcrefs` and `getParseData`, but neither solution naturally fits this hierarchical framework. You effectively end up having to recreate huge chunks of R's internal code in order to handle the majority of R code.  So the above approach can be useful in simple cases (particularly when you don't care what the output code looks like), but it's very hard to automatically transform R code, and is hence beyond the scope of this book.


## Source references and `getParseData()`

Srcrefs are a fourth property of functions. R also stores a pointer to the original location of the function (this is why you can see the comments and when you get errors you see the line number).  You can read more about srcrefs in `?srcref` and in the [R journal][srcrefs].

We're going to focus on `getParseData` (a new feature in R 3.0) which gives us the parsed data in a flat data frame format, and includes all of the original comments.


  [srcrefs]: http://journal.r-project.org/archive/2010-2/RJournal_2010-2_Murdoch.pdf


<!-- Idea:
Destructuring assignment
  https://gist.github.com/4543212
  https://github.com/crowding/ptools/blob/master/R/bind.R -->


## Creating a function

Building up a function by hand is also useful when you can't use a closure because you don't know in advance what the arguments will be. We'll use `pryr::make_function` to build up a function from its components pieces: an argument list, a quoted body (the code to run) and the environment in which it is defined (which defaults to the current environment):

```R
add <- make_function(alist(a = 1, b = 2), quote(a + b))
```

Note that to use this function we need to use the special `alist` function to create an **a**rgument list.

```R
add2 <- make_function(alist(a = 1, b = a), quote(a + b))
add(1)
add(1, 2)

add3 <- make_function(alist(a = , b = ), quote(a + b))
```

We could use `make_function` to create an `unenclose` function that takes a closure and modifies it so when you look at the source you can see what's going on: 

```R
unenclose <- function(f) {
  stopifnot(is.function(f))
  env <- environment(f)

  make_function(formals(f), substitute2(body(f), env), parent.env(env))
}

f <- function(x) {
  function(y) { 
    x + y
  }
}
f(1)
unenclose(f(1))
```

(Note we need to use `substitute2` here, because `substitute` uses non-standard evaluation).

(Exercise: modify this function so it only substitutes in atomic vectors, not more complicated objects.)

Look at the source code for partial.


## Formulas

There is one other approach we could use: a formula. `~` works much like quote, but it also captures the environment in which it is created. We need to extract the second component of the formula because the first component is `~`.

```R
subset <- function(x, f) {
  r <- eval(f'[[2]], x, environment(f))
  x[r, ]
}
subset(mtcars, ~ cyl == x)
```

### Exercises

* Read the source code for `pryr::partial` and refer to XXX for uses. How does it work?