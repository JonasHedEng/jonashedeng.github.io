+++
title = "In praise of Gleam's decode"
date = 2025-04-07T20:03:51+02:00
tags = ["gleam", "decode", "rust"]
draft = false
+++

When first encountering Gleam's approach to encoding/decoding data to/from outside of Gleam, I (just as many other people new to Gleam) think it strange that such a modern and well-thought out language still has a relatively verbose and manual-ish[[1](#footnotes)] way of doing it. I've now realized I much prefer this approach over some combination of terseness, weird dedicated syntax, macros, magics, etc.

While working on a little side project I'm doing just for fun (and to learn), I encountered an "interesting" data structure that I wanted to extract a couple of fields from. Initially I thought it to be a basic dictionary-like data structure but I soon realized it was a bit more convoluted.

A bit of brain wrestling later, I managed to write this somewhat unreadable snippet that seemingly does the job. At the time I didn't think much about it, I was mostly satisfied that I got something working.

```gleam
fn decoder() {
  let file_decoder = {
    use name <- decode.field("name", decode.string)
    decode.success(name)
  }

  use length <- decode.field("length", decode.int)

  let idcs =
    list.range(0, length - 1)
    |> list.map(fn(i) { int.to_string(i) })

  let assert [first, ..rest] = idcs

  let initial = fn(acc: List(String)) -> decode.Decoder(List(String)) {
    use next <- decode.field(first, file)
    decode.success([next, ..acc])
  }

  let final =
    list.fold(rest, initial, fn(acc_fn, index) -> fn(List(String)) ->
      decode.Decoder(List(String)) {
      fn(acc: List(String)) {
        use next <- decode.field(index, file)
        acc_fn([next, ..acc])
      }
    })

  final([])
}
```

After about a day though, I went back to the snippet, trying to determine how unreadable it is. On a first glance, even with the implementation somewhat recent in memory, I struggled to understand and break down _how_ it worked fully. A good start, you might think(it is), would be to make sure you understand the incoming data first. Let's take a look:

```json
{
  "length": 3,
  "0": {"name": "file1.txt"},
  "1": {"name": "file2.txt"},
  "2": {"name": "file3.txt"}
}
```

The object entries are dynamic and depend on what `length` is set to. Now, it clicked. The reason I struggled so with getting the decoder decent was that the incoming data structure was "interesting" at best and rather not at all how I would've preferred it. What I also realized, was that although having worked professionally in Rust for about 6-7 years, I would dread having to implement the decoder for such a data structure (at least for a few seconds, before realizing it's not that difficult, serde is great!). With Gleam's `decode` however, I did it without fully realizing how inconvenient the incoming data was. I simply took it step by step.
- First, decode the `length`.
- Then, generate the decoder for the first field.
- Now, figure out how to pass that to the decoder of the next field. Wait, or should the next field take the current decoder as a parameter? (This was the brain wrestling part)
- All of a sudden, there were no compilation errors and it "Just Worked".

Revisiting the solution, I wanted to address the `assert` that should pretty much always be avoided since it can cause a panic.

```gleam
fn decoder() {
  let file_decoder = {
    use name <- decode.field("name", decode.string)
    decode.success(name)
  }

  use length <- decode.field("length", decode.int)

  let indices =
    list.range(0, length - 1)
    |> list.map(int.to_string)

  let files =
    list.fold(indices, decode.success, fn(acc_fn, index) {
      fn(acc: List(String)) {
        use next <- decode.field(index, file_decoder)
        acc_fn([next, ..acc])
      }
    })

  files([])
}
```

The result might still not be easy to read and grasp(especially if you're new to Gleam, functional programming or the `use` semantics) but I really liked how consistent it felt with the otherwise "regular" decoding of a simple structure. We don't need an extra special decode implementation, we just combine the regular building pieces in a slightly different way!
Gleam's `decode` might not be as terse and convenient as decoders or deserializers found in other languages but what you get instead is consistency, no magic, and that the regular approach to decoding looks and works the same no matter how "interesting" the data structure might be.
If you would like to understand more about the implementation details and Gleam in general I highly recommend diving in! Gleam is a wonderful language, ecosystem and community. Though do be aware, returning to your previous go-to language might induce(including, but not limited to):
- the feeling of bloat,
- uneasyness around "automagic" features,
- frustration of having too many choices to implement that one feature,
- confusion about why there are so many language features doing what one smarter one would achieve(yes, I miss `use` in Rust these days...)

###### Footnotes
1. The Gleam Language Server has a bunch of really cool code actions that can do some, or in some cases, all of the work for you!
