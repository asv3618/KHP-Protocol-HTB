# KHP-Protocol-HTB
In this challenge, we have a server application called "khp_server" (Keys Holder Protocol Server) that manages authentication keys. The vulnerability lies in its key handling functions, which we can exploit to gain shell access.

**Understanding the Target:**
Opening the provided binary in Ghidra, we see:
1.	It listens on a specified port (default 8080)
2.	Accepts client connections
3.	Spawns a thread for each new client to handle its requests
4.	Provides a key management service using functions defined in "khp_func.h"
![image](https://github.com/user-attachments/assets/ba6b38d8-a06f-418a-8f72-0c2d2519b396)

The server appears to implement a custom protocol for key management operations with commands like:
•	REKE - Register a key
•	DEKE - Delete a key
•	RLDB - List all keys in the database
•	AUTH - Authenticate using a key
•	EXEC - Execute commands (when authenticated)

**Vulnerability Discovery:**
After analyzing the server code and conducting initial testing, I identified a heap-based buffer overflow vulnerability in the key registration functionality.
The issue occurs in the REKE command, which parses input in the format REKE username:role data. When registering a key with carefully crafted data, we can overflow the heap buffer and corrupt adjacent memory structures.

**Exploitation Strategy:**
The exploitation strategy involves:
1.	Creating several key entries to set up the heap layout
![image](https://github.com/user-attachments/assets/d34198c0-cdc4-43fe-9ac2-2ad1afbe589b)
                               Figure 1:Connecting to the Server

![image](https://github.com/user-attachments/assets/6c4600d3-2a81-44f5-b65a-12e302abcf05)
                                    Figure 2: View Heap

2.	Deleting one entry to create a free chunk
![image](https://github.com/user-attachments/assets/33bdfeaa-475f-4788-9c55-78008c55b146)
                                  Figure 3: Deleted Entry

4.	Crafting a special entry that overflows into adjacent memory
5.	Authenticating with a manipulated key structure
6.	Executing shell commands

**Step-by-Step Exploitation:**
**1. Setting Up the Environment**
First, I connected to the target server and created three initial key entries:
**Register three keys with consistent data**
    REKE('chaika', 'admin', b'abcdef')
    REKE('chaika', 'admin', b'abcdef')
    REKE('chaika', 'admin', b'abcdef')

**2. Preparing the Heap**
After creating the initial entries, I listed the database contents using RLDB to verify the keys are registered correctly. Then I deleted the third key to create a free chunk in the heap:
    RLDB()
    DEKE(3)

**3. Triggering the Heap Overflow**
The critical part was to craft a special key registration that would overflow into adjacent memory. By sending carefully calculated data, I could overwrite the metadata of adjacent heap chunks:
    REKE('CHA', 'IKA', b'\x01'*0x70 + b'chaika:admin abcdef')

This command sends:
•	A different username/role combination ('CHA', 'IKA')
•	A large padding of 0x70 bytes filled with 0x01
•	Followed by a string that resembles a valid key entry
The overflow corrupts the heap management structures, allowing me to gain control over the memory allocator.

**4. Exploiting the Vulnerability**
After setting up the heap and triggering the overflow, I authenticated with key ID 1: Note this fails sometimes for reasons I don’t know so I had to AUTH manually.
    AUTHENTICATE(1)

Due to the corruption of heap metadata, the authentication succeeds with elevated privileges. With authentication complete, I could execute shell commands:
    EXEC()
This provided an interactive shell on the target system.

**Technical Analysis:**
The vulnerability is a classic heap-based buffer overflow. The server doesn't properly validate the size of data being stored in the heap-allocated buffers during key registration.
When the REKE command is processed, it allocates a fixed-size buffer but doesn't enforce proper bounds checking when copying user-supplied data. This allows writing beyond the allocated buffer and corrupting adjacent heap metadata.
By carefully crafting the overflow data, we can manipulate the heap in a way that leads to arbitrary code execution when the EXEC command is processed.

**Debugging Details:**
During exploitation, I attached GDB to analyze the heap structure before and after the overflow. The key observations are:
1.	The key structures are stored as heap-allocated objects
2.	The authentication check verifies the username and role fields
3.	When a key is deleted, its memory is freed but not zeroed out
4.	The overflow allows overwriting critical fields in adjacent chunks
![image](https://github.com/user-attachments/assets/45608f22-07b5-4d03-8e46-abb0ef3f1148)
                          Figure 4: Breakpoints after DEKE and REKE

![image](https://github.com/user-attachments/assets/7d1a9cc3-5a34-4f9a-aca2-beeb69a7db3e)
                          Figure 5: Overflow into the adjacent chunk

**Exploit Code Analysis:**
The final exploit script uses a class-based approach to interact with the server:
1.	Establishes a connection to the target
2.	Creates three initial entries to set up the heap
3.	Deletes one entry to create a free space
4.	Injects carefully crafted data that overflows into adjacent memory
5.	Authenticates with a manipulated key
6.	Gain shell access via the EXEC command
![image](https://github.com/user-attachments/assets/dd346ea8-2186-40dd-a95b-64c597b7ce21)
                                Figure 6: Code execution

**Lessons Learned:**
This challenge demonstrates several important security concepts:
1.	**Heap Memory Management**: Understanding how the heap allocator works is crucial for exploiting heap-based vulnerabilities
2.	**Input Validation**: The server failed to validate and bound-check user input properly, leading to the overflow
3.	**Privilege Separation**: The application allowed execution of commands with elevated privileges after authentication, making the vulnerability more severe
4.	**Memory Corruption**: By corrupting heap metadata, it was possible to manipulate the application's behavior

**Conclusion:**
The Keys Holder Protocol server contained a classic heap-based buffer overflow vulnerability that could be exploited to gain shell access. The exploit involved careful manipulation of the heap layout and corrupting key structures through a buffer overflow in the key registration functionality.
This challenge demonstrates the importance of proper input validation and memory management in security-critical applications.
