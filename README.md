# System Design Study

## Zero Copy

### Traditional Approach

* The `file.read()` call causes a context switch from user mode to kernel mode
* The first copy is performed by the direct memory access (DMA) engine, which reads file contents from the disk and stores them into a kernel address space buffer
* The requested amount of data is copied from the read buffer into the user buffer
* The `read()` call returns causing a context switch from kernel back to user mode
* The `socket.send()` call causes a context switch from user mode to kernel mode
* A third copy is performed to put the data into a kernel address space buffer that is associated with the socket
* The `send()` system call returns, creating the fourth context switch
* Independently and asynchronously, a fourth copy happens as the DMA engine passes the data from the kernel buffer to the protocol engine
* During the reads, the kernel buffer is meant to serve as a read ahead buffer cache as the application reads small amount of data at a time
* During the writes, the kernel buffer makes asynchronous writes possible

![Traditional Copy Semantics](traditional_copy_semantics.gif)

![Traditional Context Switches](traditional_context_switches.gif)

### Zero Copy Approach

* The `fileChannel.transferTo()` method transfers data from the file channel to the given writable byte channel
* Internally, it depends on the underlying operating system’s support for zero copy; in UNIX and various flavors of Linux, this call is routed to the `sendfile()` system call
* The action of the `file.read()` and `socket.send()` calls can be replaced by a single `transferTo()` call
* The `transferTo()` method causes the file contents to be copied into a read buffer by the DMA engine
* The data is copied by the kernel into the kernel buffer associated with the output socket
* The third copy happens as the DMA engine passes the data from the kernel socket buffers to the protocol engine
* Number of context switches reduced from four to two and the number of data copies reduced from four to three (only one of which involves the CPU)

![Zero Copy Semantics](zero_copy_semantics.gif)

![Zero Copy Context Switches](zero_copy_context_switches.gif)

* In Linux kernels 2.4 and later, 
  * The underlying network interface card supports gather operations. Thus, no data is copied into the socket buffer. Instead, only descriptors with information about the location and length of the data are appended to the socket buffer
  * The DMA engine passes data directly from the kernel buffer to the protocol engine, thus eliminating the remaining final CPU copy
* The `transferTo()` API brings down the time approximately 65 percent compared to the traditional approach

![Linux 2.6 Zero Copy Semantics](linuz2.6_zero_copy_semantics.gif)


## References

https://developer.ibm.com/articles/j-zerocopy/