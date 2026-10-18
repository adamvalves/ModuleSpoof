# Contributing Guidelines

Contributions to `ModuleSpoof` must adhere to modern C++20 systems programming standards.

## Engineering Guidelines

1. **Language & Dialect**: Modern C++20 (MSVC v143).
2. **Resource Management**: Strictly adhere to RAII. Cleanly close handles (`CloseHandle`) and free mapped memory (`VirtualFree`) upon error paths.
3. **PE Parser Robustness**:
   - Every pointer offset calculated from PE headers (`IMAGE_DOS_HEADER`, `IMAGE_NT_HEADERS`, `IMAGE_SECTION_HEADER`) must perform bounds checks against the buffer length before dereference.
4. **Style & Aesthetics**:
   - Zero emojis in commit messages, PRs, comments, or documentation.
   - All documentation in formal technical English.
