Desgin Patterns
===============

## Visitor Pattern

* Suppose there is a tree of IRs (e.g., GLSL IRs)
  * each IR has a type: variable, expression, operator, function, ...
  * A IR can contain children IRs, such as a function IR and the function body
* There are infinitely many operations one wants to apply on the tree
  * different optimiziation techniques
  * high level IR to low level IR
* Pattern
  * each IR has an `accept` method
    * `void xxx_ir::accept(visitor *v) { v->visit(this); }`
  * each operation is represented by a visitor subclass
    * there are overloaded `visit` methods
    * `LowerVisitor::visit(xxx_ir *ir)`
    * `LowerVisitor::visit(yyy_ir *ir)`
* This is double-dispatched because the final `visit` method being called
  depends on
  * the type of the IR
  * the type of the visitor
* Hierarchical Visitor Pattern
  * visitor pattern has no concept of tree depth
  * visitor pattern does not allow a branch to be skipped
  * it still defines `visit` method for leaf nodes
  * it defines `visit_enter` and `visit_leave` for internal nodes
    * the above two combined solves the tree depth issue
  * the `accept` method and all `visit` methods return a boolean to indicate
    whether to continue with the branch
    * this allows a branch to be skipped
* Discussions
  * we would like to keep `class visitor` abstract
    * thus, when a new type of node is added, all visitors must be explicitly
      updated.  Imagine otherwise.  Suppose `class visitor` implements a default
      `visit` method for each node type.  When a new type is added and
      `class visitor` updated, there will be no warning if we forget to add the
      new type to a certain visitor.
