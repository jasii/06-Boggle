# Boggle solver HOWTO

[Boggle](https://en.wikipedia.org/wiki/Boggle) is a word game
invented in 1972. Players compete to see how many words they can 
form from the letters displayed on the top of 16 dice in a 4x4 
pattern. Variations with 3x3 ("Boggle junior"), 5x5, and 6x6 
("Super Big Boggle") grids exist, 
but we will build a solver for the original 4x4 version sometimes 
called "Classic Boggle".  

![Sample board](img/sample-board.jpeg)

Our solver will make one simplification:  Modern Boggle dice 
include one face with the letter pair "QU".  Like earlier Boggle 
games, we will treat "Q" as an ordinary letter. 

## Interaction

Input to our Boggle solver will a sequence of 16 letters, which we 
will treat as four rows of four letters.  The output will be a list 
of words that can be formed from those letters according to Boggle 
rules, and a score.  The interaction will look like this: 

```commandline
Boggle board letters (or 'return' to exit)> oydliexennoktati
['ANENT', 'ANNEX', 'ANT', 'ANTI', 'ATONE', 'DEN', 'DENT', 'DYE', 
'EON', 'IKON', 'INANE', 'INN', 'INTO', 'ION', 'IOTA', 'KIT', 'LED', 
'LEONINE', 'NATION', 'NEON', 'NINE', 'NOEL', 'NOT', 'OAT', 'ONE', 
'OXEN', 'TAN', 'TANNED', 'TAT', 'TIKE', 'TOE', 'TOED', 'TOKE', 'TON',
'TONE', 'YEN', 'YIN']
49 points
```

Boggle rules allow a word to be formed by any path through the grid, 
including diagonal as well as horizontal and vertical moves, but 
each die may be used only once in each word.  For example, here is 
how the word "LEONINE" is found in the play above: 

![LEONINE path in "oydliexennoktati"](img/sample-path.jpg)

In addition to the textual interaction, our Boggle solver will 
display its progress graphically.  

![Snapshot of boggler display](img/play-snapshot.png)

## Two-stage project

This project has several moving parts, so we will complete it in two 
stages.  The first stage will mainly concern quickly determining 
whether a sequence of characters is a word, or potentially the first 
part of a word (a _prefix_), or whether no word can be created by 
extending that sequence of characters.  In addition we will build 
some other support functions for the Boggle solver. 

In the second stage, we will create the actual Boggle solver,
using a recursive depth-first search.  This will be similar to the 
search we used to fill cavern chambers with water, but with some 
extra bookkeeping because a single die may be used only once in a 
single word but multiple times to find different words.

## Stage 1:  Quickly searching a word list

Our main objective in the first stage of the project is to very 
quickly determine whether a sequence of letters like _ANT_ is a word 
or a prefix of a word.  As we have before, we will base our 
judgments on a provided word list, which we will keep in
`data/dict.txt`.  Unlike prior projects, though, we will be looking 
up _many_ strings in the same word list.  Instead of checking words 
as we read the word list, we will read it once and store it in 
memory to use many times.

## Getting a start

We will start in the usual way, creating a program file called 
`boggler.py`.  

```python
"""Boggler:  Boggle game solver. CS 210, Fall 2022.
Your name here.
Credits: TBD
"""
import doctest

def main():
    pass

if __name__ == "__main__":
    doctest.testmod()
    print("Doctests complete")
    main()
```

Again we will need access to an external word list file, and we 
don't want to hard-code the file path in our program, so we'll 
create a separate `config.py` file with the file path: 

```python
""""Configuration of Boggle player"""

# List of words to search for
DICT_PATH = "data/dict.txt"
```

Import `config` in `boggler.py` the same way we have imported 
configuration paths in prior projects. 

Internally the word list will be a list of strings.  We will create 
a function to obtain that list of strings from an external word list 
file like `dict.txt`. 

### Reading the word list

```python
def read_dict(path: str) -> list[str]:
    """Returns ordered list of valid, normalized words from dictionary.

    >>> read_dict("data/shortdict.txt")
    ['ALPHA', 'BETA', 'DELTA', 'GAMMA', 'OMEGA']
    """
```

There are a few things to notice about `read_dict`.  Here is the 
actual content of `shortdict.txt`:

```commandline
BETA
alpha
omega
big-time
dEltA
gamma
is
```

### Filtering the word list

The list of words returned by `read_dict` is _filtered_ to contain 
only words that are valid in Boggle:  They must contain at least 
three letters, and they must contain only letters.  For this we 
will write a short supporting function `allowed`:

```python
def allowed(s: str) -> bool:
    """Is s a legal Boggle word?

    >>> allowed("am")  ## Too short
    False

    >>> allowed("de novo")  ## Non-alphabetic
    False

    >>> allowed("about-face")  ## Non-alphabetic
    False
    """
```

Although we are solving only the classic version of Boggle for now, 
we want to avoid _magic numbers_ that would make it harder to adapt 
our program to other versions in the future.  Therefor the our 
function `allowed` should refer to a _symbolic constant_ `MIN_WORD` 
for the minimum allowed length of a Boggle word.  We will place the 
definition of `MIN_WORD` near the beginning of the program file, 
just after the `import`s.  

```python
# Boggle rules
MIN_WORD = 3   # A word must be at least 3 characters to count
```

To check that a word contains only letters, we can use the string 
method `isalpha`, e.g., `s.isapha()`.  

If `read_dict` includes only alphabetic strings of at least 3 
characters, it will exclude "is" and "big-time" from `shortdict.txt`. 

### Normalizing the word list

We have seen _normalization_ before, when we searched for anagrams 
in the jumbler project.  Normalization for Boggle is simpler --- we 
just need to decide whether to store words in upper-case or 
lower-case and use that normalization consistently.  Since 
traditional Boggle dice use upper-case letters, our normalization 
function will likewise use upper-case letters.  The string function 
`upper` can do this for us, so our `normalize` function can be
trivial: 

```python
def normalize(s: str) -> str:
    """Canonical for strings in dictionary or on board
    >>> normalize("filter")
    'FILTER'
    """
    return s.upper()
```

Despite its simplicity, having a single function `normalize` that we 
call whenever we need _any_ string in normal form helps us be 
consistent in our choice of normal form. 

With `allowed` and `normalize`, you have the pieces you need to 
complete the `read_dict` function.  You may want to refer to prior 
projects for a reminder of how to open a file, read each line from 
the file, and strip off the newline.  Sort the word list before 
returning it. 

## Checkpoint

At this point you should have three functions, `allowed`, 
`normalize`, and `read_dict`.  You also have a symbolic constant 
`MIN_WORD` as a global variable near the beginning of the program. 

## Searching the word list

Our Boggle solver will search the word list repeatedly as it traces 
paths in the Boggle board.  Consider the sample board above, 
starting with the "L" in the upper right-hand corner.  It will first 
search for "L", which is not a word but can be the start of many 
legal Boggle words.  Three directions from the "L" are outside the 
board, but "D", "X", and "E" are adjacent to the left, diagonally 
left and down, and below the "L".  When our solver searches for "LD",
the result should indicate that no words in the word list start with 
"LD", and likewise for "LX".  When it searches for "LE", the result 
should indicate that "LE" is not a legal Boggle word, but there are 
words that begin with "LE".  Eventually the solver will search for 
"LEONINE", and the result should be an indication that "LEONINE" is 
a valid Boggle word that can be added to the list of words found by 
solver. 

If there were only two possible outcomes of a search, we would make 
a search function that returned type `bool`.  Since there are three 
distinct outcomes, we need another approach.

We could define a new type to encode the three outcomes, and we will 
do that later when we study object-oriented programming in Python.  
For now we will take a simple but adequate approach of defining 
three distinct values to represent the outcomes.  Since we do _not_ 
want to sprinkle these special values around in the code as magic 
numbers or magic values, we will define them as symbolic constants 
near the beginning of the source code file (maybe right after the 
symbolic constant for the minimum length of a Boggle word): 

```python
# Possible search outcomes
NOPE = "Nope"       # Not a match, nor a prefix of a match
MATCH = "Match"     # Exact match to a valid word
PREFIX = "Prefix"   # Not an exact match, but a prefix (keep searching!)
```

In addition to make our code more readable, using these symbolic 
constants will prevent us from creating mysterious bugs with typos. 
An error message from Python complaining of an undefined variable 
(because we misspelled it) is vastly preferable to unanticipated 
program behavior caused by a typographical error. 

Now our search function can return a string, but it should only 
return one of the three special strings called `NOPE`, `MATCH`, or 
`PREFIX`. 

```python
def search(candidate: str, word_list: list[str]) -> str:
    """Determine whether candidate is a MATCH, a PREFIX of a match, or a big NOPE
    Note word list MUST be in sorted order.

    >>> search("ALPHA", ['ALPHA', 'BETA', 'GAMMA']) == MATCH
    True

    >>> search("BE", ['ALPHA', 'BETA', 'GAMMA']) == PREFIX
    True

    >>> search("FOX", ['ALPHA', 'BETA', 'GAMMA']) == NOPE
    True

    >>> search("ZZZZ", ['ALPHA', 'BETA', 'GAMMA']) == NOPE
    True
    """
```

### Make it fast! 

A _linear search_ function could check the candidate string against 
each element of the word list.  Our file `dict.txt` contains 39,391 
valid Boggle words.  Since we keep our word list in sorted 
(alphabetical) order, we could stop the search when we encounter a 
word that should come after the string we are searching for.  That 
would require, on average, a little less than 20,000 comparisons each 
time we called `search`.  But we can do much better. 

A _binary search_ can determine whether the candidate string is in 
our list of 39,391 words in 16 or fewer comparisons.  It does this 
by very rapidly discarding large parts of the word list, cutting it 
in half with each operation.  For example, when I add a print statement
to my `search` function, I can trace the search for "LEONINE":

```commandline
boggler_solution.search("LEONINE", words)
LEONINE must be between AARDVARK and ZYMURGY (39391 words)
LEONINE must be between AARDVARK and LIBERATE (19695 words)
LEONINE must be between DISPLAY and LIBERATE (9847 words)
LEONINE must be between GRAHAM and LIBERATE (4923 words)
LEONINE must be between INCAUTIOUS and LIBERATE (2461 words)
LEONINE must be between IRREDUCIBLE and LIBERATE (1230 words)
LEONINE must be between KNEE and LIBERATE (615 words)
LEONINE must be between LAUNCH and LIBERATE (307 words)
LEONINE must be between LEGALIZE and LIBERATE (153 words)
LEONINE must be between LEGALIZE and LEPROSY (76 words)
LEONINE must be between LEGUME and LEPROSY (38 words)
LEONINE must be between LENGTHINESS and LEPROSY (19 words)
LEONINE must be between LENT and LEPROSY (9 words)
LEONINE must be between LENT and LEOPARD (4 words)
LEONINE must be between LEONINE and LEOPARD (2 words)
'Match'
```

We can use _almost_ the standard binary search algorithm, as 
outlined in this pseudocode: 

- Initially, `low` is the index of the first element, and `high` is 
  the index of the last element in the list.
- While `low` ≤ `high`
  - pick an index `mid` roughly half-way between low and high
  - if the element at `mid` matches the candidate, we are done (MATCH)
  - if the element at `mid` is _before_ the candidate in 
    alphabetical order, we can eliminate all elements up to `mid` by 
    setting `low` to `mid + 1`
  - if the element at `mid` is _after_ the candidate in alphabetical 
    order, we can eliminate all elements after `mid` by setting 
    `high` to `mid - 1`
- If we finish the loop with `low` > `high`, the candidate was not 
  found in the list. 

If we set to `mid` to the mid-point between `low` and `high` each 
time through the loop, each iteration of the loop will either find 
the word or cut the portion of the list under consideration in half.
We see that above in the search for "LEONTINE", as the portion of 
the word list under consideration is cut from 39,391 words to 19,695 
words, then 9,847 words, then 4,923, 2,461, 1,230, 615, 307, 153, 76,
38, 19, 9, 4, and finally just 2 words.  It took just 14 comparisons 
to find "LEONTINE" in the list of 39,391 words.  

How do we know that the maximum number of loop iterations required 
is 16? Because 
$$2^{15} = 32,768 < 39,391 < 65,536 = 2^{16}$$
Each comparison cuts the number of elements we must consider in half,
and cutting a number between $2^{15}$ and $2^{16}$ in half 16 times 
will reduce it to less than 1.
The number of operations required for binary 
search is proportional to the logarithm base 2 of the length of the 
list to be searched. 

There is just one adjustment we must make to this standard algorithm.
When it finishes without finding an exact match, we must distinguish 
between a string that is a dead end (no use continuing to search 
after "LX") and a string that, while not a word, could be the prefix 
of a word (e.g., "LEO" as a possible prefix of "LEOPARD" and 
"LEONINE").  Python provides a useful string method `startswith` 
that we can use for this check. 

In sorted order, a prefix of a word comes before the 
full word (e.g., "LEO" < "LEONINE").  Since we have eliminated all 
words that would occur _before_ `low`, the current value of `low` is 
where a longer word starting with the candidate would appear.  
However, we have to be careful:  If we search for a candidate that 
would come at the very end of the word list, `low` could be equal to 
the length of the list.  Thus we should return the symbolic constant 
`PREFIX` only if `low` is less than the length of the word list 
_and_ the word at index `low` starts with the candidate we searched 
for.   

## Checkpoint

At this point you should have the following functions: 
- `allowed` determines whether a string is a legal Boggle word. The 
  argument to `allowed` does not have to be normalized. 
- `normalize` converts a string to the "normal form" that we will 
  use in our word list and each candidate word.  We are using upper 
  case as the normal form (but this was an arbitrary decision)
- `read_dict` reads a file containing a word list and returns a list 
  of valid Boggle words in normal form, and in sorted order. 
- `search` performs a binary search for normalized string in a 
  sorted list of normalized strings.  It returns a string, which 
  must be one of `NOPE`, `PREFIX`, or `MATCH`. 

You also have the following symbolic constants as global variables 
created near the beginning of your program file.

- `MIN_WORD` is the minimum length of a valid Boggle word, which 
  should be 3. 
- `NOPE` is a string we use as a search result indicating that a 
  candidate string is neither in our word list nor a prefix of any 
  word in our word list.
- `PREFIX` is a string we use as a search result indicating that the
  candidate string is not in our word list, but is a prefix of at 
  least one word in our word list.  It must not have the same value 
  as `NOPE`.  
- `MATCH` is a string we use as a search result indicating that the 
  candidate string is a word in our word list.  It must not have the 
  same value as `PREFIX` or `NOPE`. 

This concludes stage 1. 




