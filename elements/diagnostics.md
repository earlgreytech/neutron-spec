# Diagnostics Element API

This element provides basic generic information about the current blockchain the smart contract is executing on.

ABI Information

Element ID: 4
Type: Neutron Standard
Provision: Mandatory in all Neutron implementations. If chosed to not be supported, then these functions should be a no-op which only clears the stacks of inputs

Standard Functions:

* 1, log_error(count: u32, inputs:...):pure
* 2, log_warning(count: u32, inputs:...):pure
* 3, log_info(count: u32, inputs:...):pure
* 4, log_debug(count: u32, inputs:...):pure
* 5, log_info_comap():comap, pure -- Will log all data in the caller's output comap as info messages in a key -> data format using attached types for decoding. The `.msg` key will be logged first as the context for the rest of the keys

## ABI Exception!

This breaks ABI standards by the inputs array being backwards from what would normally be used. This is done for optimization purposes and to avoid allocation and string reordering being a absolute requirement on smart contract implementations. 

For example, given a C-style format string like "Hello world the result of X is %i and Y is %s", this would be accomplished by the following set of data on the stack (in push order):

* "Hello world the result of X is "
* "100" -- %i (must be passed as an integer converted to a string)
* " and Y is "
* "foobar" -- %s
* 4 -- the count of diagnostic messages

Without this break in ABI standards, it would be a requirement that to implement a "printf" or "println!" style function, it would require storing each part of the diagnostic message until receiving the last message, then finally pushing each part in reverse order. This is very troublesome for such an essential function for development as allocation support is not necessarily guaranteed in each smart contract environment, not to mention the memory cost could be needlessly expensive. So, the alternative presented here is to push items to the stack as they come up and then for the Diagnostic Element API to reverse the stack internally within native code. 

## Details

These functions must be provided by all Neutron implementations, but they are not used for any concensus or purpose that can modify contract behavior and thus can be a no-op for performance reasons in a non-debug build of a blockchain wallet etc. 

