Title: Elixir Tip - Generate a list filled by the same value
Date:2017-02-15
Tags: elixir, elixir-tip
Category: elixir
Authors: José San Gil

I've been learning Elixir and functional programing (for real) recently, and stumbled upon this simple problem of generating a list of n elements with the same value. At first, I implemented an overcomplicated solution using ranges and map, but then realized it was super easy.

So, this is my first (begineer) post on elixir and it's just a tip *cough* (reminder for me) on how to generate a list filled by the same character, number, string, etc., n times.

Let's for example generate a list filled by 8 'x' characters (unicode 120), that's 'xxxxxxxx':

```elixir
iex> List.duplicate 120, 8
'xxxxxxxx'
```

That's it. `List.duplicate` is the answer (there is also a `duplicate` function for `String` and `Tuple`).
