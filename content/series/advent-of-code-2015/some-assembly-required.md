---
title: "Advent of Code 2015: Some Assembly Required"
date: "2020-04-14T00:39:10Z"
tags: [advent-of-code, ruby]
fileurl: https://github.com/tinychameleon/advent-of-code-2015/blob/131c7710c5b6c29f83e5bdb0ffbb10adc9f80a38/2015/7/solution.rb
---

The seventh Advent of Code [challenge](https://adventofcode.com/2015/day/7) has a few very different solutions; I chose one that would let me experiment with Ruby lambda features.
The solution presented here isn't too long, but much shorter answers are possible if you're willing to use the `eval` method to create assignment statements.
The presentation format is going to be a little different than the last posts --- we're going to move through the solution in a top-down fashion instead of from input-to-output.


## Part A: What's on the Wire
This is a wordy problem, so I have eliminated a couple paragraphs to save space; if you want the full description go look at the Advent of Code website.
Like always, I suggest you try to work out the requirements before reading below, but this problem has quite a few which are interwoven.

> Each wire has an identifier (some lowercase letters) and can carry a 16-bit signal (a number from 0 to 65535). A signal is provided to each wire by a gate, another wire, or some specific value. Each wire can only get a signal from one source, but can provide its signal to multiple destinations. A gate provides no signal until all of its inputs have a signal.
>
> The included instructions booklet describes how to connect the parts together: x AND y -> z means to connect wires x and y to an AND gate, and then connect its output to wire z.
>
> ...what signal is ultimately provided to wire a?
>
> --- _Advent of Code, 2015, Day 7_

Here are the requirements I see:

- Wires have names consisting of one or more lower-case letters
- Wires carry a 16-bit, unsigned integer value from a signal
- Signals are either a gate, another wire or a literal value
- Wires may only have one source signal
- Gates only provide a value when all inputs have a value

By following those requirements and creating an interface to ask for a specific wire's value, we should be able to solve the problem.
What I would like to do, at the top-level of the solution, is pass the problem input and the wire name into a function and get back the wire value.

{{< coderef >}}{{< var fileurl >}}#L47{{</ coderef >}}
```
TEST_INPUT = <<~DATA.freeze
  123 -> x
  456 -> y
  x AND y -> d
  x OR y -> e
  x LSHIFT 2 -> f
  y RSHIFT 2 -> g
  NOT x -> h
  NOT y -> i
DATA

def test_solve
  assert solve(TEST_INPUT, :g), 114
  assert solve(TEST_INPUT, :i), 65_079
end
```

The test input is pulled directly from the Advent of Code challenge page, and the test describes precisely the no-nonsense interface above.
I decided to use symbols (`:g`) as wire names, though strings would have been completely acceptable, as it just felt a bit cleaner to me.
While the `solve` method feels like a good place to deal with breaking apart the input, it doesn't feel like the spot for any of the signal value calculation logic.

{{< coderef >}}{{< var fileurl >}}#L155{{</ coderef >}}
```
def solve(input, wire)
  c = Circuit.new
  input.split("\n").each { |l| c << parse_line(l) }
  c.read(wire)
end
```

I've decided to delegate the real work of parsing each input line to a `parse_line` method and the work of calculating the wire values to a `Circuit` class.
This split seemed reasonable to me, but my idea is that the `Circuit` class will store the wire values and handle delegation of the logic gate calculations to a `Gate` class.

### Logic Gates
To create these logic gates each line of input will need to be analysed to determine what kind of gate it should be --- recall from above that a wire can receive a signal from a literal value, another wire, or a logic gate.
I would like to model all of these as a single `Gate` class that can be parametrized by a lambda.

{{< coderef >}}{{< var fileurl >}}#L150{{</ coderef >}}
```
def parse_line(line)
  wire, parts = components(line)
  [wire.to_sym, build_gate(parts)]
end
```

Parsing each individual line turns out to be rather easy to write because it delegates to two other methods; remember that I also decided to represent wires as symbols, not strings.
Of the two methods `components` and `build_gate` only the latter requires tests as it will be the factory method that needs to correctly parametrize the `Gate` instances.

{{< coderef >}}{{< var fileurl >}}#L133{{</ coderef >}}
```
def components(line)
  gate, wire = line.split(' -> ')
  [wire, gate.split]
end
```

As you can see both `parse_line` and `components` are simple and, at least in my opinion, don't need tests for this solution.
On the other hand, the `build_gate` method I have chosen to test because it will require a reference to the currently non-existent `Circuit` class.
This kind of relationship should have some tests around it to ensure the communication between classes works correctly.

{{< coderef >}}{{< var fileurl >}}#L21{{</ coderef >}}
```
def test_build_gate
  g = build_gate(['123'])
  assert g.inputs, [123]

  g = build_gate(%w[x AND y])
  assert g.inputs, %i[x y]

  g = build_gate(%w[x OR 5])
  c = Circuit.new
  c << [:x, build_gate(['10'])]
  c.read(:x)
  assert g.signal(c), 15
end
```

The tests I've written exercise the `Gate` class to retrieve the signal value and ensure the input record is correct.
It doesn't fully exercise the class for all possible inputs, but it's good enough to give me confidence in the code I will be writing.

{{< coderef >}}{{< var fileurl >}}#L120{{</ coderef >}}
```
DIGIT = /^[0-9]/.freeze

OPERATIONS = {
  'AND' => :&,
  'OR' => :|,
  'LSHIFT' => :<<,
  'RSHIFT' => :>>
}.freeze

def val(v)
  DIGIT.match?(v) ? v.to_i : v.to_sym
end

def build_gate(parts)
  case parts.length
  when 1
    Gate.new(->(x) { x }, val(parts[0]))
  when 2
    Gate.new(->(x) { ~x }, val(parts[1]))
  when 3
    op = OPERATIONS[parts[1]]
    Gate.new(->(l, r) { l.send(op, r) }, val(parts[0]), val(parts[2]))
  end
end
```

Here I finally get to look at using lambda functions to parametrize `Gate` operations by encoding the particular logic operation.
There is a small helper method `val` which is used to determine the resultant data type of each gate input which can either be an integer literal or another wire.
Since there are no requirements for wires supporting nested signal expressions using parenthesis, I've opted to look at the number of components on the left-hand side of the `\->` to determine what kind of strategy is necessary for the `Gate`.

When there is only one item, like "Y \-> x", then it represents a simple pass-through of a literal integer or wire value.
Two items is only represented by "NOT Y \-> x", making that an easy logic gate to detect.
Finally, three items can be any other logic gate, all of which take two parameters, and I `send` the operation by looking up the correct method via the string representation.
Each of these `Gate.new` statements passes in the pre-requisite inputs for the lambda operation.

The last piece of the input-to-gate translation is to create the `Gate` class itself.

{{< coderef >}}{{< var fileurl >}}#L63{{</ coderef >}}
```
class Gate
  attr_reader :inputs

  def initialize(op, *inputs)
    @op = op
    @inputs = inputs
  end

  def signal(circuit)
    args = inputs.map { |i| circuit.signal(i) }
    @op.call(*args) & 0xffff
  end
end
```

Remember, as shown above, that the `Gate` class requires `inputs` and `signal` as the interface, and that we want to pass a lambda as an operation and the inputs.
The `Gate` itself doesn't keep track of any values, so inside `signal` it asks the given `Circuit` for the value of every relied upon input.
Importantly, the `Gate` class also applies the 16-bit restriction by dropping any extraneous bits via `& 0xffff`.
The remaining portion of code to write is the `Circuit` which records all the signal values.


### The Circuit
This class is the most complicated part of the solution because it only calculates necessary wire values for the requested read operation.
I've decided on a simple interface to `Circuit` which only has two methods: `<<` as a short-hand for "add-a-wire-and-gate", `read` to ask for a wire value, and `signal` to fetch the value of a particular input for `Gate`.

{{< coderef >}}{{< var fileurl >}}#L35{{</ coderef >}}
```
def test_circuit
  c = Circuit.new
  c << [:z, build_gate(%w[x AND y])]
  c << [:x, build_gate(['123'])]
  c << [:w, build_gate(%w[NOT z])]
  c << [:y, build_gate(['456'])]

  assert c.read(:x), 123
  assert c.read(:z), 72
  assert c.read(:w), 65_463
end
```

This test also doubles as a check that `Circuit` will work with the output of the `parse_line` method that we wrote above; if the class works with our test data then it should work with `parse_line`, unless that method has a bug.

{{< coderef >}}{{< var fileurl >}}#L77{{</ coderef >}}
```
class Circuit
  def initialize
    @wires = {}
    @signals = {}
  end

  def <<(signal)
    @wires[signal[0]] = signal[1]
  end

  def signal(s)
    s.is_a?(Symbol) ? @signals[s] : s
  end
...
```

The first chunk of the `Circuit` class is fairly easy to understand --- initialization of the wire and signal state in `initialize`, poking around the signals state values for `Gate` in `signal`, and storing the associations of wire names to signal representations using `<<`.
The `read` method is the bulk of the class and implements a depth-first approach to calculating only the necessary wire values.
Before looking at `read` I want to look at two helper methods because they will make understanding the depth-first approach a little easier.

{{< coderef >}}{{< var fileurl >}}#L111{{</ coderef >}}
```
def missing_signals(inputs)
  inputs.filter { |x| x.is_a?(Symbol) && !@signals.key?(x) }
end

def calculate_signal(wire, gate)
  @signals[wire] = gate.signal(self)
end
```

The `calculate_signal` method stores the result of asking the `Gate` for it's signal to re-use it if the wire is referenced multiple times.
More important to the algorithm is the `missing_signals` method --- it determines which inputs still need to be calculated so that we only deal with each wire once.
With these two methods, we can now look at the `read` implementation.

{{< coderef >}}{{< var fileurl >}}#L91{{</ coderef >}}
```
def read(wire)
  queue = [wire]
  until queue.empty?
    w = queue[-1]
    unless @signals.key?(w)
      gate = @wires[w]
      missing = missing_signals(gate.inputs)
      unless missing.empty?
        queue.concat(missing)
        next
      end
      calculate_signal(w, gate)
    end
    queue.pop
  end
  @signals[wire].to_i
end
```

This is, at its heart, a depth-first search --- we have a work `queue` that dictates how long we stay in the method for and we take some actions until it is empty.
Each time around we look at the last wire in the work queue and if it's already in the `@signals` cache then we can skip all the work and remove the wire from the queue.
When we do need to calculate the wire signal, we first retrieve the `Gate` and find any missing signal dependencies; if there are missing signal dependencies we can't calculate the gate's value yet, we need to calculate those dependencies first, so we add them to the back of the queue and jump to the top of the loop.
Otherwise, we have all the dependencies for the current wire and we can calculate its signal.

That's the entire implementation, and we can run the code to figure out what wire `:a` has as a value.

```
$ run -y 2015 -q 7 -a
956
```

## Part B: Override the Signal
This implementation worked out well because the second part of this challenge requires the same thing, but with slightly different inputs.

> Now, take the signal you got on wire a, override wire b to that signal, and reset the other wires (including wire a). What new signal is ultimately provided to wire a?
>
> --- _Advent of Code, 2015, Day 7_

To solve this, I didn't write any code --- I duplicated the input file, manually replaced the value for wire `:b` with `956`, and loaded that file for `part_b`.

```
$ run -y 2015 -q 7 -b
40149
```

## Assembly Complete
Top-down design in this solution worked out for me, but I can't say I particularly enjoyed writing the code this way --- I didn't bother showing the refactoring steps I took to improve the code, but you can see them [in this commit](https://github.com/tinychameleon/advent-of-code-2015/commit/d0d4a5afc10aa9b01d47338549d7870b631cdd0e).
While the lambda support in Ruby is nice, I'm not particularly happy with the cyclical dependency represented by `Circuit` and `Gate`.
It feels to me like object-orientation led me toward this design, but the solution works and is relatively quick.
Sometimes that's enough.
