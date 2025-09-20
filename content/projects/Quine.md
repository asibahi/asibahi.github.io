+++
title = "Quine"
weight = 100 # completely unimportant
+++

There are a couple of [quines](https://en.wikipedia.org/wiki/Quine_(computing)) I wrote. Not the shortest or most code golfed implementations, but they are mine. They generally follow the same idea anyway.

## Rust

Passes `rustfmt` and `clippy::pedantic`.

```rust
fn main() {
    println!("{Q}\nconst Q: &str = r{0}\"{Q}\"{0};", '#');
}
const Q: &str = r#"fn main() {
    println!("{Q}\nconst Q: &str = r{0}\"{Q}\"{0};", '#');
}"#;
```

[The shortest implementation is fairly cool.](https://codegolf.stackexchange.com/questions/69/golf-you-a-quine-for-great-good/97833#97833)

## Zig

This one is fairly longwinded due to Zig's otherwise very nice multiline string literals syntax. Passes `zig fmt`. This works for 0.15.1.

```zig
pub fn main() !void {
    try o.interface.print("{s}\nconst Q =\n", .{Q});
    var it = @import("std").mem.splitScalar(u8, Q, '\n');
    while (it.next()) |l| try o.interface.print("    \\\\{s}\n", .{l});
    try o.interface.writeAll(";\nvar o = @import(\"std\").fs.File.stdout().writer(&.{});\n");
}
const Q =
    \\pub fn main() !void {
    \\    try o.interface.print("{s}\nconst Q =\n", .{Q});
    \\    var it = @import("std").mem.splitScalar(u8, Q, '\n');
    \\    while (it.next()) |l| try o.interface.print("    \\\\{s}\n", .{l});
    \\    try o.interface.writeAll(";\nvar o = @import(\"std\").fs.File.stdout().writer(&.{});\n");
    \\}
;
var o = @import("std").fs.File.stdout().writer(&.{});
```

## Swift

No main function but different order.

```swift
let Q = #"""
print("let Q = #\"\"\"\n\(Q)\n\"\"\"#\n\(Q)")
"""#
print("let Q = #\"\"\"\n\(Q)\n\"\"\"#\n\(Q)")
```
