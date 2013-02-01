---
layout: post
title: structure assignment and its pitfall in C language
date: 2013-01-28 21:47
comments: true
categories: programming
footer: true
---

###Problem

There is a structure type defined as below:
{% codeblock lang:c %}
typedef struct __map_t {
    int code;
    char name[NAME_SIZE];
    char *alias;
}map_t;
{% endcodeblock %}
If we want to assign <code>map_t</code> type variable <code>struct2</code> to <code>sturct1</code>, we usually have below 3 ways:
{% codeblock lang:c %}
/* Way #1: assign the members one by one */
struct1.code = struct2.code;
strncpy(struct1.name, struct2.name, NAME_SIZE);
struct1.alias = struct2.alias;

/* Way #2: memcpy the whole memory content of struct2 to struct1 */
memcpy(&struct1, &struct2, sizeof(struct1));

/* Way #3: straight assignment with '=' */
struct1 = struct2;
{% endcodeblock %}
Consider above ways, most of programmer won't use way #1, since it's so stupid ways compare to other twos, only if we are defining an structure assignment function. So, ***what's the difference between way #2 and way #3? And what's the pitfall of the structure assignment once there is array or pointer member existed?*** Coming sections maybe helpful for your understanding.

### The difference between ***'=' straight assignment*** and ***memcpy***

The <code>struct1=struct2;</code> notation is not only **more concise**, but also **shorter** and **leaves more optimization opportunities to the compiler**. The semantic meaning of = is an assignment, while <code>memcpy</code> just copies memory. That's a huge **difference in readability** as well, although <code>memcpy</code> does the same in this case.

Copying by straight assignment is probably best, since it's shorter, easier to read, and has a higher level of abstraction. Instead of saying (to the human reader of the code) "copy these bits from here to there", and requiring the reader to think about the size argument to the copy, you're just doing a straight assignment ("copy this value from here to here"). There can be no hesitation about whether or not the size is correct.

Consider that, above source code also has pitfall about the pointer alias, it will lead dangling pointer problem (*It will be introduced below section*). If we use straight structure assignment '=' in C++, we **can consider to overload the <code>operator=</code> function**, that can dissolve the problem, and the structure assignment usage does not need to do any changes, but structure memcpy does not have such opportunity.

### The pitfall of structure assignment:
Beware though, that copying structs that contain pointers to heap-allocated memory can be a bit dangerous, since by doing so you're aliasing the pointer, and typically making it ambiguous who owns the pointer after the copying operation.

If the structures are of compatible types, yes, you can, with something like:
{% codeblock lang:c %}
memcpy (dest_struct, source_struct, sizeof(dest_struct));
{% endcodeblock %}}
The only thing you need to be aware of is that this is a shallow copy. In other words, if you have a <code>char *</code> pointing to a specific string, both structures will point to the same string.

And changing the contents of one of those string fields (the data that the <code>char *</code> points to, not the <code>char *</code> itself) will change the other as well. For these situations a "deep copy" is really the only choice, and that needs to go in a function. If you want a easy copy without having to manually do each field but with the added bonus of non-shallow string copies, use strdup:

{% codeblock lang:c %}
memcpy (dest_struct, source_struct, sizeof (dest_struct));
dest_struct->strptr = strdup(source_struct->strptr);
{% endcodeblock %}

This will copy the entire contents of the structure, then deep-copy the string, effectively giving a separate string to each structure. And, if your C implementation doesn't have a strdup (it's not part of the ISO standard), you have to allocate new memory for <code>dest_struct</code> pointer member, and copy the data to memory address.

#### Example of trap:
{% codeblock lang:c %}
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define NAME_SIZE   16
typedef struct _map_t {
    int code;
    char name[NAME_SIZE];
    char *alias;
} map_t;

int main()
{
    map_t a, b, c;

    /* initialize the a's members value */
    a.code = 1024;
    snprintf(a.name, NAME_SIZE, "Controller SW3");
    char *alias = "RNC&IPA";
    a.alias = alias;

    /* assign the value via memcpy */
    memcpy(&b, &a, sizeof(b));

    /* assign the value via '=' */
    c = a;
    return 0;
}
{% endcodeblock %}

Below diagram illustrates above source memory layout, if there is a pointer field member, either the straight assignment or <code>memcpy</code>, that will be alias of pointer to point same address. For example, <code>b.alias</code> and <code>c.alias</code> both points to address of <code>a.alias</code>. Once one of them free the pointed address, it will cause another pointer as dangling pointer. **It's dangerous!!**


![Alt text](/images/2013-01-28-structure-assignment-and-its-pitfall-in-C-language/structure_assignment.1-1.png "Illustrates above source code memory layout.")

### Conclusion
* Recommend use straight assignment '=' instead of memcpy.
* If structure has pointer or array member, please consider the pointer alias problem, it will lead dangling pointer once incorrect use. Better way is implement structure assignment function in C, and overload the operator= function in C++.

### Reference:
* [stackoverflow.com: structure assignment or memcpy](http://stackoverflow.com/questions/5383318/struct-assignment-or-memcpy "struct assignment")
* [stackoverflow.com: assign one struct to another in C](http://stackoverflow.com/questions/2302351/assign-one-struct-to-another-in-c "assign one struct to another")
* [bytes.com: structures assignment](http://bytes.com/topic/c/answers/215832-structures-assignment "structures assignment")
* [wikipedia: struct in C programming language](http://en.wikipedia.org/wiki/Struct_%28C_programming_language%29 "struct in C programming language")

