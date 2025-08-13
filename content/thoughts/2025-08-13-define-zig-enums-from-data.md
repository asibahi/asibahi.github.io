+++
title = "Define Zig Enums from Data"
date = 2025-08-13
+++

I am working currently on a small Zig library which I am not quite ready to talk about yet. However, it involves parsing a big table of data and then doing operations based on that data.

For the remainder of this article, I shall pretend the library is about wind directions.

> Note that this article uses Zig `0.14.1`. Big changes have happened in the `master` branch since that was released. Though the parts discussed are probably still the same.

## Library Structure Outline

There is a data file called `WindData.txt`. It looks something like this.[^gemini]

[^gemini]: Generated with Gemini help.

```
00;Mistral;N
01;Bora;N
02;Sirocco;S
03;Khamaseen;S
04;Chinook;W
05;Zonda;W
06;Föhn;S
07;Santa Ana;E
08;Harmattan;E
09;Leste;E
0A;Pampero;S
0B;Nor'easter;N
0C;Southerly Buster;S
0D;Berg Wind;E
0E;Vendavales;S
```

This text file is parsed by a `generate.zig` file that organizes winds by their direction. The data is put into a `wind_data.zon` file, which is added by the build system as an `Import` so it is simply imported into the code base and .. calculated on.

The `enum` that describes wind directions is duplicated between the main code base and `generate.zig`.

```zig
const Direction = enum { N, E, S, W };
```

## Problem Statement

What if a new version of the data adds a new wind direction: `U` (for Upwards)? Now the code fails! It needs a way to create the type out of the data.

A rational person would just add the new direction manually, because the algorithms depending on it are going to need to change anyway.

The normal solution (in widely used rust crate `wind-direction`, for example) is good old fashioned code generation using a `generate.py` script that is run manually every once in a while by the maintainers, which generates a `tables.rs` file that includes the above enum.

But this is Zig. Things are not done with code generation. Things are done with `comptime` and build systems!

---

## `generate.zig`

I will spare you the boring mechanical parts of `generate.zig` that read a file and iterate over each line and collects the data into some sort of hashmap. The interesting bit I want to talk about is generating `type-definition.zon`.

During file reading, all the direction strings are collected into a set to make sure there are no repeats. This is simple enough code.

```zig
var set: std.StringArrayHashMapUnmanaged(void) = .empty;

// later in the loop,
try set.put(gpa, wind_direction_string, {});
```

