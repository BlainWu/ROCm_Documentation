
.. _ROCm-Native-ISA:



GCN Native ISA LLVM Code Generator
===================================


    * :ref:`Introductio`
    * :ref:`LLVM`
       *  :ref:`Target-Triples`
       *  :ref:`Processors`
       *  :ref:`Address-Spaces`
       *  :ref:`Memory-Scopes`
       *  :ref:`AMDGPU-Intrinsics`
    * :ref:`Code-Object`
       *  :ref:`Header`
       *  :ref:`Sections`
       *  :ref:`Note Records`
       *  :ref:`Symbols`
       *  :ref:`Relocation Records`
       *  :ref:`DWARF`
           * :ref:`Address Space Mapping`
           * :ref:`Register Mapping`
           * :ref:`Source Text`
    * :ref:`Code Conventions`
       * :ref:`AMDHSA`
           *  :ref:`Code Object Metadata`
           *  :ref:`Kernel Dispatch`
           *  :ref:`Memory Spaces`
           *  :ref:`Image and Samplers`
           *  :ref:`HSA Signals`
           *  :ref:`HSA AQL Queue`
           *  :ref:`Kernel Descriptor`
           *  :ref:`Kernel Descriptor for GFX6-GFX9`
           *  :ref:`Initial Kernel Execution State`
           *  :ref:`Kernel Prolog`
               *  :ref:`M0`
               *  :ref:`Flat-Scratch`
           * :ref:`Memory Model`
           * :ref:`Trap Handler ABI1`
       * :ref:`AMDPAL`
           * :ref:`User-Data`
           * :ref:`Compute-User-Data`
           * :ref:`Graphics-User-Data`
           * :ref:`Global-Internal-Table`
       * :ref:`Unspecified OS`
           * :ref:`Trap Handler ABI2`
    * :ref:`Source Languages`
       * :ref:`OpenCL`
       * :ref:`HCC`
       * :ref:`Assembler`
           * :ref:`Instructions`
           * :ref:`Operans`
           * :ref:`Modifers`
           * :ref:`Instruction Examples`
               * :ref:`DS`
               * :ref:`FLAT`
               * :ref:`MUBUF`
               * :ref:`SMRD/SMEM`
               * :ref:`SOP1`
               * :ref:`SOP2`
               * :ref:`SOPC`
               * :ref:`SOPP`
               * :ref:`VALU`
         * :ref:`Code Object V2 Predefined Symbols (-mattr=-code-object-v3)`
               * :ref:`.option.machine_version_major`
               * :ref:`.option.machine_version_minor`
               * :ref:`.option.machine_version_stepping`
               * :ref:`.kernel.vgpr_count`
               * :ref:`.kernel.sgpr_count`
         * :ref:`Code Object V2 Directives (-mattr=-code-object-v3)`
               * :ref:`.hsa_code_object_version major, minor`
               * :ref:`.hsa_code_object_isa [major, minor, stepping, vendor, arch]`
               * :ref:`.amdgpu_hsa_kernel (name)`
               * :ref:`.amd_kernel_code_t`
         * :ref:`Code Object V2 Example Source Code (-mattr=-code-object-v3)`
         * :ref:`Code Object V3 Predefined Symbols (-mattr=+code-object-v3)`
               * :ref:`.amdgcn.gfx_generation_number`
               * :ref:`.amdgcn.gfx_generation_minor`
               * :ref:`.amdgcn.gfx_generation_stepping`
               * :ref:`.amdgcn.next_free_vgpr`
               * :ref:`.amdgcn.next_free_sgpr`
        * :ref:`Code Object V3 Directives (-mattr=+code-object-v3)`
               * :ref:`.amdgcn_target`
               * :ref:`.amdhsa_kernel`
               * :ref:`.amdgpu_metadata`
        * :ref:`Code Object V3 Example Source Code (-mattr=+code-object-v3)`
        * :ref:`Additional Documentation`
               

.. _Introductio:

Introduction
#############
The AMDGPU backend provides ISA code generation for AMD GPUs, starting with the R600 family up until the current GCN families. It lives in the lib/Target/AMDGPU directory.

.. _LLVM:

LLVM
#####

.. _Target-Triples:

Target Triples
---------------
Use the clang -target <Architecture>-<Vendor>-<OS>-<Environment> option to specify the target triple:

    **AMDGPU Target Triples**
============== ======= ======== ==============
 Architecture 	Vendor 	OS 	Environment
============== ======= ======== ==============
    r600 	amd 	<empty>  <empty>
    amdgcn 	amd 	<empty>  <empty>
    amdgcn 	amd 	amdhsa 	 <empty>
    amdgcn 	amd 	amdhsa 	 opencl
    amdgcn 	amd 	amdhsa 	 amdgizcl
    amdgcn 	amd 	amdhsa 	 amdgiz
    amdgcn 	amd 	amdhsa 	 hcc
============== ======= ======== ==============

r600-amd--
    Supports AMD GPUs HD2XXX-HD6XXX for graphics and compute shaders executed on the MESA runtime.

amdgcn-amd--
    Supports AMD GPUs GCN GFX6 onwards for graphics and compute shaders executed on the MESA runtime.

amdgcn-amd-amdhsa-
    Supports AMD GCN GPUs GFX6 onwards for compute kernels executed on HSA [HSA] compatible runtimes such as AMD’s ROCm [AMD-ROCm].

amdgcn-amd-amdhsa-opencl
    Supports AMD GCN GPUs GFX6 onwards for OpenCL compute kernels executed on HSA [HSA] compatible runtimes such as AMD’s ROCm 	    	[AMD-ROCm]. See OpenCL.

amdgcn-amd-amdhsa-amdgizcl
    Same as amdgcn-amd-amdhsa-opencl except a different address space mapping is used (see Address Spaces).

amdgcn-amd-amdhsa-amdgiz
    Same as amdgcn-amd-amdhsa- except a different address space mapping is used (see Address Spaces).

amdgcn-amd-amdhsa-hcc
    Supports AMD GCN GPUs GFX6 onwards for AMD HC language compute kernels executed on HSA [HSA] compatible runtimes such as AMD’s  	ROCm [AMD-ROCm]. See HCC.

.. _Processors:

Processors
------------
Use the clang -mcpu <Processor> option to specify the AMD GPU processor. The names from both the Processor and Alternative Processor can be used.

**AMDGPU Processors Processor**

==================================== =========== ================ ============== ================== ================================ 
 Processor 		  	  	      	   Triple 
					    	   Architecture     dGPU/ APU 	   Runtime Support    Example Products
==================================== =========== ================ ============== ================== ================================ 
   **R600**
    r600 	  			 		r600 		dGPU 	  	 
    r630 	  					r600 		dGPU 	  	 
    rs880 	  					r600 		dGPU 	  	 
    rv670 	  					r600 		dGPU 	  	 
   **R700**
    rv710 	  					r600 		dGPU 	  	 
    rv730 	  					r600 		dGPU 	  	 
    rv770 	  					r600 		dGPU 	  	 
   **Evergreen**	
    cedar 	  					r600 		dGPU 	  	 
    redwood 	  					r600 		dGPU 	  	 
    sumo 	  					r600 		dGPU 	  	 
    juniper 	  					r600 		dGPU 	  	 
    cypress 	  					r600 		dGPU 	  	 
**Northern Islands**	
    barts 	  					r600 		dGPU 	  	 
    turks 	  					r600 		dGPU 	  	 
    caicos 	  					r600 		dGPU 	  	 
    cayman 	  					r600 		dGPU 	  	 
**GCN GFX6(Southern Islands (SI))**   
    gfx600 				tahiti	       amdgcn 		dGPU 	

    gfx601 			      * pitcairn		
       				      * verde
        			      * oland
        			      * hainan
**GCN GFX7 (Sea Islands (CI))**
   gfx700 			      * bonaire        amdgcn 		dGPU 	   			* Radeon HD 7790	
  													* Radeon HD 8770
   													* R7 260
  													* R7 260X
  
   gfx700  			      *	kaveri	       amdgcn 		APU 	  			* A6-7000
													* A6 Pro-7050B
   												        * A8-7100
    													* A8 Pro-7150B
    													* A10-7300
    													* A10 Pro-7350B
    													* FX-7500
   													* A8-7200P
    													* A10-7400P
    													* FX-7600P	

   gfx701 			      * hawaii	       amdgcn 		dGPU 		ROCm 		* FirePro W8100
   													* FirePro W9100
    													* FirePro S9150
    													* FirePro S9170

   gfx702 	  	  						dGPU 		ROCm 		* Radeon R9 290
													* Radeon R9 290x
  													* Radeon R390
   													* Radeon R390x
  gfx703 			      * kabini		amdgcn 		APU 				*  E1-2100

    				      *	mullins								*  E1-2200
   													*  E1-2500
   													*  E2-3000
   													*  E2-3800
   													*  A4-5000
   													*  A4-5100
													*  A6-5200
 GCN GFX8 (Volcanic Islands (VI))
   gfx800 			      * iceland		amdgcn 		dGPU 	 			* FirePro S7150
													* FirePro S7100
    													* FirePro W7100
 	    												* Radeon R285
   													* Radeon R9 380
   													* Radeon R9 385
    													* Mobile FirePro M7170

 gfx801 			      * carrizo		amdgcn 		APU 	  			* A6-8500P
   													* Pro A6-8500B
   													* A8-8600P
    													* Pro A8-8600B
   													* FX-8800P
    												        * Pro A12-8800B
							amdgcn 		APU 		ROCm 		* A10-8700P
    													* Pro A10-8700B
    													* A10-8780P
						        amdgcn 		APU 	  		        * A10-9600P	
													* A10-9630P
													* A12-9700P
													* A12-9730P
													* FX-9800P
													* FX-9830P
							amdgcn 		APU 	   			* E2-9010
												        * A6-9210
    													* A9-9410	
  
  gfx802 			     * tonga 	 	amdgcn 		dGPU 		ROCm 		Same as gfx800

  gfx803 	                     * fiji    		amdgcn 		dGPU 		ROCm 	        * Radeon R9 Nano
    													* Radeon R9 Fury
    												 	* Radeon R9 FuryX
    													* Radeon Pro Duo
    													* FirePro S9300x2
   													* Radeon Instinct MI8

				     * polaris10 	amdgcn 		dGPU 		ROCm 		* Radeon RX 470	
  				     * polaris11 	amdgcn 		dGPU 		ROCm 		* Radeon RX 460
 gfx804 	  					amdgcn 		dGPU 	  			  Same as gfx803
 gfx810 			     * stoney 		amdgcn 		APU


 **GCN GFX9 [AMD-Vega]**
 gfx900 	  					amdgcn 		dGPU 	  		     * Readeon vega Frontieredition
												     * Radeon RX Vega 56
												     * Radeon RX Vega 64
												     * Radeon RX Vega 64 Liquid
												     * Radeon Instinct MI25

 gfx901 	  					amdgcn 		dGPU 		ROCm 	    Same as gfx900 except 
												    XNACK is enabled
 gfx902 	  					amdgcn 		APU 	  			  TBA
 gfx903 	  					amdgcn 		APU 	  		     Same as gfx902 except
												     XNACK is enabled


==================================== =========== ================ ============== ================== ================================ 
 
       

    	
    
  

.. _Address-Spaces:

Address Spaces
----------------

The AMDGPU backend uses the following address space mappings.

The memory space names used in the table, aside from the region memory space, is from the OpenCL standard.

LLVM Address Space number is used throughout LLVM (for example, in LLVM IR).

**Address Space Mapping**


			Memory Space
=================== =================== ====================== ================== ===================
LLVM Address Space    Current Default 	  amdgiz/amdgizcl 	   hcc 	   	    Future Default
=================== =================== ====================== ================== ===================
    0 	   	     Private (Scratch) 	 Generic (Flat) 	Generic (Flat) 	   Generic (Flat)
    1 	    	     Global 		 Global 		Global 		   Global
    2 	   	     Constant 		 Constant 		Constant 	   Region (GDS)
    3 	    	     Local (group/LDS) 	 Local (group/LDS) 	Local (group/LDS)  Local (group/LDS)
    4 	    	     Generic (Flat) 	 Region (GDS) 		Region (GDS) 	   Constant
    5 	    	     Region (GDS) 	 Private (Scratch) 	Private (Scratch)  Private (Scratch)
=================== =================== ====================== ================== ===================

Current Default
    This is the current default address space mapping used for all languages except hcc. This will shortly be deprecated.
amdgiz/amdgizcl
    This is the current address space mapping used when amdgiz or amdgizcl is specified as the target triple environment value.
hcc
    This is the current address space mapping used when hcc is specified as the target triple environment value.This will shortly be deprecated.
Future Default
    This will shortly be the only address space mapping for all languages using AMDGPU backend.

.. _Memory-Scopes:

Memory Scopes
--------------

This section provides LLVM memory synchronization scopes supported by the AMDGPU backend memory model when the target triple OS is amdhsa (see Memory Model and Target Triples).

The memory model supported is based on the HSA memory model [HSA] which is based in turn on HRF-indirect with scope inclusion [HRF]. The happens-before relation is transitive over the synchonizes-with relation independent of scope, and synchonizes-with allows the memory scope instances to be inclusive (see table AMDHSA LLVM Sync Scopes for AMDHSA).

This is different to the OpenCL [OpenCL] memory model which does not have scope inclusion and requires the memory scopes to exactly match. However, this is conservatively correct for OpenCL.

    **AMDHSA LLVM Sync Scopes for AMDHSA LLVM Sync Scope** 	
================   =================================================================================================================  
LLVM Sync Scope 	Description
================   =================================================================================================================  
none 		      The default: system.
		      Synchronizes with, and participates in modification and seq_cst total orderings with, other operations (except 			      image operations) for all address spaces (except private, or generic that accesses private) provided the other 			      operation’s sync scope is:
    			* system.
    			* agent and executed by a thread on the same agent.
    			* workgroup and executed by a thread in the same workgroup.
   			* wavefront and executed by a thread in the same wavefront.

agent 		     Synchronizes with, and participates in modification and seq_cst total orderings with, other operations (except 			     image operations) for all address spaces (except private, or generic that accesses private) provided the other 			     operation’s sync scope is:
			* system or agent and executed by a thread on the same agent.
    			* workgroup and executed by a thread in the same workgroup.
   			* wavefront and executed by a thread in the same wavefront.

workgroup 	     Synchronizes with, and participates in modification and seq_cst total orderings with, other operations (except 			     image operations) for all address spaces (except private, or generic that accesses private) provided the other 			     operation’s sync scope is:
			* system, agent or workgroup and executed by a thread in the same workgroup.
   			* wavefront and executed by a thread in the same wavefront.

wavefront            Synchronizes with, and participates in modification and seq_cst total orderings with, other operations (except 			     image operations) for all address spaces (except private, or generic that accesses private) provided the other 			     operation’s sync scope is:
			* system, agent, workgroup or wavefront and executed by a thread in the same wavefront.

singlethread 	     Only synchronizes with, and participates in modification and seq_cst total orderings with, other operations (except 	     image operations) running in the same thread for all address spaces (for example, in signal handlers).

================   =================================================================================================================  





.. _AMDGPU-Intrinsics:

AMDGPU Intrinsics
------------------

The AMDGPU backend implements the following intrinsics.

This section is WIP.

 .. _Code-Object:

Code Object
#############
The AMDGPU backend generates a standard ELF [ELF] relocatable code object that can be linked by lld to produce a standard ELF shared code object which can be loaded and executed on an AMDGPU target.

.. _Header:

Header
-------
The AMDGPU backend uses the following ELF header:

   **AMDGPU ELF Header**

=========================== ===================================
 	  Field 			Value
=========================== ===================================
    e_ident[EI_CLASS] 		ELFCLASS64
    e_ident[EI_DATA] 		ELFDATA2LSB
    e_ident[EI_OSABI] 		ELFOSABI_AMDGPU_HSA
    e_ident[EI_ABIVERSION] 	ELFABIVERSION_AMDGPU_HSA
    e_type 			ET_REL or ET_DYN
    e_machine 			EM_AMDGPU
    e_entry 			0
    e_flags 			0
=========================== ===================================

    **AMDGPU ELF Header Enumeration Values**
    
========================== ===============
 Name 			             Value
========================== ===============
EM_AMDGPU                                224
LFOSABI_AMDGPU_HSA 		           64
ELFABIVERSION_AMDGPU_HSA                  1
========================== ===============

e_ident[EI_CLASS]
    The ELF class is always ELFCLASS64. The AMDGPU backend only supports 64 bit applications.

e_ident[EI_DATA]
    All AMDGPU targets use ELFDATA2LSB for little-endian byte ordering.

e_ident[EI_OSABI]
    The AMD GPU architecture specific OS ABI of ELFOSABI_AMDGPU_HSA is used to specify that the code object conforms to the AMD HSA runtime ABI [HSA].

e_ident[EI_ABIVERSION]
    The AMD GPU architecture specific OS ABI version of ELFABIVERSION_AMDGPU_HSA is used to specify the version of AMD HSA runtime ABI to which the code object conforms.

e_type

    Can be one of the following values:
    ET_REL
        The type produced by the AMD GPU backend compiler as it is relocatable code object.
    ET_DYN
        The type produced by the linker as it is a shared code object.

    The AMD HSA runtime loader requires a ET_DYN code object.

e_machine
    The value EM_AMDGPU is used for the machine for all members of the AMD GPU architecture family. The specific member is specified in the NT_AMD_AMDGPU_ISA entry in the .note section (see Note Records).
e_entry
    The entry point is 0 as the entry points for individual kernels must be selected in order to invoke them through AQL packets.

e_flags
    The value is 0 as no flags are used.


.. _Sections:

Sections
---------

An AMDGPU target ELF code object has the standard ELF sections which include:

    **AMDGPU ELF Sections** 
    
=============== ================ ====================================
Name 		     Type			Attributes
=============== ================ ====================================
    .bss 	SHT_NOBITS 	  SHF_ALLOC + SHF_WRITE
    .data 	SHT_PROGBITS 	  SHF_ALLOC + SHF_WRITE
    .debug_* 	SHT_PROGBITS 	  none
    .dynamic 	SHT_DYNAMIC 	  SHF_ALLOC
    .dynstr 	SHT_PROGBITS 	  SHF_ALLOC
    .dynsym 	SHT_PROGBITS 	  SHF_ALLOC
    .got 	SHT_PROGBITS 	  SHF_ALLOC + SHF_WRITE
    .hash 	SHT_HASH 	  SHF_ALLOC
    .note 	SHT_NOTE 	  none
    .relaname 	SHT_RELA 	  none
    .rela.dyn 	SHT_RELA 	  none
    .rodata 	SHT_PROGBITS 	  SHF_ALLOC
    .shstrtab 	SHT_STRTAB 	  none
    .strtab 	SHT_STRTAB 	  none
    .symtab 	SHT_SYMTAB 	  none
    .text 	SHT_PROGBITS 	  SHF_ALLOC + SHF_EXECINSTR
=============== ================ ====================================

These sections have their standard meanings and are only generated if needed.

.debug*
    The standard DWARF sections. See DWARF for information on the DWARF produced by the AMDGPU backend.

.dynamic, .dynstr, .dynsym, .hash
    The standard sections used by a dynamic loader.

.note
    See Note Records for the note records supported by the AMDGPU backend.

.relaname, .rela.dyn
    For relocatable code objects, name is the name of the section that the relocation records apply. For example, .rela.text is the section name for relocation records associated with the .text section.
    For linked shared code objects, .rela.dyn contains all the relocation records from each of the relocatable code object’s .relaname sections.
    See Relocation Records for the relocation records supported by the AMDGPU backend.

.text
    The executable machine code for the kernels and functions they call. Generated as position independent code. See Code Conventions for information on conventions used in the isa generation.

.. _Note Records:

Note Records
--------------

As required by ELFCLASS64, minimal zero byte padding must be generated after the name field to ensure the desc field is 4 byte aligned. In addition, minimal zero byte padding must be generated to ensure the desc field size is a multiple of 4 bytes. The sh_addralign field of the .note section must be at least 4 to indicate at least 8 byte alignment.

The AMDGPU backend code object uses the following ELF note records in the .note section. The Description column specifies the layout of the note record’s desc field. All fields are consecutive bytes. Note records with variable size strings have a corresponding *_size field that specifies the number of bytes, including the terminating null character, in the string. The string(s) come immediately after the preceding fields.

Additional note records can be present.

				**AMDGPU ELF Note Records**
		
================ ============================== ========================================== 
   Name  	       Type 		                 Description
================ ============================== ========================================== 
 “AMD” 	           NT_AMD_AMDGPU_HSA_METADATA 	  <metadata null terminated string>
 “AMD” 	           NT_AMD_AMDGPU_ISA 		      <isa name null terminated string>
================ ============================== ========================================== 




	**AMDGPU ELF Note Record Enumeration Values**
============================= ==================
Name			            Value
============================= ================== 
reserved 		               0-9
NT_AMD_AMDGPU_HSA_METADATA   	     10
NT_AMD_AMDGPU_ISA	               11
============================= ==================



NT_AMD_AMDGPU_ISA

    Specifies the instruction set architecture used by the machine code contained in the code object.

    This note record is required for code objects containing machine code for processors matching the amdgcn architecture in table    	  Processors.

    The null terminated string has the following syntax:

        architecture-vendor-os-environment-processor

    where:

        ``architecture``
           The architecture from table AMDGPU Target Triples. 
           This is always amdgcn when the target triple OS is amdhsa (see Target Triples).

        ``vendor``
           The vendor from table AMDGPU Target Triples.
           For the AMDGPU backend this is always amd.

        ``OS``
           The OS from table AMDGPU Target Triples.
        
        ``environment``
           An environment from table AMDGPU Target Triples, or blank if the environment has no affect on the execution of the code 		   object.
           For the AMDGPU backend this is currently always blank.
   	
        ``processor``
           The processor from table AMDGPU Processors.

    For example::
        
        amdgcn-amd-amdhsa--gfx901


