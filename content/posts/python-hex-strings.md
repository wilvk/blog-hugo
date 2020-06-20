+++
title = "Using Python for hex, strings, bytes and integers"
date = 2019-10-10T21:27:52+11:00
draft = false
tags = []
categories = []
+++

I've taken an interest recently in cryptography, strings, signing strings and using prime numbers for cryptography. I had to keep searching online for the best way to do a transform of my data for:

- interpreting
- modifying and
- appending

to my results. This was frustrating.

So, I'll start at the start an end at the end. It all starts and ends with strings; usually via arrays of bytes and strings of hex.. but how can we operate with and easily transform these in a sensible way?

In the case I was looking at, I would have a signature (say from a cookie) that was both base64 encoded and url quoted. This is done so that the binary representation can be sent across the wire (internet) without too much fuss.

I'd start by unquoting the url encoding on the string (to remove, for example the `%2D` type characters).

Going into the Python REPL and doing something like the following:

```python
>>> parse.unquote("AAA%2DBBB")
'AAA-BBB'
```

... returns a string. We can tell as there are single quotes around the result - with nothing else.

What if we wanted to base64 encode this string ('AAA-BBB')?

Trying the following fails:

```python
>>> import base64
>>> base64.b64encode('AAA-BBB')
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/Library/Frameworks/Python.framework/Versions/3.7/lib/python3.7/base64.py", line 58, in b64encode
    encoded = binascii.b2a_base64(s, newline=False)
TypeError: a bytes-like object is required, not 'str'
>>>
```

The reason being that the `b64encode` method expects a byte array to be present.

There are several ways to achieve this.

The easiest is to put a `b` out the front of our string to tell the interpreter that this string should be used as an array of bytes, like so:

```python
>>> base64.b64encode(b'AAA-BBB')
b'QUFBLUJCQg=='
```

Notice that the result is also an array of bytes, with the `b` at the front of the string.

To confirm this, we can get the type of the result and display it's name:

```python
>>> test = base64.b64encode(b'AAA-BBB')
>>> type(test).__name__
'bytes'
```

Using the `b` at the start is ok if we are using a string that has been hard-coded; but in most instances we would need to convert a string to a byte array programatically.

For this, there is the `.encode()` string method. As an example, consider the following:

```python
>>> test_string = 'AAA-BBB'
>>> test_bytes = test_string.encode()
>>> base64.b64encode(test_bytes)
b'QUFBLUJCQg=='
```

That works as expected. Great. But... *what if we need to convert back from a byte array to a string?* ...I hear you ask. Well, it turns out there's an accompanying `.decode()` string method just for that case.

```python
>>> test_bytes = b'QUFBLUJCQg=='
>>> test_string = test_bytes.decode()
>>> type(test_string).__name__
'str'
```

So now we're brushed-up on strings and bytes, let's delve into strings and byte arrays of hex values. This is where it gets a bit trickier.

If we start again with our original string `AAA-BBB` and we want to convert that to it's hexadecimal representation, it turns out there are a few ways to do this.

The simplest way is to call `.encode()` on the string, and then `.hex()` on the byte array. The result is a string of hexadecimal values.

```python
>>> test_string = 'AAA-BBB'
>>> hex_string = test_string.encode().hex()
>>> type(hex_string).__name__
'str'
>>> hex_string
'4141412d424242'
```

We can see here `41` repeated and `42` repeated as the ASCII values `A` and `B` respectively, with a `2d` in the middle, the hyphen (-). I find the hex-string representation of a string of bytes very useful for working with. Given the string is not too long, it is useful for inspecting and understanding any patterns in the bytes you are working with. When working with forms of cryptography that depend on numbers (and most does to some extent), this is especially useful.

Another way to get a hex-string is to iterate all the characters of the string, convert them to ordinals then convert them to hex strings then remove all the `0x`'s at the start and join them back together:

```python
>>> ''.join(hex(ord(c))[2:] for c in 'AAA-BBB')
'4141412d424242'
```

or as:

```python
>>> ''.join("{:x}".format(ord(c)) for c in 'AAA-BBB')
'4141412d424242'
```

This approach is less neat but more explicit of what is happening.

There is also a helper module in Python 3 called `hexlify` that takes a byte array and converts this to a string array:

```python
>>> import binascii
>>> binascii.hexlify(b'AAA-BBB')
b'4141412d424242'
>>> binascii.hexlify(b'AAA-BBB').decode()
'4141412d424242'
```

and it's compliment `unhexlify`, which takes a string, not a byte array, and outputs a byte array.

```python
>>> binascii.unhexlify('4141412d424242')
b'AAA-BBB'
>>> binascii.unhexlify('4141412d424242').decode()
'AAA-BBB'
```

If we want to go the other way, and convert a hex-string or byte array to its string representation there are also many ways to do this. I'll quickly go through these ways:

given a hex string:

```python
hex_string = '1234ABCD'
byte_array = b'1234ABCD'
```

From a byte array to string, we can use `byte_array.decode()`

From a string of hex to a byte array of characters, we can use `bytes.fromhex(hex_string)`

From a string of hex to a string of characters, we can use `bytes.fromhex(hex_string).decode()`

Note that the last `.decode()` will fail as there are bytes that cannot be converted back to a character representation.

A string representation of a series of hex values may not be printable, and may not even convert neatly to an encoding such a `utf-8`. In many cases, this is not an issue as the string is a binary representation of something else anyway and is not dissimilar to a fixed-length array of characters without a null-terminator - but I digress. This is partly why base64 is commonly used across the wire.

## Conversion between hex and integers for calculation of long numbers.

As a final step, we may want to use this hex string representation as an integer value. This is probably the most straightforwad step, where we just cast our string to an `int()` and define that it is of base `16`, for example:

```python
>>> int('4141412d424242', 16)
18367621674189378
```

## Number base conversions

Numbers in Python can be represented many ways. A standard number in Python uses a base of `10`, such that `10` is the number ten. Putting a `0x` out the front makes that number now a hex number, like shown below. If we have a hexadecimal value as a string that we want to convert to a decimal, we can use the `int('hex', 16)` approach below.

```python
>>> int('A', 16)
10
>>> 0x10
16
```

## Hex characters in a string, in a byte array, etc.

Values in a string can also be represented by their hexadecimal equivalent by escaping the value with `\x`, as shown.

```python
 string = '\x4a\x82\xfd\xfe\xff\x00'
```

There is a continuum of data types when handling data between a website and it's use and analysis:

## Decoding:

- url-encoded, base 64 encoded string
- base 64 encoded string
- non-localization-encoded binary string
- byte array of characters
- string of hexadecimal values (0-9, a-f)
- hexadecimal representation
- integer representation

## Encoding (the reverse process):

- integer representation
- hexadecimal representation
- string of hexadecimal values (0-9, a-f)
- byte array of characters
- non-localization-encoded binary string
- base 64 encoded string
- url-encoded, base 64 encoded string

## Refs:

https://stackoverflow.com/questions/9641440/convert-from-ascii-string-encoded-in-hex-to-plain-ascii

https://stackoverflow.com/questions/35536670/how-to-convert-ascii-to-hex-in-python/35536716
