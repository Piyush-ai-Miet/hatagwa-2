# File101 CTF Challenge - Quick Reference

## Instance Details
- **URL:** 34.252.33.37:31712
- **Status:** Instance appears unreachable at time of analysis
- **Challenge:** PWN category, FILE structure manipulation

## Vulnerability Summary
The challenge contains a critical vulnerability where user input is written directly into stdout and stderr FILE structures, allowing complete control over standard I/O operations.

## Key Files Generated
1. **WALKTHROUGH.md** - Comprehensive guide for Kali Linux exploitation
2. **SOLUTION.md** - Detailed technical analysis and theoretical exploitation
3. **exploit.py** - Advanced Python exploit using pwntools
4. **simple_exploit.py** - Basic network-based exploitation script

## Quick Exploitation Steps
1. Connect to the instance: `nc 34.252.33.37 31712`
2. Send crafted FILE structures to corrupt stdout/stderr
3. Leverage I/O redirection to read `/flag-<hash>.txt`
4. Extract flag through manipulated output operations

## Flag Location
Based on Dockerfile analysis: `/flag-<md5hash>.txt`

## Exploitation Techniques Required
- FILE structure layout knowledge
- FSOP (File Stream Oriented Programming)
- glibc internals understanding
- I/O redirection techniques

## Tools Needed
- GDB with GEF for analysis
- pwntools for exploit development
- Understanding of x86_64 assembly
- Knowledge of glibc FILE structures

The challenge demonstrates advanced PWN techniques requiring deep system-level knowledge.