``NT_AMD_AMDGPU_HSA_METADATA``

    Specifies extensible metadata associated with the code objects executed on HSA [HSA] compatible runtimes such as AMD’s ROCm [AMD-ROCm]. It is required when the target triple OS is amdhsa (see Target Triples). See Code Object Metadata for the syntax of the code object metadata string.

.. _Symbols:

Symbols
--------

Symbols include the following:

    **AMDGPU ELF Symbols**
    
+----------------+------------+-----------+--------------------+
| Name           | Type       | Section   | Description        |
+================+============+===========+====================+
| `link-name`    | STT_OBJECT | * .data   | Global variable    |
+                +            +           +                    +
|                |            | * .rodata |                    |
+                +            +           +                    +
|                |            | * .bss    |                    |
+----------------+------------+-----------+--------------------+
| `link-name@kd` | STT_OBJECT | * .rodata | Kernel descriptor  |
+----------------+------------+-----------+--------------------+
| `link-name`    | STT_FUNC   | * .text   | Kernel entry point |
+----------------+------------+-----------+--------------------+

Global variable

    Global variables both used and defined by the compilation unit.

    If the symbol is defined in the compilation unit then it is allocated in the appropriate section according to if it has initialized data or is readonly.

    If the symbol is external then its section is STN_UNDEF and the loader will resolve relocations using the definition provided by another code object or explicitly defined by the runtime.

    All global symbols, whether defined in the compilation unit or external, are accessed by the machine code indirectly through a GOT table entry. This allows them to be preemptable. The GOT table is only supported when the target triple OS is amdhsa (see Target Triples).


Kernel descriptor

    Every HSA kernel has an associated kernel descriptor. It is the address of the kernel descriptor that is used in the AQL dispatch packet used to invoke the kernel, not the kernel entry point. The layout of the HSA kernel descriptor is defined in Kernel Descriptor.

Kernel entry point

    Every HSA kernel also has a symbol for its machine code entry point.


.. _Relocation Records:

Relocation Records
-------------------
AMDGPU backend generates Elf64_Rela relocation records. Supported relocatable fields are:

word32
    This specifies a 32-bit field occupying 4 bytes with arbitrary byte alignment. These values use the same byte order as other word values in the AMD GPU architecture.

word64
    This specifies a 64-bit field occupying 8 bytes with arbitrary byte alignment. These values use the same byte order as other word values in the AMD GPU architecture.

Following notations are used for specifying relocation calculations:

**A**
    Represents the addend used to compute the value of the relocatable field.
**G**
    Represents the offset into the global offset table at which the relocation entry’s symbol will reside during execution.
**GOT**
    Represents the address of the global offset table.
**P**
    Represents the place (section offset for et_rel or address for et_dyn) of the storage unit being relocated (computed using r_offset).
**S**
    Represents the value of the symbol whose index resides in the relocation entry.

The following relocation types are supported:


		    AMDGPU ELF Relocation Records
+------------------------+-------+--------+--------------------------------+
| Relocation Type        | Value | Field  | Calculation                    |
+========================+=======+========+================================+
| R_AMDGPU_NONE          | 0     | none   | none                           |
+------------------------+-------+--------+--------------------------------+
| R_AMDGPU_ABS32_LO      | 1     | word32 | (S + A) & 0xFFFFFFFF           |
+------------------------+-------+--------+--------------------------------+
| R_AMDGPU_ABS32_HI      | 2     | word32 | (S + A) >> 32                  |
+------------------------+-------+--------+--------------------------------+
| R_AMDGPU_ABS64         | 3     | word64 | S + A                          |
+------------------------+-------+--------+--------------------------------+
| R_AMDGPU_REL32         | 4     | word32 | S + A - P                      |
+------------------------+-------+--------+--------------------------------+
| R_AMDGPU_REL64         | 5     | word64 | S + A - P                      |
+------------------------+-------+--------+--------------------------------+
| R_AMDGPU_ABS32         | 6     | word32 | S + A                          |
+------------------------+-------+--------+--------------------------------+
| R_AMDGPU_GOTPCREL      | 7     | word32 | G + GOT + A - P                |
+------------------------+-------+--------+--------------------------------+
| R_AMDGPU_GOTPCREL32_LO | 8     | word32 | (G + GOT + A - P) & 0xFFFFFFFF |
+------------------------+-------+--------+--------------------------------+
| R_AMDGPU_GOTPCREL32_HI | 9     | word32 | (G + GOT + A - P) >> 32        |
+------------------------+-------+--------+--------------------------------+
| R_AMDGPU_REL32_LO      | 10    | word32 | (S + A - P) & 0xFFFFFFFF       |
+------------------------+-------+--------+--------------------------------+
| R_AMDGPU_REL32_HI      | 11    | word32 | (S + A - P) >> 32              |
+------------------------+-------+--------+--------------------------------+

 
.. _DWARF:

DWARF
------
Standard DWARF [DWARF] Version 2 sections can be generated. These contain information that maps the code object executable code and data to the source language constructs. It can be used by tools such as debuggers and profilers.

.. _Address Space Mapping:

Address Space Mapping
++++++++++++++++++++++
The following address space mapping is used:

		AMDGPU DWARF Address Space Mapping
======================== ========================		
 DWARF Address Space   	   Memory Space
======================== ========================
    1 			       Private (Scratch)
    2 			       Local (group/LDS)
    omitted 		       lobal
    omitted 		       Constant
    omitted 		       Generic (Flat)
    not supported 	       Region (GDS)
======================== ========================

See ``Address Spaces`` for information on the memory space terminology used in the table.

An address_class attribute is generated on pointer type DIEs to specify the DWARF address space of the value of the pointer when it is in the private or local address space. Otherwise the attribute is omitted.

An XDEREF operation is generated in location list expressions for variables that are allocated in the private and local address space. Otherwise no XDREF is omitted.

.. _Register Mapping:

Register Mapping
+++++++++++++++++
This section is WIP.


.. _Source Text:

Source Text
++++++++++++
This section is WIP.



.. _Code Conventions:

Code Conventions
#################
This section provides code conventions used for each supported target triple OS (see Target Triples).

.. _AMDHSA:

AMDHSA
-------
This section provides code conventions used when the target triple OS is amdhsa (see Target Triples).


.. _Code Object Metadata:

Code Object Metadata
+++++++++++++++++++++
The code object metadata specifies extensible metadata associated with the code objects executed on HSA [HSA] compatible runtimes such as AMD’s ROCm [AMD-ROCm]. It is specified by the NT_AMD_AMDGPU_HSA_METADATA note record (see Note Records) and is required when the target triple OS is amdhsa (see Target Triples). It must contain the minimum information necessary to support the ROCM kernel queries. For example, the segment sizes needed in a dispatch packet. In addition, a high level language runtime may require other information to be included. For example, the AMD OpenCL runtime records kernel argument information.

The metadata is specified as a YAML formatted string (see [YAML] and YAML I/O).

The metadata is represented as a single YAML document comprised of the mapping defined in table AMDHSA Code Object Metadata Mapping and referenced tables.

For boolean values, the string values of false and true are used for false and true respectively.

Additional information can be added to the mappings. To avoid conflicts, any non-AMD key names should be prefixed by “vendor-name.”.

    AMDHSA Code Object Metadata Mapping
+------------+------------------------+-----------+------------------------------------------------------------------------------------------------------------------------------------------------+
| String Key | Value Type             | Required? | Description                                                                                                                                    |
+============+========================+===========+================================================================================================================================================+
| “Version”  | sequence of 2 integers | Required  | * The first integer is the major version. Currently 1.                                                                                         |
|            |                        |           | * The second integer is the minor version. Currently 0.                                                                                        |
+------------+------------------------+-----------+------------------------------------------------------------------------------------------------------------------------------------------------+
| “Printf”   | sequence of strings    |           | Each string is encoded information about a printf function call.                                                                               |
|            |                        |           | The encoded information is organized as fields separated by colon                                                                              |
|            |                        |           |                                                                                                                                                |
|            |                        |           | (‘:’):ID:N:S[0]:S[1]:...:S[N-1]:FormatString                                                                                                   |
|            |                        |           |                                                                                                                                                |
|            |                        |           | where:                                                                                                                                         |
|            |                        |           | ID                                                                                                                                             |
|            |                        |           |      A 32 bit integer as a unique id for each printf function call                                                                             |
|            |                        |           | N                                                                                                                                              |
|            |                        |           |      A 32 bit integer equal to the number of arguments of printf function call minus 1                                                         |
|            |                        |           | S[i] (where i = 0, 1, ..., N-1)                                                                                                                |
|            |                        |           |      32 bit integers for the size in bytes of the i-th FormatString argument of the printf function call                                       |
|            |                        |           |                                                                                                                                                |
|            |                        |           | FormatString                                                                                                                                   |
|            |                        |           | The format string passed to the printf function call.                                                                                          |
+------------+------------------------+-----------+------------------------------------------------------------------------------------------------------------------------------------------------+
| “Kernels”  | sequence of mapping    | Required  | Sequence of the mappings for each kernel in the code object. See AMDHSA Code Object Kernel Metadata Mapping for the definition of the mapping. |
+------------+------------------------+-----------+------------------------------------------------------------------------------------------------------------------------------------------------+




**AMDHSA Code Object Kernel Metadata Mapping**

+-------------------+------------------------+-----------+----------------------------------------------------------------------------------------------------------------------------------------------------+
| String Key        | value Type             | Required? | Description                                                                                                                                        |
+===================+========================+===========+====================================================================================================================================================+
| “Name”            | string                 | Required  | Source name of the kernel.                                                                                                                         |
+-------------------+------------------------+-----------+----------------------------------------------------------------------------------------------------------------------------------------------------+
| “SymbolName”      | string                 | Required  | Name of the kernel descriptor ELF symbol.                                                                                                          |
+-------------------+------------------------+-----------+----------------------------------------------------------------------------------------------------------------------------------------------------+
| “Language”        | string                 |           | Source language of the kernel. Values include:                                                                                                     |
|                   |                        |           | * “OpenCL C”                                                                                                                                       |
|                   |                        |           | * “OpenCL C++”                                                                                                                                     |
|                   |                        |           | * “HCC”                                                                                                                                            |
|                   |                        |           | * “OpenMP”                                                                                                                                         |
+-------------------+------------------------+-----------+----------------------------------------------------------------------------------------------------------------------------------------------------+
| “LanguageVersion” | sequence of 2 integers |           | * The first integer is the major version.                                                                                                          |
|                   |                        |           | * The second integer is the minor version.                                                                                                         |
+-------------------+------------------------+-----------+----------------------------------------------------------------------------------------------------------------------------------------------------+
| “Attrs”           | mapping                |           | Mapping of kernel attributes. See AMDHSA Code Object Kernel Attribute Metadata Mapping for the mapping definition.                                 |
+-------------------+------------------------+-----------+----------------------------------------------------------------------------------------------------------------------------------------------------+
| “Arguments”       | sequence of mapping    |           | Sequence of mappings of the kernel arguments. See AMDHSA Code Object Kernel Argument Metadata Mapping for the definition of the mapping.           |
+-------------------+------------------------+-----------+----------------------------------------------------------------------------------------------------------------------------------------------------+
| “CodeProps”       | mapping                |           | Mapping of properties related to the kernel code. See AMDHSA Code Object Kernel Code Properties Metadata Mapping for the mapping definition.       |
+-------------------+------------------------+-----------+----------------------------------------------------------------------------------------------------------------------------------------------------+
| “DebugProps”      | mapping                |           | Mapping of properties related to the kernel debugging. See AMDHSA Code Object Kernel Debug Properties Metadata Mapping for the mapping definition. |
+-------------------+------------------------+-----------+----------------------------------------------------------------------------------------------------------------------------------------------------+





    **AMDHSA Code Object Kernel Attribute Metadata Mapping**

+---------------------+------------------------+-----------+-----------------------------------------------------------------------------+
| String Key          | Value Type             | Required? | Description                                                                 |
+=====================+========================+===========+=============================================================================+
| “ReqdWorkGroupSize” | sequence of 3 integers |           | The dispatch work-group size X,Y,Z must correspond to the specified values. |
|                     |                        |           | Corresponds to the OpenCL reqd_work_group_size attribute.                   |
+---------------------+------------------------+-----------+-----------------------------------------------------------------------------+
| “WorkGroupSizeHint” | sequence of 3 integers |           | The dispatch work-group size X,Y,Z is likely to be the specified values.    |
|                     |                        |           | Corresponds to the OpenCL work_group_size_hint attribute.                   |
+---------------------+------------------------+-----------+-----------------------------------------------------------------------------+
| “VecTypeHint”       | string                 |           | The name of a scalar or vector type.                                        |
|                     |                        |           | Corresponds to the OpenCL vec_type_hint attribute.                          |
+---------------------+------------------------+-----------+-----------------------------------------------------------------------------+


   
   
   **AMDHSA Code Object Kernel Argument Metadata Mapping**
   
   
+-----------------+------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| String Key      | Value Type | Required? | Description                                                                                                                                                                                                                                                                                                                                       |
+=================+============+===========+===================================================================================================================================================================================================================================================================================================================================================+
| “Name”          | string     |           | Kernel argument name.                                                                                                                                                                                                                                                                                                                             |
+-----------------+------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| “TypeName”      | string     |           | Kernel argument type name.                                                                                                                                                                                                                                                                                                                        |
+-----------------+------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| “Size”          | integer    | Required  | Kernel argument size in bytes.                                                                                                                                                                                                                                                                                                                    |
+-----------------+------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| “Align”         | integer    | Required  | Kernel argument alignment in bytes. Must be a power of two.                                                                                                                                                                                                                                                                                       |
+-----------------+------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| “ValueKind”     | string     | Required  | Kernel argument kind that specifies how to set up the corresponding argument. Values include :                                                                                                                                                                                                                                                    |
|                 |            |           |  “ByValue”                                                                                                                                                                                                                                                                                                                                        |
|                 |            |           |     The argument is copied directly into the kernarg.                                                                                                                                                                                                                                                                                             |
|                 |            |           |  “GlobalBuffer”                                                                                                                                                                                                                                                                                                                                   |
|                 |            |           |     A global address space pointer to the buffer data is passed in the kernarg.                                                                                                                                                                                                                                                                   |
|                 |            |           |  “DynamicSharedPointer”                                                                                                                                                                                                                                                                                                                           |
|                 |            |           |     A group address space pointer to dynamically allocated LDS is passed in the kernarg.                                                                                                                                                                                                                                                          |
|                 |            |           |  “Sampler”                                                                                                                                                                                                                                                                                                                                        |
|                 |            |           |     A global address space pointer to a S# is passed in the kernarg.                                                                                                                                                                                                                                                                              |
|                 |            |           |  “Image”                                                                                                                                                                                                                                                                                                                                          |
|                 |            |           |     A global address space pointer to a T# is passed in the kernarg.                                                                                                                                                                                                                                                                              |
|                 |            |           |  “Pipe”                                                                                                                                                                                                                                                                                                                                           |
|                 |            |           |     A global address space pointer to an OpenCL pipe is passed in the kernarg.                                                                                                                                                                                                                                                                    |
|                 |            |           |  “Queue”                                                                                                                                                                                                                                                                                                                                          |
|                 |            |           |     A global address space pointer to an OpenCL device enqueue queue is passed in the kernarg.                                                                                                                                                                                                                                                    |
|                 |            |           |  “HiddenGlobalOffsetX”                                                                                                                                                                                                                                                                                                                            |
|                 |            |           |     The OpenCL grid dispatch global offset for the X dimension is passed in the kernarg.                                                                                                                                                                                                                                                          |
|                 |            |           |  “HiddenGlobalOffsetY”                                                                                                                                                                                                                                                                                                                            |
|                 |            |           |     The OpenCL grid dispatch global offset for the Y dimension is passed in the kernarg.                                                                                                                                                                                                                                                          |
|                 |            |           |  “HiddenGlobalOffsetZ”                                                                                                                                                                                                                                                                                                                            |
|                 |            |           |     The OpenCL grid dispatch global offset for the Z dimension is passed in the kernarg.                                                                                                                                                                                                                                                          |
|                 |            |           |  “HiddenNone”                                                                                                                                                                                                                                                                                                                                     |
|                 |            |           |     An argument that is not used by the kernel. Space needs to be left for it, but it does not need to be set up.                                                                                                                                                                                                                                 |
|                 |            |           |  “HiddenPrintfBuffer”                                                                                                                                                                                                                                                                                                                             |
|                 |            |           |     A global address space pointer to the runtime printf buffer is passed in kernarg.                                                                                                                                                                                                                                                             |
|                 |            |           |  “HiddenDefaultQueue”                                                                                                                                                                                                                                                                                                                             |
|                 |            |           |     A global address space pointer to the OpenCL device enqueue queue that should be used by the kernel by default is passed in the kernarg.                                                                                                                                                                                                      |
|                 |            |           |  “HiddenCompletionAction”                                                                                                                                                                                                                                                                                                                         |
|                 |            |           |     TBD                                                                                                                                                                                                                                                                                                                                           |
+-----------------+------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| “ValueType”     | Value Type | Required  | Kernel argument value type. Only present if “ValueKind” is “ByValue”. For vector data types, the value is for the element type.Values include:                                                                                                                                                                                                    |
|                 |            |           |   * “Struct”                                                                                                                                                                                                                                                                                                                                      |
|                 |            |           |   * “I8”                                                                                                                                                                                                                                                                                                                                          |
|                 |            |           |   * “U8”                                                                                                                                                                                                                                                                                                                                          |
|                 |            |           |   * “I16”                                                                                                                                                                                                                                                                                                                                         |
|                 |            |           |   * “U16”                                                                                                                                                                                                                                                                                                                                         |
|                 |            |           |   * “F16”                                                                                                                                                                                                                                                                                                                                         |
|                 |            |           |   * “I32”                                                                                                                                                                                                                                                                                                                                         |
|                 |            |           |   * “U32”                                                                                                                                                                                                                                                                                                                                         |
|                 |            |           |   * “F32”                                                                                                                                                                                                                                                                                                                                         |
|                 |            |           |   * “I64”                                                                                                                                                                                                                                                                                                                                         |
|                 |            |           |   * “U64”                                                                                                                                                                                                                                                                                                                                         |
|                 |            |           |   * “F64”                                                                                                                                                                                                                                                                                                                                         |
+-----------------+------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| “PointeeAlign”  | integer    |           | Alignment in bytes of pointee type for pointer type kernel argument. Must be a power of 2. Only present if “ValueKind” is “DynamicSharedPointer”.                                                                                                                                                                                                 |
+-----------------+------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| “AddrSpaceQual” | string     |           | Kernel argument address space qualifier. Only present if “ValueKind” is “GlobalBuffer” or “DynamicSharedPointer”.Values are :                                                                                                                                                                                                                     |
|                 |            |           |   * “Private”                                                                                                                                                                                                                                                                                                                                     |
|                 |            |           |   * “Global”                                                                                                                                                                                                                                                                                                                                      |
|                 |            |           |   * “Constant”                                                                                                                                                                                                                                                                                                                                    |
|                 |            |           |   * “Local”                                                                                                                                                                                                                                                                                                                                       |
|                 |            |           |   * “Generic”                                                                                                                                                                                                                                                                                                                                     |
|                 |            |           |   * “Region”                                                                                                                                                                                                                                                                                                                                      |
+-----------------+------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| “AccQual”       | string     |           | Kernel argument access qualifier. Only present if “ValueKind” is “Image” or “Pipe”. Values are :                                                                                                                                                                                                                                                  |
|                 |            |           |   * “ReadOnly”                                                                                                                                                                                                                                                                                                                                    |
|                 |            |           |   * “WriteOnly”                                                                                                                                                                                                                                                                                                                                   |
|                 |            |           |   * “ReadWrite”                                                                                                                                                                                                                                                                                                                                   |
+-----------------+------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| “ActualAcc”     | string     |           | The actual memory accesses performed by the kernel on the kernel argument.Only present if “ValueKind” is “GlobalBuffer”, “Image”, or “Pipe”. This may be more restrictive than indicated by “AccQual” to reflect what the kernel actual does.If not present then the runtime must assume what is implied by “AccQual” and “IsConst”. Values are : |
|                 |            |           |   * “ReadOnly”                                                                                                                                                                                                                                                                                                                                    |
|                 |            |           |   * “WriteOnly”                                                                                                                                                                                                                                                                                                                                   |
|                 |            |           |   * “ReadWrite”                                                                                                                                                                                                                                                                                                                                   |
+-----------------+------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| “IsConst”       | boolean    |           | Indicates if the kernel argument is const qualified. Only present if “ValueKind” is “GlobalBuffer”.                                                                                                                                                                                                                                               |
+-----------------+------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| “IsRestrict”    | boolean    |           | Indicates if the kernel argument is restrict qualified. Only present if “ValueKind” is “GlobalBuffer”.                                                                                                                                                                                                                                            |
+-----------------+------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| “IsVolatile”    | boolean    |           | Indicates if the kernel argument is volatile qualified. Only present if “ValueKind” is “GlobalBuffer”.                                                                                                                                                                                                                                            |
+-----------------+------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| “IsPipe”        | boolean    |           | Indicates if the kernel argument is pipe qualified. Only present if “ValueKind” is “Pipe”.                                                                                                                                                                                                                                                        |
+-----------------+------------+-----------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+  
  
   **AMDHSA Code Object Kernel Code Properties Metadata Mapping**
   
   
