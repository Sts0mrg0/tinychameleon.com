---
title: "Advent of Code 2015: Doesn't He Have Intern-Elves For This?"
date: "2020-03-20T03:59:21Z"
tags: [advent-of-code, ruby]
part-a-url: https://github.com/tinychameleon/advent-of-code-2015/blob/9a4e4e918f829ee6667d17dc4917edb8295558a4/2015/5/solution.rb
part-b-url: https://github.com/tinychameleon/advent-of-code-2015/blob/188e65ef8d32c5f2c62c3c09ab875e1dacf54050/2015/5/solution.rb
---

We're going to get to exercise some regular expression skills in the fifth Advent of Code [challenge](https://adventofcode.com/2015/day/5).
This one is also a bit on the simplistic side, but I'm learning Ruby as I go and it'll be good practice for slinging regular expressions with Ruby's `String` class methods.

## Part A: Niceties
Surprisingly, this problem already has a bulleted list of things to implement, so let's look at the description from Advent of Code:

> Santa needs help figuring out which strings in his text file are naughty or nice.
>
> A nice string is one with all of the following properties:
>
> - It contains at least three vowels (aeiou only), like aei, xazegov, or aeiouaeiouaeiou.
> - It contains at least one letter that appears twice in a row, like xx, abcdde (dd), or aabbccdd (aa, bb, cc, or dd).
> - It does not contain the strings ab, cd, pq, or xy, even if they are part of one of the other requirements.
>
> How many strings are nice?
>
> --- _Advent of Code, 2015, Day 5_

The challenge is to determine how many strings in the input match all of the bulleted requirements, which should be pretty simple to express thanks to regular expressions.
I want to start with some tests for the examples the challenge gives, to codify the API for the `nice?` method.

{{< coderef >}}{{< var part-a-url >}}#L5{{</ coderef >}}
```
assert nice?('ugknbfddgicrmopn'), true
assert nice?('aaa'), true
assert nice?('jchzalrnumimnmhp'), false
assert nice?('haegwjzuvuyypxyu'), false
assert nice?('dvszwmarrgswjxmb'), false
```

I'm choosing to implement the bulk of the requirements as a predicate method to keep things nice and simple.
Let's concisely repeat the requirements for a nice-string in order to better understand the necessary regular expressions.

- Must have 3+ vowels; vowels being "a", "e", "i", "o", or "u"
- Must have a doubled letter, like "aa"
- Does not contain "ab", "cd", "pq", or "xy"

The trick will be to convert these statements into regular expressions which work with particular methods on the `String` class.
For example, the first statement can be turned into the regular expression `/[aeiou]/`, but this doesn't work to detect three or more vowels.
To detect 3 or more vowels, `/[aeiou]/` can be paired with `String#scan` and `Array#count` to give a concise check of `word.scan(/[aeiou]/).count >= 3`.

The second statement can be easily checked by using `String#match?` which returns a boolean indicating if there was a match. The regular expression can use a capture group to determine if a letter is repeated by constructing it as `/([a-z])\1/`.

Finally, the third statement ensures that we don't have any of the four bigrams mentioned, so we can use the alternation operator `|` to construct a regular expression like `/ab|cd|pq|xy/`.
Put all of these together and you get a `nice?` method that looks something like this:

{{< coderef >}}{{< var part-a-url >}}#L23{{</ coderef >}}
```
def nice?(word)
  [
    word.scan(/[aeiou]/).count >= 3,
    word.match?(/([a-z])\1/),
    !word.match(/ab|cd|pq|xy/)
  ].all?
end
```

This method constructs an array of boolean values and applies `Array#all?` to verify that each element is `true`.
All that's left is to wire up the `solve_a` method to use our `nice?` method as a filter.

{{< coderef >}}{{< var part-a-url >}}#L31{{</ coderef >}}
```
def solve_a(input)
  input.split.filter { |w| nice?(w) }.count
end
```

One thing I've found so far to be a minor annoyance is missing syntax for passing methods into things like `Array#filter`.
I would like the ability to write this as `input.split.filter(nice?).count`, but Ruby only supports this kind of thing for methods on the iterated element type.
Had I implemented `nice?` on the `String` class I could have used `filter(&:nice?)`.
Ruby is an object-oriented language, so I understand the decision to support iterated element methods only, but it still bugs me.

However, with `solve_a` completed we can find out how many nice-strings are in the input to complete the first part.

```
$ run -y 2015 -q 5 -a
236
```

## Part B: Nice 2.0
The second part defines an entirely different set of rules for nice-strings, but it follows the same structure as the previous part.

> Realizing the error of his ways, Santa has switched to a better model of determining whether a string is naughty or nice. None of the old rules apply, as they are all clearly ridiculous.
>
> Now, a nice string is one with all of the following properties:
>
> - It contains a pair of any two letters that appears at least twice in the string without overlapping, like xyxy (xy) or aabcdefgaa (aa), but not like aaa (aa, but it overlaps).
> - It contains at least one letter which repeats with exactly one letter between them, like xyx, abcdefeghi (efe), or even aaa.
>
> How many strings are nice under these new rules?
>
> --- _Advent of Code, 2015, Day 5_

We also get a nice pre-bulleted requirements list and a set of examples to turn into tests:

{{< coderef >}}{{< var part-b-url >}}#L11{{</ coderef >}}
```
assert better_nice?('qjhvhtzxzqqjkmpb'), true
assert better_nice?('xxyxx'), true
assert better_nice?('aaaya'), false
assert better_nice?('uurcxstgmygtbstg'), false
assert better_nice?('ieodomkazucvgmuy'), false
```

Thinking about the regular expression requirements for the first statement, we'll need a capture group and to separate the two bigram instances by at least 0 characters.
Direct translation of those two ideas leads to the regular expression `/([a-z]{2}).&ast;\1/`.

The second statement is similar, except it matches one character on either side of another single character.
Translating that into a regular expression gives `/([a-z])[a-z]\1/`.
Putting these together, like the prior `nice?` method, gives us the `better_nice?` method.

{{< coderef >}}{{< var part-b-url >}}#L37{{</ coderef >}}
```
def better_nice?(word)
  [
    word.match?(/([a-z]{2}).*\1/),
    word.match?(/([a-z])[a-z]\1/)
  ].all?
end
```

Of course, we also need to wire up the `solve_b` method, which is very similar to `solve_a`.

{{< coderef >}}{{< var part-b-url >}}#L48{{</ coderef >}}
```
def solve_b(input)
  input.split.filter { |w| better_nice?(w) }.count
end
```

I'm also still wishing I didn't have to create an additional block just to pass the parameter through, but at least we can solve part B now and finish the challenge.

```
$ run -y 2015 -q 5 -b
51
```

## Very Nice 👍
There's not much to day 5, but learning about the different `String` methods that take regular expressions was worth it.
I am sure they will be handy with future challenges, and it's always great to get some practice slinging regular expressions in the programming language you're using.

I doubt I will immediately remember to add methods to classes in order to avoid creating pass-through blocks, but maybe it will slowly become ingrained into my solutions.
