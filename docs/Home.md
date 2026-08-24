# Ring Buffer CAN

Welcome to the **Ring Buffer CAN** project! This repository provides a circular buffer implementation designed for efficient queuing of CAN (Controller Area Network) messages in embedded systems.

## Overview

The Ring Buffer CAN is a FIFO (First-In-First-Out) circular buffer implementation optimized for managing CAN bus transmit messages. It's particularly useful in embedded systems where you need to queue multiple CAN messages and transmit them asynchronously.

### Key Features

- **Circular FIFO Buffer**: Efficiently manages a fixed-size queue of CAN messages
- **Fixed Buffer Size**: Pre-allocated 64-entry buffer for predictable memory usage
- **Safe Operations**: Handles buffer full and empty conditions gracefully
- **CAN Message Support**: Stores complete CAN messages including headers and 8-byte data payloads
- **STM32 HAL Compatible**: Integrates seamlessly with STM32 CubeMX HAL libraries

## Project Structure

```
ring_buffer_can/
├── ring_buffer.h          # Header file with data structures and function declarations
├── ring_buffer.c          # Implementation of ring buffer operations
└── docs/
    └── Home.md           # This file
```

## Core Components

### Data Structures

#### `can_queue`
Represents a single CAN message in the buffer:
```c
struct can_queue {
    uint32_t TX_MAILBOX;              // CAN transmit mailbox ID
    CAN_TxHeaderTypeDef header;       // CAN message header
    uint8_t tx_data[8];               // 8-byte CAN data payload
};
```

#### `ring`
The main ring buffer structure:
```c
struct ring {
    uint32_t head;                    // Write pointer
    uint32_t tail;                    // Read pointer
    uint32_t counter;                 // Number of messages in buffer
    struct can_queue tx_queue[MAX_SIZE];  // Message queue (MAX_SIZE = 64)
};
```

### API Functions

#### `void can_buffer_init(struct ring *ring_buffer)`
Initializes the ring buffer to an empty state.
- **Parameters**: Pointer to the ring buffer structure
- **Sets**: head, tail, and counter to 0

#### `void can_buffer_push(struct ring *ring_buffer, CAN_TxHeaderTypeDef tx_header, uint8_t data[8])`
Adds a CAN message to the buffer.
- **Parameters**: 
  - `ring_buffer`: Pointer to the ring buffer
  - `tx_header`: CAN message header
  - `data`: 8-byte message data
- **Behavior**: 
  - Returns silently if buffer is full (counter >= MAX_SIZE)
  - Advances head pointer in circular fashion
  - Increments counter

#### `void can_buffer_pop(struct ring *ring_buffer)`
Removes the oldest message from the buffer and transmits it via CAN.
- **Parameters**: Pointer to the ring buffer
- **Behavior**:
  - Returns silently if buffer is empty (counter == 0)
  - Checks for available CAN mailboxes
  - Transmits message if mailbox available
  - Advances tail pointer only on successful transmission
  - Decrements counter

## Usage Example

```c
// 1. Declare and initialize the ring buffer
struct ring my_can_buffer;
can_buffer_init(&my_can_buffer);

// 2. Push CAN messages to the buffer
CAN_TxHeaderTypeDef header;
uint8_t data[8] = {0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08};
header.StdId = 0x123;
header.IDE = CAN_ID_STD;
header.RTR = CAN_RTR_DATA;
header.DLC = 8;

can_buffer_push(&my_can_buffer, header, data);

// 3. Pop messages to transmit them
// (Can be called from a timer interrupt or polling loop)
can_buffer_pop(&my_can_buffer);
```

## Design Patterns

### Circular Buffer Operation
The buffer uses modulo arithmetic to wrap pointers around:
```c
head = (head + 1) % MAX_SIZE;  // Circular increment
tail = (tail + 1) % MAX_SIZE;
```

### Back-pressure Handling
- **Push**: Returns silently if buffer is full, preventing message loss notification
- **Pop**: Only advances tail pointer after successful CAN transmission

### Mailbox Checking
Before transmitting, the implementation checks for available CAN mailboxes using STM32 HAL functions to avoid transmission failures.

## Configuration

### Buffer Size
The maximum buffer size is defined by the `MAX_SIZE` constant (currently set to 64 messages):
```c
#define MAX_SIZE 64
```

To change the buffer size, modify this constant in `ring_buffer.h`.

## Hardware Requirements

- **STM32 Microcontroller** with CAN interface
- **CAN Transceiver** (e.g., MCP2551, TJA1051)
- **Proper CAN Termination** (120Ω resistors)

## Integration Notes

- This implementation assumes you have STM32 CubeMX HAL initialized with CAN2 (`hcan2`)
- The `main.h` header file is expected to contain necessary STM32 definitions
- Compatible with standard CAN communication at various baud rates

## Thread Safety

**Note**: This implementation is not thread-safe. If using an RTOS or multiple execution contexts, add appropriate synchronization mechanisms (mutexes, critical sections) around buffer operations.

## Future Enhancements

- Thread-safe implementation with mutex support
- Support for multiple CAN peripherals
- Configurable buffer size at initialization time
- Return status codes from operations
- Statistics tracking (overflow count, transmission success rate)

---

**Author**: TC-Desenvolvimento  
**Created**: April 22, 2026  
**Language**: C