+---------------------------+------------+-----------+--------------------------------------------------------------------------------------------------------------------------+
| String Key                | Value Type | Required? | Description                                                                                                              |
+===========================+============+===========+==========================================================================================================================+
| “KernargSegmentSize”      | integer    | Required  | The size in bytes of the kernarg segment that holds the values of the arguments to the kernel.                           |
+---------------------------+------------+-----------+--------------------------------------------------------------------------------------------------------------------------+
| “GroupSegmentFixedSize”   | integer    | Required  | The amount of group segment memory required by a work-group in bytes.                                                    |
|                           |            |           | This does not include any dynamically allocated group segment memory that may be added when the kernel is dispatched.    |
+---------------------------+------------+-----------+--------------------------------------------------------------------------------------------------------------------------+
| “PrivateSegmentFixedSize” | integer    | Required  | The amount of fixed private address space memory required for a work-item in bytes.                                      |
|                           |            |           |  If IsDynamicCallstack is 1 then additional space must be added to this value for the call stack.                        |
+---------------------------+------------+-----------+--------------------------------------------------------------------------------------------------------------------------+
| “KernargSegmentAlign”     | integer    | Required  | The maximum byte alignment of arguments in the kernarg segment. Must be a power of 2.                                    |
+---------------------------+------------+-----------+--------------------------------------------------------------------------------------------------------------------------+
| “WavefrontSize”           | integer    | Required  | Wavefront size. Must be a power of 2.                                                                                    |
+---------------------------+------------+-----------+--------------------------------------------------------------------------------------------------------------------------+
| “NumSGPRs”                | integer    |           | Number of scalar registers used by a wavefront for GFX6-GFX9.                                                            |
|                           |            |           | This includes the special SGPRs for VCC, Flat Scratch (GFX7-GFX9) and XNACK (for GFX8-GFX9).                             |
|                           |            |           |  It does not include the 16 SGPR added if a trap handler is enabled. It is not rounded up to the allocation granularity. |
+---------------------------+------------+-----------+--------------------------------------------------------------------------------------------------------------------------+
| “NumVGPRs”                | integer    |           | Number of vector registers used by each work-item for GFX6-GFX9                                                          |
+---------------------------+------------+-----------+--------------------------------------------------------------------------------------------------------------------------+
| “MaxFlatWorkgroupSize”    | integer    |           | Maximum flat work-group size supported by the kernel in work-items.                                                      |
+---------------------------+------------+-----------+--------------------------------------------------------------------------------------------------------------------------+
| “IsDynamicCallStack”      | boolean    |           | Indicates if the generated machine code is using a dynamically sized call stack.                                         |
+---------------------------+------------+-----------+--------------------------------------------------------------------------------------------------------------------------+
| “IsXNACKEnabled”          | boolean    |           | Indicates if the generated machine code is capable of supporting XNACK.                                                  |
+---------------------------+------------+-----------+--------------------------------------------------------------------------------------------------------------------------+
 
    **AMDHSA Code Object Kernel Debug Properties Metadata Mapping**
    
+-------------------------------------+------------+-----------+-------------+
| String Key                          | Value Type | Required? | Description |
+=====================================+============+===========+=============+
| “DebuggerABIVersion”                | string     |           |             |
+-------------------------------------+------------+-----------+-------------+
| “ReservedNumVGPRs”                  | integer    |           |             |
+-------------------------------------+------------+-----------+-------------+
| “ReservedFirstVGPR”                 | integer    |           |             |
+-------------------------------------+------------+-----------+-------------+
| “PrivateSegmentBufferSGPR”          | integer    |           |             |
+-------------------------------------+------------+-----------+-------------+
| “WavefrontPrivateSegmentOffsetSGPR” | integer    |           |             |
+-------------------------------------+------------+-----------+-------------+ 


.. _Kernel Dispatch:

Kernel Dispatch
++++++++++++++++
The HSA architected queuing language (AQL) defines a user space memory interface that can be used to control the dispatch of kernels, in an agent independent way. An agent can have zero or more AQL queues created for it using the ROCm runtime, in which AQL packets (all of which are 64 bytes) can be placed. See the HSA Platform System Architecture Specification [HSA] for the AQL queue mechanics and packet layouts.

The packet processor of a kernel agent is responsible for detecting and dispatching HSA kernels from the AQL queues associated with it. For AMD GPUs the packet processor is implemented by the hardware command processor (CP), asynchronous dispatch controller (ADC) and shader processor input controller (SPI).

The ROCm runtime can be used to allocate an AQL queue object. It uses the kernel mode driver to initialize and register the AQL queue with CP.

To dispatch a kernel the following actions are performed. This can occur in the CPU host program, or from an HSA kernel executing on a GPU.

   1. A pointer to an AQL queue for the kernel agent on which the kernel is to be executed is obtained.
   2. A pointer to the kernel descriptor (see Kernel Descriptor) of the kernel to execute is obtained. It must be for a kernel that is contained in a code object that that was loaded by the ROCm runtime on the kernel agent with which the AQL queue is associated.
   3. Space is allocated for the kernel arguments using the ROCm runtime allocator for a memory region with the kernarg property for the kernel agent that will execute the kernel. It must be at least 16 byte aligned.
   4. Kernel argument values are assigned to the kernel argument memory allocation. The layout is defined in the HSA Programmer’s Language Reference [HSA]. For AMDGPU the kernel execution directly accesses the kernel argument memory in the same way constant memory is accessed. (Note that the HSA specification allows an implementation to copy the kernel argument contents to another location that is accessed by the kernel.)
   5. An AQL kernel dispatch packet is created on the AQL queue. The ROCm runtime api uses 64 bit atomic operations to reserve space in the AQL queue for the packet. The packet must be set up, and the final write must use an atomic store release to set the packet kind to ensure the packet contents are visible to the kernel agent. AQL defines a doorbell signal mechanism to notify the kernel agent that the AQL queue has been updated. These rules, and the layout of the AQL queue and kernel dispatch packet is defined in the HSA System Architecture Specification [HSA].
   6. A kernel dispatch packet includes information about the actual dispatch, such as grid and work-group size, together with information from the code object about the kernel, such as segment sizes. The ROCm runtime queries on the kernel symbol can be used to obtain the code object values which are recorded in the Code Object Metadata.
   7. CP executes micro-code and is responsible for detecting and setting up the GPU to execute the wavefronts of a kernel dispatch.
   8. CP ensures that when the a wavefront starts executing the kernel machine code, the scalar general purpose registers (SGPR) and vector general purpose registers (VGPR) are set up as required by the machine code. The required setup is defined in the Kernel Descriptor. The initial register state is defined in Initial Kernel Execution State.
   9. The prolog of the kernel machine code (see Kernel Prolog) sets up the machine state as necessary before continuing executing the machine code that corresponds to the kernel.
   10. When the kernel dispatch has completed execution, CP signals the completion signal specified in the kernel dispatch packet if not 0.


.. _Memory Spaces:

Memory Spaces
++++++++++++++

The memory space properties are:

    AMDHSA Memory Spaces Memory Space
+----------+------------------+----------------+--------------+------------------------------+
| Name     | HSA Segment Name | Hardware Name  | Address Size | NULL Value                   |
+==========+==================+================+==============+==============================+
| Private  | private          | scratch        | 32           | 0x00000000                   |
+----------+------------------+----------------+--------------+------------------------------+
| Local    | group            | LDS            | 32           | 0xFFFFFFFF                   |
+----------+------------------+----------------+--------------+------------------------------+
| Global   | global           | global         | 64           | 0x0000000000000000           |
+----------+------------------+----------------+--------------+------------------------------+
| Constant | constant         | same as global | 64           | 0x0000000000000000           |
+----------+------------------+----------------+--------------+------------------------------+
| Generic  | flat             | flat           | 64           | 0x0000000000000000           |
+----------+------------------+----------------+--------------+------------------------------+
| Region   | N/A              | GDS            | 32           | `not implemented for AMDHSA` |
+----------+------------------+----------------+--------------+------------------------------+


The global and constant memory spaces both use global virtual addresses, which are the same virtual address space used by the CPU. However, some virtual addresses may only be accessible to the CPU, some only accessible by the GPU, and some by both.

Using the constant memory space indicates that the data will not change during the execution of the kernel. This allows scalar read instructions to be used. The vector and scalar L1 caches are invalidated of volatile data before each kernel dispatch execution to allow constant memory to change values between kernel dispatches.

The local memory space uses the hardware Local Data Store (LDS) which is automatically allocated when the hardware creates work-groups of wavefronts, and freed when all the wavefronts of a work-group have terminated. The data store (DS) instructions can be used to access it.

The private memory space uses the hardware scratch memory support. If the kernel uses scratch, then the hardware allocates memory that is accessed using wavefront lane dword (4 byte) interleaving. The mapping used from private address to physical address is:

   ``wavefront-scratch-base + (private-address * wavefront-size * 4) + (wavefront-lane-id * 4)``

There are different ways that the wavefront scratch base address is determined by a wavefront (see `Initial Kernel Execution State` ). This memory can be accessed in an interleaved manner using buffer instruction with the scratch buffer descriptor and per wave scratch offset, by the scratch instructions, or by flat instructions. If each lane of a wavefront accesses the same private address, the interleaving results in adjacent dwords being accessed and hence requires fewer cache lines to be fetched. Multi-dword access is not supported except by flat and scratch instructions in GFX9.

The generic address space uses the hardware flat address support available in GFX7-GFX9. This uses two fixed ranges of virtual addresses (the private and local appertures), that are outside the range of addressible global memory, to map from a flat address to a private or local address.

FLAT instructions can take a flat address and access global, private (scratch) and group (LDS) memory depending in if the address is within one of the apperture ranges. Flat access to scratch requires hardware aperture setup and setup in the kernel prologue (see Flat Scratch). Flat access to LDS requires hardware aperture setup and M0 (GFX7-GFX8) register setup (see M0).

To convert between a segment address and a flat address the base address of the appertures address can be used. For GFX7-GFX8 these are available in the HSA AQL Queue the address of which can be obtained with Queue Ptr SGPR (see Initial Kernel Execution State). For GFX9 the appature base addresses are directly available as inline constant registers SRC_SHARED_BASE/LIMIT and SRC_PRIVATE_BASE/LIMIT. In 64 bit address mode the apperture sizes are 2^32 bytes and the base is aligned to 2^32 which makes it easier to convert from flat to segment or segment to flat.

.. _Image and Samplers:

Image and Samplers
+++++++++++++++++++
Image and sample handles created by the ROCm runtime are 64 bit addresses of a hardware 32 byte V# and 48 byte S# object respectively. In order to support the HSA query_sampler operations two extra dwords are used to store the HSA BRIG enumeration values for the queries that are not trivially deducible from the S# representation.

.. _HSA Signals:

HSA Signals
++++++++++++
HSA signal handles created by the ROCm runtime are 64 bit addresses of a structure allocated in memory accessible from both the CPU and GPU. The structure is defined by the ROCm runtime and subject to change between releases (see [AMD-ROCm-github]).


.. _HSA AQL Queue:

HSA AQL Queue
++++++++++++++

The HSA AQL queue structure is defined by the ROCm runtime and subject to change between releases (see [AMD-ROCm-github]). For some processors it contains fields needed to implement certain language features such as the flat address aperture bases. It also contains fields used by CP such as managing the allocation of scratch memory.

.. _Kernel Descriptor:

Kernel Descriptor
++++++++++++++++++
A kernel descriptor consists of the information needed by CP to initiate the execution of a kernel, including the entry point address of the machine code that implements the kernel.

.. _Kernel Descriptor for GFX6-GFX9:

Kernel Descriptor for GFX6-GFX9
++++++++++++++++++++++++++++++++
CP microcode requires the Kernel descritor to be allocated on 64 byte alignment.

    Kernel Descriptor for GFX6-GFX9
 
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Bits    | Size                     | Field Name                          | Description                                                                                                                                                                                                    |
+=========+==========================+=====================================+================================================================================================================================================================================================================+
| 31:0    | 4 bytes                  | group_segment_fixed_size            | The amount of fixed local address space memory required for a work-group in bytes. This does not include any dynamically allocated local address space memory that may be added when the kernel is dispatched. |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 63:32   | 4 bytes                  | private_segment_fixed_size          | The amount of fixed private address space memory required for a work-item in bytes. If is_dynamic_callstack is 1 then additional space must be added to this value for the call stack.                         |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 95:64   | 4 bytes                  | max_flat_workgroup_size             | Maximum flat work-group size supported by the kernel in work-items.                                                                                                                                            |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 96      | 1 bit                    | is_dynamic_call_stack               | Indicates if the generated machine code is using a dynamically sized call stack.                                                                                                                               |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 97      | 1 bit                    | is_xnack_enabled                    | Indicates if the generated machine code is capable of suppoting XNACK.                                                                                                                                         |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 127:98  | 30 bits                  |                                     | Reserved. Must be 0.                                                                                                                                                                                           |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 191:128 | 8 bytes                  | kernel_code_entry_byte_offset       | Byte offset (possibly negative) from base address of kernel descriptor to kernel’s entry point instruction which must be 256 byte aligned.                                                                     |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 383:192 | 24 bytes                 |                                     | Reserved. Must be 0.                                                                                                                                                                                           |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 415:384 | 4 bytes                  | compute_pgm_rsrc1                   | Compute Shader (CS) program settings used by CP to set up COMPUTE_PGM_RSRC1 configuration register. See compute_pgm_rsrc1 for GFX6-GFX9.                                                                       |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 447:416 | 4 bytes                  | compute_pgm_rsrc2                   | Compute Shader (CS) program settings used by CP to set up COMPUTE_PGM_RSRC2 configuration register. See compute_pgm_rsrc2 for GFX6-GFX9.                                                                       |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 448     | 1 bit                    | enable_sgpr_private_segment _buffer | Enable the setup of the SGPR user data registers (see Initial Kernel Execution State).                                                                                                                         |
|         |                          |                                     |                                                                                                                                                                                                                |
|         |                          |                                     | The total number of SGPR user data registers requested must not exceed 16 and match value in compute_pgm_rsrc2.user_sgpr.user_sgpr_count. Any requests beyond 16 will be ignored.                              |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 449     | 1 bit                    | enable_sgpr_dispatch_ptr            | see above                                                                                                                                                                                                      |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 450     | 1 bit                    | enable_sgpr_queue_ptr               | see above                                                                                                                                                                                                      |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 451     | 1 bit                    | enable_sgpr_kernarg_segment_ptr     | see above                                                                                                                                                                                                      |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 452     | 1 bit                    | enable_sgpr_dispatch_id             | see above                                                                                                                                                                                                      |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 453     | 1 bit                    | enable_sgpr_flat_scratch_init       | see above                                                                                                                                                                                                      |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 454     | 1 bit                    | enable_sgpr_private_segment _size   | see above                                                                                                                                                                                                      |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 455     | 1 bit                    | enable_sgpr_grid_workgroup _count_X | Not implemented in CP and should always be 0.                                                                                                                                                                  |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 456     | 1 bit                    | enable_sgpr_grid_workgroup _count_Y | Not implemented in CP and should always be 0.                                                                                                                                                                  |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 457     | 1 bit                    | enable_sgpr_grid_workgroup _count_Z | Not implemented in CP and should always be 0.                                                                                                                                                                  |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 463:458 | 6 bits                   |                                     | Reserved. Must be 0.                                                                                                                                                                                           |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 511:464 | 4 bytes                  |                                     | Reserved. Must be 0.                                                                                                                                                                                           |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 512     | **Total size 64 bytes.** |                                     |                                                                                                                                                                                                                |
+---------+--------------------------+-------------------------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+


    **compute_pgm_rsrc1 for GFX6-GFX9**
    
