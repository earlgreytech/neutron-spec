In this folder all of the various elements will be described


Element Function Modifiers:

These are only used in these specifications and are not necessarily strict. 

* :const -- Conveys that it will express data which may change in later blocks or transactions, but will be constant across the entire current chain of execution. Introduces no current execution data dependencies, but does introduce whole-blockchain data dependencies. Thus these can not be used by pure execution contexts. These types of functions are effectively less dynamic than static, but more dynamic than pure. 
* :pure -- Conveys that the function is purely computation and thus given the same input should give the same output at any point in the blockchain history. Can be used by all execution contexts (short of forks etc for adding new algorithms or bug fixes)
* :static -- Conveys that the function will not modify or alter potentially upstream data dependencies, but also that this function's results may change through the current chain of execution. In other words, it may read execution-mutable state, but not modify it. This can not be used by pure execution contexts
* :mutable -- Conveys that the function will modify data which may be relied upon in other places in the execution history. In other words, it may both read and write to execution-mutable state. This can not be used by pure nor static execution contexts. 

