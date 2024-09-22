+++
title = "Code Coverage Isn't Everything"
slug = "code-coverage-isnt-everything"
aliases = ['/articles/code-coverage-isnt-everything']
date = 2016-04-16
draft = false
tags = ["testing"]
+++

Over the past few months, the people within my circle of friends (myself included) have started to put more of an emphasis on increasing the code coverage for their own projects. I'd like to take a moment here how a code coverage metric is important, but it should not be the only goal when writing unit tests.

For those that are unaware, code coverage is simply a metric that measures the percentage of lines of code that are executed when your unit test framework is run. You can see an example coverage report at the [Codecov page for InstrumentKit](https://app.codecov.io/gh/instrumentkit/InstrumentKit).

Coverage is a great indicator that you are heading in the correct direction with your tests. After all, if a line of code is ran once in a test environment, that's better than zero! The problem arises when one assumes that is sufficient. The idea is that unit tests should capture all the behaviour of your code, not just touch each line once.

Alright, lets look at an example of what I'm talking about. A great example is a function that uses ternary expressions (basic ``if`` statements compressed into a single line).

```python
def foobar(value):
    return value.lower() if isinstance(value, str) else value
```

All this function is doing is casting the input to lowercase if the input is a string, else it just returns the input. Not the most useful function in the world, but a good demonstration.

Now lets say our corresponding tests look like this:

```python
def test_foobar():
    assert foobar(123) == 123
```


So now if we were to check our code coverage, we'd see that we have 100%! Wow great job! But wait...we haven't actually captured all the behaviour. Someone in the future could change our ``foobar`` function to call ``str.upper()``, or not do anything with strings at all. We need some more tests.

```python
def test_foobar_integer():
    assert foobar(123) == 123

def test_foobar_lower_case_string():
    assert foobar("abc") == "abc"

def test_foobar_casts_to_lower():
    assert foobar("AbCd") == "abcd"
```

There we go, much better. Even though our code coverage hasn't changed, our tests are in a much better shape. Naturally we can keep adding more (what's the desired behaviour with lists of strings? What about byte strings?) but to show my point this is sufficient.

Now, this would have been much less of a problem if we had followed Test Driven Development (basically write tests before you write your implementation, and yes this is a huge oversimplification of TDD). But because we're trying to add tests to an existing project after its already been written, we don't get that privilege.
