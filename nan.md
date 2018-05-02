For [Mask](https://github.com/scizzorz/mask), I only have plans for one
compound data type, a hash table, which should operate in a familiar way to
most dynamic languages with a similar construct: dicts in Python, objects in
JS, tables in Lua, etc. In all of these languages, you can index into the
construct using floats. This is slightly more complicated than I expected.

Hash tables typically use an equality test to pull out the right value.

The float magic value `NaN` drops the reflexive property, which means
`NaN == NaN` is false.

So what happens when `NaN` is used as a key?

Python
------

    #!python
    >>> nan = float('nan')
    >>> d = {}
    >>> d[nan] = 'banana'
    >>> d
    {nan: 'banana'}
    >>> d[nan]
    'banana'
    >>> nan == nan
    False

Somehow, you can correctly insert and get a value out of a dict using a `NaN`
key, despite `NaN` not being equal to itself. However...

    #!python
    >>> nan2 = float('nan')
    >>> d[nan2]
    KeyError: nan

If you use a *different* `NaN`, it doesn't work! The secret is that Python
actually uses an identity check *before* an equality check.

    #!python
    >>> nan is nan
    True
    >>> nan2 is nan
    False

Because `nan` is the same object as `nan`, the identity check passes and the
dict lookup succeeds. Because `nan2` is a different object, the identity check
fails, and lookup moves to the equality check. Because `NaN` is not equal to
itself, the equality check also fails, and the lookup aborts with a `KeyError`.
This means that we can use *both* `NaN` objects as keys successfully.

    #!python
    >>> d[nan2] = 'apple'
    >>> d
    {nan: 'banana', nan: 'apple'}
    >>> d[nan]
    'banana'
    >>> d[nan2]
    'apple'

Neat, I guess, but it's inconsistent with my expectations: either `NaN` keys
are impossible to retrieve from a dict, or all `NaN` keys resolve to the same
thing.

JavaScript
----------

    #!javascript
    >>> nan = 0/0
    >>> d = {}
    >>> d[nan] = 'banana'
    >>> d
    {NaN: 'banana'}
    >>> d[nan]
    'banana'
    >>> nan === nan
    false

Again, somehow you can correctly insert and get a value out of an object using
a `NaN` key. However...

    #!javascript
    >>> nan2 = 0/0
    >>> d[nan2]
    'banana'

This time, different `NaN`s both index to the same value! This is because
JavaScript likes to stringify its keys before hashing or checking anything, so
both of these `NaN`s get turned into `'NaN'`:

    #!javascript
    >>> d['NaN']
    'banana'

Less neat, but it's more intuitive than Python's implementation, which makes it
less likely to cause hard-to-find bugs.

Lua
---

    #!lua
    >>> nan = 0/0
    >>> d = {}
    >>> d[nan] = 'banana'
    table index is NaN

Lua throws an error. That's okay too.

Mask
----

I haven't quite decided yet, but I'm leaning towards using the
[ordered-float](https://rust-bio.github.io/rust-bio/ordered_float/struct.OrderedFloat.html)
crate, which contradicts the IEEE standard and adds the reflexive property back
into `NaN`s. This would provide similar behavior to JS regarding `NaN`, but
would avoid some of the nasty consequences of stringifying all keys.

