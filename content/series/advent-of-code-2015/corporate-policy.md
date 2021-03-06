---
title: "Advent of Code 2015: Corporate Policy"
date: "2020-06-13T02:58:46Z"
tags: [advent-of-code, ruby]
fileurl: https://github.com/tinychameleon/advent-of-code-2015/blob/d8062d68e5a286afade928b292e951663a28cb93/2015/11/solution.rb
---

During the [eleventh Advent of Code challenge](https://adventofcode.com/2015/day/11) we get to think about the concept of a "sequence" and exercise more regular expression muscle.
Translating text verification steps into regular expressions can be pretty overwhelming without practice, so this is a great opportunity.

## Part A: You Shall Not Password
This challenge is extremely verbose, to give you all the detail needed for a correct solution; try to pull the requirements out of this and you'll see that it's dense compared to other challenges.

> Santa's previous password expired, and he needs help choosing a new one.
>
> To help him remember his new password after the old one expires, Santa has devised a method of coming up with a password based on the previous one. Corporate policy dictates that passwords must be exactly eight lowercase letters (for security reasons), so he finds his new password by incrementing his old password string repeatedly until it is valid.
>
> Incrementing is just like counting with numbers: xx, xy, xz, ya, yb, and so on. Increase the rightmost letter one step; if it was z, it wraps around to a, and repeat with the next letter to the left until one doesn't wrap around.
>
> Unfortunately for Santa, a new Security-Elf recently started, and he has imposed some additional password requirements:
>
> - Passwords must include one increasing straight of at least three letters, like abc, bcd, cde, and so on, up to xyz. They cannot skip letters; abd doesn't count.
> - Passwords may not contain the letters i, o, or l, as these letters can be mistaken for other characters and are therefore confusing.
> - Passwords must contain at least two different, non-overlapping pairs of letters, like aa, bb, or zz.
>
> --- _Advent of Code, 2015, Day 11_

All of that can be reduced to the following requirements for a password:

- Must be 8 characters long.
- Must use only lower-case letters.
- Must include one sequence of 3 consecutive letters.
- Must not contain the letters i, o, or l.
- Must include at least 2 occurrences of different pairs of letters.

The problem statement is followed by several examples, so I've pushed all of them into the normal `test` method that gets defined.

{{< coderef >}}{{< var fileurl >}}#L6{{</ coderef >}}
```
def tests
  assert valid('hijklmmn'), false
  assert valid('abbceffg'), false
  assert valid('abbcdegk'), false
  assert valid('abcdffaa'), true
  assert valid('ghjaabcc'), true
  assert solve_a('abcdefgh'), 'abcdffaa'
  assert solve_a('ghijklmn'), 'ghjaabcc'
  :ok
end
```

From those tests you can see that we need to implement `valid` and `solve_a`, and we'll start with the `valid` method.
We need to translate the restrictions in the last three bullet points above into regular expressions, so let's start with the easiest: recognizing i, o, or l.
We can create a character class containing those letters to build the regular expression `/[iol]/`.
Finding a two letter sequence is easy using a capture group, `/(.)\1/`, but validating the number of pairs and validating the three consecutive letters will require a bit of code. 

{{< coderef >}}{{< var fileurl >}}#L27{{</ coderef >}}
```
SEQUENCES = Regexp.union((?a..?z).to_a.each_cons(3).map(&:join))
DISALLOWED = /[iol]/
DOUBLES = /(.)\1/

def valid(s)
  !!s[SEQUENCES] && s !~ DISALLOWED && s.scan(DOUBLES).uniq.length > 1
end
```

That `SEQUENCES` definition is compact and deserves a long-form explanation:

- The chunk `(?a..?z).to_a` creates a `Range` of characters from 'a' to 'z' and then converts that range into an `Array`.
- Calling `Array#each_cons` results in an `Enumerator` which yields _each consecutive_ sequence of three letters; the result looks like `[['a', 'b', 'c'], ['b', 'c', 'd'], ...]`.
- Then `Enumerator#map` is called on our `Array` of consecutive letter groups, which calls `Array#join` on each group to give `["abc", "bcd", ...]`.
- Finally, `Regexp.union` is called, which creates a regular expression out of a sequence by inserting the `|` operator between each value resulting in a regular expression like `/abc|def|.../`.

With the `SEQUENCES` definition understood the implementation of `valid` should be fairly easy to understand, if you know Ruby's shorthand regular expression operations.
The `!!s[SEQUENCES]` expression checks that there is a match; the `s !~ DISALLOWED` expression ensures there is no match; finally, the `s.scan(DOUBLES).uniq.length` expression yields the number of unique matches of the `DOUBLES` regular expression across the entire value of `s`.

There's no real need to actually check the length of the password for this challenge, because getting to a 9 character password would require checking many billions of strings, and these challenges are meant to be solvable without high-powered machines.

The last thing to do is iterate until we find the next valid password, which we can do in the `solve_a` method easily.

{{< coderef >}}{{< var fileurl >}}#L35{{</ coderef >}}
```
def solve_a(input)
  input.succ! until valid(input)
  input
end
```

Ruby has a successor interface which works on `String` that can give you the next value in the sequence, which you can see from the `input.succ!` call.
The code iterates through a succession of password values until it finds one that is valid, then returns it.
The result when we try to solve part A is:

```
$ run -y 2015 -q 11 -a
vzbxxyzz
```

## Part B: Expired, Again
The second part is very easy, with a light description.

> Santa's password expired again. What's the next one?
>
> --- _Advent of Code, 2015, Day 11_

There's no code for this one, aside from renaming `solve_a` to `solve`, because the answer is `solve(solve(INPUT))`.
With that simple statement, we can get figure out part B:

```
$ run -y 2015 -q 11 -b
vzcaabcc
```

## Text & Ruby
Each time I have to reach for regular expressions during these challenges I'm always happy about the first-class support Ruby provides.
In particular, the `Regexp.union` static method is very cool; I can't see myself needing it often, but I'm glad I can reach for it.
