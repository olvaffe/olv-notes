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
  - `targets : prerequisites ; recipe` is also supported
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
- 4.10 Multiple Targets in a Rule
  - `target1 target2: prerequisite`
- 4.11 Multiple Rules for One Target
  - `target: prerequisite1`
  - `target: prerequisite2`
  - the rules are merged into one
- 4.12 Static Pattern Rules
  - `targets ...: target-pattern: prereq-patterns ...`
    - e.g., `foo.o bar.o: %.o: %.c`
  - allow multiple targets in a rule to have analogous, but not identical
    prerequisites
- 4.13 Double-Colon Rules

## Chapter 5. Writing Recipes in Rules

- 5.1 Recipe Syntax
  - most lines are in make syntax
  - except those begin with a tab, which are in shell syntax
  - rule-context: after a rule is defined, and before another rule or variable
    definition
- 5.2 Recipe Echoing
  - `@` silences a command
- 5.5 Errors in Recipes
  - `-` makes a failue non-fatal
- 5.7 Recursive Use of make
  - `export <variable>` exports a variable to submake

## Chapter 6. How to Use Variables

- 6.3 Advanced Features for Reference to Variables
  - `$(var:.c=.o)` is equivalent to `$(patsubst %.c,%.o,$(var))`
- 6.11 Target-specific Variable Values
  - `target: VAR := 3`

## Chapter 7. Conditional Parts of Makefiles

## Chapter 8. Functions for Transforming Text

## Chapter 9. How to Run make

## Chapter 10. Using Implicit Rules

- 10.5 Defining and Redefining Pattern Rules
  - implicit rule is defined by a pattern rule
    - has exactly one `%` in its target
  - automatic variables
    - `$@` is the target
    - `$<` is the first prerequisite
    - `$^` is all prerequisites
    - `$*` matches the stem of a (static) pattern rule
- 10.7 Old-Fashioned Suffix Rules
  - `.c.o:` is equivalent to `%.o: %.c`

## Chapter 11. Using make to Update Archive Files

## Chapter 12. Extending GNU make

## Chapter 13. Integrating GNU make

## Chapter 14. Features of GNU make

## Chapter 15. Incompatibilities and Missing Features

## Chapter 16. Makefile Conventions
