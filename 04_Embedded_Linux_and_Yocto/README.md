# 04. Embedded Linux & Yocto Project

## Overview

This section covers system-level embedded software development with Linux, including kernel programming, Yocto build system, and Linux system programming.

## Contents

### 4.1 Embedded Linux Basics
- Kernel space vs user space architecture
- Boot process and bootloaders
- Device tree configuration
- Kernel modules and device drivers

### 4.2 Yocto Project (Build System)
- Yocto Project concepts and architecture
- Metadata and recipe development
- Layer management
- Custom image building

### 4.3 Linux System Programming
- POSIX API and system calls
- Inter-process communication
- Process and thread management
- System initialization

## Learning Objectives

By the end of this section, you should be able to:
- Understand embedded Linux architecture
- Build custom Linux distributions with Yocto
- Write Linux kernel modules and drivers
- Develop user-space applications for embedded Linux
- Configure and customize the boot process
- Create Board Support Packages (BSPs)

## Prerequisites

- Strong C programming skills
- Linux command-line proficiency
- Understanding of operating system concepts
- Familiarity with build systems (make, CMake)

## When to Use Embedded Linux

**Use Embedded Linux when:**
- Need networking, filesystem, or complex middleware
- Processor has MMU (ARM Cortex-A, x86, RISC-V with MMU)
- Sufficient resources (32MB+ RAM, 8MB+ Flash minimum)
- Open-source ecosystem benefits outweigh complexity

**Don't use Embedded Linux when:**
- Hard real-time requirements (<1ms response)
- Very resource-constrained (use RTOS instead)
- Simple, deterministic application
- Battery-powered with strict power budgets

## Resources

- Linux kernel documentation
- Yocto Project documentation
- Device tree specifications
- Embedded Linux primers
