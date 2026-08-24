# Getting Started with Ring Buffer CAN

Welcome! This guide will help you get up and running with Ring Buffer CAN.

## Prerequisites

- C compiler (GCC, Clang, or MSVC)
- Make or CMake build system
- Git for cloning the repository

## Installation

### Clone the Repository

```bash
git clone https://github.com/totalcentralinasdev/ring_buffer_can.git
cd ring_buffer_can
```

### Build the Project

Using Make:
```bash
make
```

Or using CMake:
```bash
mkdir build
cd build
cmake ..
make
```

## Basic Usage

### Creating a Ring Buffer

```c
#include "ring_buffer.h"

int main() {
    // Create a ring buffer with capacity for 256 messages
    ring_buffer_t *buffer = ring_buffer_create(256);
    
    if (buffer == NULL) {
        return -1;
    }
    
    // ... use buffer ...
    
    // Clean up
    ring_buffer_destroy(buffer);
    return 0;
}
```

### Writing Data

```c
// Write a CAN message to the buffer
can_message_t msg = {
    .id = 0x123,
    .dlc = 8,
    .data = {0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08}
};

int result = ring_buffer_write(buffer, &msg);
if (result == 0) {
    // Message written successfully
}
```

### Reading Data

```c
can_message_t msg;

int result = ring_buffer_read(buffer, &msg);
if (result == 0) {
    // Message read successfully
    printf("ID: 0x%X, DLC: %d\n", msg.id, msg.dlc);
}
```

## Next Steps

- Read the [API Documentation](API-Documentation) for detailed function reference
- Check out the [Examples](Examples) for more code samples
- Review the [Architecture](Architecture) to understand the design
- See [FAQ](FAQ) for common questions and troubleshooting

## Troubleshooting

### Build Fails
- Ensure you have a C compiler installed
- Check that all dependencies are available
- Try cleaning the build: `make clean` then `make`

### Buffer Operations Return Errors
- Verify the buffer was created successfully
- Check that you're not reading from an empty buffer
- Ensure the buffer hasn't been destroyed before use

## Getting Help

If you encounter issues:
1. Check the [FAQ](FAQ) page
2. Review the [Examples](Examples) for reference implementations
3. Open an issue on the project repository

---

**Last Updated:** August 24, 2026