+-------+------------------------+---------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Bits  | Size                   | Field Name                      | Description                                                                                                                                                                                                                                                                          |
+=======+========================+=================================+======================================================================================================================================================================================================================================================================================+
| 5:0   | 6 bits                 | granulated_workitem_vgpr_count  | Number of vector registers used by each work-item, granularity is device specific:                                                                                                                                                                                                   |
|       |                        |                                 |  GFX6-9 roundup                                                                                                                                                                                                                                                                      |
|       |                        |                                 |         ((max-vgpg + 1) / 4) - 1                                                                                                                                                                                                                                                     |
|       |                        |                                 | Used by CP to set up COMPUTE_PGM_RSRC1.VGPRS.                                                                                                                                                                                                                                        |
+-------+------------------------+---------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 9:6   | 4 bits                 | granulated_wavefront_sgpr_count | Number of scalar registers used by a wavefront, granularity is device specific:                                                                                                                                                                                                      |
|       |                        |                                 | GFX6-8 roundup                                                                                                                                                                                                                                                                       |
|       |                        |                                 |      ((max-sgpg + 1) / 8) - 1                                                                                                                                                                                                                                                        |
|       |                        |                                 | GFX9 roundup                                                                                                                                                                                                                                                                         |
|       |                        |                                 |      ((max-sgpg+1)/16) - 1                                                                                                                                                                                                                                                           |
|       |                        |                                 | Includes the special SGPRs for VCC, Flat Scratch (for GFX7 onwards) and XNACK (for GFX8 onwards).                                                                                                                                                                                    |
|       |                        |                                 | It does not include the 16 SGPR added if a trap handler is enabled.                                                                                                                                                                                                                  |
|       |                        |                                 |  Used by CP to set up COMPUTE_PGM_RSRC1.SGPRS.                                                                                                                                                                                                                                       |
+-------+------------------------+---------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 11:10 | 2 bits                 | priority                        | Must be 0.                                                                                                                                                                                                                                                                           |
|       |                        |                                 | Start executing wavefront at the specified priority.                                                                                                                                                                                                                                 |
|       |                        |                                 |  CP is responsible for filling in COMPUTE_PGM_RSRC1.PRIORITY.                                                                                                                                                                                                                        |
+-------+------------------------+---------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 13:12 | 2 bits                 | float_mode_round_32             | Wavefront starts execution with specified rounding mode for single (32 bit) floating point precision floating point operations.                                                                                                                                                      |
|       |                        |                                 |  Floating point rounding mode values are defined in Floating Point Rounding Mode Enumeration Values.                                                                                                                                                                                 |
|       |                        |                                 | Used by CP to set up ``COMPUTE_PGM_RSRC1.FLOAT_MODE.``                                                                                                                                                                                                                               |
+-------+------------------------+---------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 15:14 | 2 bits                 | float_mode_round_16_64          | Wavefront starts execution with specified rounding denorm mode for half/double (16 and 64 bit)  floating point precision floating point operations.                                                                                                                                  |
|       |                        |                                 | Floating point rounding mode values are defined in Floating Point Rounding Mode Enumeration Values.Used by CP to set up ``COMPUTE_PGM_RSRC1.FLOAT_MODE.``                                                                                                                            |
+-------+------------------------+---------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 17:16 | 2 bits                 | float_mode_denorm_32            | Wavefront starts execution with specified denorm mode for single (32 bit) floating point precision floating point operations.                                                                                                                                                        |
|       |                        |                                 | Floating point denorm mode values are defined in Floating Point Denorm Mode Enumeration Values.                                                                                                                                                                                      |
|       |                        |                                 | Used by CP to set up ``COMPUTE_PGM_RSRC1.FLOAT_MODE.``                                                                                                                                                                                                                               |
+-------+------------------------+---------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 19:18 | 2 bits                 | float_mode_denorm_16_64         | Wavefront starts execution with specified denorm mode for half/double (16 and 64 bit) floating point precision floating point operations.                                                                                                                                            |
|       |                        |                                 | Floating point denorm mode values are defined in Floating Point Denorm Mode Enumeration Values.                                                                                                                                                                                      |
|       |                        |                                 | Used by CP to set up ``COMPUTE_PGM_RSRC1.FLOAT_MODE.``                                                                                                                                                                                                                               |
+-------+------------------------+---------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 20    | 1 bit                  | priv                            | Must be 0.                                                                                                                                                                                                                                                                           |
|       |                        |                                 | Start executing wavefront in privilege trap handler mode.                                                                                                                                                                                                                            |
|       |                        |                                 | CP is responsible for filling in ``COMPUTE_PGM_RSRC1.PRIV.``                                                                                                                                                                                                                         |
+-------+------------------------+---------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 21    | 1 bit                  | enable_dx10_clamp               | Wavefront starts execution with DX10 clamp mode enabled.                                                                                                                                                                                                                             |
|       |                        |                                 | Used by the vector ALU to force DX-10 style treatment of NaN’s (when set, clamp NaN to zero, otherwise pass NaN through).                                                                                                                                                            |
|       |                        |                                 | Used by CP to set up`` COMPUTE_PGM_RSRC1.DX10_CLAMP.``                                                                                                                                                                                                                               |
+-------+------------------------+---------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 22    | 1 bit                  | debug_mode                      | Must be 0.                                                                                                                                                                                                                                                                           |
|       |                        |                                 | Start executing wavefront in single step mode.                                                                                                                                                                                                                                       |
|       |                        |                                 | CP is responsible for filling in ``COMPUTE_PGM_RSRC1.DEBUG_MODE.``                                                                                                                                                                                                                   |
+-------+------------------------+---------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 23    | 1 bit                  | enable_ieee_mode                | Wavefront starts execution with IEEE mode enabled. Floating point opcodes that support exception flag gathering will quiet and propagate signaling-NaN inputs per IEEE 754-2008. Min_dx10 and max_dx10 become IEEE 754-2008 compliant due to signaling-NaN propagation and quieting. |
|       |                        |                                 | Used by CP to set up COMPUTE_PGM_RSRC1.IEEE_MODE.                                                                                                                                                                                                                                    |
+-------+------------------------+---------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 24    | 1 bit                  | bulky                           | Must be 0.                                                                                                                                                                                                                                                                           |
|       |                        |                                 | Only one work-group allowed to execute on a compute unit.                                                                                                                                                                                                                            |
|       |                        |                                 | CP is responsible for filling in ``COMPUTE_PGM_RSRC1.BULKY.``                                                                                                                                                                                                                        |
+-------+------------------------+---------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 25    | 1 bit                  | cdbg_user                       | Must be 0.                                                                                                                                                                                                                                                                           |
|       |                        |                                 | Flag that can be used to control debugging code.                                                                                                                                                                                                                                     |
|       |                        |                                 | CP is responsible for filling in ``COMPUTE_PGM_RSRC1.CDBG_USER.``                                                                                                                                                                                                                    |
+-------+------------------------+---------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 31:26 | 6 bits                 |                                 | Reserved. Must be 0.                                                                                                                                                                                                                                                                 |
+-------+------------------------+---------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 32    | **Total size 4 bytes** |                                 |                                                                                                                                                                                                                                                                                      |
+-------+------------------------+---------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
 
 **compute_pgm_rsrc2 for GFX6-GFX9**

    
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Bits  | Size                | Field Name                                      | Description                                                                                                                                                                                   |
+=======+=====================+=================================================+===============================================================================================================================================================================================+
| 0     | 1 bit               | enable_sgpr_private_segment _wave_offset        | Enable the setup of the SGPR wave scratch offset system register (see Initial Kernel Execution State).                                                                                        |
|       |                     |                                                 | Used by CP to set up COMPUTE_PGM_RSRC2.SCRATCH_EN.                                                                                                                                            |
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 5:1   | 5 bits              | user_sgpr_count                                 | The total number of SGPR user data registers requested.                                                                                                                                       |
|       |                     |                                                 | This number must match the number of user data registers enabled.                                                                                                                             |
|       |                     |                                                 |  Used by CP to set up COMPUTE_PGM_RSRC2.USER_SGPR.                                                                                                                                            |
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 6     | 1 bit               | enable_trap_handler                             | Set to 1 if code contains a TRAP instruction which requires a trap handler to be enabled.                                                                                                     |
|       |                     |                                                 | CP sets COMPUTE_PGM_RSRC2.TRAP_PRESENT if the runtime has installed a trap handler regardless of the setting of this field.                                                                   |
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 7     | 1 bit               | enable_sgpr_workgroup_id_x                      | Enable the setup of the system SGPR register for the work-group id in the X dimension (see Initial Kernel Execution State).Used by CP to set up COMPUTE_PGM_RSRC2.TGID_X_EN.                  |
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 8     | 1 bit               | enable_sgpr_workgroup_id_y                      | Enable the setup of the system SGPR register for the work-group id in the Y dimension                                                                                                         |
|       |                     |                                                 |  (see Initial Kernel Execution State).Used by CP to set up COMPUTE_PGM_RSRC2.TGID_Y_EN.                                                                                                       |
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 9     | 1 bit               | enable_sgpr_workgroup_id_z                      | Enable the setup of the system SGPR register for the work-group id in the Z dimension                                                                                                         |
|       |                     |                                                 | (see Initial Kernel Execution State).                                                                                                                                                         |
|       |                     |                                                 |                                                                                                                                                                                               |
|       |                     |                                                 | Used by CP to set up COMPUTE_PGM_RSRC2.TGID_Z_EN.                                                                                                                                             |
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 10    | 1 bit               | enable_sgpr_workgroup_info                      | Enable the setup of the system SGPR register for work-group information (see Initial Kernel Execution State).                                                                                 |
|       |                     |                                                 | Used by CP to set up COMPUTE_PGM_RSRC2.TGID_SIZE_EN.                                                                                                                                          |
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 12:11 | 2 bits              | enable_vgpr_workitem_id                         | Enable the setup of the VGPR system registers used for the work-item ID.                                                                                                                      |
|       |                     |                                                 |                                                                                                                                                                                               |
|       |                     |                                                 | System VGPR Work-Item ID Enumeration Values defines the values.                                                                                                                               |
|       |                     |                                                 | Used by CP to set up COMPUTE_PGM_RSRC2.TIDIG_CMP_CNT.                                                                                                                                         |
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 13    | 1 bit               | enable_exception_address_watch                  | Must be 0.                                                                                                                                                                                    |
|       |                     |                                                 | Wavefront starts execution with address watch exceptions enabled which are generated when L1 has witnessed a thread access an address of interest.                                            |
|       |                     |                                                 | CP is responsible for filling in the address watch bit in COMPUTE_PGM_RSRC2.EXCP_EN_MSB according to what the runtime requests.                                                               |
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 14    | 1 bit               | enable_exception_memory                         | Must be 0.                                                                                                                                                                                    |
|       |                     |                                                 | Wavefront starts execution with memory violation exceptions exceptions enabled                                                                                                                |
|       |                     |                                                 | which are generated when a memory violation has occurred for this wave from L1 or LDS                                                                                                         |
|       |                     |                                                 | (write-to-read-only-memory, mis-aligned atomic, LDS address out of range, illegal address, etc.).                                                                                             |
|       |                     |                                                 | CP sets the memory violation bit in COMPUTE_PGM_RSRC2.EXCP_EN_MSB according to what the runtime requests.                                                                                     |
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 23:15 | 9 bits              | granulated_lds_size                             | Must be 0.                                                                                                                                                                                    |
|       |                     |                                                 | CP uses the rounded value from the dispatch packet, not this value, as the dispatch may contain dynamically allocated group segment memory. CP writes directly to COMPUTE_PGM_RSRC2.LDS_SIZE. |
|       |                     |                                                 | Amount of group segment (LDS) to allocate for each work-group. Granularity is device specific:                                                                                                |
|       |                     |                                                 |  GFX6:                                                                                                                                                                                        |
|       |                     |                                                 |      roundup(lds-size / (64 * 4))                                                                                                                                                             |
|       |                     |                                                 |  GFX7-GFX9:                                                                                                                                                                                   |
|       |                     |                                                 |      roundup(lds-size / (128 * 4))                                                                                                                                                            |
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 24    | 1 bit               | enable_exception_ieee_754_fp _invalid_operation | Wavefront starts execution with specified exceptions enabled.                                                                                                                                 |
|       |                     |                                                 | Used by CP to set up COMPUTE_PGM_RSRC2.EXCP_EN (set from bits 0..6).                                                                                                                          |
|       |                     |                                                 | IEEE 754 FP Invalid Operation                                                                                                                                                                 |
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 25    | 1 bit               | enable_exception_fp_denormal _source            | FP Denormal one or more input operands is a denormal number                                                                                                                                   |
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 26    | 1 bit               | enable_exception_ieee_754_fp _division_by_zero  | IEEE 754 FP Division by Zero                                                                                                                                                                  |
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 27    | 1 bit               | enable_exception_ieee_754_fp _overflow          | IEEE 754 FP FP Overflow                                                                                                                                                                       |
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 28    | 1 bit               | enable_exception_ieee_754_fp _underflow         | IEEE 754 FP Underflow                                                                                                                                                                         |
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 29    | 1 bit               | enable_exception_ieee_754_fp _inexact           | IEEE 754 FP Inexact                                                                                                                                                                           |
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 30    | 1 bit               | enable_exception_int_divide_by _zero            | Integer Division by Zero (rcp_iflag_f32 instruction only)                                                                                                                                     |
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 31    | 1 bit               |                                                 | Reserved. Must be 0.                                                                                                                                                                          |
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 32    | Total size 4 bytes. |                                                 |                                                                                                                                                                                               |
+-------+---------------------+-------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   
   
    Floating Point Rounding Mode Enumeration Values 
    
+-------------------------------------+-------+------------------------+
| Enumeration Name                    | Value | Description            |
+=====================================+=======+========================+
| AMD_FLOAT_ROUND_MODE_NEAR_EVEN      | 0     | Round Ties To Even     |
+-------------------------------------+-------+------------------------+
| AMD_FLOAT_ROUND_MODE_PLUS_INFINITY  | 1     | Round Toward +infinity |
+-------------------------------------+-------+------------------------+
| AMD_FLOAT_ROUND_MODE_MINUS_INFINITY | 2     | Round Toward -infinity |
+-------------------------------------+-------+------------------------+
| AMD_FLOAT_ROUND_MODE_ZERO           | 3     | Round Toward 0         |
+-------------------------------------+-------+------------------------+

	Floating Point Denorm Mode 
+-------------------------------------+-------+--------------------------------------+
| Enumeration Values Enumeration Name | Value | Description                          |
+=====================================+=======+======================================+
| AMD_FLOAT_DENORM_MODE_FLUSH_SRC_DST | 0     | Flush Source and Destination Denorms |
+-------------------------------------+-------+--------------------------------------+
| AMD_FLOAT_DENORM_MODE_FLUSH_DST     | 1     | Flush Output Denorms                 |
+-------------------------------------+-------+--------------------------------------+
| AMD_FLOAT_DENORM_MODE_FLUSH_SRC     | 2     | Flush Source Denorms                 |
+-------------------------------------+-------+--------------------------------------+
| AMD_FLOAT_DENORM_MODE_FLUSH_NONE    | 3     | No Flush                             |
+-------------------------------------+-------+--------------------------------------+


    			System VGPR Work-Item ID 
    
+---------------------------------------+-------+-----------------------------------------+
| Enumeration Values Enumeration Name   | Value | Description                             |
+=======================================+=======+=========================================+
| AMD_SYSTEM_VGPR_WORKITEM_ID_X         | 0     | Set work-item X dimension ID.           |
+---------------------------------------+-------+-----------------------------------------+
| AMD_SYSTEM_VGPR_WORKITEM_ID_X_Y       | 1     | Set work-item X and Y dimensions ID.    |
+---------------------------------------+-------+-----------------------------------------+
| AMD_SYSTEM_VGPR_WORKITEM_ID_X_Y_Z     | 2     | Set work-item X, Y and Z dimensions ID. |
+---------------------------------------+-------+-----------------------------------------+
| AMD_SYSTEM_VGPR_WORKITEM_ID_UNDEFINED | 3     | Undefined.                              |
+---------------------------------------+-------+-----------------------------------------+



.. _Initial Kernel Execution State:

Initial Kernel Execution State
+++++++++++++++++++++++++++++++

This section defines the register state that will be set up by the packet processor prior to the start of execution of every wavefront. This is limited by the constraints of the hardware controllers of CP/ADC/SPI.

The order of the SGPR registers is defined, but the compiler can specify which ones are actually setup in the kernel descriptor using the enable_sgpr_* bit fields (see Kernel Descriptor). The register numbers used for enabled registers are dense starting at SGPR0: the first enabled register is SGPR0, the next enabled register is SGPR1 etc.; disabled registers do not have an SGPR number.

The initial SGPRs comprise up to 16 User SRGPs that are set by CP and apply to all waves of the grid. It is possible to specify more than 16 User SGPRs using the enable_sgpr_* bit fields, in which case only the first 16 are actually initialized. These are then immediately followed by the System SGPRs that are set up by ADC/SPI and can have different values for each wave of the grid dispatch.

SGPR register initial state is defined in SGPR Register Set Up Order.

    SGPR Register Set Up Order
+------------+----------------------------------------------------------------------------------------------+-----------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| SGPR Order | Name (kernel descriptor enable field)                                                        | Number of SGPRs | Description                                                                                                                                                                                                                                                 |
+============+==============================================================================================+=================+=============================================================================================================================================================================================================================================================+
| First      | Private Segment Buffer (enable_sgpr_private _segment_buffer)                                 | 4               | V# that can be used, together with Scratch Wave Offset as an offset, to access the private memory space using a segment address.                                                                                                                            |
|            |                                                                                              |                 | CP uses the value provided by the runtime.                                                                                                                                                                                                                  |
+------------+----------------------------------------------------------------------------------------------+-----------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| then       | Dispatch Ptr (enable_sgpr_dispatch_ptr)                                                      | 2               | 64 bit address of AQL dispatch packet for kernel dispatch actually executing.                                                                                                                                                                               |
+------------+----------------------------------------------------------------------------------------------+-----------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| then       | Queue Ptr (enable_sgpr_queue_ptr)                                                            | 2               | 64 bit address of amd_queue_t object for AQL queue on which the dispatch packet was queued.                                                                                                                                                                 |
+------------+----------------------------------------------------------------------------------------------+-----------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| then       | Kernarg Segment Ptr (enable_sgpr_kernarg _segment_ptr)                                       | 2               | 64 bit address of Kernarg segment. This is directly copied from the kernarg_address in the kernel dispatch packet.                                                                                                                                          |
|            |                                                                                              |                 |                                                                                                                                                                                                                                                             |
|            |                                                                                              |                 | Having CP load it once avoids loading it at the beginning of every wavefront.                                                                                                                                                                               |
+------------+----------------------------------------------------------------------------------------------+-----------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| then       | Dispatch Id (enable_sgpr_dispatch_id)                                                        | 2               | 64 bit Dispatch ID of the dispatch packet being executed.                                                                                                                                                                                                   |
+------------+----------------------------------------------------------------------------------------------+-----------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| then       | Flat Scratch Init (enable_sgpr_flat_scratch _init)                                           | 2               | This is 2 SGPRs:                                                                                                                                                                                                                                            |
|            |                                                                                              |                 |  GFX6                                                                                                                                                                                                                                                       |
|            |                                                                                              |                 |     Not supported.                                                                                                                                                                                                                                          |
|            |                                                                                              |                 |  GFX7-GFX8                                                                                                                                                                                                                                                  |
|            |                                                                                              |                 |  The first SGPR is a 32 bit byte offset from SH_HIDDEN_PRIVATE_BASE_VIMID to per SPI base of memory for scratch for the queue executing the kernel dispatch. CP obtains this from the runtime.                                                              |
|            |                                                                                              |                 |  (The Scratch Segment Buffer base address is SH_HIDDEN_PRIVATE_BASE_VIMID plus this offset.) The value of Scratch Wave Offset must be added to this offset by the kernel machine code, right shifted by 8, and moved to the FLAT_SCRATCH_HI SGPR register.  |
|            |                                                                                              |                 |  FLAT_SCRATCH_HI corresponds to SGPRn-4 on GFX7, and SGPRn-6 on GFX8 (where SGPRn is the highest numbered SGPR allocated to the wave).                                                                                                                      |
|            |                                                                                              |                 |  FLAT_SCRATCH_HI is multiplied by 256 (as it is in units of 256 bytes) and added to SH_HIDDEN_PRIVATE_BASE_VIMID to calculate the per wave FLAT SCRATCH BASE in flat memory instructions that access the scratch apperture.                                 |
|            |                                                                                              |                 |                                                                                                                                                                                                                                                             |
|            |                                                                                              |                 |  The second SGPR is 32 bit byte size of a single work-item’s scratch memory usage.                                                                                                                                                                          |
|            |                                                                                              |                 |  CP obtains this from the runtime, and it is always a multiple of DWORD. CP checks that the value in the kernel dispatch packet Private Segment Byte Size is not larger, and requests the runtime to increase the queue’s scratch size if necessary.        |
|            |                                                                                              |                 |  The kernel code must move it to FLAT_SCRATCH_LO which is SGPRn-3 on GFX7 and SGPRn-5 on GFX8. FLAT_SCRATCH_LO is used as the FLAT SCRATCH SIZE in flat memory instructions.                                                                                |
|            |                                                                                              |                 |  Having CP load it once avoids loading it at the beginning of every wavefront. GFX9 This is the 64 bit base address of the per SPI scratch backing memory managed by SPI for the queue executing the kernel dispatch. CP obtains this from the runtime      |
|            |                                                                                              |                 |  (and divides it if there are multiple Shader Arrays each with its own SPI).                                                                                                                                                                                |
|            |                                                                                              |                 |  The value of Scratch Wave Offset must be added by the kernel machine code and the result moved to the FLAT_SCRATCH SGPR which is SGPRn-6 and SGPRn-5.                                                                                                      |
|            |                                                                                              |                 |  It is used as the FLAT SCRATCH BASE in flat memory instructions. then Private Segment Size 1 The 32 bit byte size of a (enable_sgpr_private single work-item’s scratch_segment_size) memory allocation.                                                    |
|            |                                                                                              |                 |  This is the value from the kernel dispatch packet Private Segment Byte Size rounded up by CP to a multiple of DWORD.                                                                                                                                       |
|            |                                                                                              |                 |  Having CP load it once avoids loading it at the beginning of every wavefront.                                                                                                                                                                              |
|            |                                                                                              |                 |                                                                                                                                                                                                                                                             |
|            |                                                                                              |                 | This is not used for GFX7-GFX8 since it is the same value as the second SGPR of Flat Scratch Init.                                                                                                                                                          |
|            |                                                                                              |                 | However, it may be needed for GFX9 which changes the meaning of the Flat Scratch Init value.                                                                                                                                                                |
+------------+----------------------------------------------------------------------------------------------+-----------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| then       | Grid Work-Group Count X (enable_sgpr_grid _workgroup_count_X)                                | 1               | 32 bit count of the number of work-groups in the X dimension for the grid being executed.                                                                                                                                                                   |
|            |                                                                                              |                 | Computed from the fields in the kernel dispatch packet as ((grid_size.x + workgroup_size.x - 1) / workgroup_size.x).                                                                                                                                        |
+------------+----------------------------------------------------------------------------------------------+-----------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| then       | Grid Work-Group Count Y (enable_sgpr_grid _workgroup_count_Y && less than 16 previous SGPRs) | 1               | 32 bit count of the number of work-groups in the Y dimension for the grid being executed.                                                                                                                                                                   |
|            |                                                                                              |                 | Computed from the fields in the kernel dispatch packet as ((grid_size.y + workgroup_size.y - 1) / workgroupSize.y).                                                                                                                                         |
|            |                                                                                              |                 | Only initialized if <16 previous SGPRs initialized.                                                                                                                                                                                                         |
+------------+----------------------------------------------------------------------------------------------+-----------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| then       | Grid Work-Group Count Z (enable_sgpr_grid _workgroup_count_Z && less than 16 previous SGPRs) | 1               | 32 bit count of the number of work-groups in the Z dimension for the grid being executed.                                                                                                                                                                   |
|            |                                                                                              |                 | Computed from the fields in the kernel dispatch packet as ((grid_size.z + workgroup_size.z - 1) / workgroupSize.z).                                                                                                                                         |
|            |                                                                                              |                 | Only initialized if <16 previous SGPRs initialized.                                                                                                                                                                                                         |
+------------+----------------------------------------------------------------------------------------------+-----------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| then       | Work-Group Id X (enable_sgpr_workgroup_id _X)                                                | 1               | 32 bit work-group id in X dimension of grid for wavefront.                                                                                                                                                                                                  |
+------------+----------------------------------------------------------------------------------------------+-----------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| then       | Work-Group Id Y (enable_sgpr_workgroup_id _Y)                                                | 1               | 32 bit work-group id in Y dimension of grid for wavefront.                                                                                                                                                                                                  |
+------------+----------------------------------------------------------------------------------------------+-----------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| then       | Work-Group Id Z (enable_sgpr_workgroup_id _Z)                                                | 1               | 32 bit work-group id in Z dimension of grid for wavefront.                                                                                                                                                                                                  |
+------------+----------------------------------------------------------------------------------------------+-----------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| then       | Work-Group Info (enable_sgpr_workgroup _info)                                                | 1               | {first_wave, 14’b0000, ordered_append_term[10:0], threadgroup_size_in_waves[5:0]}                                                                                                                                                                           |
+------------+----------------------------------------------------------------------------------------------+-----------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| then       | Scratch Wave Offset (enable_sgpr_private _segment_wave_offset)                               | 1               | 32 bit byte offset from base of scratch base of queue executing the kernel dispatch.                                                                                                                                                                        |
|            |                                                                                              |                 | Must be used as an offset with Private segment address when using Scratch Segment Buffer.                                                                                                                                                                   |
|            |                                                                                              |                 | It must be used to set up FLAT SCRATCH for flat addressing (see Flat Scratch).                                                                                                                                                                              |
+------------+----------------------------------------------------------------------------------------------+-----------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

The order of the VGPR registers is defined, but the compiler can specify which ones are actually setup in the kernel descriptor using the enable_vgpr* bit fields (see Kernel Descriptor). The register numbers used for enabled registers are dense starting at VGPR0: the first enabled register is VGPR0, the next enabled register is VGPR1 etc.; disabled registers do not have a VGPR number.

VGPR register initial state is defined in VGPR Register Set Up Order.

    VGPR Register Set Up Order
    
+------------+----------------------------------------------+-----------------+----------------------------------------------------------------------+
| VGPR Order | Name (kernel descriptor enable field)        | Number of VGPRs | Description                                                          |
+============+==============================================+=================+======================================================================+
| First      | Work-Item Id X (Always initialized)          | 1               | 32 bit work item id in X dimension of work-group for wavefront lane. |
+------------+----------------------------------------------+-----------------+----------------------------------------------------------------------+
| then       | Work-Item Id Y (enable_vgpr_workitem_id > 0) | 1               | 32 bit work item id in Y dimension of work-group for wavefront lane. |
+------------+----------------------------------------------+-----------------+----------------------------------------------------------------------+
| then       | Work-Item Id Z (enable_vgpr_workitem_id > 1) | 1               | 32 bit work item id in Z dimension of work-group for wavefront lane. |
+------------+----------------------------------------------+-----------------+----------------------------------------------------------------------+

The setting of registers is is done by GPU CP/ADC/SPI hardware as follows:

    1. SGPRs before the Work-Group Ids are set by CP using the 16 User Data registers.
    2. Work-group Id registers X, Y, Z are set by ADC which supports any combination including none.
    3. Scratch Wave Offset is set by SPI in a per wave basis which is why its value cannot included with the flat scratch init value which is per queue.
    4. The VGPRs are set by SPI which only supports specifying either (X), (X, Y) or (X, Y, Z).

Flat Scratch register pair are adjacent SGRRs so they can be moved as a 64 bit value to the hardware required SGPRn-3 and SGPRn-4 respectively.

