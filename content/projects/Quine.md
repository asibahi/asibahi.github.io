+++
title = "Quine"
weight = 100 # completely unimportant
+++

This is a [quine](https://en.wikipedia.org/wiki/Quine_(computing)) in Rust:

```rust
fn main() {
    println!("{Q}\nconst Q: &str = r{0}\"{Q}\"{0};", '#');
}
const Q: &str = r#"fn main() {
    println!("{Q}\nconst Q: &str = r{0}\"{Q}\"{0};", '#');
}"#;
```

Not the [shortest implementation](https://codegolf.stackexchange.com/questions/69/golf-you-a-quine-for-great-good/97833#97833) but it is mine. Passes `rustfmt` and `clippy` too!
