---
categories: Tech
---
# Operating Systems Notes

## Background

This is a series of notes on Georgia Tech's course- CS 6200, Introduction to Operating Systems.

I'm trying to create these notes from memory based upon the lectures I watched in an attempt to practice information retrieval and thus improve my ability to retain information.

##Processes
An operating system (OS) process is the representation of an executing program on a machine.  The manifestation of a process on a computer involves the process's "footprint" in memory.  

## Process representation and creation
Basically, the creation of a process involves the allocation of the footprint in memory by the OS.  The components of the process in memory include the stack, heap, data, and text of code.  

### Virtual Address Space and Page Table
When a process is created, the operating system assigns to a process a virtual address space which consists of a contiguous range of addresses that the process is aware of. The OS also creates a mapping from the virtual addresses to the underlying physical addresses via a page table.  All parts of the process live at a particular address within the virtual address space.  The virtual address space is from the process's point of view unique to it.  It doesn't have to worry about sharing the virtual address space with other processes, for example. During its execution, the process may refer to variables defined by its executing program that live at a particular virtual address.  In order to look up the values for the variables, the process will look up the virtual address of the variable and retrieve it by that memory address. The OS will have to resolve the physical address of that variable by looking at the page table for the process, and then accessing the physical memory indicated by the page table, in order for the process to be able to resolve the variable's value.

### Process Control Blocks (PCBs)
The OS maintains a summary of each of its processes within a process control block.  The data stored within a PCB includes things such as process state, process id, program counter, registers, memory limits, and open files.

## Process State
The process state diagram is shown: 
![alt text](https://github.com/carrpet/carrpet.github.io/blob/main/assets/img/diagram-of-process-state.jpg "Process State Diagram")
