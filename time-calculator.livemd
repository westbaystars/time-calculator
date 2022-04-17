# Time Calculator

## Forward

Let's start with the problem to be solved.

I would like to introduce someone to the techniques that Functional Programming 
offers despite her being in an Object Oriented environment. After a short code review
using [The Code Review Pyramid](https://www.morling.dev/blog/the-code-review-pyramid/), it was
determined that her code could be made much more clear by separating some of the concerns,
such as by extracting a simple time calculating API that
doesn't know anything about the rest of the web application. She's tried a number of things
with her current project, but separating out an API for such a single-purpose SPA (Single Page
Application) doesn't seem worth the effort. I'd like to change her mind.

Oh, and one thing that she is very adamant about is that I don't give her the code to do it. 
With that in mind, I plan to show the concepts in a different language, Elixir.

As my goal isn't to teach Elixir or any of its wonderful features, this is going to be some
quick and dirty code that explains some of the concepts of Functional Programming that can also
apply to the Object Oriented world.

So let's get started.

## Setup

I am writing this in [LiveBook](https://livebook.dev/). As such, I need to include `Kino` to
provide a method for inputing values to test and execute the APIs created. This section is
only if you want to play along at home with your own instance of `LiveBook`.

```elixir
Mix.install([{:kino, "~> 0.5.2"}])
```

## The TimeCalculator API

The first thing we'll need is the TimeCalculator API. The core functionality of this module
is to convert a `Map` structure of:

<!-- livebook:{"force_markdown":true} -->

```elixir
%{
  hours: 9,
  minutes: 41,
  seconds: 0
}
```

into seconds and back again.

As I said in the forward, this isn't an Elixir tutorial. But it would still be nice to
explain some of the code which may look unusual to those outside of Elixir.

First of all, a structure comprising of `hours`, `minutes`, and `seconds` is defined, with
zeros provided as the default values for each of the properties.

Next, the `calculate_seconds` function is defined, taking a `TimeCalculator` structure as its
only parameter and calculating the number of seconds based on the values within the structure.
I am aware that I'm not considering malformed data being passed in. Building out an API that
will get used by others will require that. But let's just concentrate on implementing the
"Happy Path" for now.

The second public function is `seconds_to_time`. This takes in the number of seconds to
convert into a `TimeCalculator` structure, extracting the number of hours, minutes, and
remaining seconds with the three private helper functions defined below.

```elixir
defmodule TimeCalculator do
  alias TimeCalculator, as: TC

  defstruct hours: 0, minutes: 0, seconds: 0

  def calculate_seconds(%TC{} = time) do
    time.seconds + time.minutes * 60 + time.hours * 60 * 60
  end

  def seconds_to_time(seconds) when is_integer(seconds) do
    %TC{
      hours: hours_in_seconds(seconds),
      minutes: minutes_in_seconds(seconds),
      seconds: remaining_seconds(seconds)
    }
  end

  defp hours_in_seconds(seconds) do
    floor(seconds / (60 * 60))
  end

  defp minutes_in_seconds(seconds) do
    floor((seconds - hours_in_seconds(seconds) * 60 * 60) / 60)
  end

  defp remaining_seconds(seconds) do
    seconds - hours_in_seconds(seconds) * 60 * 60 - minutes_in_seconds(seconds) * 60
  end
end
```

## Tests

Before going any further, I'd like to quickly run some tests to make sure that the 
calculations are working properly.

Notice that values can be negative within the `TimeCalculator` structure? I don't consider 
that to be bug but a feature. It's like saying "quarter to noon" can be expressed either 
`%{hours: 11, minutes: 45, seconds: 0}` or `%{hours: 12, minutes: -15, seconds: 0}`.

```elixir
alias TimeCalculator, as: TC

IO.puts("Test Results")

IO.puts(
  "is 1 hour, 3 minutes, 45 seconds = 3,825 seconds? #{TC.calculate_seconds(%TC{hours: 1, minutes: 3, seconds: 45})}"
)

IO.puts(
  "is 0 hours, 59 minutes, 59 seconds = 3,599 seconds? #{TC.calculate_seconds(%TC{hours: 0, minutes: 59, seconds: 59})}"
)

IO.puts(
  "is 2 hours, 91 minutes, -60 seconds = 3.5 hours = 12,600 seconds? #{TC.calculate_seconds(%TC{hours: 2, minutes: 91, seconds: -60})}"
)

IO.puts(
  "is 3,825 second = %{hours: 1, minutes: 3, seconds: 45}? #{inspect(TC.seconds_to_time(3_825))}"
)

IO.puts(
  "is 3,599 seconds = %{hours: 0, minutes: 59, seconds: 59}? #{inspect(TC.seconds_to_time(3_599))}"
)

IO.puts(
  "is 12,600 seconds = %{hours: 3, minutes: 30, seconds: 0}? #{inspect(TC.seconds_to_time(12_600))}"
)
```

The output should look something like this:

<!-- livebook:{"force_markdown":true} -->

```elixir
Test Results
is 1 hour, 3 minutes, 45 seconds = 3,825 seconds? 3825
is 0 hours, 59 minutes, 59 seconds = 3,599 seconds? 3599
is 2 hours, 91 minutes, -60 seconds = 3.5 hours = 12,600 seconds? 12600
is 3,825 second = %{hours: 1, minutes: 3, seconds: 45}? %TimeCalculator{hours: 1, minutes: 3, seconds: 45}
is 3,599 seconds = %{hours: 0, minutes: 59, seconds: 59}? %TimeCalculator{hours: 0, minutes: 59, seconds: 59}
is 12,600 seconds = %{hours: 3, minutes: 30, seconds: 0}? %TimeCalculator{hours: 3, minutes: 30, seconds: 0}
```

## User Input

The test cases above all appear to work. Excellent.

But I don't want to hard wire the user input. This API (or a similar one) needs to be called
from a web application. So let's create a form with a `textarea` and a `submit` button.

```elixir
inputs = [
  times: Kino.Input.textarea("Enter times, one per line:")
]

form = Kino.Control.form(inputs, submit: "Calculate")
```

## Result Output

Now that we have a method to input times, we need a place for the output to go.

```elixir
frame = Kino.Frame.new()
```

## Controller

Next we need a way to take the inputs and display the outputs. This is our controller function
which executes when the user clicks on the `Calculate` button.

Let's start with taking in the value of the `textarea`, breaking it up into multiple lines,
then `reduce` each line into a sum of the number of seconds each line represents. (The `reduce`
function in JavaScript is very similar to Elixir's `Enum.reduce` function. If you are not
familiar with the function, please see 
[Array.prototype.reduce](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce)
[developer.mozilla.org] for a full description.)

```elixir
Kino.Frame.render(frame, "Welcome to time calculator via clock")

for %{data: data} <- Kino.Control.stream(form) do
  %{times: times} = data

  result =
    times
    |> String.split("\n")
    |> Enum.reduce(0, fn time, secs -> secs + TC.clock_to_seconds(time) end)
    |> TC.seconds_to_time()

  Kino.Frame.render(frame, result)
end
```

## But First

If you try to calculate at this point, you're going to get an error. The `TC.clock_to_seconds`
function hasn't yet been defined.

Within the above parser, each line should hold an amount of time in the `H:MM:SS` format. 
Well, let's not be so strict with the format. Leaving out leading zeros will be fine. Let's
add the missing `clock_to_seconds` function to our `TimeCalculator` module. This will first
convert the passed clock formatted string to our time structure, then call our 
`calculate_seconds` function with that struct to get the number of seconds the time represents.

<!-- livebook:{"force_markdown":true} -->

```elixir
  @doc "Convert a string in HH:MM:SS format to seconds"
  def clock_to_seconds(time) do
    time
    |> clock_to_time()
    |> calculate_seconds()
  end
```

Okay. Now we need to define `clock_to_time`. As stated above, it's in charge of converting the
clock formatted string to our `TimeCalculator` struct which can then get passed to our
`calculate_seconds` function. Let's add this function as well.

<!-- livebook:{"force_markdown":true} -->

```elixir
  @doc "Convert a string in HH:MM:SS format to `TimeCalculator` struct"
  def clock_to_time(time) do
    tokens = String.split(time, ":")
    %TC{
      hours: String.to_integer(Enum.at(tokens, 0)),
      minutes: String.to_integer(Enum.at(tokens, 1)),
      seconds: String.to_integer(Enum.at(tokens, 2))
    }
  end
```

If you're playing along at home, run each part of this LiveBook document from the module
definition to the controller. Once that is done, enter `1:01:01` in the `textarea` above 
and click `Calculate`. The result in the `Result Output` section above should read:

<!-- livebook:{"force_markdown":true} -->

```elixir
%TimeCalculator{hours: 1, minutes: 1, seconds: 1}
```

Success!

Now let's add 25 seconds to that by entering `0:00:25` on the next line and re-calculating.

The new result is:

<!-- livebook:{"force_markdown":true} -->

```elixir
%TimeCalculator{hours: 1, minutes: 1, seconds: 26}
```

Go ahead and enter more lines, confirming the result as you go along.

## Different Inputs

Let's say that we want to enter the hours, minutes, and seconds in a comma separated list.
That's very simple.

First, comment out the `clock_to_seconds` reduction and add one that reduces with the
`csv_to_seconds` API helper function instead:

<!-- livebook:{"force_markdown":true} -->

```elixir
#  |> Enum.reduce(0, fn time, secs -> secs + TC.clock_to_seconds(time) end)
  |> Enum.reduce(0, fn time, secs -> secs + TC.csv_to_seconds(time) end)
```

Naturally, we're going to need to implement the two helper functions that will handle 
comma separated values instead of colon separated values:

<!-- livebook:{"force_markdown":true} -->

```elixir
  @doc "Convert a string in HH,MM,SS format to seconds"
  def csv_to_seconds(time) do
    time
    |> csv_to_time()
    |> calculate_seconds()
  end

  @doc "Convert a string in H,M,S format to `TimeCalculator` struct"
  def csv_to_time(line) do
    tokens = String.split(line, ",")
    %TC{
      hours: String.to_integer(Enum.at(tokens, 0)),
      minutes: String.to_integer(Enum.at(tokens, 1)),
      seconds: String.to_integer(Enum.at(tokens, 2))
    }
  end
```

Reevaluate each block similarly to above, enter `1,1,1` in the form and click `Calculate`.

This results in:

<!-- livebook:{"force_markdown":true} -->

```elixir
%TimeCalculator{hours: 1, minutes: 1, seconds: 1}
```

Success!

What happens if we enter a negative number? Try `0,-30,0` on the next line and recalculate.

<!-- livebook:{"force_markdown":true} -->

```elixir
%TimeCalculator{hours: 0, minutes: 31, seconds: 1}
```

It subtracted 30 seconds! This feels more comfortable than 0:-30:00 would have, but that
would have worked as well.

## Conclusion

I hope that this shows the power of extracting the functionality of doing time calculations
from the main program and modularizing them.

To do this in JavaScript, you're going to need to remember the `static` keyword when building
a `class`. As a reminder, here is how to create a purely functional class:

```javascript
class TimeCalculator {
  static calculateSeconds(time) {
    let seconds = 0;
    /* Do calculations here ... */
    return seconds;
  }

  static secondsToTime(seconds) {
    /* Calculate hours, minutes, seconds here ... */
    return {
      hours: hours,
      minutes: minutes,
      seconds: seconds
    }
  }
}
```

Your class should have **NO** properties in it. Only `static` functions. That way you can
access it with `TimeCalculator.calculateSeconds({hours: 1, minutes: 1, seconds: 1})` and
the like.

Once you have this API "module" setup, turn to your event handlers. These are your 
`Controllers`. They know the layout of your page and can gather the information, run
a `reducer`, and output the results back to the page.

These are the lines of division of labor. API knows how to do the business logic of
transforming time structures. Controllers are the middle-men between what is on the page
and what needs processing.

I hope that this helps to bring order to your code, so that it may be more easily 
maintained and expanded on in the future.