After the set is collected, it needs to be serialized into `zon` (which is Zig's own `json`-like data type). This is as simple as outputting text to a file, but the Zig standard library provides some helpful builtins.[^json] This is easiest way I found to serialize the strings to what Zig calls `enum_literal`s. (I will get back to that).

[^json]: Unlike formatting and `json` serialization, `zon` serialization does not seem to depend on defining [specific magic functions](https://www.openmymind.net/Writing-Json-To-A-Custom-Output-in-Zig/) to customize the behaviour.

```zig
const out = std.io.getStdOut().writer(); // <-- replace with output file

// after finishing up the set.
const directions = set.keys();

var sz = std.zon.stringify.serializer(out, .{});
var container = try sz.beginTuple(.{});
for (directions, 0..) |direction, idx| {
    if (idx != 0) try sz.writer.writeByte(','); // a bit hacky but w/e
    try sz.ident(direction);
}
try container.end();
// maybe should print a new line here too?
```

And voila! Here is the happy `data.zon`. Note that this is written to an output file, not `stdout` like the code snippet below.

```
.{.N,.E,.S,.W}
```

## `build.zig`

This is the least interesting part. It is just plumbing. Here is the code. It would get invoked whenever `zig build` commands are called, and cached accordingly, hopefully?

```zig
// calling genrate script
const generate_wind_data = b.addExecutable(.{
    .name = "wind_dir_data",
    .root_source_file = b.path("tools/generate.zig"),
    .target = b.graph.host,
});
const generate_step = b.addRunArtifact(generate_wind_data);
generate_step.addFileArg(b.path("WindData.txt"));
const wind_zon = generate_step.addOutputFileArg("data.zon");

// Main library.
const lib_mod = b.createModule(.{
    .root_source_file = b.path("src/lib.zig"),
    .target = target,
    .optimize = optimize,
});

// piping the two
lib_mod.addAnonymousImport(
    "wind_tables",
    .{ .root_source_file = wind_zon },
);
```

And that's it. Fairly simple code if you're familiar with the Zig build system.

## The Library

Here is the cool stuff, if I may. Since this for defining a normal, non-generic type, we shall start it thus. All the code in the following snippets will be in there.

```zig
const WindData = WD: {
    // All following code is in here.
};
```

First, we need to `@import` the data. That's why it is in a `zon` file to begin with! But import as what type? In `0.14.1`, we still need to define the types `zon` imports (a restriction I understand is lifted in `master`).

Zig has a special `comptime` only type called `enum_literal`. Oddly enough, unlike `comptime_int` and `comptime_float`, you cannot just do this:

```zig
// does not compile
const foo: enum_literal = .bar;
```

The type can actually be written out as `@TypeOf(.enum_literal)`. Sure. A `const` slice of those would be `[]const @TypeOf(.enum_literal)`. Not that you need the `const` here for deserialization, as you cannot deserialize into a mutable slice.

```zig
const data: []const @TypeOf(.enum_literal) = @import("test.zon");
```

Then we need to iterate over the fields and convert them, one by one, to another type. This time it is `std.builtin.Type.EnumField`. An array of which will become our final `enum`. We create a `BoundedArray` at home with just an index and an iterator. This code was pretty much lifted from [Mitchell Hashimoto's 'Tagged Union Subsets with Comptime in Zig' article](https://mitchellh.com/writing/zig-comptime-tagged-union-subset), which is actually the inspiration for _this_ post.[^matklad]

[^matklad]: Also, slightly related, is [matklad's Partially Matching Zig Enums](https://matklad.github.io/2025/08/08/partially-matching-zig-enums.html)

```zig
var i: usize = 0;
var fields: [data.len]std.builtin.Type.EnumField = undefined;

// outer: // uneeded block label. see below
for (data) |literal| {
    const name = @tagName(literal);

    // // Deduplication code. It is not needed because the file is
    // // generated from a hash set.
    // for (fields[0..i]) |f|
    //     if (std.mem.eql(u8, f.name, name))
    //         continue :outer;

    fields[i] = .{
        .name = name,
        .value = i,
    };
    i += 1;
}
```

Now we simply create the `enum` with the helpful builtin `@Type`.

```zig
break :WD @Type(.{ .@"enum" = .{
    .tag_type = usize,
    .fields = fields[0..i],
    .decls = &.{},
    .is_exhaustive = true, // maybe?
} });
```

Fairly simple. One small trick remains. `tag_type` is not an optional value, so we need to specify it. It is the integer that should hold the results of the enum.. `usize` is playing it safe, because it is extremely unlikely to hold more than .. whatever the largest `usize` is. But that's a very big enum for something than potentially fit into one byte. So it is incumbent upon us to create the smallest integer possible.

```zig
const backing_int: type = @Type(.{ .int = .{
    .bits = // what to put here??
    .signedness = .unsigned,
} });
```

To calculate the smallest number of bits that can hold our `fields` count, two simple (for computers at least) math operations are needed. First, `i` (our counter) is raised to the nearest power of two. Then the `log2` is taken from it. Boom. Two `std.math` function calls and that is IT.

```zig
const next_p_of_2 = std.math.ceilPowerOfTwoAssert(usize, i + 1);
const bits = std.math.log2_int(usize, next_p_of_2);
```

And this does it. Here is the full code.

```zig
const WindData = WD: {
    const data: []const @TypeOf(.enum_literal) = @import("test.zon");

    var i: usize = 0;
    var fields: [data.len]std.builtin.Type.EnumField = undefined;

    for (data) |literal| {
        const name = @tagName(literal);

        fields[i] = .{
            .name = name,
            .value = i,
        };
        i += 1;
    }

    const next_p_of_2 = std.math.ceilPowerOfTwoAssert(usize, i + 1);
    const bits = std.math.log2_int(usize, next_p_of_2);

    break :WD @Type(.{ .@"enum" = .{
        .tag_type = @Type(.{ .int = .{
            .bits = bits,
            .signedness = .unsigned,
        } }),
        .fields = fields[0..i],
        .decls = &.{},
        .is_exhaustive = true,
    } });
};
```

Your `enum` is ready. Now your data definition is resilient to new types added to the database. You still need to fix all the `switch` statements, tho.

---

## Conclusion

In the actual library I just put out the enum definition into its own separate file that is imported by both `generate.zig` and the library. But hey, this was a fun exercise.

Until later.

---
