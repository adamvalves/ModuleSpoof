# Architecture & Execution Mechanics

## PE Redirection & In-Memory Relocation

`ModuleSpoof` bifurcates execution into two discrete memory regions: a legitimate-appearing decoy module mapping and an isolated, manually mapped executable image.

```text
[ External Scanner / Inspection Routine ]
                 │
                 ▼
  Calls GetModuleHandle / GetProcAddress / Disk Verification
                 │
                 ├───────────────────────────────┐
                 ▼ (Intercepted APIs)            ▼ (Direct Disk / Memory Scan)
   [ Intercepted Resolution Layer ]      [ Decoy Honeypot Module ]
                 │                               │
                 ▼                               ▼ (Trips Guard / Integrity Trap)
     Points to Decoy Module Base        [ Alerts Security Monitor ]
                 │
                 ▼
[ Active Executable Memory (Manually Mapped) ]
  - Base relocations recomputed to private VAs
  - PE headers sanitized or stripped
  - Execution redirected through stealth detours
```

## Manual PE Mapping Sequence

1. **Header Parsing**: Validates `IMAGE_DOS_SIGNATURE` (`MZ`) and `IMAGE_NT_SIGNATURE` (`PE\0\0`).
2. **Virtual Memory Allocation**: Allocates memory using `VirtualAlloc` with `PAGE_EXECUTE_READWRITE` sized to `OptionalHeader.SizeOfImage`.
3. **Section Mapping**: Iterates across `IMAGE_SECTION_HEADER` array, mapping each section from file offset `PointerToRawData` to in-memory offset `VirtualAddress`.
4. **Base Relocation**: Parses the `IMAGE_DIRECTORY_ENTRY_BASERELOC` table. Calculates delta:
   `delta = (ULONG_PTR)MappedBase - NtHeaders->OptionalHeader.ImageBase`
   Iterates through relocation blocks, applying `IMAGE_REL_BASED_DIR64` or `IMAGE_REL_BASED_HIGHLOW` fixups.
5. **Import Resolution**: Iterates across `IMAGE_DIRECTORY_ENTRY_IMPORT`, loading dependency modules via `LoadLibraryA` and resolving function virtual addresses via `GetProcAddress`.

## Honeypot Decoy Trap Architecture

The honeypot module deploys page-level protection traps (`PAGE_NOACCESS` or `PAGE_GUARD`) across dummy export entries. Any external agent, scanner, or debugger attempting to dump, read, or set hardware breakpoints on the decoy address triggers a structured exception (`STATUS_GUARD_PAGE_VIOLATION`), exposing the inspection attempt.
