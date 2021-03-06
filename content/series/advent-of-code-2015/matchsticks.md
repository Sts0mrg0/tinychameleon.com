---
title: "Advent of Code 2015: Matchsticks"
date: "2020-04-21T04:04:48Z"
tags: [advent-of-code, ruby]
fileurl: https://github.com/tinychameleon/advent-of-code-2015/blob/e1cff83ecdaba81b7c6cdb49b1fb77b241a5aa55/2015/8/solution.rb
---

The eighth Advent of Code [challenge](https://adventofcode.com/2015/day/8) is all about character escapes in strings and provides a good opportunity to discuss why you should avoid `eval`.
The problem itself is rather small and has little surface area, so I thought it would be a good opportunity to try out top-down and procedural design using Ruby.

## Part A: Literal Overhead
This challenge is a bit different, since it asks you to find two things about each item in the input list, but the description is mostly focused on explaining the input data --- I have elided this for brevity.

> Santa's list is a file that contains many double-quoted string literals, one on each line. The only escape sequences used are \\ (which represents a single backslash), \" (which represents a lone double-quote character), and \x plus two hexadecimal characters (which represents a single character with that ASCII code).
>
> Disregarding the white-space in the file, what is the number of characters of code for string literals minus the number of characters in memory for the values of the strings in total for the entire file?
>
> --- _Advent of Code, 2015, Day 8_

We need to find the result of subtracting the string literal length from the string in-memory length, but to do that we need to understand how the input strings are formatted:

- One double-quoted string per line
- Three escape sequences in total, `\\`, `\"`, and `\xAB`
- Each escape sequence represents a single character

I want to build out methods that return a hash of data values per string, which I can use to aggregate data to determine the correct answer.
As a first step, I've pulled the example data from the challenge page into a reusable test input.

{{< coderef >}}{{< var fileurl >}}#L4{{</ coderef >}}
```
TEST_INPUT = <<~'DATA'.freeze
  ""
  "abc"
  "aaa\"aaa"
  "\x27"
DATA
```

The solver method should be able to return an answer from the whole input and lean on a method called `char_counts` to determine the literal and in-memory values for each string; let's represent that in some tests.

{{< coderef >}}{{< var fileurl >}}#L17{{</ coderef >}}
```
def test_counts
  assert char_counts('""'), { literal: 2, memory: 0 }
  assert char_counts('"abc"'), { literal: 5, memory: 3 }
  assert char_counts('"aaa\"aaa"'), { literal: 10, memory: 7 }
  assert char_counts('"\x27"'), { literal: 6, memory: 1 }
  assert solve_a(TEST_INPUT), 12
end
```

That `char_counts` method exclusively prepares the hash of key-value pairs and pushes the in-memory length calculation into another helper method.
It also nicely highlights how literal data structures and implicit `return` statements improve the legibility of code by stripping away unnecessary syntax.

{{< coderef >}}{{< var fileurl >}}#L43{{</ coderef >}}
```
def char_counts(string)
  { literal: string.length, memory: char_scan(string[1...-1]) }
end
```

Exploring Ruby's support for procedural code pushes me toward implementing `char_scan` using a while-loop control-flow statement instead of the more functional `Enumerable` methods.
It's important to understand what tools your language provides for expressing problems in their most natural way; sometimes problems are easier to express procedurally than functionally or logically.

{{< coderef >}}{{< var fileurl >}}#L47{{</ coderef >}}
```
def char_scan(string)
  len = 0
  i = 0
  while i < string.length
    c = string[i]
    i += 1
    len += 1
    next unless c == '\\'

    i += string[i] == 'x' ? 3 : 1
  end
  len
end
```

This algorithm relies on the `len` variable to track the in-memory string length and the `i` variable to track the current string index for the while-loop.
If you focus on `len`, notice that it's used in three places: initialization to zero at the top of the function, returned at the bottom of the function, and incremented by one inside the while-loop.
With `len` incremented by one only inside of the while-loop it will always have a value equal to the number of loop iterations.

All the other code is dedicated to figuring out the correct index in the `i` variable which also is incremented by one for each loop iteration; the additional code is all for handling escape sequences.
The algorithm stores the current character in the variable `c` so that it can apply the increment-by-one operation to `i` unconditionally and still check to see if the character is a backslash.
When a backslash character is found there are only two possible cases that are represented by the in-line conditional operator at the end of the while-loop.

Recall from above that the possible escape sequences are `\\`, `\"`, and `\xAB`, and that `i` has already been incremented by one so `string[i]` is referencing the character following the backslash.
If this character is an `x` then we need to move three characters ahead to have `i` reference the first character after the `\xAB` escape sequence, and otherwise it is one of the two-character escape sequences which requires skipping ahead by one.

Having implemented `char_scan` and `char_counts` we can focus on implementing `solve_a` and the aggregation of line data counts.

{{< coderef >}}{{< var fileurl >}}#L73{{</ coderef >}}
```
def solve_a(input)
  lines = input.split("\n")
  totals = lines.each_with_object({ literal: 0, memory: 0 }) do |line, agg|
    char_counts(line).each { |k, v| agg[k] += v }
  end
  totals[:literal] - totals[:memory]
end
```

This may seem like it does a lot, but if you eliminate the first and last lines of the method what remains is a single method call to `each_with_object`.
This method operates similarly to `reduce`, but you do not need to return the aggregate value from the interior block.
Instead you supply a mutable default value which you can freely change within that block --- in this case the `agg` variable refers to the `{ literal: 0, memory: 0 }` hash created in the call to `each_with_object`.

The block itself can be broken into two parts: a call to the method `char_counts(line)` and a call to the `.each` method.
We know that the former chunk of code returns another `{ literal: L, memory: M }` hash, so we can rely on the equivalent keys to add the values into our `agg` hash.
With those sums and the subsequent subtraction we can calculate the answer for part A.

```
$ run -y 2015 -q 8 -a
1342
```

### Security Dangers
The `char_scan` method has another implementation which is significantly shorter, highly legible, and a huge security exploit in production code.

```
def char_scan(string)
  (eval string).length
end
```

Production software must be 100% certain that all values `string` receives are system controlled, otherwise the `eval` statement can do almost anything.
Here's an example irb session showing that `eval` works to calculate the in-memory length, but then is hijacked to issue a shell command and exit the irb program.

```
$ irb
irb(main):001:0> s = '"aaa\"aaa"'
=> "\"aaa\\\"aaa\""
irb(main):002:0> s.length
=> 10
irb(main):003:0> (eval s).length
=> 7
irb(main):002:0> bad = '"abc"; exec "python"'
=> "\"abc\"; exec \"python\""
irb(main):003:0> (eval bad).length
Python 2.7.17 (default, Oct 24 2019, 12:57:38)
[GCC 4.2.1 Compatible Apple LLVM 11.0.0 (clang-1100.0.33.8)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>>
```

In the `bad` case this replaces the `irb` process with a `python` process, which is about as mean as I want to be to myself, but you can imagine how terrible this would be if it occurred in a server-side application.
These types of exploits allow an attacker to run arbitrary code and are classified as "Remote Code Execution"[^1] bugs.

## Part B: Double Encoding Overhead
The second part of this challenge goes in the opposite direction --- the escaped input string length needs to be calculated instead of the in-memory length.

> In addition to finding the number of characters of code, you should now encode each code representation as a new string and find the number of characters of the new encoded representation, including the surrounding double quotes.
>
> Your task is to find the total number of characters to represent the newly encoded strings minus the number of characters of code in each original string literal.
>
> --- _Advent of Code, 2015, Day 8_

Similar to part A, I want to have a solver method that relies on a method called `encode_counts` to create a hash containing the literal and encoded character counts.
The implementation was fairly good last time, so replicating it seems like a great way to speed up finding the answer to part B.

{{< coderef >}}{{< var fileurl >}}#L61{{</ coderef >}}
```
def encode_counts(string)
  { literal: string.length, encoded: char_encode(string) }
end
```

The method called `char_encode` will calculate the total length of the escaped string, but before I can confidently work on it I want to duplicate prior tests to figure out the expected results.

{{< coderef >}}{{< var fileurl >}}#L25{{</ coderef >}}
```
def test_encodings
  assert char_encode('""'), 6 # "\"\""
  assert char_encode('"abc"'), 9 # "\"abc\""
  assert char_encode('"aaa\"aaa"'), 16 # "\"aaa\\\"aaa\""
  assert char_encode('"\x27"'), 11 # "\"\\x27\""
  assert solve_b(TEST_INPUT), 19
end
```

Using the commented examples above to think about how to calculate escaped string length turned out to be valuable --- the solution is surprisingly simple.
Any backslash or double-quote character counts as two characters and everything else counts as one.

{{< coderef >}}{{< var fileurl >}}#L65{{</ coderef >}}
```
ENCODE = ['\\', '"'].freeze

def char_encode(string)
  string.each_char.reduce(0) do |sum, c|
    sum + (ENCODE.include?(c) ? 2 : 1)
  end + 2
end
```

Each character in the string is compared against the two necessary to escape characters and the sum of all characters is found.
The final piece is to take into account the surrounding double-quotes of a string literal by adding two to the sum.

To tie everything together the `solve_b` method can be written almost identically to the previous `solve_a` method --- the only differences being the `:encoded` symbol and the `encode_counts` call.

{{< coderef >}}{{< var fileurl >}}#L81{{</ coderef >}}
```
def solve_b(input)
  lines = input.split("\n")
  totals = lines.each_with_object({ literal: 0, encoded: 0 }) do |line, agg|
    encode_counts(line).each { |k, v| agg[k] += v }
  end
  totals[:encoded] - totals[:literal]
end
```

Running the solution for part B gives us our answer.

```
$ run -y 2015 -q 8 -b
2074
```

## Escaping Escaping
Procedural algorithms can be written quite nicely in Ruby thanks to conditional modifier clauses which reduce nested indentation levels for single conditional statements.
Some of the de facto standard tools surrounding the language, like Rubocop, are not particularly friendly toward procedural code, having defaults which make code more verbose.
For example, Rubocop will flag multiple assignments per line as a problem, which is why I did not write `len, i = 0, 0` in the `char_scan` method.
It highlights the importance of tuning tool settings for your use-case --- Rubocop is still very useful for procedural code with some adjustments.

I'm quite happy that this challenge allowed a bit of discussion around `eval` and security because it's always good to think about these kinds of constructs.
Having a low risk place to try out things and learn why they're bad and how to avoid them is critical to becoming a better programmer, no matter the language.

[^1]: Frequently people use the initialism "RCE" instead.
