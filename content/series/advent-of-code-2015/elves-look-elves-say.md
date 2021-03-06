---
title: "Advent of Code 2015: Elves Look, Elves Say"
date: "2020-05-29T02:16:10Z"
tags: [advent-of-code, ruby]
part-a-url: https://github.com/tinychameleon/advent-of-code-2015/blob/74adc84c0b8e66ac3c045427703af96a7bcced01/2015/10/solution.rb
part-b-url: https://github.com/tinychameleon/advent-of-code-2015/blob/857d93e72c443ede0a4ad82578f5fc469c83dabd/2015/10/solution.rb
---

The [tenth Advent of code challenge](https://adventofcode.com/2015/day/10) is a short, but fun, problem involving translations of numbers as strings.
This entry will continue delving into wall-clock profiling by looking at a simple regular-expression-based solution and optimizing toward a procedural loop which is much faster.

## Part A: Look, Say, Repeat
This problem is interesting because it seems like it could be difficult to solve, but the solution is succinct.
Let's take a look at the problem statement to get a handle on what we need to do:

> Today, the Elves are playing a game called look-and-say. They take turns making sequences by reading aloud the previous sequence and using that reading as the next sequence. For example, 211 is read as "one two, two ones", which becomes 1221 (1 2, 2 1s).
>
> Look-and-say sequences are generated iteratively, using the previous value as input for the next step. For each step, take the previous value, and replace each run of digits (like 111) with the number of digits (3) followed by the digit itself (1).
>
> Starting with the digits in your puzzle input, apply this process 40 times. What is the length of the result?
>
> --- _Advent of Code, 2015, Day 10_

The final paragraph highlights what we need to do: apply the look-and-say algorithm forty times and find the length of the resulting string.
The middle paragraph tells us the details of the look-and-say algorithm: for each run of a given number replace it with the length of the run and the number.
Let's set up some tests based on the example data from the challenge to implement the look-and-say algorithm.

{{< coderef >}}{{< var part-a-url >}}#L6{{</ coderef >}}
```
def tests
  assert look_and_say('1'), '11'
  assert look_and_say('11'), '21'
  assert look_and_say('21'), '1211'
  assert look_and_say('1211'), '111221'
  assert look_and_say('111221'), '312211'
  :ok
end
```

Matching the same number repeatedly is easy with regular expressions, so let's create one to solve our problem.
We need to match a number, which can be done via `\d`, and then need to match more of that number if it repeats.
Something that may or may not repeat can be matched using `*`, and ensuring we match the same number means we need to capture the first number.
Now, capturing the first number uses groups via `(...)`, and referring to that match uses the reference `\1` since we only have one group.
Putting these things together, the regular expression we can use is `(\d)\1*`.

With the regular expression built, we only need to apply it in a loop to extract the length and number for each run:

{{< coderef >}}{{< var part-a-url >}}#L25{{</ coderef >}}
```
def look_and_say(number)
  result = ''
  until (m = /(\d)\1*/.match(number)).nil?
    result << "#{m[0].length}#{m[1]}"
    number = m.post_match
  end
  result
end
```

There are a few things to understand about this implementation:

- We're assigning the match to `m` and checking for `nil` in the same statement.
- The `m` variable represents a `MatchData` instance, which can be indexed like an array.
- The entire match exists as `m[0]` and the first group capture as `m[1]`.
- The input is overridden each loop iteration via `m.post_match`, which contains the remaining characters from `number`.

Once this is done we only need to implement the forty-time iteration on the challenge input and obtain the length of the output string.

{{< coderef >}}{{< var part-a-url >}}#L34{{</ coderef >}}
```
def solve_a(input)
  40.times { input = look_and_say(input) }
  input.length
end
```

Finally, our answer is:
 
```
$ run -y 2015 -q 10 -a
252594
```

## Part B: Must Go Faster
The second part of the challenge is very terse:

> Now, starting again with the digits in your puzzle input, apply this process 50 times. What is the length of the new result?
>
> --- _Advent of Code, 2015, Day 10_

Seems easy!
All we need to do is change our `solve_a` method to take a configurable number of iterations:

{{< coderef >}}{{< var part-b-url >}}#L38{{</ coderef >}}
```
def solve(input, iterations)
  iterations.times { input = look_and_say(input) }
  input.length
end
```

Now we run the solution and...

```
$ run -y 2015 -q 10 -b
3579328
```

We get the right answer after 15 seconds --- a 25% increase in iterations resulted in approximately a 1100% increase in run-time --- this is not good.
Let's use the `time` utility to gather some wall-clock times to optimize the solution:

```
$ time run -y 2015 -q 10 -a
252594

real    0m1.266s
user    0m1.192s
sys     0m0.006s

$ time run -y 2015 -q 10 -b
3579328

real    0m15.297s
user    0m15.204s
sys     0m0.007s
```

Instead of using regular expression objects to match every group in the look-and-say algorithm, we should try to be more efficient, so lets use a more procedural approach to handling the number input.
What we'll do is record where we are in the string, and then jump ahead using a second index while the numeric character is the same --- this will give us the character we need to emit and the run length without needing any fancier concepts.

{{< coderef >}}{{< var part-b-url >}}#L25{{</ coderef >}}
```
def look_and_say(number)
  result = ''
  i = 0
  while i < number.length
    c = number[i]
    j = i
    j += 1 while j < number.length && number[j] == c
    result << "#{j - i}#{c}"
    i = j
  end
  result
end
```

The crucial thing to understand about this approach is that the inner loop, which runs `j += 1`, stops when `number[j]` points to the first character which is not equal to `c`.
This means that `j - i` is equal to the number of times `c` is sequentially repeated.
Visually, if we were looking for the number of times B is repeated sequentially, the indices would look like this:

```
      i = 2
[ A A B B B B B C C ]
                j = 7
```

Now, to measure and see what this attempt at optimization has gained us; first lets run part A again.

```
$ time run -y 2015 -q 10 -a
252594

real    0m0.631s
user    0m0.552s
sys     0m0.006s
```

We've got around a 50% speed-up for the forty iteration case, which is pretty nice considering we've made a fairly direct translation from regular expressions into procedural matching code.
What about part B, which caused us to optimize?
Take a guess quickly before looking at the result.

```
$ time run -y 2015 -q 10 -b
3579328

real    0m6.664s
user    0m6.572s
sys     0m0.007s
```

Approximately a 56% speed-up for the fifty iteration case, which is very similar to part A.
If you don't understand why the speed-up isn't much larger for the second case, think about what we've done:

- We're still checking every character in the string, so both look-and-say algorithms are linear in complexity.
- We've shaved off a consistent amount of work from each loop iteration.

Neither of those things will result in an order-of-magnitude improvement because they don't change the fundamental amount of iterative work we do.
Still, a 50% speed-up is a wonderful thing to have and makes running this problem's solutions a lot more bearable.

## Speed Isn't Everything
Now, that original 15 second run-time wasn't a particularly big deal, as there is no SLA or user request waiting on a look-and-say service to respond.
I wouldn't bother optimizing a script like this at my day job, even if it ran every 5 minutes, because many things can run in the background.
The important thing is to practice using profiling tools, preferably in your spare time with a language you enjoy, so that when they are required you're prepared and know what tools to apply.

It's also occasionally important to be able to drop down a level of abstraction to eke out performance, like from regular expressions to index-based string handling.
That skill requires practice, but is useful for more than just writing loops --- many applications can benefit from a small handful of ORM-managed queries being written directly in SQL.

Take the time to practice some of these skills because they'll make you better at programming even if you never have to optimize code.
