# Parallelism for Free? Understanding interaction combinators and the Bend programming language

TODO
- [] Split
- [] Acknowledgements/sources
- [] Rewrite the bend portions after understanding ICs
- [] Lessons for the future?

Bend is a programming language, developed by HigherOrderCo (HOC), that 
claims that any code you write that _can_ be parallelized,
_will_ be parallelized, even if you haven't written it 
specifically with a parallel execution in mind. How does 
it do this?

The main idea is this: Bend runs on the Higher-Order Virtual Machine
(HVM), also developed by HOC, which uses a bytecode that implements
something called the "Interaction Calculus." Interaction calculus is a 
Turing-complete computation system (just like Lambda calculus) with
a property called _strong confluence_ which implies that reductions
can be done in parallel in a consistent manner. Because Bend 
compiles down to this bytecode, the HVM can then perform reductions
on the bytecode in parallel, which gives Bend its (theoretical) 
speedup.

So how does this all work?

## Interaction nets and Interaction combinators
Interaction nets are a system of 

## Interaction Calculus: Representing this textually

## Bend syntax and features
Now that we understand how the underlying system works, it's time to 
understand how Bend translates programs to this interaction calculus
layer. Before we can do that, we need to be able to read programs written
in Bend. Thankfully, Bend syntax isn't that hard to understand. It emulates 
python for the most part, though it _is_ a purely functional language with
no mutation (there's also _another_ LISP-like syntax with worse documentation. I 
decided to stick with the pythonic syntax because I can only cover one of them
and the python syntax is better documented).
It also has _affine types_, which means there is also some
syntax for move semantics (the "open" keyword) and discarding
(which bend calls "erasing")

Most of the syntax is very basic: function definitions, if-else blocks,
struct declarations (here are only three datatypes (u24, i24, f24) and some ways
of combining those into larger structures), lambdas, and the "fork" and "bend" keywords
which need a little bit of explaining. Many of the examples below are from the Bend
project repository.

A program enters at main. There is no IO, but the 
return value of main is printed to stdout at the end. Function definitions
and lambdas works as expected.

```py
def main:
  sum = add(2, 3)
  return sum

# These two are equivalent
def add(x, y):
  return x + y

def add2:
  return lambda x, y: x + y

```

Data types (which are algebraic) can be made:

```py
# With a tuple
def tuple_fst(x):
  # This destructures the tuple into the two values it holds.
  # '*' means that the value is discarded and not bound to any variable.
  (fst, *) = x
  return fst

# With an object (similar to what other languages call a struct, a class or a record)
object Pair { fst, snd }

def Pair/fst(x):
  match x:
    case Pair:
      return x.fst

# We can also access the fields of an object after we `open` it.
def Pair/fst_2(x):
  open Pair: x
  return x.fst

# This is how we can create new objects.
def Pair/with_one(x):
  return Pair{ fst: x, snd: 1 }

# The function can be named anything, but by convention we use Type/function_name.
def Pair/swap(x):
  open Pair: x
  # We can also call the constructor like any normal function.
  return Pair(x.snd, x.fst)

type MyTree:
  Node { val, ~left, ~right }
  Leaf


```

In the above example, "open" is used to bring the fields of an 
object into scope and is a move semantic to maintain the "affine-ness" of
a type. The asterisk "*" is called an eraser and discards the type. The
"~" in the definition of trees indicates a recursive data type.

Pattern matching works exactly as expected:

```py
def Maybe/or_default(x, default):
  match x:
    case Maybe/Some:
      # We can access the fields of the variant using 'matched.field'
      return x.val
    case Maybe/None:
      return default
```

There are literals for lists, maps, and trees (the exclamation
marks in the list-like structure make it a tree. It needs to be 
put before the integer literals too, to wrap them in a Tree/Leaf)

```py 
[1,2,3,4]
{'a':'z', 'b':'y'}
![![!1, !2], !3]
```

There are two related keywords: fold and bend. Fold consumes a 
recursive datatype, recursively replacing its self-similar
fields with a computation, and bend does the opposite, _creating_
a recursive datatype by replacing steps in the creation recursively.
Ultimately they are both just sugar to avoid writing boilerplate
recursive functions to create and consume recursive datatypes.

```py
def MyTree.sum(x):
  # Sum all the values in the tree.
  fold x:
    # The fold is implicitly called for fields marked with '~' in their definition.
    case MyTree/Node:
      return x.val + x.left + x.right
    case MyTree/Leaf:
      return 0

def main:
  bend val = 0:
    when val < 10:
      # 'fork' calls the bend recursively with the provided values.
      x = MyTree/Node { val:val, left:fork(val + 1), right:fork(val + 1) }
    else:
      # 'else' is the base case, when the condition fails.
      x = MyTree/Leaf

  return MyTree.sum(x)
```

You can also do accumulators with fold (making it basically
what a fold would mean in most functional languages)

```py
# This function substitutes each value in the tree with the sum of all the values before it.
def MyTree.map_sum(x):
  acc = 0
  fold x with acc:
    case MyTree/Node:
      # `x.left` and `x.right` are called with the new state value.
      # Note that values are copied if you use them more than once, so you don't want to pass something very large.
      return MyTree/Node{ val: x.val + acc, left: x.left(x.val + acc), right: x.right(x.val + acc) }
    case MyTree/Leaf:
      return x
```

There are some other subtleties, like duplications and 
superpositions, or scopeless lambdas, that I will get to 
in the next section. They are just the hvm implementation
leaking out into the language, but in some cases this 
leaking is really interesting.


## Bend under the hood 

- [] How are some basic programs compiled
- [] Compilation of, say, fork and bend
- [] How are lambdas compiled? How are scopeless lambdas compiled?
- [] Open and Erasers
- [] Dups and sups

## Wrapping up: Does Bend do what it promises?

Bend is still deeply experimental. Its current implementation
is not very well optimized, execution can be different (and buggy!)
based on which interpreter (Rust, C, or CUDA) you use, and it doesn't 
yet support IO! Still, Bend is the only implementation of the interaction
combinator idea I could find, and in some hand-picked examples it does have 
immense speedup over (say) GHC.

- [] Include benchmarks