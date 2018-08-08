# Writeup of Challenge 8 of SecTalks 0x00BER

by Nils Magnus and Kester Habermann

Some credits go to Malte Wirz, who supported us on the preceding challenges.

## Outer environment

The [starting point of the
challenge](https://0x00.randomcompanyna.me/base-64-is-great.html) was
the result of seven previous CTF-challenges.

It consists of a single website with a single block of JavaScript
inside a `<script>` container that is invoked directly when the page
is loaded. All other elements have no meaning (well, the image
contains a hint to the solution, but that hint is not really useful in
the resolving process).

There are a number of definitions of data structures and functions,
but control flow starts at the very end with

```javascript
alert(a(b)(prompt()))
```

The builtin function `prompt()` creates the dialog popping up after
loading the page, asking for a string. This string is fed through a
number of complex mechanics to the parameter `c` of the function
`b()`. That is where the control flow actually starts.

If you want to run through the single steps explained in this writeup,
just load the website mentioned above (or a local copy), enter an
arbitrary text in the pop-up dialog (most probably resulting in a
`false`) and then hit Ctrl+Shift-I (on Firefox) or F12 (on Chrome).
Click on `Console` and you are ready to type JavaScript code. All
mentioned data structures are initialized by then.

## First review of the code

The code is obviously written in (modern) JavaScript and uses some
really nasty twists provided by the language.

```javascript
String.prototype._=function(i){return this.charCodeAt(i)};
Array.prototype._=function(){return this.join("")};
Number.prototype._=function(){return String.fromCharCode(this)};
let r=r=>Array.apply([],{length:r}).map(Function.call,_=>(1&Math.random()*10^0|0)._())._(),
    x=(a,b)=>{for(_=[],i=0,j=a.length,k=b.length,l=j^((j^k)&-(j<k));i<l;_.push((a._(i%j)^b._(i++%k))._())){};return _._()},
    s=[44,49,119,107,11,127,37,48,78,120,114,39,101,56,126,55,98,47,57,99,103,120].map(_=>_._())._(),
    z=[r(22),(''+x).slice(42,64),s],b=b=>c=>b==z.reduce(x,c),a=a=>a(a);
alert(a(b)(prompt()))
```
Let's run through the code quickly line by line, just to understand what we have here:

The first three lines add new methods to the existing data types
`String`, `Array`, and `Number`. The new functions have all the same
name `_()`. Except for `String` they take no arguments and just
convert the data structure into an "natural" representation of some
other type: An `Array` of characters is transformed into a
string. `["f", "o", "o"]._()` results in just `"foo"`. A `Number` is
transformed into a character (actually a one character string) based
on its Unicode value. `(65)._()` results in `"A"`. `String`s implement
the reverse function of the latter at the provided position (started
at 0). Thus `"dragon"._(3)` results in 103, the Unicode number for
`g`.

The next block stared with the single `let` defines four functions and
two static values. For clearity let's divide the parts into single
statements by breaking up by comma:

```javascript
let r=r=>Array.apply([],{length:r}).map(Function.call,_=>(1&Math.random()*10^0|0)._())._();

let x=(a,b)=>{for(_=[],i=0,j=a.length,k=b.length,l=j^((j^k)&-(j<k));i<l;_.push((a._(i%j)^b._(i++%k))._())){};return _._()};

let s=[44,49,119,107,11,127,37,48,78,120,114,39,101,56,126,55,98,47,57,99,103,120].map(_=>_._())._();

let z=[r(22),(''+x).slice(42,64),s];

let b=b=>c=>b==z.reduce(x,c);

let a=a=>a(a);
```

The `let` keyword defines a local scoped variable (as apposed to the
global `var`), but that is not important here. You could as well
completely delete the `let` keywords altogether.

The code uses a lot of so-called arrow operators to do function
definition. In JavaScript functions are more or less anonymous lambda
closures assigned to a name. Another trick is that the code uses more
than once parameters that have the same name as the function itself.
To distinguish that, we renamed some of them if it looked ambigous.
That said, you can transform the
definitions of `r()`, `x()`, `b()`, and `a()` into more readable
equivalents:

```javascript
r = function(r_arg) {
    Array.apply([],{length:r_arg}).map(Function.call,_=>(1&Math.random()*10^0|0)._())._();
}

x = function(a, b) {
    for(_=[],i=0,j=a.length,k=b.length,l=j^((j^k)&-(j<k));i<l;_.push((a._(i%j)^b._(i++%k))._())){};return _._()
};

b = function(b_arg) {
    c=>b_arg==z.reduce(x,c);
}

a = function(a_arg) {
    a_arg(a_arg);
}
```

## The functions

Now that we've divided up the single parts, let's have a look to each
of the function and find out what they are doing:

### Creating random strings: `r()` 
*Function `r()`* creates an strings of random bits. Don't be confused
by the double use of the name `r`: It's both the name of the function
as well as its only parameter inside the function. The parameter holds
the length of the array to be created.

The first part `Array.apply([],{length:r})` creates an empty array
`[]` and then changes the `length` attribute of this very object. It
is thus the equivalent of just `new Array(r)`.

After this, the method `map()` applies some code on each empty element
of the array and collects the results in a new array of the same size.

The code calculates first a random bit of either 0 or 1. The code
`1&Math.random()*10^0|0` is just a lot of obfuscating code equivalent
of `Math.floor(Math.random() * 2)`: It creates a random float number
between 0 and 1, multiplies it by 10, gets rid of the float part by
xoring it with 0, chops of all but the least significant bit by `&1`
and does a redundant `|0` which maps each value on itself.

The random bit is then transformed into Unicode characters, collected
in an array and finally into a string. Note that this strings is not
really printable, since it just consists of Unicode codepoints of 0x0000
and 0x0001. These strings have some importance later on.

So an alternative notation of `r()` is

```javascript
r = function(len) {
    arr = [];
    for (i = 0; i < len; i++) {
        arr[i] = String.fromCharCode(Math.random() * 2); 
    }
    return arr.join("");
}
```

### Applying the crypto: `x()`
The function `x(a, b)` is the heart of the actual "encryption", taking
two strings and creating a new one. There's actually some code to deal
with strings of arbitrary length, but it's easier to think of each
string being exactly 22 characters long. We will see later on that
this a reasonable assumption.

Let us defuscate the function first a bit: We can safely extract most
of the initializing values out of the for-loop and move code of the
increment part into the body of the loop (leaving the `i++` part in
the increment):

```javascript
x = function(a, b) {
    _ = [];
    j = a.length;
    k = b.length;
    l = j^((j^k)&-(j<k));
    for (i = 0; i < l; i++) {
        _.push((a._(i%j)^b._(i%k))._()))
    };
    return _._()
};
```

Now with the better readable code, we see that `j` and `k` are the
sizes of the parameter strings. `l` is just the maximum of both
notated in a obfuscated way. But as all strings are 22 characters each
anyway, it's ok to think of `l = k = j = 22;` Now, we iterate `i` from
0 to 21 and build up the result array `_` that starts empty, element
by element.

As we assume both `j` and `k` to be 22 and `i` is always smaller than
22, we can get rid of the moduloes `%j` and `%k` without any
harm. Actually each character at every position is calculated as an
exclusive-or of the corresponding characters of the input
parameters. Once the calculation is done on the numbers, everything is
marshalled again into an 22-element array and finally again into a
22-character string.

Observe that this function `x(a, b)` has a number of properties that
are important for the further solution:

1. The function is commutative, meaning that `x(a, b) == x(b, a)`.

2. The function is a involution, meaning that if you apply the the
  result of `x(a, b)` again to one of its arguments (regardless
  which), you can retrieve the other original value: `x(a, x(a, b)) ==
  b` and `x(x(a, b), b) == a`. Another easier example of an involution
  is ROT13: Twice applied on a cleartext it results again in the
  cleartext.

3. The function behaves like what we call "local": If you modify just a
  single character at position `n` in either `a` or `b` inputs, the
  result differs only in position `n` as well. No other characters are
  modified.

The function `b()` defines the way the crypto function `x(a, b)` is
called. It feeds by means of the method `reduce()` three 22-char
strings of the array `z` (we get later to that) and the parameter
`c` to `x()` and compare the result to another 22-char string `b`
(not to be confused with the name of the function, which is also `b`).

Effectively this results in this code:

```javascript
b = function(input) {
    accu = x(input, z[0]);
    accu = x(accu,  z[1]);
    accu = x(accu,  z[2]);
    if (accu == b) {
       return true;
    } else {
       return false;
    }
}
```

The overall aim is to create a `true` here by feeding the correct
input into the function. Please observe that the result is also
depending on the global three-element array `z[]`. Later we will
transform this function once more to reverse it, but first let's check
the final function.

### Obfuscating the invocation: `a()` and `b()`
Function `a()` is a function that returns a function itself, in our
case the previous function `b()`. 

## The static values

Besides the four functions there are also three (kind of) static
values `s`, `z` and `b` defined:

```javascript
s=[44,49,119,107,11,127,37,48,78,120,114,39,101,56,126,55,98,47,57,99,103,120].map(_=>_._())._();
```

This just creates a 22-char string based on the numbers, each
converted by `map()` into chars and finally into a single string by
the `_()` methods. `s` does not depend on any other input and is thus
static.

The second static value consists actally of three values, packed into
an array of three 22-char strings:

```javascript
    z=[r(22), ('' + x).slice(42, 64), s]
```

The first element `z[0]` is created by function `r()` we analyzed
before. It contains a 22-character string of bits. Please observe that
this string is (pseudo) random and changes at each invocation.

The second element `z[1]` takes the function `x(a, b)` and casts it
into a string data type, resulting in a textual representation of the
founction source code itself. **This is one of the craziest effects of
a programming language I have seen in a while!** Please note that its
mandatory to use the original definition of the function `x(a, b)` to
solve the challenge, not the simplified version we developed above as
the source code differs. As we need exactly a 22-char string, we
`slice()` a piece from position 42 to 64 out of it.

The third element `z[2]` we discussed already and is just the static
string `s`.

Finally there's the string `b` which is in fact the function `b()` we
discussed before, but since it is within this very function evaluated
in a boolean context and compared with another string, `b` is just the
complete source definition of the function which also appears to
consist magically of 22 characters!

## The decoding of the secret passphrase

Now with all basic dataflow, functions, and values in place, we're now
up to reverse the mechanism to obtain the secret passphrase. Let's
first recapture the data flow, though:

We take the input (22 chars), mix it with the fixed string `s`, mix it
with parts of `x()` which is also fixed, mix it with a random string
of bits and compare the result with another fixed string made out of
`b()`.

Let's assume for a moment we know the "right" random bit-string we
create with `r(22)`. We mix all values including the input together
with `x()` and compare the result to `b`. But taking into account the
properties of `x()` we observed ealier, we could also mix together `z`
and `b` and compare against the passphrase, since we are allowed to
reorder the application of `x()`.

Let's assume further for a second that all bits of `r(22)` are
zero. That renders the application of `r(22)` unnecessary and the
following holds:

```javascript
x(a, r_all_zero) == a
```

With that in mind let's calculate what we call the zero-result (all
variables are just the way they are after being executed once):

