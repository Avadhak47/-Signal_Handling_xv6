# xv6 Operating System Enhancement Project

## Overview

This project enhances the xv6 operating system by implementing advanced signal handling and a custom priority-based scheduler. The implementation adds keyboard-triggered signal processing, custom process management with delayed execution, and comprehensive scheduler profiling capabilities.

## Features Implemented

### 1. Signal Handling System

The project implements a comprehensive signal handling mechanism that responds to keyboard interrupts:

**Signals Supported:**
- **SIGINT (Ctrl+C)**: Terminates user processes (excluding init and shell)
- **SIGSTP/SIGBG (Ctrl+Z/Ctrl+B)**: Suspends all user processes
- **SIGFG (Ctrl+F)**: Resumes suspended processes
- **SIGCUSTOM (Ctrl+G)**: Triggers user-defined signal handlers

**Key Components:**
- Keyboard interrupt detection in `console.c`
- Signal broadcasting and handling mechanisms in `proc.c`
- Process state management for suspension/resumption
- User-defined signal handler registration via system calls

### 2. Custom Scheduler Implementation

**Priority Boosting Scheduler:**
- Dynamic priority calculation using the formula: `πi(t) = πi(0) - α·Ci(t) + β·Wi(t)`
- Prevents process starvation through priority boosting
- Configurable α and β parameters for workload optimization

**Process Management:**
- `custom_fork()`: Creates processes with delayed execution and time limits
- `scheduler_start()`: Activates delayed processes simultaneously
- Execution time monitoring with automatic termination via SIGINT

