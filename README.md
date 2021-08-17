# C_File_Transfer_A3
Building a xmodem server to receive the file from client. 
Assignment 3: Socket Programming
File Transfer Protocols
A file transfer protocol is an agreement between sender and receiver for transferring a file across a (possibly unreliable) connection. You use HTTP or HTTPS to receive files every day when visiting webpages, and you may be familiar with FTP and SFTP which are examples of other popular protocols for file transfers.

In this assignment, you'll be writing a server for a file transfer protocol called xmodem. When finished, you will be able to connect an xmodem client to your server and send a file to your server using the xmodem protocol.

Your Ports
To avoid port conflicts, we're asking you to use the following number as the port on which your server will listen: take the last four digits of your student number, and add a 5 in front. For example, if your student number is 1007123456, your port would be 53456.

To have the OS release your server's port as soon as your server terminates, you should put the following socket option in your server code (where listenfd is the server's listening socket):
    int yes = 1;
    if((setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(int))) == -1) {
        perror("setsockopt");
    }
If it turns out that you still find a port in use, you may add 1 or 2 to your port number as necessary in order to use new ports (for example, the fictitious student here could also use 53457, 53458, 53459, 53460). When you shutdown your server (e.g. to compile and run it again), the OS may not release the old port immediately, so you may have to cycle through ports a bit.

What is xmodem?
xmodem was a popular protocol for sending and receiving files back when the majority of consumer networking occurred over phone lines using dial-up modems. It was possible for bursts of data to be garbled or lost completely because the connection was unreliable and subject to line noise. Unchecked, such garbled data would prevent you from receiving an exact copy of a file being sent to you! xmodem was introduced as a simple protocol for more reliable transfers. When the receiver detects data corruption, it signals to the sender to re-send the current packet so that each packet is received cleanly.

There is a lot of xmodem information out there. You can Read More, but there are a confusing number of versions of the protocol in use. Use only what is in this handout.

CRC16
CRC16 is an algorithm for detecting transmission errors in a file sent from a sender to a receiver. A CRC16 of a message is 16 bits (2 bytes) long. The sender of a message sends the message concatenated with its CRC16. When received, the receiver calculates its own CRC16 on the message and checks that its CRC16 matches what the sender said it should be. If it matches, it is likely that there was no transmission error. If the CRC16 doesn't match, the receiver can request that the sender re-send the message.

The xmodem protocol uses CRC16 to detect data corruption. We have provided an implementation of CRC16 for you to use in crc16.c.

Operation of the Client
In this assignment, there is an xmodem client and an xmodem server. The client sends files; the server receives and stores them. You are writing the server; we are providing one sample client and your server must work with this client and all other clients that satisfy the protocol.

Take a look at client1.c for the following discussion.

The behaviour of an xmodem client (i.e. a sender) is governed by a state machine. In the client1.c sample, there are five states, each of which has associated actions and transitions to other states. The client starts in the initial state. We briefly describe the states here. This is important information for you because your server is the "other half" of this protocol and you'll want to understand what the client is doing.

