<!--
SPDX-FileCopyrightText: 2023 Rivos Inc.

SPDX-License-Identifier: Apache-2.0
-->

# CoVE (RISC-V Confidential VM Extension)

[![REUSE status](https://api.reuse.software/badge/github.com/rivosinc/cove)](https://api.reuse.software/info/github.com/rivosinc/cove)

Architecture:
====================
The CoVE APIs are designed to be implementation and architecture agnostic,
allowing for different deployment models while retaining common host and guest
kernel code. Two examples are shown in Figure 1 and Figure 2.
As shown in both figures, the architecture introduces a new software component
called the "TEE Security Manager" (TSM) that runs in HS mode. The TSM has minimal
hw attested footprint on TCB as it is a passive component that doesn't support
scheduling or interrupts. Both example deployment models provide memory isolation
between the host and the TVM.

```

        
	Non secure world       |         Secure world         |
                               |                              |
        Non                    |                              |
    Virtualized |  Virtualized |   Virtualized  Virtualized   |            
        Env     |      Env     |       Env          Env       |                
   +----------+ | +----------+ |  +----------+ +----------+   |  --------------        
   |          | | |          | |  |          | |          |   |  
   | Host Apps| | |   Apps   | |  |   Apps   | |   Apps   |   |        VU-Mode
   |  (VMM)   | | |          | |  |          | |          |   |         
   +----------+ | +----------+ |  +----------+ +----------+   |  --------------
        |       | +----------+ |  +----------+ +----------+   |                
        |       | |          | |  |          | |          |   |      
        |       | |          | |  |    TVM   | |    TVM   |   |      
        |       | |   Guest  | |  |   Guest  | |   Guest  |   |       VS-Mode
     Syscalls   | +----------+ |  +----------+ +----------+   |      
        |              |       |        |                     |
        |             SBI      |   SBI(COVG + COVI)           |   
        |              |       |        |                     |
  +--------------------------+ |  +---------------------------+  --------------
  |     Host (Linux)         | |  |       TSM (Salus)         |        
  +--------------------------+ |  +---------------------------+
             |                 |            |                       HS-Mode
     SBI (COVH + COVI)         |     SBI (COVH + COVI)            
             |                 |            |
  +-----------------------------------------------------------+  --------------
  |                    Firmware(OpenSBI) + TSM Driver         |        M-Mode
  +-----------------------------------------------------------+  --------------
 +-----------------------------------------------------------------------------
  |                    Hardware (RISC-V CPU + RoT + IOMMU)
  +---------------------------------------------------------------------------- 
 		Figure 1: Host in HS model


``` 
The deployment model shown in Figure 1 runs the host in HS mode where it is peer
to the TSM which also runs in HS mode. It requires another component known as TSM
Driver running in higher privilege mode than host/TSM. It is responsible for switching
the context between the host and the TSM. TSM driver also manages the platform
specific hardware solution via confidential domain bit as described in the specification[0]
to provide the required memory isolation.

If a platform consider M-mode as part of TCB, TSM driver can just be part of the
firmware running in M-mode. However, it is likely that entire M-mode may not be
part of TCB. Thus, TSM driver should run in carved out a secure world in M-mode
while the firmware runs in non-secure world. The detailed proposal to achieve that is
still under discussion.

```
 
	     Non secure world  |         Secure world
                               |
         Virtualized Env       |   Virtualized   Virtualized  |                  
             		              Env           Env       |                       
   +-------------------------+ |  +----------+  +----------+  |    ------------ 
   |          | | |          | |  |          |  |          |  |                           
   | Host Apps| | |   Apps   | |  |   Apps   |  |   Apps   |  |        VU-Mode              
   +----------+ | +----------+ |  +----------+  +----------+  |    ------------ 
        |                      |        |             |       |                          
    Syscalls             SBI   |      	|             |       |                           
        |                      |        |             |       |                           
  +--------------------------+ |  +-----------+ +-----------+ |                          
  |     Host (Linux)         | |  |  TVM Guest| |  TVM Guest| |       VS-Mode                
  +--------------------------+ |  +-----------+ +-----------+ |               
             |                 |        |             |       |               
     SBI (COVH + COVI)         |       SBI           SBI      |              
             |                 |   (COVG + COVI) (COVG + COVI)|              
	     |                 |        |             |       |              
  +-----------------------------------------------------------+    --------------
  |                    TSM(Salus)	                      |        HS-Mode
  +-----------------------------------------------------------+    --------------
 			      | 
  			     SBI
			      |
  +---------------------------------------------------------+    --------------
  |                    Firmware(OpenSBI)                  |        M-Mode
  +---------------------------------------------------------+    --------------
 +-----------------------------------------------------------------------------
  |                    Hardware (RISC-V CPU + RoT + IOMMU)
  +---------------------------------------------------------------------------- 
 			Figure 2: Host in VS model

```

The deployment model shown in Figure 2 simplfies the context switch and memory isolation
by running the host in VS mode as a guest of TSM. Thus, the memory isolation is
achieved by gstage mapping by the TSM. We don't need any additional hardware confidential
domain bit to provide memory isolation. The downside of this model the host has to run the
non-confidential VMs in nested environment which may have lower performance (yet to be measured).
The current implementation Salus(TSM) doesn't support full nested virtualization yet.

The platform must have a RoT to provide attestation in either model.
This patch series implements the APIs defiend by CoVE. The current TSM implementation
allows the host to run TVMs as shown in figure 2. We are working on deployment
model 1 in parallel. We do not expect any significant changes in the host/guest side
due to that.

Shared memory between the host & TSM:
=====================================
To accelerate the H-mode CSR/GPR access, CoVE also reuses the Nested Acceleration (NACL)
SBI extension[1]. NACL defines a per physical cpu shared memory area that is allocated
at the boot. It allows the host running in VS mode to access H-mode CSR/GPR easily
without trap/emulate by TSM. The CoVE specification clearly defines the exact
state of the shared memory with r/w permissions at every call. 

Secure Interrupt management:
===========================
The CoVE specification relies on MSI based interrupt scheme defined in Advanced Interrupt
Architecture specification[2]. The COVI SBI extension adds functions to bind
a guest interrupt file to a TVMs. After that, only TCB components (TSM, TVM, TSM driver)
can modify that. The host can inject an interrupt via TSM only. 
The TVMs are also in complete control of which interrupts it can recieve. By default,
all interrupts are denied. In this proof-of-concept implementation, all the interrupts
are allowed by the guest at boot time to keep it simple.

Device I/O: 
===========
In order to support paravirt I/O devices, SWIOTLB bounce buffer must be used by the
guest. As the host can not access confidential memory, this buffer memory
must be shared with the host via share/unshare functions defined in COVG SBI specification.
RISC-V implementation achieves this generalizing mem_encrypt_init() similar to TDX/SEV/CCA.
That's why, the CoVE Guest is only allowed to use virtio devices with VIRTIO_F_ACCESS_PLATFORM
and VIRTIO_F_VERSION_1 as they force virtio drivers to use the DMA API.

MMIO emulation:
======================
TVM can register regions of address space as MMIO regions to be emulated by
the host. TSM provides explict SBI functions i.e. SBI_EXT_COVG_ADD_MMIO_REGION and 
SBI_EXT_COVG_REMOVE_MMIO_REGION to request/remove MMIO regions. Any reads or
writes to the MMIO region after SBI_EXT_COVG_ADD_MMIO_REGION call are forwarded
to the host for emulation. 

This series allows any ioremapped memory to be emulated as MMIO region with
above APIs via arch hookups inspired from pKVM work. We are aware that this model
doesn't address all the threat vectors. We have also implemented the device 
filtering/authorization approach adopted by TDX[4]. However, we have dropped those
patches for now as it is being redesigned as well. RISC-V CoVE will also adapt
the revamped device filtering work once it is accepted by the Linux community
in the future.

The direct assignment of devices are a work in progress and will be added in the future[4].

VMM support:
============
This series only tested with kvmtool support. Other VMM support (qemu-kvm, crossvm/rust-vmm)
will be added later.

Test cases:
===========
We are working on kvm selftest for CoVE. We will post them as soon as they are ready.
We haven't started any work on kvm unit-tests as RISC-V doesn't have basic infrastrcture
to support that. Once the kvm uni-test infrastructure is in place, we will add
support for CoVE as well.

Running the stack
====================

To run/test the stack, you would need the following components :

1) Qemu
2) Common Host & Guest Kernel
3) kvmtool
4) Host RootFS with KVMTOOL and Guest Kernel
5) Salus

