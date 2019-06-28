---
published: true
title: clang vs gcc diagnostic flags
layout: post
tags: [compiler, gcc, clang, diagnostic, ]
---

I switched to clang instead of gcc at some point (whenever at least it is possible). This was around the time when I had still worked for Nokia. The aim was to improve the codebase,
while I cannot report on that for numerous reasons like I no longer work there.

I still remember a few differences like clang liked to depend on include order, and gcc was more relaxed; or that clang was keen on dropping objects it did not thing is reachable causing a linking nightmare (that was the last thing I tried to resolve).

Since that time, the current project that I tend to compile regularly is the syslog-ng. It does in fact compile with clang, even before I know the project.
The part that bugs me, is rather connected to the diagnostics(warnings).

The project has a nice set of warning configured, but some are gcc-ism, and keep my console busy every time I compile. I set a one day project to clean up, but first to understand those warnings, and fine possible alternatives in clang.

The list of warnings that bugs me:
* -Wcast-align
* -Wmissing-parameter-type
* -Wold-style-declaration
* -Wsuggest-attribute=noreturn
* -Wunused-but-set-parameter

## My environment

### on my computer
```
clang version 8.0.0 (tags/RELEASE_800/final)
gcc (GCC) 9.1.0
```

###  travis (which we use to compile)
```
clang version 5.0.0 (tags/RELEASE_500/final)
gcc (Ubuntu 4.8.4-2ubuntu1~14.04.3) 4.8.4
```

Oh and I am also limited to C99, well at least I can thing of other project (c11).


---


## -Wold-style-declaration

Sample code:
```
const static int foo = 2;
```
This tries to prevent the switch between `const` and `static` as the `static const int` would be the new standard order to follow.

Mostly gcc is going to support it, as explained above:
```
gcc -Wold-style-declaration -c -o /dev/null old-style.c
old-style.c:2:1: warning: ‘static’ is not at beginning of declaration [-Wold-style-declaration]
    2 | const static int foo = 2;
      | ^~~~~
```

clang also give a warning, but not what we would expect:

```
clang -Wold-style-declaration -c -o /dev/null old-style.c
warning: unknown warning option '-Wold-style-declaration'; did you mean '-Wout-of-line-declaration'? [-Wunknown-warning-option]
1 warning generated.
```

I could not find an alternative for this in `clang`, even tried with `-Weverything` (thanks for that `clang`).


## -Wmissing-parameter-type


As per `gcc` documentation suggest, it detects the following issue
```
void foo(bar) { }
```

```
gcc -c -o /dev/null old-style.c -Wmissing-parameter-type
old-style.c: In function ‘foo’:
old-style.c:2:7: warning: type of ‘bar’ defaults to ‘int’ [-Wimplicit-int]
    2 | float foo(bar) {}
      |       ^~~
```

Let's try again

```
gcc -c -o /dev/null old-style.c -Wmissing-parameter-type -Wno-implicit-int
```

No, silencing the implicit int does not help either.

Maybe the following would trigger:
```
float foo(int bar);

float foo(bar)
{
   return bar;
}
```

```
gcc -c -o /dev/null old-style.c -Wmissing-parameter-type -Wno-implicit-int
gcc -c -o /dev/null old-style.c -Wmissing-parameter-type
old-style.c: In function ‘foo’:
old-style.c:4:7: warning: type of ‘bar’ defaults to ‘int’ [-Wimplicit-int]
    4 | float foo(bar)
      |       ^~~
```

Nope!

Let's see the source of gcc to find out what is happening here. Gcc is nice enough to have a test case for this option: 
```
int foo(bar) { return bar; }
```

Which in fact just results the same `-Wimplicit-int` warning, even the tests asserts for that. Something seems fishy here.

Finally after digging up the code responsible to trigger this warning:
```
          if (flag_isoc99)
            pedwarn (DECL_SOURCE_LOCATION (decl),
                     OPT_Wimplicit_int, "type of %qD defaults to %<int%>",
                     decl);
          else
            warning_at (DECL_SOURCE_LOCATION (decl),
                        OPT_Wmissing_parameter_type,
                        "type of %qD defaults to %<int%>", decl);
```

