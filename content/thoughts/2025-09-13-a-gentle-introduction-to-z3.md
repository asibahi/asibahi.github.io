+++
title = "A Gentle Introduction to z3"
description = "Exploring the world of constraint solvers with very simple examples."
date = 2025-09-13
draft = true
+++

Recently I have come across a nice article: [Many Hard Leetcode Problems are Easy Constraint Problems](https://buttondown.com/hillelwayne/archive/many-hard-leetcode-problems-are-easy-constraint/), and I figured, I really should learn how to use these things! What else do I really have to do? I have had use for solvers (or as they are commonly called: theorem provers) [In a previous article](@/thoughts/2024-10-17-the-hanging-gardens-problem/index.md), but then I tried to prove the things with good old algorithms. I looked at `z3` at the time, but found the whole concept a bit too opaque. Now however, it seemed a bit easier to get into.

To be clear, as of writing these words, I have only been looking at `z3` reading mateiral for two days. I am in no way an expert, and I have not written anything more complex than a solver for the change counter problem (the first example in the article listed above). So I am writing this really knowing nothing about the underlying theory, theorem provers, or whatever the hell "unificaion" is. There is a good chance you know more about this than I do.

There are `z3` bindings in many popular languages. I will be using [`z3`'s Rust bindings](https://docs.rs/z3/latest/z3/), because I am more comfortable in Rust than, say, Python or JavaScript. The examples I worked with to understand `z3` however, can be found in two nice documents:

1. [First is in Python](https://z3prover.github.io/papers/programmingz3.html)
2. [Second in JavaScript](https://microsoft.github.io/z3guide/programming/Z3%20JavaScript%20Examples/)

## What are Solvers?

Solvers are a class of .. tools? libraries? where you input some rules and constraints and have the tool just .. solve it for you. It is not going to be a faster or more optimized solution than a custom made algorithm, but it is much easier to change the rules on the fly.

There are many real world uses. They are often used for scheduling and resource allocation problems. Consider the common scenario of a school schedule: Mary cannot work on Tuesdays because she needs to take care of her father; John lives far so he cannot give classes before 10; Class 3-A is full of nerds so their math hours are double; city council regulates no outdoor activity after 12; Susan and Sarah hate each other so you should not have them teach the same class; etc. You can either have two teachers work on it for a week, or just pop it in a solver!

The [MiniZinc homepage](https://www.minizinc.org) (another popular solver) has a couple of nice examples: a seating chart, rostering, vehicle routing, grid coloring.

On that note, you might wonder: why did I go with `z3` when MiniZinc has a more colorful homepage and is actually referenced by the article linked at the start of this article? The answer is because `z3` has bindings in Rust. That is pretty much it.

## A Note on Terminology

Documentation on `z3` and its API use a lot of jargon,[^jargon] which makes the whole thing really difficult to wade into without a previous background. I will explain things as I understand when I get to them, but two things really stand out.

[^jargon]: Domain specific termonology.

The first is the word `Sort`. You see this in the context of arrays and function declarations (we will get to those, I hope). But it has nothing to do with .. well .. sorting. `Sort` is just the jargon word for _types_.

The second one is **constands**. They are not what a normal person would call constants: they are actually the knobs the solvers use to solve problems. There are two types of contants: _free_, which are what one would call variables; and _interpreted_, which is when you'd type an integer literal and clever type machinations turns it into a coonstant in the solver.

Note that also solvers do not work within the regular type system of the programming anguage. They have their own types (sorry, sorts), and operations that may or may not map nicely to the language's types and operations. Much of the actual code you are writing is about expressing things in the target solver's language. `z3` uses a language called "SMT-LIB2" (henceforth called `smt2`), apparently. And you can actually write your constraints immediately in said language and have the library consume it. Much of what the bindings is take your code and translate internally to this language before feeding it to the solver.[^dsl]

[^dsl]: Probably not true. For all I know the C API constructs the model directly without the intermediate step. However, I found it much easier to understand them in this manner.

---

## A Simple Equation

Let's start with what might be the simplest, dumbest equation. Solve for `x`:

```
x + 4 = 7
```

Yes, a child (literally) can solve this. But it is nice. Here is the Rust program to solve it.

```rs
use z3::{Solver, ast::Int};
fn main() {
	let solver = Solver::new();

	// define the variable.
	let x = Int::new_const("x");

	// define the equation
	solver.assert((&x + 4).eq(7));

	// run the solver
	_ = solver.check();
	let model = solver.get_model().unwrap();

	println!("{model:?}");
}
```

This prints out the solution. `x` equals three. Who would have guessed?

```smt2
x -> 3
```

The Rust bindings have some nice ergonomics here. You can simply do `&x + 4` and it would do all the bookkeeping behind closed doors to transform the `4` (and the `7`) into an interpreted constant and have them inserted into the internal model.

The reason you have to pass in a string in `new_const` is that this is the name given to the variable in `smt2`. It does not have to be `"x"`, it can be anything. Why do the bindings not autogenerate the name for you? Who knows.

If you print the solver (as in `println!("{solver:?}");`), you will get the following output in the `smt2`.

```smt2
(declare-fun x () Int)
(assert (= (+ x 4) 7))
```

Note that the variable you declared is declared as a _function_. A free constant is basically a function that takes no input and gives an output (here of ~~type~~ sort `Int`). The solver finds which version of the function satisfies the assertions. This also explains the arrow in `x -> 3` earlier. `x` _evaluates to_ 3.

---

## A Jump in Complexity

In school, jumping from solving equations with a single variable to equations with two variables was a real jump on complexity. Everything was doubled! Here is a pair of equations we will try to solve next:

```
x + y = 17
y = 2 * x
```

Here is the program. I am going to print the result of `solver.check()` first, tho. I just made up those numbers!

```rs
use z3::{Solver, ast::Int};

fn main() {
	let solver = Solver::new();

	// define the variable.
	let x = Int::new_const("x");
	let y = Int::new_const("y");

	// define the equation
	solver.assert((&x + &y).eq(17));
	solver.assert((&x * 2).eq(&y));

	println!("{solver:?}");

	let c = solver.check();
	println!("; {c:?}");
}
```

This prints out the following:

```smt2
(declare-fun y () Int)
(declare-fun x () Int)
(assert (= (+ x y) 17))
(assert (= (* x 2) y))

; Unsat
```

Oh it is `Unsat`. Unsatisfiable. Bummer. This means this cannot be solved as defined.

Let's try changing the type to `Real`. The `Real` type does not have the same nice ergonomics as `Int` apparently, so the code will look slightly uglier. This is the new updated code.

```rs
use z3::{Solver, ast::Real};

fn main() {
	let solver = Solver::new();

	// define the variable.
	let x = Real::new_const("x");
	let y = Real::new_const("y");

	let seventeen = Real::from_rational(17, 1);
	let two = Real::from_rational(2, 1);

	// define the equation
	solver.assert((&x + &y).eq(&seventeen));
	solver.assert((&x * &two).eq(&y));

	println!("{solver:?}");

	let c = solver.check();
	println!("; {c:?}");
}
```

Which prints

```smt2
(declare-fun y () Real)
(declare-fun x () Real)
(assert (= (+ x y) 17.0))
(assert (= (* x 2.0) y))

; Sat
```

Excellent! Using `get_model()` and printing the model as before gives us the following answer, presented as a nice rational number.

```smt2
y -> (/ 34.0 3.0)
x -> (/ 17.0 3.0)
```

To actually extract the values programmatically, instead of debug printing `model`, requires some song and dance with the API, but it is simple, really. This is what it would look like.

```rust
// to avoid panicking on unsatisfiable models
if let z3::SatResult::Sat = solver.check() {
	let model = solver.get_model().unwrap();

	// do not ask me what the `true` is for. I don't know.
	let x = model.eval(&x, true).unwrap().approx_f64();
	let y = model.eval(&y, true).unwrap().approx_f64();

	println!("x: {x:.3}\ty: {y:.3}");
}
```

Which prints

```
x: 5.667	y: 11.333
```

Nice. Isn't this grand?

---
