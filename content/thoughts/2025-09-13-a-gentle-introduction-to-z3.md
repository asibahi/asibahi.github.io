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

Let's try changing the type to `Real`.[^float] The `Real` type does not have the same nice ergonomics as `Int` apparently, so the code will look slightly uglier. This is the new updated code.

[^float]: Can also use `Float`.

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

## Multiple Solutions

As I am sure you know from your high school math, some equations have multiple solutions. Here is a simple one.

```
x * x = 4
```

The Rust bindings have a nice method for getting multiple solutions out of a solver, simple called `solutions`. It works similarly to `model.eval()` above, and takes the same paramaters with the same output. Here is the complete program. (I am going back to `Int` because I am not cool enough for `Real` numbers.)

```rust
use z3::{Solver, ast::Int};
fn main() {
	let solver = Solver::new();
	let x = Int::new_const("x");

	solver.assert((&x * &x).eq(4));
	println!("{solver:?}");

	// This terminates when `check` does not return `Sat`
	for (idx, s) in solver.solutions(x, true).enumerate() {
		let s = s.as_i64().unwrap();
		println!(";{}:\t{s}", idx + 1);
	}
}
```

Which prints:

```smt2
(declare-fun x () Int)
(assert (= (* x x) 4))

;1:	-2
;2:	2
```

I am not clear really on how to get multiple solutions with the regular `check` followed by `get_model` method, but this one is easy enough to use. Also, some problems might have infinitely many solutions, so it is advisable to use `take` with the `solutions` iterator. To demonstrate, I will use the circle equation.

```
x * x + y * y = 25
```

Here is the straightforward script followed by the printed out result. Note that the `fresh_const` API creates a unique name for every invocation.

```rust
use z3::{Solver, ast::Int};
fn main() {
	let solver = Solver::new();

	let x = Int::fresh_const("x");
	let y = Int::fresh_const("x");

	let area = &x * &x + &y * &y;
	solver.assert(area.eq(25));

	println!("{solver:?}");

	for (idx, (x,y)) in solver.solutions((x,y), true).enumerate() {
		let x = x.as_i64().unwrap();
		let y = y.as_i64().unwrap();
		println!("; {}:\t{x:>2},{y:?2} ", idx + 1);
	}
}
```

```smt2
(declare-fun x!1 () Int)
(declare-fun x!0 () Int)
(assert (= (+ (* x!0 x!0) (* x!1 x!1)) 25))

; 1:	 0, 5
; 2:	-5, 0
; 3:	-3,-4
; 4:	-4,-3
; 5:	 0,-5
; 6:	 3,-4
; 7:	 4,-3
; 8:	 5, 0
; 9:	 3, 4
; 10:	 4, 3
; 11:	-3, 4
; 12:	-4, 3
```

This goes without saying, but if I used the `Real` type in this example it would generate infinite solutions.

---

## Coin Change Problem

The Coin Change problem is a simple one: given a list of denominations and a total, find the smallest number of coins that add up to said total. Emphasis on _smallest_. Unlike previous problems, this is an optimizatin problem. We are looking for a solution that satisfies specific criteria instead of just _a_ solution. Conventiently enough, `z3` provides an `Optimize` object which we can use to optimize.

Let us set up the parameters of the problem in plain language. The denominations we have are 1, 5, and 10. We need to give 37 money in the least amount of coins. This is simple enough that we can know the solution is three 10 coins, one 5 coin, and two 1 coins. Let's see if we can get the same result. As usual, code followed by output:

```rust
use z3::{Optimize, ast::Int};
fn main() {
	let opt = Optimize::new();

	let c1 = Int::new_const("c1");
	let c5 = Int::new_const("c5");
	let c10 = Int::new_const("c10");

	let total = (&c1 * 1) + (&c5 * 5) + (&c10 * 10);
	let count = &c1 + &c5 + &c10;

	opt.assert(&total.eq(37));
	opt.minimize(&count);

	println!("{opt:?}");

	if let z3::SatResult::Sat = opt.check(&[]) {
		let model = opt.get_model().unwrap();
		let c1 = model.eval(&c1, true).unwrap();
		let c5 = model.eval(&c5, true).unwrap();
		let c10 = model.eval(&c10, true).unwrap();

		println!("; c1: {c1:?}, c5: {c5:?}, c10: {c10:?}");
	} else {
		println!("; woe for us");
	}
}
```

```smt2
(declare-fun c10 () Int)
(declare-fun c5 () Int)
(declare-fun c1 () Int)
(assert (= (+ (* c1 1) (* c5 5) (* c10 10)) 37))
(minimize (+ c1 c5 c10))
(check-sat)

; c1: 37, c5: 0, c10: 0
```

Oops. This cannot be right.

I do not really understand why the answer is so nonsensical here. The problem is that `Int` really spans the entire natural integers range, so it is accounting for negative amounts of coins. This still does not explain how the optimal solution given is 37 coins. (If you can explain, please let me know.)

The solution for this[^mystery] is to constrain the amount of coins to be non-negative. So let's do that. Add these assertions somewhere before `check`, and Bob's your uncle.

[^mystery]: Again, for reasons I do not understand.

```rust
opt.assert(&c1.ge(0));
opt.assert(&c5.ge(0));
opt.assert(&c10.ge(0));
```

```smt2
(declare-fun c1 () Int)
(declare-fun c5 () Int)
(declare-fun c10 () Int)
(assert (>= c1 0))
(assert (>= c5 0))
(assert (>= c10 0))
(assert (= (+ (* c1 1) (* c5 5) (* c10 10)) 37))
(minimize (+ c1 c5 c10))
(check-sat)

; c1: 2, c5: 1, c10: 3
```

