
Work in progress.

Version 2 will extend Version 1, most likely in a compatible fashion.  Here's what's going on.

### Tables-of-anyref

Table can now be "anyref" in addition to "anyfunc".

Tables that are "anyref" can be targeted by element segments holding function values, 
and the values stored in such tables are the function values that would be obtained from the host side
if the host side reached into a corresponding anyfunc table and extracted functions.

Element segments are *not* further extended, they can reference only function values.

