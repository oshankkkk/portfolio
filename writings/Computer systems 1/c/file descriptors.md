when you give a path name the OS gives a file descriptor
its a number and its unique for your process
using that you can run functions, syscalls to it, you can read run write to that. not all fd represet a actual files. 
So fd is this unified interfaces for a lot of io stuff in the OS. This is whats meant by everything is a file. 

> It is a process local handle to a kernal object

When you fork the process, it can inherit those file descriptors.  
> when you run a cmd with a file path like /etc/passwd we fork it???

Those numbers are not random
Does linux closes the file for me when the code dies, relying on the kernal to do it. dont have to free memory??


