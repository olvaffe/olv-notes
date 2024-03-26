GNU Make
========

## Chapter 1. Overview of make

## Chapter 2. An Introduction to Makefiles

## Chapter 3. Writing Makefiles

## Chapter 4. Writing Rules

- 4.2 Rule Syntax

    targets : prerequisites
        recipe
        ...
  - `targets` are filenames separated by spaces
  - `prerequisites` are filenames separated by spaces
    - they determine if `targets` are out-of-date
  - `recipe` are executed by shell
    - recipe lines start with a tab
    - they determine how to update `targets`
  - long lines can be splitted by a backslash followed by a newline
- 4.3 Types of Prerequisites
  - `targets : normal-prerequisites | order-only-prerequisites`
  - normal prerequisites
    - they are updated before targets
    - if they are updated, targets are considered out-of-date and are updated
      as well
  - order-only prerequisites
    - they are updated before targets
- 4.4 Using Wildcard Characters in File Names
  - `*`, `?`, `[]`, and `~` are supported as in shell
- 4.6 Phony Targets
  - `.PHONY: foo`
  - `foo` is always considered out of date
- 4.11 Multiple Rules for One Target
  - the rules are merged into one
- 4.12 Static Pattern Rules
  - `targets: target-pattern: prereq-pattern`

## Chapter 5. Writing Recipes in Rules

## Chapter 6. How to Use Variables

## Chapter 7. Conditional Parts of Makefiles

## Chapter 8. Functions for Transforming Text

## Chapter 9. How to Run make

## Chapter 10. Using Implicit Rules

## Chapter 11. Using make to Update Archive Files

## Chapter 12. Extending GNU make

## Chapter 13. Integrating GNU make

## Chapter 14. Features of GNU make

## Chapter 15. Incompatibilities and Missing Features

## Chapter 16. Makefile Conventions

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
