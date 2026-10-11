+++
title = "Bytes in Motion"
date = 2026-08-19
template = "blog/page.html"

+++

## Description

Coming from Java's String class to C's character arrays taught me more about memory than any textbook. Here's what I learned about string copy, the null terminator and why memmove exists.

## Motivation

I worked with strings in Java for a long time without ever thinking about them. They were immutable, garbage collected and safe to copy around. I never wondered where the characters lived or what really happened when I assigned one string to another.

C changed that quickly. Copying a string meant deciding where the bytes go, how much room there is, and who is to blame when there isn't enough. A single line of copying suddenly involved memory allocation, pointers and raw data all at once.

I wanted to slow down and understand it properly, so this post is me working through it. What does a string look like in memory? What is the zero at the end for? How does a copy actually work, and why do both memcpy and memmove exist?

## A string is just bytes

In Java, a String is an object that knows its own length and won't let you read past the end. In C, a string is an array of char that happens to end with a zero byte.

```c
char s[] = "Hi";
```

I printed what is actually stored:

```
sizeof(s) = 3
strlen(s) = 2
s[0] = 72   ('H')
s[1] = 105  ('i')
s[2] = 0
```

Three bytes for a two-letter word. The last byte is the terminator. Nothing in C stores the length of a string, so functions like strlen and printf just walk forward until they hit that zero.

This also changed how I think about comparing strings. In Java I knew == compared references and equals compared content. C has the same trap. Two arrays holding "Hello" are at different addresses, so a == b is false, while strcmp(a, b) returns 0 because the contents match.

## The "escape character"

When I started writing this, I called that last byte the escape character. That's wrong. The escape character is the backslash. It begins escape sequences like \n and \t. The sequence \0 is just a way of writing the byte with value zero, and that byte is the null terminator.

What does it mean? Simply "the string ends here". And if it's missing or overwritten, nothing stops the functions from reading on.

```c
char x[8] = {'A','B','C','\0','X','Y','Z','\0'};
printf("%s\n", x);     // ABC
x[3] = '-';
printf("%s\n", x);     // ABC-XYZ
```

The second printf just kept going and picked up the XYZ next to it. I was lucky that there was another zero at x[7]. If there hadn't been one, it would have read whatever happened to sit after the array, which could be garbage or a crash.

## Copying a string

Once you know about the terminator, copying a string is easy to picture. You copy the characters one at a time, and then you copy the zero at the end.

```c
while (*src != '\0') {
    *dst = *src;
    dst++;
    src++;
}
*dst = '\0';
```

For "Hi", this copies 'H', then 'i', then sees the zero in the source and stops. The last line writes a zero into the destination so it becomes a proper string too. The real strcpy does the same job, just written more compactly and tuned for speed.

The problem is what strcpy never checks. It looks at the source to know when to stop, but it has no idea how big the destination is.

```c
char dst[4];
strcpy(dst, "Hello");     // needs 6 bytes, dst only has 4
```

"Hello" needs six bytes including the zero. The array has four. strcpy happily writes the extra two bytes past the end, over whatever was stored next to it.

I assumed strncpy was the fix, since it takes a size limit. It does stop at the limit, but it has a trap of its own.

```c
char dst[4];
strncpy(dst, "Hello", 4);
```

This copies H, e, l, l and stops because it reached the limit of four. There is no room left for the zero, and strncpy doesn't add one. When I checked, the last byte was 'l'. So dst isn't a valid string anymore, and printing it would read on past the array looking for a zero.

The habit I settled on is snprintf.

```c
char dst[4];
snprintf(dst, sizeof dst, "%s", "Hello");     // dst holds "Hel"
```

It never writes more than the size you give it, and it always leaves room for the zero. If the text is too long, it gets cut short instead of overflowing. Losing the end of a string is a much better problem to have than writing over memory that isn't yours.

## memcpy

memcpy(dest, src, n) is different because it doesn't care about zero bytes. It copies exactly n bytes, whatever they are. That makes it usable for structs, integer arrays and binary data, not only text. The price is that you must say how many bytes to copy.

I used to think memcpy was the slow, risky one, but that isn't true. It's usually very fast. The catch is a rule: the source and destination must not overlap. If they do, the behaviour is undefined.

## Why overlap matters

Say I want to shift part of a buffer two places to the right, inside the same buffer.

```
index:   0 1 2 3 4 5 6 7
before:  A B C D E F G H
```

I want to move the first six bytes so they start at index 2. If I copy from the front, one byte at a time:

```c
for (int i = 0; i < 6; i++)
    buf[2 + i] = buf[i];
```

I get ABABABAB. The first write puts an A at index 2, which was a C I hadn't copied yet. From then on I'm copying the damage forward.

If I copy from the back instead, I get ABABCDEF, which is right. But if I wanted to shift left, copying from the front would be the correct choice. So the right direction depends on how the two regions overlap.

## memmove

memmove takes the same arguments as memcpy, but it is guaranteed to give the right answer even when the source and destination overlap. The trick is that it looks at the two addresses before it copies anything, and uses them to decide which end to start from.

Take the same buffer as before.

```
index:   0 1 2 3 4 5 6 7
before:  A B C D E F G H
```

If the destination is after the source, I'm shifting right. The danger is writing over source bytes I haven't read yet, and that happens at the front of the region, so memmove starts from the back instead. Moving the first six bytes to index 2, it copies F to index 7, then E to index 6, then D to index 5, and so on down to A. By the time a write lands on a byte that was part of the source, that byte has already been copied. The result is ABABCDEF.

If the destination is before the source, I'm shifting left, and the danger flips to the other end. So memmove starts from the front. Moving the last six bytes to index 0, it copies C to index 0, D to index 1, and so on. Each write only lands on a byte that has already been read. The result is CDEFGHGH.

If the two regions don't overlap at all, either direction works.

Written out by hand, the whole idea is only a few lines.

```c
void *my_memmove(void *dest, const void *src, size_t n)
{
    unsigned char *d = (unsigned char *)dest;
    const unsigned char *s = (const unsigned char *)src;

    if (n == 0 || dest == src)
        return dest;

    if (d > s && d < s + n)
    {
        // Copy backward when memory overlaps
        d += n;
        s += n;

        while (n--)
            *--d = *--s;
    }
    else
    {
        // Copy forward
        while (n--)
            *d++ = *s++;
    }

    return dest;
}
```
The check d > s && d < s + n is the overlap test. It asks whether the destination starts inside the source region. If it does, copying forward would overwrite source bytes I haven’t read yet, so I copy backward. In every other case, forward is safe.

The real memmove does the same thing, just faster because it copies many bytes at a time.

A common use is deleting from the middle of a string. To remove the C from "ABCDEF", move everything after it one place to the left.

```c
char s[] = "ABCDEF";
memmove(s + 2, s + 3, strlen(s + 3) + 1);
printf("%s\n", s);                 // ABDEF
```

The plus one matters. memmove only moves bytes and knows nothing about strings, so the terminator has to be included in the count or the result won't end properly.

## What memmove doesn't do

It fixes overlap, not overflow. It will copy exactly n bytes and trust you that both buffers are big enough. It can't check the destination size, and it can't tell whether the source really has n readable bytes.

A few mistakes I want to avoid. Because n is unsigned, a subtraction that goes below zero wraps into a huge number. Copying strlen(s) bytes leaves out the terminator, so it needs to be strlen(s) + 1. And the size should come from sizeof or a length I track myself, checked before I copy.

## Closing thoughts

Java handled memory safety for me, and I never noticed. In C it's all on me. I need to know where the data starts, where it ends and where it's going. That was the real lesson from strings.