That's more like it.  Now let's try with different denominations. Something like 10. 9, and 1. Note that the optimal solution for 37 would be: one 10 coin, three 9 coins, and no 1 coins. The greedy solution would fail to catch that. Here is the output of printing the optimizer and the result.

```smt2
(declare-fun c1 () Int)
(declare-fun c9 () Int)
(declare-fun c10 () Int)
(assert (>= c1 0))
(assert (>= c9 0))
(assert (>= c10 0))
(assert (= (+ (* c1 1) (* c9 9) (* c10 10)) 37))
(minimize (+ c1 c9 c10))
(check-sat)

; c1: 0, c9: 3, c10: 1
```

Success!

## `push` and `pop`

Currently, the total 37 is hardcoded. But what if I want the answers for a number of different totals? Thankfully, you do not need to build the optimizer from scratch for every total. Instead, use the magical functions `push` and `pop`. The first one essentially creates a book mark in the stack of assertions. The second removes everything above said bookmark, and the bookmark. It is simple really. Here are the solutions from 30 to 39, because why not.

Here is the full `main`. I will spare you the output.

```rust
let opt = Optimize::new();

let c1 = Int::new_const("c1");
let c9 = Int::new_const("c9");
let c10 = Int::new_const("c10");

opt.assert(&c1.ge(0));
opt.assert(&c9.ge(0));
opt.assert(&c10.ge(0));

let total = (&c1 * 1) + (&c9 * 9) + (&c10 * 10);
let count = &c1 + &c9 + &c10;

opt.minimize(&count);

println!("; total\tcount\tc1\tc9\tc10");

for t in (30u32..).take(10) {
	print!("; {t}\t");

	opt.push();

	opt.assert(&total.eq(t));

	if let SatResult::Sat = opt.check(&[]) {
		let model = opt.get_model().unwrap();
		let c1 = model.eval(&c1, true).unwrap().as_u64().unwrap();
		let c9 = model.eval(&c9, true).unwrap().as_u64().unwrap();
		let c10 = model.eval(&c10, true).unwrap().as_u64().unwrap();

		let count = c1 + c9 + c10;

		println!("{count}\t{c1:?}\t{c9:?}\t{c10:?}");
	} else {
		println!("; woe for us");
	}

	opt.pop(); // no RAII for you
}
```

Note that the `push` and `pop` API is available for `Solver` as well. At any rate, back to solving.

---

## Sudoku

This is a significant jump in complexity, so bear with me. We are going to solve a Sudoku.[^js] So let's write the constraints first. We can use Rust's arrays or `Vec` to organize our `z3.Int`s and check their constraints. First, this is the puzzle we are solving:

[^js]: This example is translated from [The `z3` Guide](https://microsoft.github.io/z3guide/programming/Z3%20JavaScript%20Examples#solve-sudoku)

```
....94.3.
...51...7
.89....4.
......2.8
.6.2.1.5.
1.2......
.7....52.
9...65...
.4.97....
```

I will forgo the steps to turn that into a `[[Option<u8>;9];9]`. Instead the code below will get that info from a `get_puzzle()` function.

```rust
let solver = Solver::new();

// Note that we're using Rust arrays here. The solver does not really know about them.
let grid: [[_; 9]; 9] =
	array::from_fn(|i| array::from_fn(|j| Int::new_const(format!("x{i}{j}"))));

// each cell contains a value 1<=x<=9
grid.iter().flatten().for_each(|i| {
	solver.assert(i.ge(1) & i.le(9));
});

// each row contains a digit only once
for row in &grid {
	solver.assert(Ast::distinct(row));
}

// each column contains a digit only once
for idx in 0..9 {
	let mut col = Vec::with_capacity(9);
	grid.iter().for_each(|r| col.push(r[idx].clone()));
	solver.assert(Ast::distinct(&col))
}

// each 3x3 contains a digit at most once
for x_s in (0..9).step_by(3) {
	for y_s in (0..9).step_by(3) {
		let mut square = Vec::with_capacity(9);
		for x in (x_s..).take(3) {
			for y in (y_s..).take(3) {
				// very nested loop
				square.push(grid[x][y].clone());
			}
		}

		solver.assert(Ast::distinct(&square))
	}
}

// Finally, assert that each cell equals a provided clue in the given puzzle
get_puzzle().iter().flatten().zip(grid.iter().flatten()).for_each(|(clue, variable)| {
	if let Some(clue) = clue {
		solver.assert(variable.eq(*clue));
	}
});

eprintln!("{solver:?}");
```

Printing the solver after each step lets you debug whether you have your constraints correctly. The printout is over 200 lines long, so let's skip that. All we have to do next is to check the value of each cell in `grid`.

```rust
if let SatResult::Sat = solver.check() {
	let model = solver.get_model().unwrap();
	for row in grid {
		for int in row {
			let result = model.eval(&int, true).unwrap();
			print!("{result:?}");
		}
		println!();
	}
} else {
	println!("Unsolvable")
}
```

And this prints out the result. You can verify for yourself whether this is correct or not. Maybe try other puzzles. Or add more constraints. [You can even try the Miracle Sudoku](https://www.youtube.com/watch?v=yKf9aUIxdb4).

```
715894632
234516897
689723145
493657218
867231954
152489763
376148529
928365471
541972386
```

One thing of note here: which is how *dumb* the solver is. Note that if you print out the solver, there is no notion of rows and columns and squares. It does not know any Sudoku tricks like X-wings and what have you. All the data is organized on the Rust side of things, and what is given to the solver is "these two variables cannot be the same" over and over and over again. And it just .. tells you what the rest of them are.

Another thing that is not obvious at first glance, is that it does not check if there is a unique solution. The puzzle may be badly constructed and have multiple solutions, and it will happily give you one, or two, or how many you ask for. It does, howver, check if it is unsolvable!

---
