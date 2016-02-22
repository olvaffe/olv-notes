Doxygen
=======

## Documentation

* brief description and detailed description

    /**
     * \brief BRIEF DESCRIPTION
     *        BRIEF DESCRIPTION CONTINUED
     * 
     * DETAILED DESCRIPTION.
     */
* To put a descript after memebers, use `/**< ... */`
* The documentation of global functions, variables, typedefs, and enums will
  only be included in the output if the file they are in is documented as well.
  * That is, use `\file`
* To document a function parameter, use
  * `\param[in] var What is it for`
* `\c` use typewriter font for the next word
* `\note` starts an indented paragraph for note
* `\sa` starts an indented paragraph for see alsos
* `\return` documents the return value
* `\name` to name a member group
