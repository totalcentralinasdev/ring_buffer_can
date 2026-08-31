# Ring Buffer CAN

## Overview

A **Ring Buffer** (also called a Circular Buffer) is a fixed-size data structure that uses a single, continuous block of memory arranged in a circular fashion. It's optimized for CAN (Controller Area Network) message queuing, providing O(1) time complexity for both enqueue and dequeue operations.

This implementation is specifically designed for STM32 microcontrollers using the HAL (Hardware Abstraction Layer) library to handle CAN bus communication efficiently.

## Key Characteristics

| Feature | Details |
|---------|---------|
| **Size** | Fixed capacity of 64 messages |
| **Structure** | Circular FIFO (First-In-First-Out) |
| **Time Complexity** | O(1) for all operations |
| **Space Complexity** | O(1) |
| **Message Format** | CAN standard (up to 8 bytes per message) |
| **Thread Safety** | Not thread-safe (requires external synchronization) |

## Data Structures

### struct can_queue

Represents a single CAN message stored in the buffer:

```c
struct can_queue {
    uint32_t TX_MAILBOX;
    CAN_TxHeaderTypeDef header;
    uint8_t tx_data[8];
};
```

### struct ring

The main ring buffer container:

```c
struct ring {
    uint32_t head;
    uint32_t tail;
    uint32_t counter;
    struct can_queue tx_queue[MAX_SIZE];
};
```

Constants:

```c
#define MAX_SIZE 64
```

## How It Works

### Circular Pointer Logic

The buffer uses modulo arithmetic to wrap pointers around, creating a circular structure:

```
New position = (current position + 1) % MAX_SIZE
```

Example Flow:

```
Buffer size: 5
Positions: 0 1 2 3 4 0 1 2 3 4 (repeats)

head = 0 -> push message -> head = (0 + 1) % 5 = 1
head = 1 -> push message -> head = (1 + 1) % 5 = 2
...
head = 4 -> push message -> head = (4 + 1) % 5 = 0 (wraps around)
```

### Buffer State Tracking

The counter variable tracks how many messages are in the buffer:

- Empty: counter == 0 (no messages)
- Partially Full: 0 < counter < MAX_SIZE (space available)
- Full: counter >= MAX_SIZE (cannot accept new messages)

## Core Functions

### 1. Initialization

```c
void can_buffer_init(struct ring *ring_buffer)
```

Purpose: Initialize a ring buffer for use.

Behavior:
- Sets head = 0
- Sets tail = 0
- Sets counter = 0

Example:

```c
struct ring my_buffer;
can_buffer_init(&my_buffer);
```

### 2. Push (Enqueue)

```c
void can_buffer_push(struct ring *ring_buffer, CAN_TxHeaderTypeDef tx_header, uint8_t data[8])
```

Purpose: Add a new CAN message to the buffer.

Parameters:
- ring_buffer: Pointer to the ring buffer structure
- tx_header: CAN message header containing ID, DLC, and flags
- data[8]: Message payload data (8 bytes)

Behavior:
- Checks if buffer is full (counter >= MAX_SIZE)
- If full, silently returns (message is not added)
- If space available:
  - Copies the header and data to the buffer at head position
  - Advances head pointer: head = (head + 1) % MAX_SIZE
  - Increments counter

Example:

```c
CAN_TxHeaderTypeDef msg_header;
msg_header.IDE = CAN_ID_STD;
msg_header.StdId = 0x123;
msg_header.DLC = 8;

uint8_t msg_data[8] = {0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08};

can_buffer_push(&my_buffer, msg_header, msg_data);
```

### 3. Pop (Dequeue)

```c
void can_buffer_pop(struct ring *ring_buffer)
```

Purpose: Remove a message from the buffer and transmit it via CAN hardware.

Behavior:
1. Checks if buffer is empty (counter == 0)
   - If empty, silently returns
2. Checks if CAN hardware has free TX mailboxes
   - If no mailboxes available, returns (message stays in queue)
3. Transmits the message at tail position using HAL function
4. On successful transmission:
   - Advances tail pointer: tail = (tail + 1) % MAX_SIZE
   - Decrements counter
5. On transmission failure, message remains in buffer for retry

Hardware Requirements:
- Requires CAN_HandleTypeDef hcan2 (STM32 CAN2 peripheral)
- Uses HAL functions:
  - HAL_CAN_GetTxMailboxesFreeLevel(): Check mailbox availability
  - HAL_CAN_AddTxMessage(): Send message to hardware

Example:

```c
can_buffer_pop(&my_buffer);

while (my_buffer.counter > 0) {
    can_buffer_pop(&my_buffer);
}
```

## Performance Analysis