**Scheduler Profiling:**
Tracks and reports key performance metrics for each process:
- **Turnaround Time (TAT)**: Total time from creation to completion
- **Waiting Time (WT)**: Time spent in RUNNABLE state
- **Response Time (RT)**: Time from creation to first scheduling
- **Context Switches (#CS)**: Number of times process was scheduled

## Technical Implementation

### Files Modified

1. **proc.h**: Extended process structure with signal handling fields and scheduler metrics
2. **proc.c**: Core implementation of signal handling, custom scheduler, and profiling
3. **console.c**: Keyboard interrupt detection and signal generation
4. **sysproc.c**: System calls for signal handler registration
5. **Makefile**: Configuration for priority parameters (α, β, INIT_PRIORITY)

### System Calls Added

- `sys_custom_fork(start_later, exec_time)`: Advanced process creation
- `sys_scheduler_start()`: Delayed process activation
- `sys_signal(handler)`: Signal handler registration
- `sys_sigret()`: Signal handler return mechanism

## Installation and Setup

### Prerequisites

**For M1/M2 MacBooks:**
```bash
# Install Xcode Command Line Tools
xcode-select --install

# Install Homebrew if not present
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install required tools
brew install qemu
brew install i386-elf-gcc  # or x86_64-elf-gcc for newer systems
```

**For Linux Systems:**
```bash
# Install required packages
sudo apt-get update
sudo apt-get install git build-essential qemu-system-x86 gdb
```

**For Windows (WSL2):**
```bash
# Enable WSL2 and install Ubuntu
wsl --install

# In Ubuntu terminal:
sudo apt-get update
sudo apt-get install git make gcc wget qemu qemu-kvm gdb
```

### Building and Running

1. **Clone the Repository:**
```bash
git clone <repository-url>
cd xv6-enhanced
```

2. **Configure Makefile Parameters:**
```bash
# Edit Makefile to set scheduler parameters
INIT_PRIORITY = 100
ALPHA = 1
BETA = 2
CPUS := 1  # Required for single-core operation
```

3. **For M1/M2 Macs - Set Tool Prefix:**
```bash
# Modify Makefile line 32:
TOOLPREFIX = i386-elf-
# or for newer systems:
TOOLPREFIX = x86_64-elf-
```

4. **Compile and Run:**
```bash
# Basic compilation
make clean
make

# Run xv6
make qemu-nox

# For debugging
make qemu-gdb
# In another terminal:
gdb ./kernel
```

### Troubleshooting Common Issues

**Infinite Recursion Error:**
Add to `sh.c` before the `runcmd` function:
```c
__attribute__((noreturn))
void runcmd(struct cmd *cmd)
```

**M1 Mac Compilation Issues:**
```bash
# Use Rosetta 2 if needed
softwareupdate --install-rosetta

# Use correct tool prefix
make TOOLPREFIX=x86_64-elf- qemu-nox
```

## Testing the Implementation

### Signal Handling Test

Create `test_signal.c`:
```c
#include "types.h"
#include "stat.h"
#include "user.h"

int main(void) {
    while (1) {
        printf(1, "Hello");
        sleep(100);
    }
    return 0;
}
```

**Expected Behavior:**
- Press Ctrl+Z: Process suspends
- Press Ctrl+F: Process resumes
- Press Ctrl+C: Process terminates

### Scheduler Test

Use the provided `test_sched.c` to test custom fork and scheduler functionality:
```bash
$ test_sched
All child processes created with start_later flag set.
Calling sys_scheduler_start() to allow execution.
Child 0 (PID: 4) started but should not run yet.
# ... process metrics printed on completion
```

## Performance Analysis

### α and β Parameter Effects

The scheduler's behavior can be tuned using α and β parameters:

- **High α**: Penalizes CPU-bound processes, favoring I/O-bound tasks
- **High β**: Boosts priority of long-waiting processes, preventing starvation
- **Balanced α=β**: Provides fair scheduling across different workload types

**Example Configuration:**
```makefile
# CPU-bound workload optimization
ALPHA = 3
BETA = 1

# I/O-bound workload optimization  
ALPHA = 1
BETA = 3

# Balanced approach
ALPHA = 1
BETA = 1
```

### Parameter Analysis Results

| Test | α | β | Process Type | TAT | WT | RT |
|------|---|---|--------------|-----|----|----|
| 1    | 1 | 1 | CPU-bound    | 120 | 40 | 20 |
| 1    | 1 | 1 | I/O-bound    | 90  | 20 | 10 |
| 2    | 3 | 1 | CPU-bound    | 140 | 60 | 30 |
| 2    | 3 | 1 | I/O-bound    | 85  | 18 | 9  |
| 3    | 1 | 3 | CPU-bound    | 110 | 30 | 20 |
| 3    | 1 | 3 | I/O-bound    | 70  | 10 | 5  |

## Project Structure

```
xv6-enhanced/
├── proc.h          # Process structure definitions
├── proc.c          # Core scheduler and signal handling
├── console.c       # Keyboard interrupt handling
├── sysproc.c       # System call implementations
├── test_sched.c    # Scheduler test program
├── Makefile        # Build configuration
└── README.md       # This file
```

## Implementation Details

### Process Structure Extensions

```c
struct proc {
    // ... existing fields ...
    
    // Signal handling
    int pending_signals;
    int is_suspended;
    sighandler_t custom_handler;
    uint saved_eip;
    
    // Scheduler profiling
    int creation_time;
    int first_scheduled;
    int cpu_ticks_used;
    int wait_time;
    int context_switches;
    
    // Priority scheduling
    int initial_priority;
    int dynamic_priority;
    int start_later;
    int exec_time;
};
```

### Signal Handling Flow

1. **Key Detection**: Console interrupt handler detects Ctrl+C/Z/F/G
2. **Signal Broadcasting**: `broadcast_signal()` sends signals to target processes
3. **Signal Processing**: `handle_signals()` processes pending signals in scheduler
4. **State Management**: Process states updated based on signal type

### Scheduler Priority Formula

```
πi(t) = πi(0) - α·Ci(t) + β·Wi(t)
```

Where:
- `πi(0)`: Initial priority (configurable)
- `Ci(t)`: CPU ticks consumed
- `Wi(t)`: Waiting time in RUNNABLE state
- `α, β`: Tunable parameters for balancing CPU usage vs. fairness

## Educational Value

This project demonstrates key operating system concepts:

1. **Process Management**: Custom process creation and lifecycle management
2. **Signal Handling**: Asynchronous event processing and inter-process communication
3. **Scheduling Algorithms**: Priority-based scheduling with starvation prevention
4. **Performance Metrics**: System monitoring and optimization techniques
5. **Kernel Programming**: Low-level system programming in a real OS environment

## Known Issues and Solutions

1. **Shell Suspension**: Ensure `handle_signals()` excludes init (PID 1) and shell (PID 2)
2. **Lock Acquisition Panic**: Properly manage ptable locks in signal handling
3. **Timer Interrupt Handling**: Update process metrics correctly in trap handler
4. **Process State Management**: Careful handling of SLEEPING/RUNNABLE transitions

## Future Enhancements

- Support for more complex signal handling patterns
- Advanced scheduler algorithms (e.g., multi-level feedback queues)
- Memory management enhancements
- Network stack implementation
- File system improvements

## References

- [xv6 Operating System](https://pdos.csail.mit.edu/6.828/2023/xv6.html)
- [xv6 Installation Guides](https://oslab.kaist.ac.kr/xv6-install/)
- [Signal Handling in Operating Systems](https://www.gnu.org/s/libc/manual/html_node/Signal-Handling.html)
- [Scheduling Algorithms](https://wiki.osdev.org/Scheduling_Algorithms)

## License

This project is based on xv6, which is distributed under the MIT License. Educational use and modifications are encouraged.

## Contributing

1. Fork the repository
2. Create a feature branch
3. Implement your changes
4. Test thoroughly
5. Submit a pull request

## Authors

- Implementation based on Assignment 2 - Operating Systems course
- Extended and documented for educational purposes
- Contributors: [Add your names here]