The detailed steps are available [here](https://github.com/rivosinc/cove/wiki/CoVE-KVM-RISCV64-on-QEMU)

TODOs
=======
As this is a very early work, the todo list is quite long :).
Here are some of them (not in any specific order)

1. Support fd based private memory interface proposed in
   https://lkml.org/lkml/2022/1/18/395
2. Align with updated guest runtime device filtering approach.
3. IOMMU integration
4. Dedicated device assignment via TDSIP & SPDM[4]
5. Support huge pages
6. Page pool allocator to avoid convert/reclaim at every fault
7. Other VMM support (qemu-kvm, crossvm)
8. Complete the PoC for the deployment model 1 where host runs in HS mode
9. Attestation integration
10. Harden the interrupt allowed list
11. kvm self-tests support for CoVE
11. kvm unit-tests support for CoVE
12. Guest hardening
13. Port pKVM on RISC-V using CoVE
14. Any other ?

Links
============

[0] CoVE architecture Specification.
    https://github.com/riscv-non-isa/riscv-ap-tee/blob/main/specification/riscv-aptee-spec.pdf

[1] NACL SBI Extension

[2] https://github.com/riscv/riscv-aia/releases/download/1.0-RC2/riscv-interrupts-1.0-RC2.pdf 

[3] https://github.com/rivosinc/linux/tree/cove_integration_device_filtering1

[4] https://github.com/intel/tdx/commits/guest-filter-upstream 