### Time Complexity

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| can_buffer_init() | O(1) | Simple initialization |
| can_buffer_push() | O(1) | Fixed memory copy (8 bytes) |
| can_buffer_pop() | O(1) | Hardware call + pointer update |

### Space Complexity

- Fixed: O(1) - Always uses exactly MAX_SIZE entries
- Per Message: approximately 20 bytes
- Total Buffer: approximately 1.28 KB for 64 messages

## Use Cases

### 1. Message Buffering During Transmission

When the CAN hardware TX mailboxes are temporarily full, queue messages and transmit them as soon as mailboxes become available.

```c
void CAN_RxISR() {
    CAN_TxHeaderTypeDef header;
    uint8_t data[8];
    can_buffer_push(&tx_queue, header, data);
}

void main() {
    can_buffer_init(&tx_queue);
    while (1) {
        can_buffer_pop(&tx_queue);
    }
}
```

### 2. Rate Limiting

Prevent flooding the CAN bus by buffering messages and transmitting at a controlled rate.

```c
void TIM_Callback() {
    can_buffer_pop(&tx_queue);
}
```

### 3. Priority Queue Simulation

Insert messages into the queue in priority order for FIFO transmission.

### 4. Interrupt Safety

Store CAN messages received in high-priority interrupts for processing in low-priority tasks.

## Important Notes and Limitations

### WARNING: Not Thread-Safe

This implementation is NOT thread-safe. If multiple threads access the buffer:
- Use a MUTEX to protect all operations:

```c
pthread_mutex_lock(&buffer_mutex);
can_buffer_push(&my_buffer, header, data);
pthread_mutex_unlock(&buffer_mutex);
```

- Or use a semaphore-based producer/consumer pattern

### WARNING: Fixed Capacity

- Cannot grow dynamically
- Messages are SILENTLY DROPPED if the buffer is full
- Monitor counter to prevent overflow:

```c
if (my_buffer.counter >= MAX_SIZE - 1) {
    // Buffer nearly full - handle overflow condition
}
```

### WARNING: Hardware Dependent

- Requires STM32 HAL and CAN2 peripheral
- Not portable to other platforms without modification
- Depends on proper CAN hardware initialization

### BENEFIT: Deterministic Timing

- All operations complete in O(1) time
- Predictable interrupt latency
- Suitable for real-time systems

### BENEFIT: Memory Efficient

- No dynamic allocation (no fragmentation)
- Fixed memory footprint
- Efficient for embedded systems

## STM32 HAL Integration

This implementation is tightly integrated with STM32 HAL:

Required:

```c
extern CAN_HandleTypeDef hcan2;
#include "main.h"
```

Initialization in main():

```c
MX_CAN2_Init();
struct ring tx_queue;
can_buffer_init(&tx_queue);
```

TX Mailbox Check:
The pop function checks mailbox availability:

```c
if (HAL_CAN_GetTxMailboxesFreeLevel(&hcan2) > 0) {
    // Proceed with transmission
}
```

## Best Practices

1. Initialize Before Use

```c
can_buffer_init(&my_buffer);
```

2. Monitor Buffer Fullness

```c
if (my_buffer.counter > 50) {
    printf("Buffer nearly full!\n");
}
```

3. Regularly Drain the Queue
- Call can_buffer_pop() frequently in main loop or timer interrupt
- Don't let buffer fill up completely

4. Handle Errors Gracefully

```c
if (my_buffer.counter >= MAX_SIZE) {
    // Implement overflow handling
}
```

5. Protect in Multi-threaded Environments

```c
sem_wait(&buffer_semaphore);
can_buffer_push(&my_buffer, header, data);
sem_post(&buffer_semaphore);
```

## Troubleshooting

### Messages Not Being Transmitted
- Cause: can_buffer_pop() not being called frequently enough
- Solution: Call pop() in main loop or faster interrupt handler

### Buffer Overflow (Messages Dropped)
- Cause: Messages being pushed faster than popped
- Solution:
  - Increase pop frequency
  - Reduce message push rate
  - Increase MAX_SIZE (requires code recompilation)

### HAL_CAN_AddTxMessage() Fails
- Cause: No TX mailboxes available
- Solution: Messages stay in queue and will be transmitted when mailboxes free up

### Stale Messages in Queue
- Cause: can_buffer_pop() not called regularly
- Solution: Ensure pop is called in timer interrupt or high-priority task

## Related Documentation

- Getting Started - Quick setup guide
- API Documentation - Detailed API reference
- Examples - Code examples and use cases
- Architecture - Design and architecture overview
- FAQ - Common questions and answers

---

Last Updated: August 24, 2026
Status: Complete
Maintained By: TC-Desenvolvimento
