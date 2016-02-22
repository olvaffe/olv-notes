## Constants in C

Actually, it's not that complicated:

1) base and suffices choose the possible types.
2) order of types is always the same: int -> unsigned -> long -> unsigned
long -> long long -> unsigned long long
3) we always choose the first type the value would fit into
4) L in suffix == "at least long"
5) LL in suffix == "at least long long"
6) U in suffix == "unsigned"
7) without U in suffix, base 10 == "signed"

That's it.  C90 differs from C99 only in one thing - long long (and LL) isn't
there.  The subtle mess Linus has mentioned is C90 gccism: gcc has allowed
unsigned long for decimal constants, as the last resort.  I.e. if you had
a plain decimal constant that wouldn't fit into long but would fit into
unsigned long, gcc generated a warning and treated it as unsigned long.
C90 would reject the damn thing.  _Bad_ extension, since in C99 the same
constant would be a legitimate signed long long.

But yes, "use the suffix when unsure" is a damn good idea, _especially_ since
the sizeof(long) actually varies between the targets we care about.