The global segment can be accessed either using buffer instructions (GFX6 which has V# 64 bit address support), flat instructions (GFX7-9), or global instructions (GFX9).

If buffer operations are used then the compiler can generate a V# with the following properties:

    * base address of 0
    * no swizzle
    * ATC: 1 if IOMMU present (such as APU)
    * ptr64: 1
    * MTYPE set to support memory coherence that matches the runtime (such as CC for APU and NC for dGPU).


.. _Kernel Prolog:

Kernel Prolog
+++++++++++++++

.. _M0:

M0
***
GFX6-GFX8
    The M0 register must be initialized with a value at least the total LDS size if the kernel may access LDS via DS or flat operations. Total LDS size is available in dispatch packet. For M0, it is also possible to use maximum possible value of LDS for given target (0x7FFF for GFX6 and 0xFFFF for GFX7-GFX8).
GFX9
    The M0 register is not used for range checking LDS accesses and so does not need to be initialized in the prolog.

.. _Flat-Scratch:

Flat Scratch
*************
If the kernel may use flat operations to access scratch memory, the prolog code must set up FLAT_SCRATCH register pair (FLAT_SCRATCH_LO/FLAT_SCRATCH_HI which are in SGPRn-4/SGPRn-3). Initialization uses Flat Scratch Init and Scratch Wave Offset SGPR registers (see Initial Kernel Execution State):

GFX6
    Flat scratch is not supported.
    
GFX7-8
    1. The low word of Flat Scratch Init is 32 bit byte offset from SH_HIDDEN_PRIVATE_BASE_VIMID to the base of scratch backing memory being managed by SPI for the queue executing the kernel dispatch. This is the same value used in the Scratch Segment Buffer V# base address. The prolog must add the value of Scratch Wave Offset to get the wave’s byte scratch backing memory offset from SH_HIDDEN_PRIVATE_BASE_VIMID. Since FLAT_SCRATCH_LO is in units of 256 bytes, the offset must be right shifted by 8 before moving into FLAT_SCRATCH_LO.
    2. The second word of Flat Scratch Init is 32 bit byte size of a single work-items scratch memory usage. This is directly loaded from the kernel dispatch packet Private Segment Byte Size and rounded up to a multiple of DWORD. Having CP load it once avoids loading it at the beginning of every wavefront. The prolog must move it to FLAT_SCRATCH_LO for use as FLAT SCRATCH SIZE.

GFX9
    The Flat Scratch Init is the 64 bit address of the base of scratch backing memory being managed by SPI for the queue executing the kernel dispatch. The prolog must add the value of Scratch Wave Offset and moved to the FLAT_SCRATCH pair for use as the flat scratch base in flat memory instructions.

.. _Memory Model:

Memory Model
++++++++++++++

This section describes the mapping of LLVM memory model onto AMDGPU machine code (see Memory Model for Concurrent Operations). The implementation is WIP.

The AMDGPU backend supports the memory synchronization scopes specified in Memory Scopes.

The code sequences used to implement the memory model are defined in table AMDHSA Memory Model Code Sequences GFX6-GFX9.

The sequences specify the order of instructions that a single thread must execute. The s_waitcnt and buffer_wbinvl1_vol are defined with respect to other memory instructions executed by the same thread. This allows them to be moved earlier or later which can allow them to be combined with other instances of the same instruction, or hoisted/sunk out of loops to improve performance. Only the instructions related to the memory model are given; additional s_waitcnt instructions are required to ensure registers are defined before being used. These may be able to be combined with the memory model s_waitcnt instructions as described above.

The AMDGPU memory model supports both the HSA [HSA] memory model, and the OpenCL [OpenCL] memory model. The HSA memory model uses a single happens-before relation for all address spaces (see Address Spaces). The OpenCL memory model which has separate happens-before relations for the global and local address spaces, and only a fence specifying both global and local address space joins the relationships. Since the LLVM memfence instruction does not allow an address space to be specified the OpenCL fence has to convervatively assume both local and global address space was specified. However, optimizations can often be done to eliminate the additional ``s_waitcnt`` instructions when there are no intervening corresponding ``ds/flat_load/store/atomic memory`` instructions. The code sequences in the table indicate what can be omitted for the OpenCL memory. The target triple environment is used to determine if the source language is OpenCL (see OpenCL).

ds/flat_load/store/atomic instructions to local memory are termed LDS operations.

buffer/global/flat_load/store/atomic instructions to global memory are termed vector memory operations.

For GFX6-GFX9:

    * Each agent has multiple compute units (CU).
    * Each CU has multiple SIMDs that execute wavefronts.
    * The wavefronts for a single work-group are executed in the same CU but may be executed by different SIMDs.
    * Each CU has a single LDS memory shared by the wavefronts of the work-groups executing on it.
    * All LDS operations of a CU are performed as wavefront wide operations in a global order and involve no caching. Completion is reported to a wavefront in execution order.
    * The LDS memory has multiple request queues shared by the SIMDs of a CU. Therefore, the LDS operations performed by different waves of a work-group can be reordered relative to each other, which can result in reordering the visibility of vector memory operations with respect to LDS operations of other wavefronts in the same work-group. A s_waitcnt lgkmcnt(0) is required to ensure synchronization between LDS operations and vector memory operations between waves of a work-group, but not between operations performed by the same wavefront.
    * The vector memory operations are performed as wavefront wide operations and completion is reported to a wavefront in execution order. The exception is that for GFX7-9 flat_load/store/atomic instructions can report out of vector memory order if they access LDS memory, and out of LDS operation order if they access global memory.
    * The vector memory operations access a vector L1 cache shared by all wavefronts on a CU. Therefore, no special action is required for coherence between wavefronts in the same work-group. A buffer_wbinvl1_vol is required for coherence between waves executing in different work-groups as they may be executing on different CUs.
    * The scalar memory operations access a scalar L1 cache shared by all wavefronts on a group of CUs. The scalar and vector L1 caches are not coherent. However, scalar operations are used in a restricted way so do not impact the memory model. See Memory Spaces.
    * The vector and scalar memory operations use an L2 cache shared by all CUs on the same agent.
    * The L2 cache has independent channels to service disjoint ranges of virtual addresses.
    * Each CU has a separate request queue per channel. Therefore, the vector and scalar memory operations performed by waves executing in different work-groups (which may be executing on different CUs) of an agent can be reordered relative to each other. A s_waitcnt vmcnt(0) is required to ensure synchronization between vector memory operations of different CUs. It ensures a previous vector memory operation has completed before executing a subsequent vector memory or LDS operation and so can be used to meet the requirements of acquire and release.
    * The L2 cache can be kept coherent with other agents on some targets, or ranges of virtual addresses can be set up to bypass it to ensure system coherence.

Private address space uses buffer_load/store using the scratch V# (GFX6-8), or scratch_load/store (GFX9). Since only a single thread is accessing the memory, atomic memory orderings are not meaningful and all accesses are treated as non-atomic.

Constant address space uses buffer/global_load instructions (or equivalent scalar memory instructions). Since the constant address space contents do not change during the execution of a kernel dispatch it is not legal to perform stores, and atomic memory orderings are not meaningful and all access are treated as non-atomic.

A memory synchronization scope wider than work-group is not meaningful for the group (LDS) address space and is treated as work-group.

The memory model does not support the region address space which is treated as non-atomic.

Acquire memory ordering is not meaningful on store atomic instructions and is treated as non-atomic.

Release memory ordering is not meaningful on load atomic instructions and is treated a non-atomic.

Acquire-release memory ordering is not meaningful on load or store atomic instructions and is treated as acquire and release respectively.

AMDGPU backend only uses scalar memory operations to access memory that is proven to not change during the execution of the kernel dispatch. This includes constant address space and global address space for program scope const variables. Therefore the kernel machine code does not have to maintain the scalar L1 cache to ensure it is coherent with the vector L1 cache. The scalar and vector L1 caches are invalidated between kernel dispatches by CP since constant address space data may change between kernel dispatch executions. See Memory Spaces.

The one execption is if scalar writes are used to spill SGPR registers. In this case the AMDGPU backend ensures the memory location used to spill is never accessed by vector memory operations at the same time. If scalar writes are used then a s_dcache_wb is inserted before the s_endpgm and before a function return since the locations may be used for vector memory instructions by a future wave that uses the same scratch area, or a function call that creates a frame at the same address, respectively. There is no need for a s_dcache_inv as all scalar writes are write-before-read in the same thread.

Scratch backing memory (which is used for the private address space) is accessed with MTYPE NC_NV (non-coherenent non-volatile). Since the private address space is only accessed by a single thread, and is always write-before-read, there is never a need to invalidate these entries from the L1 cache. Hence all cache invalidates are done as *_vol to only invalidate the volatile cache lines.

On dGPU the kernarg backing memory is accessed as UC (uncached) to avoid needing to invalidate the L2 cache. This also causes it to be treated as non-volatile and so is not invalidated by *_vol. On APU it is accessed as CC (cache coherent) and so the L2 cache will coherent with the CPU and other agents.

    **AMDHSA Memory Model Code Sequences GFX6-GFX9**

+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| LLVM Instr   | LLVM Memory Ordering | LLVM Memory Sync Scope | AMDGPU Address Space | AMDGPU Machine Code                                                                                                                                                                         |
+==============+======================+========================+======================+=============================================================================================================================================================================================+
| **Non-Atomic**                                                                                                                                                                                                                                                                    |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Load         | none                 | none                   |  * global            | non-volatile                                                                                                                                                                                |
|              |                      |                        |  * generic           |    1. buffer/global/flat_load                                                                                                                                                               |
|              |                      |                        |                      | volatile                                                                                                                                                                                    |
|              |                      |                        |                      |    2. buffer/global/flat_load  glc=1                                                                                                                                                        |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Load         | none                 | none                   |  * Local             | 1. ds_load                                                                                                                                                                                  |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| store        | none                 | none                   |  * global            | 1. buffer/global/flat_store                                                                                                                                                                 |
|              |                      |                        |  * generic           |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| store        | none                 | none                   |                      | 1. ds_store                                                                                                                                                                                 |
|              |                      |                        |  * local             |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| **Unordered Atomic**                                                                                                                                                                                                                                                              |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| load atomic  | unordered            | any                    | any                  | Same as non-atomic                                                                                                                                                                          |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| store atomic | unordered            | any                    | any                  | Same as non-atomic                                                                                                                                                                          |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| atomicrmw    | unordered            | any                    | any                  | Same as monotonic atomic.                                                                                                                                                                   |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| **Monotonic Atomic**                                                                                                                                                                                                                                                              |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| load atomic  | monotonic            |  * singlethread        |  * global            |  1. buffer/global/flat_load                                                                                                                                                                 |
|              |                      |  * wavefront           |  * generic           |                                                                                                                                                                                             |
|              |                      |  * workgroup           |                      |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| load atomic  | monotonic            |  * singlethread        |  * local             | 1. ds_load                                                                                                                                                                                  |
|              |                      |  * wavefront           |                      |                                                                                                                                                                                             |
|              |                      |  * workgroup           |                      |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| load atomic  | monotonic            |  * agent               |  * global            |  1. buffer/global/flat_load glc=1                                                                                                                                                           |
|              |                      |  * system              |  * generic           |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| store atomic | monotonic            |  * singlethread        |  * global            |  1. buffer/global/flat_store                                                                                                                                                                |
|              |                      |  * wavefront           |  * generic           |                                                                                                                                                                                             |
|              |                      |  * workgroup           |                      |                                                                                                                                                                                             |
|              |                      |  * agent               |                      |                                                                                                                                                                                             |
|              |                      |  * system              |                      |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| store atomic | monotonic            |  * singlethread        |  * local             |   1. ds_store                                                                                                                                                                               |
|              |                      |  * wavefront           |                      |                                                                                                                                                                                             |
|              |                      |  * workgroup           |                      |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| atomicrmw    | monotonic            |  * singlethread        |  * global            | 1. buffer/global/flat_atomic                                                                                                                                                                |
|              |                      |  * wavefront           |  * generic           |                                                                                                                                                                                             |
|              |                      |  * workgroup           |                      |                                                                                                                                                                                             |
|              |                      |  * agent               |                      |                                                                                                                                                                                             |
|              |                      |  * system              |                      |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| atomicrmw    | monotonic            |  * singlethread        |  * local             | 1. ds_atomic                                                                                                                                                                                |
|              |                      |  * wavefront           |                      |                                                                                                                                                                                             |
|              |                      |  * workgroup           |                      |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| **Acquire Atomic**                                                                                                                                                                                                                                                                |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| load atomic  | acquire              |  * singlethread        |  * global            | 1. buffer/global/ds/flat_load                                                                                                                                                               |
|              |                      |  * wavefront           |  * local             |                                                                                                                                                                                             |
|              |                      |                        |  * generic           |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| load atomic  | acquire              |  * workgroup           |  * global            | 1. buffer/global_load                                                                                                                                                                       |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| load atomic  | acquire              |  * workgroup           |  * local             | 1. ds/flat_load                                                                                                                                                                             |
|              |                      |                        |  * generic           | 2. s_waitcnt lgkmcnt(0)                                                                                                                                                                     |
|              |                      |                        |                      |  * If OpenCL, omit waitcnt.                                                                                                                                                                 |
|              |                      |                        |                      |  * Must happen before any following global/generic load/load atomic/store/store atomic/atomicrmw.                                                                                           |
|              |                      |                        |                      |  * Ensures any following global data read is no older than the load atomic value being acquired.                                                                                            |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| load atomic  | acquire              |  * agent               |  * global            | 1. buffer/global_load glc=1                                                                                                                                                                 |
|              |                      |  * system              |                      | 2. s_waitcnt vmcnt(0)                                                                                                                                                                       |
|              |                      |                        |                      |  * Must happen before following buffer_wbinvl1_vol.                                                                                                                                         |
|              |                      |                        |                      |  * Ensures the load has completed before invalidating the cache.                                                                                                                            |
|              |                      |                        |                      | 3. buffer_wbinvl1_vol                                                                                                                                                                       |
|              |                      |                        |                      |  * Must happen before any following global/generic load/load atomic/atomicrmw.                                                                                                              |
|              |                      |                        |                      |  * Ensures that following loads will not see stale global data.                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| load atomic  | acquire              |  * agent               |  * generic           | 1. flat_load glc=1                                                                                                                                                                          |
|              |                      |  * system              |                      | 2. s_waitcnt vmcnt(0) & lgkmcnt(0)                                                                                                                                                          |
|              |                      |                        |                      |   * If OpenCL omit lgkmcnt(0).                                                                                                                                                              |
|              |                      |                        |                      |   * Must happen before following buffer_wbinvl1_vol.                                                                                                                                        |
|              |                      |                        |                      |   * Ensures the flat_load has completed before invalidating the cache.                                                                                                                      |
|              |                      |                        |                      | 3. buffer_wbinvl1_vol                                                                                                                                                                       |
|              |                      |                        |                      |   * Must happen before any following global/generic load/load atomic/atomicrmw.                                                                                                             |
|              |                      |                        |                      |   * Ensures that following loads will not see stale global data                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| atomicrmw    | acquire              |  * singlethread        |  * global            | 1. buffer/global/ds/flat_atomic                                                                                                                                                             |
|              |                      |  * wavefront           |  * local             |                                                                                                                                                                                             |
|              |                      |                        |  * generic           |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| atomicrmw    | acquire              |  * workgroup           |  * global            | 1. buffer/global_atomic                                                                                                                                                                     |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| atomicrmw    | acquire              |  * workgroup           |  * local             | 1. ds/flat_atomic                                                                                                                                                                           |
|              |                      |                        |  *generic            | 2. waitcnt lgkmcnt(0)                                                                                                                                                                       |
|              |                      |                        |                      |   * If OpenCL, omit waitcnt.                                                                                                                                                                |
|              |                      |                        |                      |   * Must happen before any following global/generic load/load atomic/store/store atomic/atomicrmw.                                                                                          |
|              |                      |                        |                      |   * Ensures any following global data read is no older than the atomicrmw value being acquired.                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| atomicrmw    | acquire              |  * agent               |  * global            | 1. buffer/global_atomic                                                                                                                                                                     |
|              |                      |  * system              |                      | 2. s_waitcnt vmcnt(0)                                                                                                                                                                       |
|              |                      |                        |                      |   * Must happen before following buffer_wbinvl1_vol.                                                                                                                                        |
|              |                      |                        |                      |   * Ensures the atomicrmw has completed before invalidating the cache.                                                                                                                      |
|              |                      |                        |                      | 3. buffer_wbinvl1_vol                                                                                                                                                                       |
|              |                      |                        |                      |   * Must happen before any following global/generic load/load atomic/atomicrmw.                                                                                                             |
|              |                      |                        |                      |   * Ensures that following loads will not see stale global data.                                                                                                                            |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| atomicrmw    | acquire              |  * agent               |  * generic           | 1. flat_atomic                                                                                                                                                                              |
|              |                      |  * system              |                      | 2. s_waitcnt vmcnt(0) & lgkmcnt(0)                                                                                                                                                          |
|              |                      |                        |                      |   * If OpenCL, omit lgkmcnt(0).                                                                                                                                                             |
|              |                      |                        |                      |   * Must happen before following buffer_wbinvl1_vol.                                                                                                                                        |
|              |                      |                        |                      |   * Ensures the atomicrmw has completed before invalidating the cache.                                                                                                                      |
|              |                      |                        |                      | 3. buffer_wbinvl1_vol                                                                                                                                                                       |
|              |                      |                        |                      |   * Must happen before any following global/generic load/load atomic/atomicrmw.                                                                                                             |
|              |                      |                        |                      |   * Ensures that following loads will not see stale global data.                                                                                                                            |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| fence        | acquire              |  * singlethread        | none                 | none                                                                                                                                                                                        |
|              |                      |  * wavefront           |                      |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| fence        | acquire              |  * workgroup           | none                 | 1. s_waitcnt lgkmcnt(0)                                                                                                                                                                     |
|              |                      |                        |                      |   * If OpenCL and address space is not generic, omit waitcnt.                                                                                                                               |
|              |                      |                        |                      |      However, since LLVM currently has no address space on the fence need to conservatively always generate.                                                                                |
|              |                      |                        |                      |      If fence had an address space then set to address space of OpenCL fence flag, or to generic if both local and global flags are specified.                                              |
|              |                      |                        |                      |   * Must happen after any preceding local/generic load atomic/atomicrmw with                                                                                                                |
|              |                      |                        |                      |     an equal or wider sync scope and memory ordering stronger than unordered (this is termed the fence-paired-atomic).                                                                      |
|              |                      |                        |                      |   * Must happen before any following global/generic load/load atomic/store/store atomic/atomicrmw.                                                                                          |
|              |                      |                        |                      |   * Ensures any following global data read is no older than the value read by the fence-paired-atomic.                                                                                      |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| fence        | acquire              |  * agent               | none                 | 1. s_waitcnt vmcnt(0) & lgkmcnt(0)                                                                                                                                                          |
|              |                      |  * system              |                      |   * If OpenCL and address space is not generic, omit lgkmcnt(0).                                                                                                                            |
|              |                      |                        |                      |     However, since LLVM currently has no address space on the fence need to conservatively always generate (see comment for previous fence).                                                |
|              |                      |                        |                      |   * Could be split into separate s_waitcnt vmcnt(0) and s_waitcnt lgkmcnt(0) to allow them to be independently moved according to the following rules.                                      |
|              |                      |                        |                      |   * s_waitcnt vmcnt(0) must happen after any preceding global/generic load atomic/atomicrmw with                                                                                            |
|              |                      |                        |                      |     an equal or wider sync scope and memory ordering stronger than unordered (this is termed the fence-paired-atomic).                                                                      |
|              |                      |                        |                      |   * s_waitcnt lgkmcnt(0) must happen after any preceding group/generic load atomic/atomicrmw with an equal or                                                                               |
|              |                      |                        |                      |     wider sync scope and memory ordering stronger than unordered (this is termed the fence-paired-atomic).                                                                                  |
|              |                      |                        |                      |   * Must happen before the following buffer_wbinvl1_vol.                                                                                                                                    |
|              |                      |                        |                      |   * Ensures that the fence-paired atomic has completed before invalidating the cache.                                                                                                       |
|              |                      |                        |                      |     Therefore any following locations read must be no older than the value read by the fence-paired-atomic.                                                                                 |
|              |                      |                        |                      | 3. buffer_wbinvl1_vol                                                                                                                                                                       |
|              |                      |                        |                      |   * Must happen before any following global/generic load/load atomic/store/store atomic/atomicrmw.                                                                                          |
|              |                      |                        |                      |   * Ensures that following loads will not see stale global data.                                                                                                                            |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| **Release Atomic**                                                                                                                                                                                                                                                                |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| store atomic | release              |  * singlethread        |  * global            | 1. buffer/global/ds/flat_store                                                                                                                                                              |
|              |                      |  * wavefront           |  * local             |                                                                                                                                                                                             |
|              |                      |                        |  * generic           |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| store atomic | release              |  * workgroup           |  * global            | 1. s_waitcnt lgkmcnt(0)                                                                                                                                                                     |
|              |                      |                        |  * generic           |   * If OpenCL, omit waitcnt.                                                                                                                                                                |
|              |                      |                        |                      |   * Must happen after any preceding local/generic load/store/load atomic/store atomic/atomicrmw.                                                                                            |
|              |                      |                        |                      |   * Must happen before the following store.                                                                                                                                                 |
|              |                      |                        |                      |   * Ensures that all memory operations to local have completed before performing the store that is being released.                                                                          |
|              |                      |                        |                      | 2. buffer/global/flat_store                                                                                                                                                                 |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| store atomic | release              |  * workgroup           |  * local             | 1. ds_store                                                                                                                                                                                 |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| store atomic | release              |  * agent               |  * global            | 1. s_waitcnt vmcnt(0) & lgkmcnt(0)                                                                                                                                                          |
|              |                      |  * system              |  * generic           |   * If OpenCL, omit lgkmcnt(0).                                                                                                                                                             |
|              |                      |                        |                      |   * Could be split into separate s_waitcnt vmcnt(0) and s_waitcnt lgkmcnt(0) to allow them to be independently moved according to the following rules.                                      |
|              |                      |                        |                      |   * s_waitcnt vmcnt(0) must happen after any preceding global/generic load/store/load atomic/store atomic/atomicrmw.                                                                        |
|              |                      |                        |                      |   * s_waitcnt lgkmcnt(0) must happen after any preceding local/generic load/store/load atomic/store atomic/atomicrmw.                                                                       |
|              |                      |                        |                      |    * Must happen before the following store.                                                                                                                                                |
|              |                      |                        |                      |   * Ensures that all memory operations to global have completed before performing the store that is being released.                                                                         |
|              |                      |                        |                      | 2. buffer/global/ds/flat_store                                                                                                                                                              |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| atomicrmw    | release              |  * singlethread        |  * global            | 1. buffer/global/ds/flat_atomic                                                                                                                                                             |
|              |                      |  * wavefront           |  * local             |                                                                                                                                                                                             |
|              |                      |                        |  * generic           |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| atomicrmw    | release              |  * workgroup           |  * global            | 1. s_waitcnt lgkmcnt(0)                                                                                                                                                                     |
|              |                      |                        |  * generic           |   * If OpenCL, omit waitcnt.                                                                                                                                                                |
|              |                      |                        |                      |   * Must happen after any preceding local/generic load/store/load atomic/store atomic/atomicrmw.                                                                                            |
|              |                      |                        |                      |   * Must happen before the following atomicrmw.                                                                                                                                             |
|              |                      |                        |                      |   * Ensures that all memory operations to local have completed before performing the atomicrmw that is being released.                                                                      |
|              |                      |                        |                      | 2. buffer/global/flat_atomic                                                                                                                                                                |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| atomicrmw    | release              |  * workgroup           |   * local            | 1. ds_atomic                                                                                                                                                                                |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| atomicrmw    | release              |  * agent               |  * global            | 1. s_waitcnt vmcnt(0) & lgkmcnt(0)                                                                                                                                                          |
|              |                      |  * system              |  * generic           |   * If OpenCL, omit lgkmcnt(0).                                                                                                                                                             |
|              |                      |                        |                      |   * Could be split into separate s_waitcnt vmcnt(0) and s_waitcnt lgkmcnt(0) to allow them to be independently moved according to the following rules.                                      |
|              |                      |                        |                      |   * s_waitcnt vmcnt(0) must happen after any preceding global/generic load/store/load atomic/store atomic/atomicrmw.                                                                        |
|              |                      |                        |                      |   * s_waitcnt lgkmcnt(0) must happen after any preceding local/generic load/store/load atomic/store atomic/atomicrmw.                                                                       |
|              |                      |                        |                      |   * Must happen before the following atomicrmw.                                                                                                                                             |
|              |                      |                        |                      |   * Ensures that all memory operations to global and local have completed before performing the atomicrmw that is being released.                                                           |
|              |                      |                        |                      | 2. buffer/global/ds/flat_atomic*                                                                                                                                                            |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| fence        | release              |  * singlethread        | none                 | none                                                                                                                                                                                        |
|              |                      |  * wavefront           |                      |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| fence        | release              |  * workgroup           | none                 | 1. s_waitcnt lgkmcnt(0)                                                                                                                                                                     |
|              |                      |                        |                      |   * If OpenCL and address space is not generic, omit waitcnt.                                                                                                                               |
|              |                      |                        |                      |     However, since LLVM currently has no address space on the fence need to conservatively always generate (see comment for previous fence).                                                |
|              |                      |                        |                      |   * Must happen after any preceding local/generic load/load atomic/store/store atomic/atomicrmw.                                                                                            |
|              |                      |                        |                      |   * Must happen before any following store atomic/atomicrmw with an equal or                                                                                                                |
|              |                      |                        |                      |     wider sync scope and memory ordering stronger than unordered (this is termed the fence-paired-atomic).                                                                                  |
|              |                      |                        |                      |   * Ensures that all memory operations to local have completed before performing the following fence-paired-atomic.                                                                         |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| fence        | release              |  * agent               | none                 | 1. s_waitcnt vmcnt(0) & lgkmcnt(0)                                                                                                                                                          |
|              |                      |  * system              |                      |   * If OpenCL and address space is not generic, omit lgkmcnt(0).                                                                                                                            |
|              |                      |                        |                      |     However, since LLVM currently has no address space on the fence need to conservatively always generate (see comment for previous fence).                                                |
|              |                      |                        |                      |   * Could be split into separate s_waitcnt vmcnt(0) and s_waitcnt lgkmcnt(0) to allow them to be independently moved according to the following rules.                                      |
|              |                      |                        |                      |   * s_waitcnt vmcnt(0) must happen after any preceding global/generic load/store/load atomic/store atomic/atomicrmw.                                                                        |
|              |                      |                        |                      |   * s_waitcnt lgkmcnt(0) must happen after any preceding local/generic load/store/load atomic/store atomic/atomicrmw.                                                                       |
|              |                      |                        |                      |   * Must happen before any following store atomic/atomicrmw with an equal or wider sync scope and memory ordering stronger than unordered                                                   |
|              |                      |                        |                      |      (this is termed the fence-paired-atomic).                                                                                                                                              |
|              |                      |                        |                      |   * Ensures that all memory operations to global have completed before performing the following fence-paired-atomic.                                                                        |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| **Acquire-Release Atomic**                                                                                                                                                                                                                                                        |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| atomicrmw    | acq_rel              |  * singlethread        |  * global            | 1. buffer/global/ds/flat_atomic                                                                                                                                                             |
|              |                      |  * wavefront           |  * local             |                                                                                                                                                                                             |
|              |                      |                        |  * generic           |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| atomicrmw    | acq_rel              |  * workgroup           | * global             | 1. s_waitcnt lgkmcnt(0)                                                                                                                                                                     |
|              |                      |                        |                      |   * If OpenCL, omit waitcnt.                                                                                                                                                                |
|              |                      |                        |                      |   * Must happen after any preceding local/generic load/store/load atomic/store atomic/atomicrmw.                                                                                            |
|              |                      |                        |                      |   * Must happen before the following atomicrmw.                                                                                                                                             |
|              |                      |                        |                      |   * Ensures that all memory operations to local have completed before performing the atomicrmw that is being released.                                                                      |
|              |                      |                        |                      | 2. buffer/global_atomic                                                                                                                                                                     |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| atomicrmw    | acq_rel              |  * workgroup           |  * local             | 1. ds_atomic                                                                                                                                                                                |
|              |                      |                        |                      | 2. s_waitcnt lgkmcnt(0)                                                                                                                                                                     |
|              |                      |                        |                      |   * If OpenCL, omit waitcnt.                                                                                                                                                                |
|              |                      |                        |                      |   * Must happen before any following global/generic load/load atomic/store/store atomic/atomicrmw.                                                                                          |
|              |                      |                        |                      |   * Ensures any following global data read is no older than the load atomic value being acquired.                                                                                           |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| atomicrmw    | acq_rel              |  *workgroup            |  * generic           | 1. s_waitcnt lgkmcnt(0)                                                                                                                                                                     |
|              |                      |                        |                      |   * If OpenCL, omit waitcnt.                                                                                                                                                                |
|              |                      |                        |                      |   * Must happen after any preceding local/generic load/store/load atomic/store atomic/atomicrmw.                                                                                            |
|              |                      |                        |                      |   * Must happen before the following atomicrmw.                                                                                                                                             |
|              |                      |                        |                      |   * Ensures that all memory operations to local have completed before performing the atomicrmw that is being released.                                                                      |
|              |                      |                        |                      | 2. flat_atomic                                                                                                                                                                              |
|              |                      |                        |                      | 3. s_waitcnt lgkmcnt(0)                                                                                                                                                                     |
|              |                      |                        |                      |   * If OpenCL, omit waitcnt.                                                                                                                                                                |
|              |                      |                        |                      |   * Must happen before any following global/generic load/load atomic/store/store atomic/atomicrmw.                                                                                          |
|              |                      |                        |                      |   * Ensures any following global data read is no older than the load atomic value being acquired.                                                                                           |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| atomicrmw    | acq_rel              |  * agent               | * global             | 1. s_waitcnt vmcnt(0) & lgkmcnt(0)                                                                                                                                                          |
|              |                      |  * system              |                      |   * If OpenCL, omit lgkmcnt(0).                                                                                                                                                             |
|              |                      |                        |                      |   * Could be split into separate s_waitcnt vmcnt(0) and s_waitcnt lgkmcnt(0) to allow them to be independently moved according to the following rules.                                      |
|              |                      |                        |                      |   * s_waitcnt vmcnt(0) must happen after any preceding global/generic load/store/load atomic/store atomic/atomicrmw.                                                                        |
|              |                      |                        |                      |   * s_waitcnt lgkmcnt(0) must happen after any preceding local/generic load/store/load atomic/store atomic/atomicrmw.                                                                       |
|              |                      |                        |                      |   * Must happen before the following atomicrmw.                                                                                                                                             |
|              |                      |                        |                      |   * Ensures that all memory operations to global have completed before performing the atomicrmw that is being released.                                                                     |
|              |                      |                        |                      | 2. buffer/global_atomic                                                                                                                                                                     |
|              |                      |                        |                      | 3. s_waitcnt vmcnt(0)                                                                                                                                                                       |
|              |                      |                        |                      |   * Must happen before following buffer_wbinvl1_vol.                                                                                                                                        |
|              |                      |                        |                      |   * Ensures the atomicrmw has completed before invalidating the cache.                                                                                                                      |
|              |                      |                        |                      | 4. buffer_wbinvl1_vol                                                                                                                                                                       |
|              |                      |                        |                      |   * Must happen before any following global/generic load/load atomic/atomicrmw.                                                                                                             |
|              |                      |                        |                      |   * Ensures that following loads will not see stale global data.                                                                                                                            |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| atomicrmw    | acq_rel              |  * agent               | * generic            | 1. s_waitcnt vmcnt(0) & lgkmcnt(0)                                                                                                                                                          |
|              |                      |  * system              |                      |   * If OpenCL, omit lgkmcnt(0).                                                                                                                                                             |
|              |                      |                        |                      |   * Could be split into separate s_waitcnt vmcnt(0) and s_waitcnt lgkmcnt(0) to allow them to be independently moved according to the following rules.                                      |
|              |                      |                        |                      |   * s_waitcnt vmcnt(0) must happen after any preceding global/generic load/store/load atomic/store atomic/atomicrmw.                                                                        |
|              |                      |                        |                      |   * s_waitcnt lgkmcnt(0) must happen after any preceding local/generic load/store/load atomic/store atomic/atomicrmw.                                                                       |
|              |                      |                        |                      |   * Must happen before the following atomicrmw.                                                                                                                                             |
|              |                      |                        |                      |   * Ensures that all memory operations to global have completed before performing the atomicrmw that is being released.                                                                     |
|              |                      |                        |                      | 2. flat_atomic                                                                                                                                                                              |
|              |                      |                        |                      | 3. s_waitcnt vmcnt(0) & lgkmcnt(0)                                                                                                                                                          |
|              |                      |                        |                      |   * If OpenCL, omit lgkmcnt(0).                                                                                                                                                             |
|              |                      |                        |                      |   * Must happen before following buffer_wbinvl1_vol.                                                                                                                                        |
|              |                      |                        |                      |   * Ensures the atomicrmw has completed before invalidating the cache.                                                                                                                      |
|              |                      |                        |                      | 4. buffer_wbinvl1_vol                                                                                                                                                                       |
|              |                      |                        |                      |   * Must happen before any following global/generic load/load atomic/atomicrmw.                                                                                                             |
|              |                      |                        |                      |   * Ensures that following loads will not see stale global data.                                                                                                                            |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| fence        | acq_rel              |  * singlethread        | none                 | none                                                                                                                                                                                        |
|              |                      |  * wavefront           |                      |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| fence        | acq_rel              |  * workgroup           | none                 | 1. s_waitcnt lgkmcnt(0)                                                                                                                                                                     |
|              |                      |                        |                      |   * If OpenCL and address space is not generic, omit waitcnt.                                                                                                                               |
|              |                      |                        |                      |     However, since LLVM currently has no address space on the fence need to conservatively always generate (see comment for previous fence).                                                |
|              |                      |                        |                      |   * Must happen after any preceding local/generic load/load atomic/store/store atomic/atomicrmw.                                                                                            |
|              |                      |                        |                      |   * Must happen before any following global/generic load/load atomic/store/store atomic/atomicrmw.                                                                                          |
|              |                      |                        |                      |   * Ensures that all memory operations to local have completed before performing any following global memory operations.                                                                    |
|              |                      |                        |                      |   * Ensures that the preceding local/generic load atomic/atomicrmw with an equal or wider sync scope and memory ordering stronger than unordered                                            |
|              |                      |                        |                      |     (this is termed the fence-paired-atomic) has completed before following global memory operations.This satisfies the requirements of acquire.                                            |
|              |                      |                        |                      |   * Ensures that all previous memory operations have completed before a following local/generic store atomic/atomicrmw with an equal or                                                     |
|              |                      |                        |                      |     wider sync scope and memory ordering stronger than unordered (this is termed the fence-paired-atomic). This satisfies the requirements of release.                                      |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| fence        | acq_rel              |  * agent               | none                 | 1. s_waitcnt vmcnt(0) & lgkmcnt(0)                                                                                                                                                          |
|              |                      |  * system              |                      |   * If OpenCL and address space is not generic, omit lgkmcnt(0).                                                                                                                            |
|              |                      |                        |                      |     However, since LLVM currently has no address space on the fence need to conservatively always generate (see comment for previous fence).                                                |
|              |                      |                        |                      |   * Could be split into separate s_waitcnt vmcnt(0) and s_waitcnt lgkmcnt(0) to allow them to be independently moved according to the following rules.                                      |
|              |                      |                        |                      |   * s_waitcnt vmcnt(0) must happen after any preceding global/generic load/store/load atomic/store atomic/atomicrmw.                                                                        |
|              |                      |                        |                      |   * s_waitcnt lgkmcnt(0) must happen after any preceding local/generic load/store/load atomic/store atomic/atomicrmw.                                                                       |
|              |                      |                        |                      |   * Must happen before the following buffer_wbinvl1_vol.                                                                                                                                    |
|              |                      |                        |                      |   * Ensures that the preceding global/local/generic load atomic/atomicrmw with an equal or wider sync scope and                                                                             |
|              |                      |                        |                      |      memory ordering stronger than unordered (this is termed the fence-paired-atomic) has completed before invalidating the cache.                                                          |
|              |                      |                        |                      |      This satisfies the requirements of acquire.                                                                                                                                            |
|              |                      |                        |                      |   * Ensures that all previous memory operations have completed before a following global/local/generic store atomic/atomicrmw with                                                          |
|              |                      |                        |                      |     an equal or wider sync scope and memory ordering stronger than unordered (this is termed the fence-paired-atomic).This satisfies the requirements of release.                           |
|              |                      |                        |                      | 2. buffer_wbinvl1_vol                                                                                                                                                                       |
|              |                      |                        |                      |   * Must happen before any following global/generic load/load atomic/store/store atomic/atomicrmw.                                                                                          |
|              |                      |                        |                      |   * Ensures that following loads will not see stale global data. This satisfies the requirements of acquire.                                                                                |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| **Sequential Consistent Atomic**                                                                                                                                                                                                                                                  |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| load atomic  | seq_cst              |  * singlethread        |  *global             | Same as corresponding load atomic acquire.                                                                                                                                                  |
|              |                      |  * wavefront           |  * local             |                                                                                                                                                                                             |
|              |                      |  * workgroup           |  * generic           |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| load atomic  | seq_cst              |  * agent               |  * global            | 1. s_waitcnt vmcnt(0)                                                                                                                                                                       |
|              |                      |  * system              |  * local             |   * Must happen after preceding global/generic load atomic/store atomic/atomicrmw with memory ordering of seq_cst and with equal or wider sync scope.                                       |
|              |                      |                        |  * generic           |     (Note that seq_cst fences have their own s_waitcnt vmcnt(0) and so do not need to be considered.)                                                                                       |
|              |                      |                        |                      |   * Ensures any preceding sequential consistent global memory instructions have completed before executing this sequentially consistent instruction.                                        |
|              |                      |                        |                      |     This prevents reordering a seq_cst store followed by a seq_cst load (Note that seq_cst is stronger than acquire/release as the reordering of load acquire                               |
|              |                      |                        |                      |     followed by a store release is prevented by the waitcnt vmcnt(0) of the release, but there is nothing preventing a store release followed by load acquire from competing out of order.) |
|              |                      |                        |                      | 2.Following instructions same as corresponding load atomic acquire.                                                                                                                         |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| store atomic | seq_cst              |  * singlethread        |  * global            | Same as corresponding store atomic release.                                                                                                                                                 |
|              |                      |  * wavefront           |  * local             |                                                                                                                                                                                             |
|              |                      |  * workgroup           |  * generic           |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| store atomic | seq_cst              |  * agent               |  * global            | Sameas corresponding store atomic release.                                                                                                                                                  |
|              |                      |  * system              |  * generic           |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| atomicrmw    | seq_cst              |  * singlethread        |  * global            | Same as corresponding atomicrmw acq_rel.                                                                                                                                                    |
|              |                      |  * wavefront           |  * local             |                                                                                                                                                                                             |
|              |                      |  * workgroup           |  * generic           |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| atomicrmw    | seq_cst              |  * agent               |  * global            | Same as corresponding atomicrmw acq_rel.                                                                                                                                                    |
|              |                      |  * system              |  * generic           |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| fence        | seq_cst              |  * singlethread        | none                 | Same as corresponding fence acq_rel.                                                                                                                                                        |
|              |                      |  * wavefront           |                      |                                                                                                                                                                                             |
|              |                      |  * workgroup           |                      |                                                                                                                                                                                             |
|              |                      |  * agent               |                      |                                                                                                                                                                                             |
|              |                      |  * system              |                      |                                                                                                                                                                                             |
+--------------+----------------------+------------------------+----------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+



The memory order also adds the single thread optimization constrains defined in table AMDHSA Memory Model Single Thread Optimization Constraints GFX6-GFX9.

    AMDHSA Memory Model Single Thread Optimization Constraints GFX6-GFX9
    
+-------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| LLVM Memory | Optimization Constraints                                                                                                                                                                   |
+=============+============================================================================================================================================================================================+
| Ordering    |                                                                                                                                                                                            |
+-------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| unordered   | none                                                                                                                                                                                       |
+-------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| monotonic   | none                                                                                                                                                                                       |
+-------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| acquire     | * If a load atomic/atomicrmw then no following load/load atomic/store/ store atomic/atomicrmw/fence instruction can be moved before the acquire.                                           |
|             | * If a fence then same as load atomic, plus no preceding associated fence-paired-atomic can be moved after the fence.                                                                      |
+-------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| release     | * If a store atomic/atomicrmw then no preceding load/load atomic/store/ store atomic/atomicrmw/fence instruction can be moved after the release.                                           |
|             | * If a fence then same as store atomic, plus no following associated fence-paired-atomic can be moved before the fence.                                                                    |
+-------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| acq_rel     | Same constraints as both acquire and release.                                                                                                                                              |
+-------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| seq_cst     | * If a load atomic then same constraints as acquire, plus no preceding sequentially consistent load atomic/store atomic/atomicrmw/fence instruction can be moved after the seq_cst.        |
|             | * If a store atomic then the same constraints as release, plus no following sequentially consistent load atomic/store atomic/atomicrmw/fence instruction can be moved before the seq_cst.  |
|             | * If an atomicrmw/fence then same constraints as acq_rel.                                                                                                                                  |
+-------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+


.. _Trap Handler ABI1:

Trap Handler ABI
+++++++++++++++++

For code objects generated by AMDGPU backend for HSA [HSA] compatible runtimes (such as ROCm [AMD-ROCm]), the runtime installs a trap handler that supports the s_trap instruction with the following usage:

**AMDGPU Trap Handler for AMDHSA OS**

+---------------------+---------------+---------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| Usage               | Code Sequence | Trap Handler Inputs | Description Description                                                                                                                                  |
+=====================+===============+=====================+==========================================================================================================================================================+
| reserved            | s_trap 0x00   |                     | Reserved by hardware.                                                                                                                                    |
+---------------------+---------------+---------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| debugtrap(arg)      | s_trap 0x01   |  SGPR0-1:           | Reserved for HSA debugtrap intrinsic (not implemented).                                                                                                  |
|                     |               |  queue_ptr          |                                                                                                                                                          |
|                     |               |  VGPR0:             |                                                                                                                                                          |
|                     |               |  arg                |                                                                                                                                                          |
+---------------------+---------------+---------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| llvm.trap           | s_trap 0x02   |  SGPR0-1:           | Causes dispatch to be terminated and its associated queue put into the error state.                                                                      |
|                     |               |  queue_ptr          |                                                                                                                                                          |
+---------------------+---------------+---------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| llvm.debugtrap      | s_trap 0x03   |                     | If debugger not installed then behaves as a no-operation. The trap handler is entered and immediately returns to continue execution of the wavefront.    |
|                     |               |                     | If the debugger is installed, causes the debug trap to be reported by the debugger and the wavefront is put in the halt state until resumed by debugger. |                                                                                                                                                 |
+---------------------+---------------+---------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| reserved            | s_trap 0x04   |                     | Reserved                                                                                                                                                 |
+---------------------+---------------+---------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| reserved            | s_trap 0x05   |                     | Reserved                                                                                                                                                 |
+---------------------+---------------+---------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| reserved            | s_trap 0x06   |                     | Reserved                                                                                                                                                 |
+---------------------+---------------+---------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| debugger breakpoint | s_trap 0x07   |                     | Reserved for debugger breakpoints.                                                                                                                       |
+---------------------+---------------+---------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| reserved            | s_trap 0x08   |                     | Reserved                                                                                                                                                 |
+---------------------+---------------+---------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| reserved            | s_trap 0xfe   |                     | Reserved                                                                                                                                                 | 
+---------------------+---------------+---------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| reserved            | s_trap 0xff   |                     | Reserved                                                                                                                                                 |
+---------------------+---------------+---------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------+

.. _AMDPAL:

AMDPAL
-------
This section provides code conventions used when the target triple OS is amdpal (see :ref:`Target-Triples`) for passing runtime parameters from the application/runtime to each invocation of a hardware shader. These parameters include both generic, application-controlled parameters called user data as well as system-generated parameters that are a product of the draw or dispatch execution.

.. _User-Data:

User Data
++++++++++++

Each hardware stage has a set of 32-bit user data registers which can be written from a command buffer and then loaded into SGPRs when waves are launched via a subsequent dispatch or draw operation. This is the way most arguments are passed from the application/runtime to a hardware shader.

.. _Compute-User-Data:

Compute User Data
+++++++++++++++++++++++

Compute shader user data mappings are simpler than graphics shaders, and have a fixed mapping.

Note that there are always 10 available user data entries in registers - entries beyond that limit must be fetched from memory (via the spill table pointer) by the shader.

        **PAL Compute Shader User Data Registers**

  ================ ===================================================== 
   User Register 	  Description	 
  ================ ===================================================== 
   0	             Global Internal Table (32-bit pointer)
   1	             Per-Shader Internal Table (32-bit pointer)
   2 - 11	     Application-Controlled User Data (10 32-bit values)
   12	             Spill Table (32-bit pointer)
   13 - 14           Thread Group Count (64-bit pointer)
   15          	     GDS Range  
  ================ =====================================================                
 
.. _Graphics-User-Data:

Graphics User Data
+++++++++++++++++++++++

Graphics pipelines support a much more flexible user data mapping:

**PAL Graphics Shader User Data Registers**

 ==============  ===============================================================================
 User Register	 Description
 ==============  ===============================================================================
   0	         Global Internal Table (32-bit pointer)
                 Per-Shader Internal Table (32-bit pointer)
   1-15          Application Controlled User Data (1-15 Contiguous 32-bit Values in Registers)
                 Spill Table (32-bit pointer)
                 Draw Index (First Stage Only)
                 Vertex Offset (First Stage Only)
                 Instance Offset (First Stage Only)
 ==============  ===============================================================================

The placement of the global internal table remains fixed in the first user data SGPR register. Otherwise all parameters are optional, and can be mapped to any desired user data SGPR register, with the following regstrictions:

* Draw Index, Vertex Offset, and Instance Offset can only be used by the first activehardware stage in a graphics pipeline (i.e. where the API vertex shader runs).
* Application-controlled user data must be mapped into a contiguous range of user data registers.
* The application-controlled user data range supports compaction remapping, so only entries that are actually consumed by the shader must be assigned to corresponding registers. Note that in order to support an efficient runtime implementation, the remapping must pack registers in the same order as entries, with unused entries removed.

.. _Global-Internal-Table:

Global Internal Table
+++++++++++++++++++++++

The global internal table is a table of shader resource descriptors (SRDs) that define how certain engine-wide, runtime-managed resources should be accessed from a shader. The majority of these resources have HW-defined formats, and it is up to the compiler to write/read data as required by the target hardware.

The following table illustrates the required format:

**PAL Global Internal Table**
  =========  ==============================================
  Offset	 Description
  =========  ==============================================
    0-3	        Graphics Scratch SRD
    4-7	        Compute Scratch SRD
    8-11	ES/GS Ring Output SRD
    12-15	ES/GS Ring Input SRD
    16-19	GS/VS Ring Output #0
    20-23	GS/VS Ring Output #1
    24-27	GS/VS Ring Output #2
    28-31	GS/VS Ring Output #3
    32-35	GS/VS Ring Input SRD
    36-39	Tessellation Factor Buffer SRD
    40-43	Off-Chip LDS Buffer SRD
    44-47	Off-Chip Param Cache Buffer SRD
    48-51	Sample Position Buffer SRD
    52	        vaRange::ShadowDescriptorTable High Bits
  =========  ==============================================

The pointer to the global internal table passed to the shader as user data is a 32-bit pointer. The top 32 bits should be assumed to be the same as the top 32 bits of the pipeline, so the shader may use the program counter’s top 32 bits.


.. _Unspecified OS:

Unspecified OS
+++++++++++++++++

This section provides code conventions used when the target triple OS is empty (see :ref:`Target-Triples`).

.. _Trap Handler ABI2:

Trap Handler ABI
+++++++++++++++++

For code objects generated by AMDGPU backend for non-amdhsa OS, the runtime does not install a trap handler. The llvm.trap and llvm.debugtrap instructions are handled as follows:

**AMDGPU Trap Handler for Non-AMDHSA OS**

+----------------+---------------+-----------------------------------------------------------------+
| Usage          | Code Sequence | Description                                                     |
+================+===============+=================================================================+
| llvm.trap      | s_endpgm      | Causes wavefront to be terminated.                              |
+----------------+---------------+-----------------------------------------------------------------+
| llvm.debugtrap | none          | Compiler warning given that there is no trap handler installed. |
+----------------+---------------+-----------------------------------------------------------------+


.. _Source Languages:

Source Languages
#################

.. _OpenCL:

OpenCL
-------

When the language is OpenCL the following differences occur:

  1. The OpenCL memory model is used (see :ref:`Memory Model`).
  2. The AMDGPU backend adds additional arguments to the kernel's explicit arguments for the AMDHSA OS.
  3. Additional metadata is generated (see :ref:`Code Object Metadata`).

**OpenCL kernel implicit arguments appended for AMDHSA OS**

 ========= ============ =================  =========================================================
  Position   Byte Size    Byte Alignment	  Description
 ========= ============ =================  =========================================================
   1	     8	         8        	     OpenCL Global Offset X
   2	     8	         8        	     OpenCL Global Offset Y
   3	     8           8        	     OpenCL Global Offset Z
   4	     8           8        	     OpenCL address of printf buffer
   5	     8           8        	     OpenCL address of virtual queue used by enqueue_kernel
   6         8           8        	     OpenCL address of AqlWrap struct used by enqueue_kernel
   7         8           8                   Pointer argument used for Multi-gird synchronization
 ========= ============ =================  =========================================================

.. _HCC:

HCC
----

When the language is OpenCL the following differences occur:

    1. The HSA memory model is used (see Memory Model).


.. _Assembler:

Assembler
----------

AMDGPU backend has LLVM-MC based assembler which is currently in development. It supports AMDGCN GFX6-GFX8.

This section describes general syntax for instructions and operands.

.. _Instructions:

Instructions
-------------
An instruction has the following `syntax <http://releases.llvm.org/8.0.1/docs/AMDGPUInstructionSyntax.html>`_:

<opcode>    <operand0>, <operand1>,...    <modifier0> <modifier1>...

`Operands <http://releases.llvm.org/8.0.1/docs/AMDGPUOperandSyntax.html>`_ are normally comma-separated while `modifiers <http://releases.llvm.org/8.0.1/docs/AMDGPUModifierSyntax.html>`_ are space-separated.

The order of operands and modifiers is fixed. Most modifiers are optional and may be omitted.

See detailed instruction syntax description for `GFX7 <http://releases.llvm.org/8.0.1/docs/AMDGPU/AMDGPUAsmGFX7.html>`_, `GFX8 <http://releases.llvm.org/8.0.1/docs/AMDGPU/AMDGPUAsmGFX8.html>`_ and `GFX9 <http://releases.llvm.org/8.0.1/docs/AMDGPU/AMDGPUAsmGFX9.html>`_.

Note that features under development are not included in this description.

For more information about instructions, their semantics and supported combinations of operands, refer to one of instruction set architecture manuals [AMD-GCN-GFX6], [AMD-GCN-GFX7], [AMD-GCN-GFX8] and [AMD-GCN-GFX9] here :ref: `Additional Documentation`.

.. _Operans:

Operands
---------

The following syntax for register operands is supported:

 * SGPR registers: s0, ... or s[0], ...
 * VGPR registers: v0, ... or v[0], ...
 * TTMP registers: ttmp0, ... or ttmp[0], ...
 * Special registers: exec (exec_lo, exec_hi), vcc (vcc_lo, vcc_hi), flat_scratch (flat_scratch_lo, flat_scratch_hi)
 * Special trap registers: tba (tba_lo, tba_hi), tma (tma_lo, tma_hi)
 * Register pairs, quads, etc: s[2:3], v[10:11], ttmp[5:6], s[4:7], v[12:15], ttmp[4:7], s[8:15], ...
 * Register lists: [s0, s1], [ttmp0, ttmp1, ttmp2, ttmp3]
 * Register index expressions: v[2*2], s[1-1:2-1]
 * ‘off’ indicates that an operand is not enabled

The following extra operands are supported:

 * offset, offset0, offset1
 * idxen, offen bits
 * glc, slc, tfe bits
 * waitcnt: integer or combination of counter values
 * VOP3 modifiers:
    * abs (| |), neg (-)
 * DPP modifiers:
    * row_shl, row_shr, row_ror, row_rol
    * row_mirror, row_half_mirror, row_bcast
    * wave_shl, wave_shr, wave_ror, wave_rol, quad_perm
    * row_mask, bank_mask, bound_ctrl
 * SDWA modifiers:
    * dst_sel, src0_sel, src1_sel (BYTE_N, WORD_M, DWORD)
    * dst_unused (UNUSED_PAD, UNUSED_SEXT, UNUSED_PRESERVE)
    * abs, neg, sext

Detailed description of operands may be found `here <http://releases.llvm.org/8.0.1/docs/AMDGPUOperandSyntax.html>`_.

.. _Modifers:

Modifers
---------

Detailed description of modifers may be found `here <http://releases.llvm.org/8.0.1/docs/AMDGPUModifierSyntax.html>`_

.. _Instruction Examples:

Instruction Examples
+++++++++++++++++++++

.. _DS:

DS
***

::
 
 ds_add_u32 v2, v4 offset:16
 ds_write_src2_b64 v2 offset0:4 offset1:8
 ds_cmpst_f32 v2, v4, v6 
 ds_min_rtn_f64 v[8:9], v2, v[4:5]
 

For full list of supported instructions, refer to “LDS/GDS instructions” in ISA Manual.

.. _FLAT:

FLAT
*****
::
 
 flat_load_dword v1, v[3:4]
 flat_store_dwordx3 v[3:4], v[5:7]
 flat_atomic_swap v1, v[3:4], v5 glc
 flat_atomic_cmpswap v1, v[3:4], v[5:6] glc slc
 flat_atomic_fmax_x2 v[1:2], v[3:4], v[5:6] glc
 

For full list of supported instructions, refer to “FLAT instructions” in ISA Manual.


.. _MUBUF:

MUBUF
******
::
  
 buffer_load_dword v1, off, s[4:7], s1
 buffer_store_dwordx4 v[1:4], v2, ttmp[4:7], s1 offen offset:4 glc tfe
 buffer_store_format_xy v[1:2], off, s[4:7], s1
 buffer_wbinvl1
 buffer_atomic_inc v1, v2, s[8:11], s4 idxen offset:4 slc

For full list of supported instructions, refer to “MUBUF Instructions” in ISA Manual.

.. _SMRD/SMEM:

SMRD/SMEM
**********
::
 
 s_load_dword s1, s[2:3], 0xfc
 s_load_dwordx8 s[8:15], s[2:3], s4
 s_load_dwordx16 s[88:103], s[2:3], s4
 s_dcache_inv_vol
 s_memtime s[4:5]

For full list of supported instructions, refer to “Scalar Memory Operations” in ISA Manual.

.. _SOP1:

SOP1
*****
::
 
 s_mov_b32 s1, s2
 s_mov_b64 s[0:1], 0x80000000
 s_cmov_b32 s1, 200
 s_wqm_b64 s[2:3], s[4:5]
 s_bcnt0_i32_b64 s1, s[2:3]
 s_swappc_b64 s[2:3], s[4:5]
 s_cbranch_join s[4:5]

For full list of supported instructions, refer to “SOP1 Instructions” in ISA Manual.

.. _SOP2:

SOP2
*****
::
 
 s_add_u32 s1, s2, s3
 s_and_b64 s[2:3], s[4:5], s[6:7]
 s_cselect_b32 s1, s2, s3
 s_andn2_b32 s2, s4, s6
 s_lshr_b64 s[2:3], s[4:5], s6
 s_ashr_i32 s2, s4, s6
 s_bfm_b64 s[2:3], s4, s6
 s_bfe_i64 s[2:3], s[4:5], s6
 s_cbranch_g_fork s[4:5], s[6:7]
 
For full list of supported instructions, refer to “SOP2 Instructions” in ISA Manual.

.. _SOPC:

SOPC
*****
::
 
 s_cmp_eq_i32 s1, s2
 s_bitcmp1_b32 s1, s2
 s_bitcmp0_b64 s[2:3], s4
 s_setvskip s3, s5
 
For full list of supported instructions, refer to “SOPC Instructions” in ISA Manual.

.. _SOPP:

SOPP
*****
::
 
 s_barrier
 s_nop 2
 s_endpgm
 s_waitcnt 0 ; Wait for all counters to be 0
 s_waitcnt vmcnt(0) & expcnt(0) & lgkmcnt(0) ; Equivalent to above
 s_waitcnt vmcnt(1) ; Wait for vmcnt counter to be 1.
 s_sethalt 9
 s_sleep 10
 s_sendmsg 0x1
 s_sendmsg sendmsg(MSG_INTERRUPT)
 s_trap 1
 
For full list of supported instructions, refer to “SOPP Instructions” in ISA Manual.

Unless otherwise mentioned, little verification is performed on the operands of SOPP Instructions, so it is up to the programmer to be familiar with the range or acceptable values.

.. _VALU:

VALU
*****
For vector ALU instruction opcodes (VOP1, VOP2, VOP3, VOPC, VOP_DPP, VOP_SDWA), the assembler will automatically use optimal encoding based on its operands. To force specific encoding, one can add a suffix to the opcode of the instruction:

 * _e32 for 32-bit VOP1/VOP2/VOPC
 * _e64 for 64-bit VOP3
 * _dpp for VOP_DPP
 * _sdwa for VOP_SDWA

VOP1/VOP2/VOP3/VOPC examples
*****************************
 
::

 v_mov_b32 v1, v2
 v_mov_b32_e32 v1, v2
 v_nop
 v_cvt_f64_i32_e32 v[1:2], v2
 v_floor_f32_e32 v1, v2
 v_bfrev_b32_e32 v1, v2
 v_add_f32_e32 v1, v2, v3
 v_mul_i32_i24_e64 v1, v2, 3
 v_mul_i32_i24_e32 v1, -3, v3
 v_mul_i32_i24_e32 v1, -100, v3
 v_addc_u32 v1, s[0:1], v2, v3, s[2:3]
 v_max_f16_e32 v1, v2, v3

VOP_DPP examples
******************
 
::
  
 v_mov_b32 v0, v0 quad_perm:[0,2,1,1]
 v_sin_f32 v0, v0 row_shl:1 row_mask:0xa bank_mask:0x1 bound_ctrl:0
 v_mov_b32 v0, v0 wave_shl:1
 v_mov_b32 v0, v0 row_mirror
 v_mov_b32 v0, v0 row_bcast:31
 v_mov_b32 v0, v0 quad_perm:[1,3,0,1] row_mask:0xa bank_mask:0x1 bound_ctrl:0
 v_add_f32 v0, v0, |v0| row_shl:1 row_mask:0xa bank_mask:0x1 bound_ctrl:0
 v_max_f16 v1, v2, v3 row_shl:1 row_mask:0xa bank_mask:0x1 bound_ctrl:0

VOP_SDWA examples
******************

::
 
 v_mov_b32 v1, v2 dst_sel:BYTE_0 dst_unused:UNUSED_PRESERVE src0_sel:DWORD
 v_min_u32 v200, v200, v1 dst_sel:WORD_1 dst_unused:UNUSED_PAD src0_sel:BYTE_1 src1_sel:DWORD
 v_sin_f32 v0, v0 dst_unused:UNUSED_PAD src0_sel:WORD_1
 v_fract_f32 v0, |v0| dst_sel:DWORD dst_unused:UNUSED_PAD src0_sel:WORD_1
 v_cmpx_le_u32 vcc, v1, v2 src0_sel:BYTE_2 src1_sel:WORD_0
 
For full list of supported instructions, refer to “Vector ALU instructions”.


.. _Code Object V2 Predefined Symbols (-mattr=-code-object-v3):

Code Object V2 Predefined Symbols (-mattr=-code-object-v3)
------------------------------------------------------------

**Warning**

::

 Code Object V2 is not the default code object version emitted by this version of LLVM. For a description of the predefined symbols
 available  with the default configuration (Code Object V3) see :ref:`Code Object V3 Predefined Symbols (-mattr=+code-object-v3)`.


The AMDGPU assembler defines and updates some symbols automatically. These symbols do not affect code generation.

.. _.option.machine_version_major:

.option.machine_version_major
++++++++++++++++++++++++++++++

Set to the GFX major generation number of the target being assembled for. For example, when assembling for a “GFX9” target this will be set to the integer value “9”. The possible GFX major generation numbers are presented in :ref:`Processors`.


.. _.option.machine_version_minor:

.option.machine_version_minor
++++++++++++++++++++++++++++++

Set to the GFX minor generation number of the target being assembled for. For example, when assembling for a “GFX810” target this will be set to the integer value “1”. The possible GFX minor generation numbers are presented in :ref:`Processors`.
.option.machine_version_stepping

Set to the GFX stepping generation number of the target being assembled for. For example, when assembling for a “GFX704” target this will be set to the integer value “4”. The possible GFX stepping generation numbers are presented in :ref:`Processors`.


.. _.option.machine_version_stepping:

.option.machine_version_stepping
+++++++++++++++++++++++++++++++++

Set to the GFX stepping generation number of the target being assembled for. For example, when assembling for a “GFX704” target this will be set to the integer value “4”. The possible GFX stepping generation numbers are presented in :ref:`Processors`.

.. _.kernel.vgpr_count:

.kernel.vgpr_count
++++++++++++++++++++

Set to zero each time a .amdgpu_hsa_kernel (name) directive is encountered. At each instruction, if the current value of this symbol is less than or equal to the maximum VPGR number explicitly referenced within that instruction then the symbol value is updated to equal that VGPR number plus one.

.. _.kernel.sgpr_count:

.kernel.sgpr_count
+++++++++++++++++++++

Set to zero each time a .amdgpu_hsa_kernel (name) directive is encountered. At each instruction, if the current value of this symbol is less than or equal to the maximum VPGR number explicitly referenced within that instruction then the symbol value is updated to equal that SGPR number plus one.

.. _Code Object V2 Directives (-mattr=-code-object-v3):

Code Object V2 Directives (-mattr=-code-object-v3)
---------------------------------------------------

**Warning**

::

  Code Object V2 is not the default code object version emitted by this version of LLVM. For a description of the directives supported 
  with  the default configuration (Code Object V3) see :ref:`Code Object V3 Directives (-mattr=+code-object-v3)`.

AMDGPU ABI defines auxiliary data in output code object. In assembly source, one can specify them with assembler directives.

.. _.hsa_code_object_version major, minor:

.hsa_code_object_version major, minor
++++++++++++++++++++++++++++++++++++++

major and minor are integers that specify the version of the HSA code object that will be generated by the assembler.

.. _.hsa_code_object_isa [major, minor, stepping, vendor, arch]:

.hsa_code_object_isa [major, minor, stepping, vendor, arch]
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

major, minor, and stepping are all integers that describe the instruction set architecture (ISA) version of the assembly program.

vendor and arch are quoted strings. vendor should always be equal to “AMD” and arch should always be equal to “AMDGPU”.

By default, the assembler will derive the ISA version, vendor, and arch from the value of the -mcpu option that is passed to the assembler.

.. _.amdgpu_hsa_kernel (name):

.amdgpu_hsa_kernel (name)
++++++++++++++++++++++++++++

This directives specifies that the symbol with given name is a kernel entry point (label) and the object should contain corresponding symbol of type STT_AMDGPU_HSA_KERNEL.

.. _.amd_kernel_code_t:

.amd_kernel_code_t
+++++++++++++++++++

This directive marks the beginning of a list of key / value pairs that are used to specify the amd_kernel_code_t object that will be emitted by the assembler. The list must be terminated by the .end_amd_kernel_code_t directive. For any amd_kernel_code_t values that are unspecified a default value will be used. The default value for all keys is 0, with the following exceptions:

    * amd_code_version_major defaults to 1.
    * amd_kernel_code_version_minor defaults to 2.
    * amd_machine_kind defaults to 1.
    * amd_machine_version_major, machine_version_minor, and amd_machine_version_stepping are derived from the value of the -mcpu option that is passed to the assembler.
    * kernel_code_entry_byte_offset defaults to 256.
    * wavefront_size defaults 6 for all targets before GFX10. For GFX10 onwards defaults to 6 if target feature wavefrontsize64 is enabled, otherwise 5. Note that wavefront size is specified as a power of two, so a value of n means a size of 2^ n.
    * call_convention defaults to -1.
    * kernarg_segment_alignment, group_segment_alignment, and private_segment_alignment default to 4. Note that alignments are specified as a power of 2, so a value of n means an alignment of 2^ n.
    * enable_wgp_mode defaults to 1 if target feature cumode is disabled for GFX10 onwards.
    * enable_mem_ordered defaults to 1 for GFX10 onwards.

The .amd_kernel_code_t directive must be placed immediately after the function label and before any instructions.

For a full list of amd_kernel_code_t keys, refer to AMDGPU ABI document, comments in lib/Target/AMDGPU/AmdKernelCodeT.h and test/CodeGen/AMDGPU/hsa.s.

.. _Code Object V2 Example Source Code (-mattr=-code-object-v3):

Code Object V2 Example Source Code (-mattr=-code-object-v3)
------------------------------------------------------------
**Warning**

::

 Code Object V2 is not the default code object version emitted by this version of LLVM. For a description of the predefined symbols 
 available with the default configuration (Code Object V3).

Here is an example of a minimal assembly source file, defining one HSA kernel:

::

  .hsa_code_object_version 1,0
  .hsa_code_object_isa
  .hsatext
  .globl  hello_world
  .p2align 8
  .amdgpu_hsa_kernel hello_world

  hello_world:

   .amd_kernel_code_t
      enable_sgpr_kernarg_segment_ptr = 1
      is_ptr64 = 1
      compute_pgm_rsrc1_vgprs = 0
      compute_pgm_rsrc1_sgprs = 0
      compute_pgm_rsrc2_user_sgpr = 2
      compute_pgm_rsrc1_wgp_mode = 0
      compute_pgm_rsrc1_mem_ordered = 0
      compute_pgm_rsrc1_fwd_progress = 1
  .end_amd_kernel_code_t

  s_load_dwordx2 s[0:1], s[0:1] 0x0
  v_mov_b32 v0, 3.14159
  s_waitcnt lgkmcnt(0)
  v_mov_b32 v1, s0
  v_mov_b32 v2, s1
  flat_store_dword v[1:2], v0
  s_endpgm
  .Lfunc_end0:
     .size   hello_world, .Lfunc_end0-hello_world


.. _Code Object V3 Predefined Symbols (-mattr=+code-object-v3):

Code Object V3 Predefined Symbols (-mattr=+code-object-v3)
-----------------------------------------------------------

The AMDGPU assembler defines and updates some symbols automatically. These symbols do not affect code generation.

.. _.amdgcn.gfx_generation_number:

.amdgcn.gfx_generation_number
++++++++++++++++++++++++++++++

Set to the GFX major generation number of the target being assembled for. For example, when assembling for a “GFX9” target this will be set to the integer value “9”. The possible GFX major generation numbers are presented in :ref:`Processors`.

.. _.amdgcn.gfx_generation_minor:

.amdgcn.gfx_generation_minor
++++++++++++++++++++++++++++++

Set to the GFX minor generation number of the target being assembled for. For example, when assembling for a “GFX810” target this will be set to the integer value “1”. The possible GFX minor generation numbers are presented in :ref:`Processors`.

.. _.amdgcn.gfx_generation_stepping:

.amdgcn.gfx_generation_stepping
+++++++++++++++++++++++++++++++++

Set to the GFX stepping generation number of the target being assembled for. For example, when assembling for a “GFX704” target this will be set to the integer value “4”. The possible GFX stepping generation numbers are presented in :ref:`Processors`.

.. _.amdgcn.next_free_vgpr:

.amdgcn.next_free_vgpr
+++++++++++++++++++++++

Set to zero before assembly begins. At each instruction, if the current value of this symbol is less than or equal to the maximum VGPR number explicitly referenced within that instruction then the symbol value is updated to equal that VGPR number plus one.

May be used to set the .amdhsa_next_free_vpgr directive in AMDHSA Kernel Assembler Directives.

May be set at any time, e.g. manually set to zero at the start of each kernel.

.. _.amdgcn.next_free_sgpr:

.amdgcn.next_free_sgpr
++++++++++++++++++++++++

Set to zero before assembly begins. At each instruction, if the current value of this symbol is less than or equal the maximum SGPR number explicitly referenced within that instruction then the symbol value is updated to equal that SGPR number plus one.

May be used to set the .amdhsa_next_free_spgr directive in AMDHSA Kernel Assembler Directives.

May be set at any time, e.g. manually set to zero at the start of each kernel.

.. _Code Object V3 Directives (-mattr=+code-object-v3):

Code Object V3 Directives (-mattr=+code-object-v3)
---------------------------------------------------

Directives which begin with .amdgcn are valid for all amdgcn architecture processors, and are not OS-specific. Directives which begin with .amdhsa are specific to amdgcn architecture processors when the amdhsa OS is specified. See Target Triples and Processors.

.. _.amdgcn_target:

.amdgcn_target <target>
++++++++++++++++++++++++

Optional directive which declares the target supported by the containing assembler source file. Valid values are described in Code Object Target Identification. Used by the assembler to validate command-line options such as -triple, -mcpu, and those which specify target features.

.. _.amdhsa_kernel:

.amdhsa_kernel <name>
++++++++++++++++++++++

Creates a correctly aligned AMDHSA kernel descriptor and a symbol, <name>.kd, in the current location of the current section. Only valid when the OS is amdhsa. <name> must be a symbol that labels the first instruction to execute, and does not need to be previously defined.

Marks the beginning of a list of directives used to generate the bytes of a kernel descriptor, as described in Kernel Descriptor. Directives which may appear in this list are described in AMDHSA Kernel Assembler Directives. Directives may appear in any order, must be valid for the target being assembled for, and cannot be repeated. Directives support the range of values specified by the field they reference in Kernel Descriptor. If a directive is not specified, it is assumed to have its default value, unless it is marked as “Required”, in which case it is an error to omit the directive. This list of directives is terminated by an .end_amdhsa_kernel directive.

**AMDHSA Kernel Assembler Directives**

=======================================================   ========== ===============    ==================================================================================================================================================
           Directive	                                   Default    Supported On	  Description
=======================================================   ========== ===============    ==================================================================================================================================================
.amdhsa_group_segment_fixed_size	                     0	      GFX6-GFX9	        Controls GROUP_SEGMENT_FIXED_SIZE in Kernel Descriptor for GFX6 GFX6-GFX9.
.amdhsa_private_segment_fixed_size	                     0	      GFX6-GFX9	        Controls PRIVATE_SEGMENT_FIXED_SIZE in Kernel Descriptor for GFX6-GFX9.
.amdhsa_user_sgpr_private_segment_buffer	             0	      GFX6-GFX9	        Controls ENABLE_SGPR_PRIVATE_SEGMENT_BUFFER in Kernel Descriptor for GFX6-GFX9.
.amdhsa_user_sgpr_dispatch_ptr	                             0	      GFX6-GFX9	        Controls ENABLE_SGPR_DISPATCH_PTR in Kernel Descriptor for GFX6-GFX9.
.amdhsa_user_sgpr_queue_ptr	                             0	      GFX6-GFX9	        Controls ENABLE_SGPR_QUEUE_PTR in Kernel Descriptor for GFX6-GFX9.
.amdhsa_user_sgpr_kernarg_segment_ptr	                     0	      GFX6-GFX9	        Controls ENABLE_SGPR_KERNARG_SEGMENT_PTR in Kernel Descriptor for GFX6-GFX9.
.amdhsa_user_sgpr_dispatch_id	                             0	      GFX6-GFX9	        Controls ENABLE_SGPR_DISPATCH_ID in Kernel Descriptor for GFX6-GFX9.
.amdhsa_user_sgpr_flat_scratch_init	                     0	      GFX6-GFX9	        Controls ENABLE_SGPR_FLAT_SCRATCH_INIT in Kernel Descriptor for GFX6-GFX9.
.amdhsa_user_sgpr_private_segment_size	                     0	      GFX6-GFX9	        Controls ENABLE_SGPR_PRIVATE_SEGMENT_SIZE in Kernel Descriptor for GFX6-GFX9.
.amdhsa_system_sgpr_private_segment_wavefront_offset	     0	      GFX6-GFX9	        Controls ENABLE_SGPR_PRIVATE_SEGMENT_WAVEFRONT_OFFSET in compute_pgm_rsrc2 for GFX6-GFX9.
.amdhsa_system_sgpr_workgroup_id_x	                     1	      GFX6-GFX9	        Controls ENABLE_SGPR_WORKGROUP_ID_X in compute_pgm_rsrc2 for GFX6-GFX9.
.amdhsa_system_sgpr_workgroup_id_y	                     0	      GFX6-GFX9	        Controls ENABLE_SGPR_WORKGROUP_ID_Y in compute_pgm_rsrc2 for GFX6-GFX9.
.amdhsa_system_sgpr_workgroup_id_z	                     0	      GFX6-GFX9  	Controls ENABLE_SGPR_WORKGROUP_ID_Z in compute_pgm_rsrc2 for GFX6-GFX9.
.amdhsa_system_sgpr_workgroup_info	                     0	      GFX6-GFX9	        Controls ENABLE_SGPR_WORKGROUP_INFO in compute_pgm_rsrc2 for GFX6-GFX9.
.amdhsa_system_vgpr_workitem_id 	                     0	      GFX6-GFX9	        Controls ENABLE_VGPR_WORKITEM_ID in compute_pgm_rsrc2 for GFX6-GFX9. Possible values are defined in System VGPR Work-Item ID Enumeration Values.
.amdhsa_next_free_vgpr	Required	                              GFX6-GFX9	        Maximum VGPR number explicitly referenced, plus one. Used to calculate GRANULATED_WORKITEM_VGPR_COUNT in compute_pgm_rsrc1 for GFX6-GFX9.
.amdhsa_next_free_sgpr	Required	                              GFX6-GFX9	        Maximum SGPR number explicitly referenced, plus one. Used to calculate GRANULATED_WAVEFRONT_SGPR_COUNT in compute_pgm_rsrc1 for GFX6-GFX9.
.amdhsa_reserve_vcc	                                     1	      GFX6-GFX9	        Whether the kernel may use the special VCC SGPR. Used to calculate GRANULATED_WAVEFRONT_SGPR_COUNT in compute_pgm_rsrc1 for GFX6-GFX9.
.amdhsa_reserve_flat_scratch	                             1	      GFX7-GFX9	        Whether the kernel may use flat instructions to access scratch memory. Used to calculate GRANULATED_WAVEFRONT_SGPR_COUNT in compute_pgm_rsrc1 for GFX6-GFX9.
.amdhsa_reserve_xnack_mask	                           Target     GFX8-GFX9	        Whether the kernel may trigger XNACK replay. Used to calculate GRANULATED_WAVEFRONT_SGPR_COUNT in compute_pgm_rsrc1 for GFX6-GFX9.
                                                           Feature
                                                           Specific
                                                           (+xnack)
.amdhsa_float_round_mode_32	                             0	      GFX6-GFX9	        Controls FLOAT_ROUND_MODE_32 in compute_pgm_rsrc1 for GFX6-GFX9. Possible values are defined in Floating Point Rounding Mode Enumeration Values.
.amdhsa_float_round_mode_16_64	                             0	      GFX6-GFX9	        Controls FLOAT_ROUND_MODE_16_64 in compute_pgm_rsrc1 for GFX6-GFX9. Possible values are defined in Floating Point Rounding Mode Enumeration Values.
.amdhsa_float_denorm_mode_32	                             0	      GFX6-GFX9	        Controls FLOAT_DENORM_MODE_32 in compute_pgm_rsrc1 for GFX6-GFX9. Possible values are defined in Floating Point Denorm Mode Enumeration Values.
.amdhsa_float_denorm_mode_16_64	                             3	      GFX6-GFX9	        Controls FLOAT_DENORM_MODE_16_64 in compute_pgm_rsrc1 for GFX6-GFX9. Possible values are defined in Floating Point Denorm Mode Enumeration Values.
.amdhsa_dx10_clamp	                                     1	      GFX6-GFX9	        Controls ENABLE_DX10_CLAMP in compute_pgm_rsrc1 for GFX6-GFX9.
.amdhsa_ieee_mode	                                     1	      GFX6-GFX9	        Controls ENABLE_IEEE_MODE in compute_pgm_rsrc1 for GFX6-GFX9.
.amdhsa_fp16_overflow	                                     0	      GFX9	        Controls FP16_OVFL in compute_pgm_rsrc1 for GFX6-GFX9.
.amdhsa_exception_fp_ieee_invalid_op	                     0	      GFX6-GFX9	        Controls ENABLE_EXCEPTION_IEEE_754_FP_INVALID_OPERATION in compute_pgm_rsrc2 for GFX6-GFX9.
.amdhsa_exception_fp_denorm_src	                             0	      GFX6-GFX9	        Controls ENABLE_EXCEPTION_FP_DENORMAL_SOURCE in compute_pgm_rsrc2 for GFX6-GFX9.
.amdhsa_exception_fp_ieee_div_zero	                     0	      GFX6-GFX9	        Controls ENABLE_EXCEPTION_IEEE_754_FP_DIVISION_BY_ZERO in compute_pgm_rsrc2 for GFX6-GFX9.
.amdhsa_exception_fp_ieee_overflow	                     0	      GFX6-GFX9	        Controls ENABLE_EXCEPTION_IEEE_754_FP_OVERFLOW in compute_pgm_rsrc2 for GFX6-GFX9.
.amdhsa_exception_fp_ieee_underflow	                     0	      GFX6-GFX9	        Controls ENABLE_EXCEPTION_IEEE_754_FP_UNDERFLOW in compute_pgm_rsrc2 for GFX6-GFX9.
.amdhsa_exception_fp_ieee_inexact	                     0	      GFX6-GFX9	        Controls ENABLE_EXCEPTION_IEEE_754_FP_INEXACT in compute_pgm_rsrc2 for GFX6-GFX9.
.amdhsa_exception_int_div_zero	                             0	      GFX6-GFX9	        Controls ENABLE_EXCEPTION_INT_DIVIDE_BY_ZERO in compute_pgm_rsrc2 for GFX6-GFX9.
=======================================================   ========== ===============    ==================================================================================================================================================

.. _.amdgpu_metadata:

.amdgpu_metadata
+++++++++++++++++

Optional directive which declares the contents of the NT_AMDGPU_METADATA note record.

The contents must be in the [YAML] markup format, with the same structure and semantics described in Code Object V3 Metadata (-mattr=+code-object-v3).

This directive is terminated by an .end_amdgpu_metadata directive.

.. _Code Object V3 Example Source Code (-mattr=+code-object-v3):

Code Object V3 Example Source Code (-mattr=+code-object-v3)
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

Here is an example of a minimal assembly source file, defining one HSA kernel:

::

 .amdgcn_target "amdgcn-amd-amdhsa--gfx900+xnack" // optional
 .text
 .globl hello_world
 .p2align 8
 .type hello_world,@function
 hello_world:
  s_load_dwordx2 s[0:1], s[0:1] 0x0
  v_mov_b32 v0, 3.14159
  s_waitcnt lgkmcnt(0)
  v_mov_b32 v1, s0
  v_mov_b32 v2, s1
  flat_store_dword v[1:2], v0
  s_endpgm
 .Lfunc_end0:
  .size   hello_world, .Lfunc_end0-hello_world

 .rodata
 .p2align 6
 .amdhsa_kernel hello_world
  .amdhsa_user_sgpr_kernarg_segment_ptr 1
  .amdhsa_next_free_vgpr .amdgcn.next_free_vgpr
  .amdhsa_next_free_sgpr .amdgcn.next_free_sgpr
 .end_amdhsa_kernel

 .amdgpu_metadata
 ---
 amdhsa.version:
  - 1
  - 0
amdhsa.kernels:
  - .name: hello_world
    .symbol: hello_world.kd
    .kernarg_segment_size: 48
    .group_segment_fixed_size: 0
    .private_segment_fixed_size: 0
    .kernarg_segment_align: 4
    .wavefront_size: 64
    .sgpr_count: 2
    .vgpr_count: 3
    .max_flat_workgroup_size: 256
 ...
 .end_amdgpu_metadata

If an assembly source file contains multiple kernels and/or functions, the .amdgcn.next_free_vgpr and .amdgcn.next_free_sgpr symbols may be reset using the .set <symbol>, <expression> directive. For example, in the case of two kernels, where function1 is only called from kernel1 it is sufficient to group the function with the kernel that calls it and reset the symbols between the two connected components:

::

 .amdgcn_target "amdgcn-amd-amdhsa--gfx900+xnack" // optional
 // gpr tracking symbols are implicitly set to zero
 .text 
 .globl kern0
 .p2align 8
 .type kern0,@function
 kern0:
  // ...
  s_endpgm
 .Lkern0_end:
  .size   kern0, .Lkern0_end-kern0
 .rodata
 .p2align 6
 .amdhsa_kernel kern0
  // ...
  .amdhsa_next_free_vgpr .amdgcn.next_free_vgpr
  .amdhsa_next_free_sgpr .amdgcn.next_free_sgpr
 .end_amdhsa_kernel
 // reset symbols to begin tracking usage in func1 and kern1
 .set .amdgcn.next_free_vgpr, 0
 .set .amdgcn.next_free_sgpr, 0

 .text
 .hidden func1
 .global func1
 .p2align 2
 .type func1,@function
 func1:
  // ...
  s_setpc_b64 s[30:31]
 .Lfunc1_end:
 .size func1, .Lfunc1_end-func1
 .globl kern1
 .p2align 8
 .type kern1,@function
 kern1:
  // ...
  s_getpc_b64 s[4:5]
  s_add_u32 s4, s4, func1@rel32@lo+4
  s_addc_u32 s5, s5, func1@rel32@lo+4
  s_swappc_b64 s[30:31], s[4:5]
  // ...
  s_endpgm
 .Lkern1_end:
  .size   kern1, .Lkern1_end-kern1

 .rodata
 .p2align 6
 .amdhsa_kernel kern1
  // ...
  .amdhsa_next_free_vgpr .amdgcn.next_free_vgpr
  .amdhsa_next_free_sgpr .amdgcn.next_free_sgpr
.end_amdhsa_kernel


These symbols cannot identify connected components in order to automatically track the usage for each kernel. However, in some cases careful organization of the kernels and functions in the source file means there is minimal additional effort required to accurately calculate GPR usage.


.. _Additional Documentation:

Additional Documentation
*************************

`[AMD-RADEON-HD-2000-3000] <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id3>`_	`AMD R6xx shader ISA <http://developer.amd.com/wordpress/media/2012/10/R600_Instruction_Set_Architecture.pdf>`_

`[AMD-RADEON-HD-4000] <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id4>`_	`AMD R7xx shader ISA <http://developer.amd.com/wordpress/media/2012/10/R700-Family_Instruction_Set_Architecture.pdf>`_

`[AMD-RADEON-HD-5000] <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id5>`_	`AMD Evergreen shader ISA <http://developer.amd.com/wordpress/media/2012/10/AMD_Evergreen-Family_Instruction_Set_Architecture.pdf>`_

[AMD-RADEON-HD-6000] <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id6>`_	`AMD Cayman/Trinity shader ISA <http://developer.amd.com/wordpress/media/2012/10/AMD_HD_6900_Series_Instruction_Set_Architecture.pdf>`_

[AMD-GCN-GFX6]	(`1 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id7>`_, `2 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id39>`_) `AMD Southern Islands Series ISA <http://developer.amd.com/wordpress/media/2012/12/AMD_Southern_Islands_Instruction_Set_Architecture.pdf>`_

[AMD-GCN-GFX7]	(`1 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id8>`_, `2 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id40>`_) `AMD Sea Islands Series ISA <http://developer.amd.com/wordpress/media/2013/07/AMD_Sea_Islands_Instruction_Set_Architecture.pdf>`_

[AMD-GCN-GFX8]	(`1 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id9>`_, `2 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id41>`_) `AMD GCN3 Instruction Set Architecture <http://amd-dev.wpengine.netdna-cdn.com/wordpress/media/2013/12/AMD_GCN3_Instruction_Set_Architecture_rev1.1.pdf>`_

[AMD-GCN-GFX9]	(`1 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id10>`_, `2 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id42>`_) `AMD “Vega” Instruction Set Architecture <http://developer.amd.com/wordpress/media/2013/12/Vega_Shader_ISA_28July2017.pdf>`_

[AMD-ROCm]	(`1 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id2>`_, `2 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id21>`_, `3 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id26>`_, `4 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id37>`_) `ROCm: Open Platform for Development, Discovery and Education Around GPU Computing <http://gpuopen.com/compute-product/rocm/>`_

[AMD-ROCm-github](`1 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id32>`_, `2 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id33>`_) `ROCm github <http://github.com/RadeonOpenCompute>`_

[HSA]	(`1 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id1>`_, `2 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id11>`_, `3 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id20>`_, `4 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id25>`_, `5 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id29>`_, `6 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id30>`_, `7 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id31>`_, `8 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id34>`_, `9 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id36>`_) `Heterogeneous System Architecture (HSA) Foundation <http://www.hsafoundation.com/>`_

[`DWARF <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id24>`_]	`DWARF Debugging Information Format <http://dwarfstd.org/>`_

[YAML]	(`1 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id27>`_, `2 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id43>`_) `YAML Ain’t Markup Language (YAML™) Version 1.2 <http://www.yaml.org/spec/1.2/spec.html>`_

[MsgPack]	(`1 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id22>`_, `2 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id23>`_, `3 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id28>`_) `Message Pack <http://www.msgpack.org/>`_

[OpenCL]	(`1 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id13>`_, `2 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id35>`_) `The OpenCL Specification Version 2.0 <http://www.khronos.org/registry/cl/specs/opencl-2.0.pdf>`_

[`HRF <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id12>`_]`Heterogeneous-race-free Memory Models <http://benedictgaster.org/wp-content/uploads/2014/01/asplos269-FINAL.pdf>`_

[CLANG-ATTR]	(`1 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id14>`_, `2 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id15>`_, `3 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id16>`_, `4 <http://releases.llvm.org/8.0.1/docs/AMDGPUUsage.html#id17>`_) `Attributes in Clang <http://clang.llvm.org/docs/AttributeReference.html>`_

