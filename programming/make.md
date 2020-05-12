GNU Make
========

- keywords

    target: VAR := 3
    target: A B | C D ; echo $(VAR)
    	echo $(VAR)
    
    keywords: target-specific variable, order-only prerequisite, first command
- most lines are in make syntax
- except those begin with a tab, which are in shell syntax
- rule-context: after a rule is defined, and before another rule or variable definition
- `$(var:.c=.o)` is equivalent to `$(patsubst %.c,%.o,$(var))`
  - keyword: `Substitution References`
- `@` silences a command; `-` makes a failue non-fatal.
  - keywords: `command echoing`, `errors in commands`
- Types of rules
  - ordinary rule
    - can have multiple targets in a rule
    - can have multiple rules for one target
  - static pattern rule
    - allow multiple targets in a rule to have analogous, but not identical
      prerequisites.
    - `targets ...: target-pattern: prereq-patterns ...`
  - double-colon rule
  - implicit rule
    - is defined by a pattern rule
    - has exactly one `%` in its target.
    - `.c.o` is equivalent to `%.o: %.c`
      - the former is called suffix rule
- `$@`, `$<`, `$^`, and `$*`
  - `$*` matches the stem of a (static) pattern rule
