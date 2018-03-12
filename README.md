[![Build Status](https://travis-ci.org/dchudz/rewritecov.svg?branch=master)](https://travis-ci.org/dchudz/rewritecov)

This package extends the notion of 'test coverage' by looking for modifications
it can make without causing errors, which may indicate a gap in testing.

Note that since it runs your code modified, you should be pretty careful
with it! (More on that below.)

## Details

`rewritecov` uses Python's `ast` library to parse your code and then
try running it with various modifications. For any modifications that
don't fail, it reports a gap in coverage.

Currently the modifications it tries are:

- delete a node
- replace function calls with `None`

Once it has tries a given type of modification, it never tries the same
type of modification on a node with the same line number. (This is too
restrictive. We might as well let it try different nodes on the same
line!)

`rewritecov` stops trying rewrites if it ever sees a functon called
`test_<anything>`. This signals that we've reached the "testing" section
of the code, where checking coverage doesn't make sense like it does for
the "implementation" section.


## Motivation

One time I had to work on a package that was mostly a bunch of
regexes. That was sad for many reasons, but one of them was that
codecov reported 90% coverage with hardly any testing.

The way it was structured, if you ran any of the functions, you
ran all of the lines in that function. The fact that the regexes executed
did nothing to tell us that it even mattered if they were there. I
didn't add much testing, but when I was done we still had around
"90% coverage".


## Usage and Example

You can install it with:

 ```
 pip install git+git://github.com/dchudz/rewritecov.git
 ```

Then given this code file `example.py`:


```
import re


def munge_some_text(text):
    replacements = [
        (r'[\s+]', r'_'),
        (r'[^a-zA-Z0-9_]', r'')
    ]
    for old, new in replacements:
        text = re.sub(old, new, text)
    return text


def test_it():
    munge_some_text('hi there!')


test_it()
```

... we can run:

```
$rewritecov example.py
```

```
Lines with stuff we could delete: 6, 7, 9, 11
Lines with function calls we could replace with None: 10
```

Running `rewritecov example.py --verbose` shows much more info about
what's going on along the way.


## Careful!

Given that `rewritecov` runs your code after modifications, it could
have very unpleasant consequences. The first time I tried it with
CPython 2.7, it segfaulted. But it could do much worse to you,
depending on the code you try it with!

## Improvements

This is probably just a toy that's going nowhere, but I still might
work on it a bit, especially if other people are interested.

- The main limitation is maybe that `rewritecov` just operates on a single
source file and isn't integrated at all with any standard testing
frameworks like `pytest`.

- I also haven't thought much about safety. It's possible this thing
should only ever be run in a container that you don't mind destroying.

- It'd be nice to show the actual code that's uncovered and/or report
coverage in a standard format that integrates with coverage-viewing tools.

- It'd be nice to consider more than just one node per line - for example,
adding `col_offset` to our keys in addition to `lineno` would be an
easy step in that direction.

- At a larger scale, it would be pretty inefficient! But I suspect that
doesn't have to be as bad as it sounds. For example, we run something like
this individually for each test after using standard `codecov` to
determine which lines are even hit at all. We wouldn't bother rewriting
lines that aren't hit anyway.

- We could also keep track of which nodes/lines are "covered"
by previous tests, and not bother rewriting anything we already know is
important to the tests.

- More rewrite rules!