The trick is to have an ancient skeleton (c89), no thanks.


What about `clang` ? Well... that does not have the exact option, but it has `-Wknr-promoted-parameter`, although it has a slightly different meaning, and as such does not provide any warning with any of the above example.

In the end the `-pedantic` is to save the day, it works with both `gcc` and `clang`, sadly I cannot use it either (there are a few things to be solved before).


## -Wunused-but-set-parameter

Sample code:
```
void foo(void)
{
   int bar;

   bar = 2;

}
```

```
gcc -c -o /dev/null old-style.c -Wextra -Wall -pedantic
old-style.c: In function ‘foo’:
old-style.c:4:8: warning: variable ‘bar’ set but not used [-Wunused-but-set-variable]
    4 |    int bar;
      |        ^~~

clang -c -o /dev/null old-style.c -Wextra -Wall -pedantic
```

This is something that `clang` does not yet support, good for `gcc` users. Better for those whom uses both.


## -Wsuggest-attribute=noreturn

Sample code:
```
void foo(void)
{
   while (1) ;
}
```

```
gcc -c -o /dev/null old-style.c -Wextra -Wall -pedantic -Wsuggest-attribute=noreturn
old-style.c: In function ‘foo’:
old-style.c:2:6: warning: function might be candidate for attribute ‘noreturn’ [-Wsuggest-attribute=noreturn]
    2 | void foo(void)
      |      ^~~
gcc -c -o /dev/null old-style.c -Wextra -Wall -pedantic -Wmissing-noreturn
old-style.c: In function ‘foo’:
old-style.c:2:6: warning: function might be candidate for attribute ‘noreturn’ [-Wsuggest-attribute=noreturn]
    2 | void foo(void)
      |      ^~~
clang -c -o /dev/null old-style.c -Wextra -Wall -pedantic -Wmissing-noreturn
old-style.c:3:1: warning: function 'foo' could be declared with attribute 'noreturn' [-Wmissing-noreturn]
{
^
1 warning generated.
```

It seems that the `-Wsuggest-attribute=noreturn` and `-Wmissing-noreturn` are the same in case of `gcc`, while `clang` only support the later.
I would rather have an option not to allow non-returning function to exist; but hey this is a good-enough alternative.

## -Wcast-align

Sample:
```
int main(void)
{
    char foo[] = "foobar";
    int bar = *(int*)(foo + 1);
    return 0;
}
```

```
gcc -c -o /dev/null old-style.c -Wcast-align
clang -c -o /dev/null old-style.c -Wcast-align
old-style.c:6:16: warning: cast from 'char *' to 'int *' increases required alignment from 1 to 4 [-Wcast-align]
    int bar = *(int*)(foo + 1);
               ^~~~~~~~~~~~~~~
1 warning generated.
```

Note: I compiled this with x86-64, and it seems `gcc` takes the ABI into account, as if I grab arm:

```
arm-gcc -c -o /dev/null old-style.c -Wcast-align
old-style.c: In function 'main':
old-style.c:6:16: warning: cast increases required alignment of target type [-Wcast-align]
     int bar = *(int*)(foo + 1);
                ^
```

Well it kinda feel bad that `gcc` despite asking it for this warning straight ignores it, but hey use `clang`! 


# Conclusion

If any can be made. First, of course I try to upstream at least a few of them [https://github.com/balabit/syslog-ng/pull/2810](https://github.com/balabit/syslog-ng/pull/2810).
I had some trouble with clang, and ended up not forcing some of these. It looks like `clang` in fact looks inside the macros and reports errors when they are expanded (probably macro expansion happens before error reporting or in parallel), while `gcc` is a lazy dog do not even bother with the macros.

If you project uses a lot of macro, you should consider `clang` at least to compile with; or better avoid macros at all cost!
But now I do not know if I should list as a pitfall of macro usage or gcc...