```javascript
z[0] = "\000".repeat(22);
z.reduce(x, b)
```

We call the result `"b``tbhiog-ius-tahl.htmm"` of that __zero-res__.

In another turn, let's now assume that all bits in `r(22)` are
set. That means that all characters are shifted by one Unicode
value. Previously even values get incremented by one, odd ones get
decremented by one. We call that result __ones-res__:
`"caucihnf,htr,u```im/iull"`.

We further know that the passphrase contains only lowercase letters,
digits, dots, or dashes and probably ends in `.html`. So here is a
table of potential values at all of the 22 slots:

Position | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 | 21
---------|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---
zero-res | b | \` | t | b | h | i | o | g | - | i | u | s | - | t | a | h | l | . | h | t | m | m
ones-res | c | a | u | c | i | h | n | f | , | h | t | r | , | u | \` | i | m | / | i | u | l | l

We can now decide row by row which character to take. Some of them are
not allowed (striked out), so the other option must be the valid
option (bold). We also know the ending extension ".html" already.

Position | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 | 21
---------|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---
zero-res | b | ~~\´~~ | t | b | h | i | o | g | **-** | i | u | s | **-** | t | **a** | h | l | **.** | **h** | **t** | **m** | m
ones-res | c | **a** | u | c | i | h | n | f | ~~,~~ | h | t | r | ~~,~~ | u | ~~\`~~ | i | m | ~~/~~ | i | u | l | **l**

To repeat: The values are generated by

```javascript
["\000".repeat(22), z[1], s].reduce(x, b+'')
"b\`tbhiog-ius-tahl.htmm"
["\001".repeat(22), z[1], s].reduce(x, b+'')
"caucihnf,htr,u\`im/iull"
```
That eliminates 9 out of 22 bits, leaving 13 bits open. Even though
there are thus now 2^13 = 8192 possible options left, in practise it
should be quite easy to figure out the original passphrase, which
reads [`catching-its-tail.html`](http://0x00.randomcompanyna.me/catching-its-tail.htm).