initial: in this state, the client is just beginning. It sends the filename to the server followed by a network newline. No null character is sent to terminate the filename. This state also sets the current block to 1, and then moves to the handshake state. (Notice here that the first block the sender sends has block number 1, not 0.)
handshake: in this state, the sender reads one character at a time from the socket, discarding the characters until a capital C is received. This C means that the server is ready to start receiving xmodem blocks. Once the C is found, the sender moves to send_block.
send_block: the goal of this state is to send a single block (or packet) of data to the server. The block includes the payload, which is the actual content of the file that we want to transfer. However, this isn't all: xmodem doesn't simply send the payload by itself because then the server has no way of knowing whether the payload has been corrupted. So, the xmodem client sends some overhead with each block. Here is what the sender sends for each block:
An ASCII symbol known as SOH (start of header). This is an ASCII control character that tells the server that a block is starting.
The current block number, as a one-byte integer. This is not an ASCII code like 49 for the number 1, but is the integer representation of 1.
The inverse of the current block number. This is 255 minus the block number. The server is going to make sure that the block number and the inverse agree; for example, if the block number is 15, then the server will expect the inverse to be 240.
The actual payload. This is always 128 bytes. You might wonder what happens if the file being sent is not a multiple of 128 bytes. In that case, the final block is padded with another ASCII control character called SUB. So, every file that successfully transfers to the server will have 0 to 127 bytes of this padding.
The high byte of the CRC16 of the payload.
The low byte of the CRC16 of the payload.
OK! So, the client has sent all of this stuff. (That's 1 byte for the block number, 1 byte for the inverse, 128 bytes for the payload, and another 2 bytes for the CRC16. 132 bytes total for each block.) Now, the client has to know whether the receiving server got the block OK or whether there were problems, so it moves to wait_reply.

There's one more state transition to mention here. If the file is at EOF, then instead of sending a block and moving to wait_reply, the client moves to finish.

wait_reply: in this state, the client is waiting for a reply from the server. Did the server receive the block correctly? With the proper block number, inverse, and CRC16? If yes, then the server will send the client an ACK (another of those ASCII control characters), and the client will increment the block number and go back to send_block to send the next block. If not, then the server sends the client a NAK and the client will have to re-send the current block, so the client goes to send_block but does not increment the block number.
finish: now the client has sent all of the blocks. The only thing left is to send an EOT (that's an end-of-transfer ASCII control code) and wait for a final ACK. When the client receives that final ACK, we know that the server has received the entire file. If the client's EOT gets NAK'd, then the client repeats by sending another EOT.
This is a well-behaved client. It always sends the proper block numbers and inverses, and always sends the correct CRC16. It is also a simplified client, because it does not take advantage of an xmodem feature that allows a block length of 1024 bytes. The server you write will work with this simple client, but also clients that send wrong blocks or wrong CRC16s or send 1024-byte blocks in addition to 128-byte blocks.

Your Task: xmodem Server
Your task is to implement an xmodem server as described in this section. The server allows clients to connect and send a file to the server using the xmodem protocol.

Using Select
The server must never block waiting for input from a particular client or the listening socket. It must allow multiple clients to connect simultaneously and send files to the server. The server won't know whether a client will talk next or whether a new client will connect. And, in the former case, it won't know which client will talk next. This means that you must use select rather than blocking on one file descriptor.

A related issue is that you cannot assume that an xmodem block will be received in a single read call. For example, if you are expecting a 132-byte block, you might get 50 bytes in the first read and then 82 bytes in the second read. You must therefore buffer what a client sends until you have the required data in the buffer.

Makefile
Create a Makefile that compiles a program called xmodemserver. In addition to building your code, your makefile must permit choosing a port at compile-time. To do this, first add a #define to your xmodemserver.c program to define the port number on which the server will expect connections (this is the port based on your student number):

#ifndef PORT
  #define PORT x
#endif
Then, in your makefile, include the following code, where y should be set to a number possibly different from x:
PORT=y
CFLAGS= -DPORT=\$(PORT) -g -Wall
Now, if you type make PORT=53456 the program will be compiled with PORT defined as 53456. If you type just make, PORT will be set to y as defined in the makefile. Finally, if you use gcc directly and do not use -D to supply a port number, it will still have the x value from your source code file. This method of setting a port value will make it possible for us to test your code by compiling with our desired port number. (It's also useful for you to know how to use -D to define macros at the commandline.)

Sample Server
There is a sample server written by Alan Rosenthal that you might find helpful (muffinman.c). Feel free to yank code from there, with two important provisos:

Don't copy-and-paste stuff into your program and then fuss with it to make it work. You should know exactly what the code does and why. We won't take kindly to junk in your code that is unnecessary or does not work, and is clearly a stayover from the sample server.
Clearly indicate the code that you took from the sample server.
Operation of the Server
We're going to describe the xmodem server (which receives a file from the sending client) using a state machine similar to the one we discussed above for the xmodem client. The server must keep track of which state it is in when communicating with each client. The xmodemserver.h file contains some important declarations including struct client that you will use to store the state for each client.

The server will be reading files from clients, and it needs to store these files somewhere. To avoid cluttering up your source directory, the server will store the files in a directory called filestore. A helper function is available in helper.c to create the directory (if it doesn't exist) and to open the file in the directory. Do not add filestore or its contents to your repository.

There are five states that correspond to the states in the client, and the server starts in the initial state when it accepts a connection from a client:

initial: in this state, the server is waiting for a filename from the client. (Immediately here you're required to use select, because a client could take a long time to send the filename, during which another faster client could connect!) Once the server receives a complete filename, it should open that file for writing, then send a C to the client and transition to pre_block. The first block that the server expects is block number 1.
Important: your server is receiving files and can easily delete an existing file if you're not careful. You're highly advised to choose some arbitrary extension for your files, like .junk, and only send those files to your server. In fact, for extra precaution, you might have your server check for this extension in a filename and drop the client if an unsuitable filename is provided. The last thing you want is accidentally deleting your .c files for the assignment -- commit often to git!

pre_block: in this state, the server scans incoming characters looking for one of three different characters.
If the server gets an EOT, there is nothing more to receive. Send ACK, drop the client, and transition to finish.
If the server gets SOH, it means a 128-byte payload is going to be sent by the client. This is the kind of block that our sample client sends -- the one with the block number, inverse, payload, and CRC16. Transition to get_block.
If the server gets STX (this is another ASCII control code; this one has integer value 2), then a 1024-byte payload is going to be sent by the client. As with SOH, transition to get_block.
get_block: the server now wants to read 132 bytes (SOH) or 1028 bytes (STX). Again, do not block waiting for 132 or 1028 bytes, because that will lock out other clients who want to send stuff in the meantime. Once the server has sufficient bytes, transition to check_block.
check_block: you now have the block number, inverse, packet, and CRC16. There's a bunch of error-checking and other conditions here:
If the block number and inverse do not correspond, then you have a serious error, so drop the client and abort this file transfer.
If the block number is the same as the block number of the previous block, then the server is receiving a duplicate block. (This can happen if the client loses an ACK that you send to acknowledge a block.) If this happens, then just ACK the duplicate.
If the block number is not the one that is expected to be next, then you have a serious error, so drop the client and abort this file transfer.
If the CRC16 is incorrect, send a NAK.
Assuming everything is OK, then the block has been successfully received. Write the block to the file and increment the block number (watch for the wrap at 255), and send an ACK to the client. Move to pre_block to wait for the next action from the client.
finish: you can perform any cleanup actions here.
Other Clients
We strongly suggest modifying our sample client to ensure that your server works when things go wrong. Note that we're using TCP (a reliable protocol), so your clients will have to intentionally "make mistakes" to properly test your server. Here is what your server must be able to handle:

Blocks with an unexpected block number or inverse. (To test this, you can have your client randomly mess up the block number of the block it is going to send next.)
Blocks where the CRC16 does not match the CRC16 of the payload. (Again, test this by having your client randomly mess up the CRC16 that it sends.)
Blocks sent across multiple TCP segments. (To do this, have your client do two or more write calls for a packet instead of one write call. Or have your client do two write calls to split the filename over two TCP segments! You might also insert a small time delay (man 3 usleep) between your write calls to ensure that the pieces are really being sent out separately.)
1024-byte payloads. (To test this, have your client sometimes send 1024-byte payloads. Remember that these blocks start with STX.)
What to submit
You will commit to your repository in the a3 directory all .h and .c files and your makefile.

Make sure that you've done git add on every file that is needed by your assignment. As a final step, Checkout your repository into a new directory to verify that everything is there. There's nothing we can do if you forget to submit a file.
