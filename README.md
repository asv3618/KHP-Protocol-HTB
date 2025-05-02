# KHP-Protocol-HTB
In this challenge, we are provided with a server application called "khp_server" (Keys Holder Protocol Server) that manages authentication keys. The vulnerability lies in its key handling functions, which we can exploit to gain shell access.
Understanding the Target
Opening the provided binary in ghidra we see:
1.	It listens on a specified port (default 8080)
2.	Accepts client connections
3.	Spawns a thread for each new client to handle its requests
4.	Provides a key management service using functions defined in "khp_func.h"
![image](https://github.com/user-attachments/assets/ba6b38d8-a06f-418a-8f72-0c2d2519b396)
