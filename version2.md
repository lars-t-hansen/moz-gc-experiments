
Work in progress.

Version 2 will extend Version 1, most likely in a compatible fashion.  Here's what's going on.

### Tables-of-anyref

Table can now be "anyref" in addition to "anyfunc".  The code for "anyref" is 0x6F, its standard type code.

"anyref" can be used in the text format.

"anyref" can be used as the type passed to the JS WebAssembly.Table constructor.

Initially, at least, `call_indirect` cannot call via a table-of-anyref.

Setting elements in a table-of-anyref from JS will store JS objects.

Can table.copy only between tables of the same type.

Can table.fill only tables of type anyfunc since the source in this case is an elem segment which can only reference function values.

TODO: What if the value being set is not an object?  Run ToObject on it?

(Eventually)  Tables that are "anyref" can be targeted by element segments holding function values, 
and the values stored in such tables are the function values that would be obtained from the host side
if the host side reached into a corresponding anyfunc table and extracted functions.

Element segments are *not* further extended, they can reference only function values.
