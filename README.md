<h1 style="text-align: center;">TL 6510 Computer</h1>
<h2 style="text-align: center;">Yet another CPU made from TTL chips</h2>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/pcb_revA.jpg" />
</div>
<br>
<h3 style="text-align: center;">Introduction</h3>
I know, this is not the first TTL CPU/Computer and not the last. But I got interested in designing my own TTL-CPU after a friend showed me his <a href="https://eater.net/8bit/">"Ben Eater 8bit Computer"</a> setup. I really liked the educational part of it, even though the CPU is very limited. I then looked into the <a href="https://gigatron.io/">"Gigatron - TTL microcomputer"</a>, which is a clever RISC-like design. <br>
Both CPU designs had not been modeled after any existing CPU architecture. Because of its nature, the Gigatron doesn't even require any micro code. It runs fast, but it also doesn't support any register display. Ben Eater's design supports binary and hex-displays, but is very limited and slow. <br>
For my own design I wanted to imitate a real existing CPU, allowing for extensive debug and display options as well as running it as fast as the original. A simple and very common CPU of the past was the <a href="https://en.wikipedia.org/wiki/MOS_Technology_6502">MOS 6502</a> with its variant 6510 running in the <a href="https://en.wikipedia.org/wiki/Commodore_64">Commodore C64</a>. Even though I liked <a href="https://en.wikipedia.org/wiki/Zilog">Zilog</a>, the complexity of the <a href="https://en.wikipedia.org/wiki/Zilog_Z80">Z80 CPU </a>was much higher and harder to bring on a board like this. However, here is a project that implements both if of interest: <a href="https://hackaday.io/project/190345-isetta-ttl-computer">Isetta TTL computer</a><br>
To allow for extensive register displays, I added headers where smaller boards can be plugged in for single step updates. They can be removed for full speed. A special logic also allows single instruction steps, single clock phase steps. For the last mode I found it better to replace the 8-bit hex display boards for the instruction register and the micro codes with special LCD module controllers displaying text lines in addition to the codes.<br>
I tried to stick to <a href="https://en.wikipedia.org/wiki/List_of_7400-series_integrated_circuits#Larger_footprints">74xx logic chips</a>, mainly the HCT series. But I had to do some exceptions. I wanted to use real <a href="https://en.wikipedia.org/wiki/74181">ALU chips</a> (because I always wanted to do something with these) and had to settle to the 74LS181, which is still available. I also had to use two <a href="https://en.wikipedia.org/wiki/Generic_Array_Logic">GALs</a>, one for the C64-like chip select decoder for memories and IOs (part of a PAL in the C64). And I had to introduce the second GAL for remapping the ALU codes because I ran out of micro code bits. Now it maps 4-bit codes plus CY to 5 ALU selection bits plus carry-in. I had to resist the temptation to replace more logic with GALs. The extreme would have been implementing the whole board into an <a href="https://en.wikipedia.org/wiki/Field-programmable_gate_array">FPGA</a>, which was not the goal :-).<br>
The goal was to create a design that comes as close to the internal processor functionality as possible, including timing. Since the hardware is not exactly the same, the micro code execution does not match 100%. But the general execution comes close and all instructions take the same number of clocks as the original. In very few cases dummy micro steps had to be included to meet the target. The only exceptions, where the original timing could not be met, are the ADC and SBC instructions in BCD mode. As far as I could find out, these are seldomly used, so this mode was implemented with more micro steps instead of more hardware.<br>
There is a great chip simulator <a href="http://www.visual6502.org/JSSim/expert.html">here</a>.
<br>
Before I even got started designing my own CPU, I spent quite some time on creating a simulation tool <a href="https://github.com/StefansAI/SimTTL">SimTTL</a> to analyze both designs above and then working on this design.  In addition, I had to create the <a href="https://github.com/StefansAI/MicroCodeGenerator">MicroCodeGenerator</a> for this design and the small boards <a href="https://github.com/StefansAI/HexDisplayController">HexDisplayController</a>, <a href="https://github.com/StefansAI/LCD-DisplayController">LCD-DisplayController</a> and <a href="https://github.com/StefansAI/ArduinoExpander">ArduinoExpander</a> (for testing bread board circuits). All of this came together in this project.<br>
If you want to know more about the 6502 processor, there is a lot out there. I personally liked <a href="http://www.6502.org/">6502.org</a> and <a href="https://www.masswerk.at/6502/">masswerk</a> and many more.<br>

<h3 style="text-align: center;">Description</h3>
<h4 style="text-align: center;">Overview</h4>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/block.png"/>
</div>
<br>
Here is a very simplified block diagram. The external data bus is connected with the internal data bus through a bi-directional buffer. There are the three 8-bit registers A, X, and Y with inputs and outputs connected to the internal data bus. There is the status register and the ALU with it's two input registers and an output buffer to place the results on the internal data bus. While the ALU_A input register is connected to iDB, it turned out to be beneficial to connect the B input register to the external data bus. This allowed loading ALU_A and ALU_B in the same clock phase. <br>
In the lower half of the block diagram are the three 16-bit address registers PC, AR, and SP and their connections to the internal data bus and external address bus. All three address registers are split into two 8-bit registers that can be loaded and read individually, except AR and the high part of the SP. In order to load a new address while the address has to be stable at the external address bus, there are registers to buffer the low part until the high part can be loaded together with the low part.<br>
At the bottom of the block diagram is just a hint to the exception handler, instruction register, micro code memory and the instruction decoder providing the signals for controlling all functionality. There is much more detail in the schematics.
<br>
<br>
<h4 style="text-align: center;">Page 2: Clock and Reset</h4>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_2/clock_reset.png"/>
</div>
<br>
While there is nothing special about the reset part, there is a bit more than just a clock generator. First, there are 2 clock sources to chose from. The crystal one for normal speed and the 555 timer for slow speeds down to seconds. The last one makes it fun to watch clock by clock execution with all displays.<br>
The crystal is four times the CPU clock of 1MHz in the C64. The board might run much faster, but that will have to be tested out. There are 2 JK-flip-flops to divide the crystal clock by 2 and 4. The last one is sampled and goes out as PHI2 as in the original 6502 processor system.<br>
Everything else is for creating single step modes for instructions or clock phases. Since the whole TTL processor is completely static, its clock can be paused indefinitely. This fact is used to implement single instruction steps or single clock phase steps. Let's check with <a href="https://github.com/StefansAI/SimTTL">SimTTL</a>.
<br>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_2/free-running.png"/>
</div>
<br>
The screenshot above is the standard mode of operation. The signals 4CLK2* and 4CLK4* are changing with the falling edge of 4CLK dividing it by 2 and 4. The signals 2CLK, iPHI2 and PHI2 sample those at the rising edge of 4CLK and the system runs freely.
<br>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_2/instruction_step.png"/>
</div>
<br>
When the rotary switch is in the second position ("Step one Instruction") iPHI2 will be held constant with the next activation of /LD_IR until the STEP button is pressed as here simulated by injecting a low pulse of the signal /STEP_BTN, which in return allows sampling the divided clocks again. The circuitry keeps a strong phase relationship with the external PHI2 clock, so it always stays synchronized.
<br>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_2/clock_phase_step.png"/>
</div>
<br>
Switching to the third position ("Step on Clock Phase") halts the internal clock iPHI2 after each transition, while 2CLCK changes twice each time and is also held constant until the next step button is pressed. This function allows stepping through each micro code phase.
<br>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_2/rdy-timing.png"/>
</div>
<br>
The same clock control is also used for the RDY function implementation. In the 6502, the RDY signal is sampled with the rising edge of the clock PHI2. As long as RDY is sampled as low, the processor has to wait for the external memory or IO port. The internal clock is held constant again as in the other cases.
<h4 style="text-align: center;">Page 3: Program Counter</h4>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_3/program_counter.png"/>
</div>
<br>
The 16-bit program counter is realized with 4 chips of the 4-bit counters 74HCT161. The two 8-bit groups can be loaded individually. As described in the overview, the lower 8-bit are loaded from the AL bus. These signals are coming from the Address Low (AL) register, which is contained on page 4 and will be explained there. The higher 8-bit counters are loaded from the internal data bus (iDB) directly. There are two 8-bit output buffers to the external address bus and two other ones to place the counter contents onto the internal data bus. The last ones allow reading the PC parts.<br>
Several control signals are used to increment the PC, to load it and to activate the bus drivers. These signals are mostly generated in the instruction decoder on page 8.<br>
Something that can be found in several parts of the schematics are female headers, mostly with 11 pins to plug in the <a href="https://github.com/StefansAI/HexDisplayController">HexDisplayController</a> or <a href="https://github.com/StefansAI/LCD-DisplayController">LCD-DisplayController</a> in two special cases. <br>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_3/jsr_rts.png"/>
</div>
<br>
The <a href="https://github.com/StefansAI/SimTTL">SimTTL</a> screenshot above shows the JSR and RTS instructions, which exercise incrementing, saving and loading of PC contents. The rising edge of /LD_IR captures the iDB contents into IR (red trigger marker, yellow marker and light blue marker). In each of these cases, the PC is incremented with the rising edge of /2CLK (falling edge of 2CLK). The JSR instruction is loaded from 0xFE41, then the low part of the JSR address from 0xFE42 (first blue marker) and the high part from 0xFE43 (second blue marker). The two parts of the PC are then written to the stack (next 2 blue marker, where R/W is low). The following blue marker is placed at the rising edge of /LD_PC_H, which loads AL and iDB into the PC to output the new execution address (0xFE01). The RTS instruction is then loaded from there (yellow marker). While the address bus continues to put out the PC of 0xFE02 for internal phases until the return address is read from the stack at the next 2 blue markers and loaded to AL and then to PCH and PCL simultanously. The return address is then incremented in the next phase before the execution can be continued fetching the next instruction from 0xFE44, the instruction after the JSR 0xFE01 instruction.
<br>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_3/mc_jsr.png"/>
</div>
<div style="text-align: center;">
  <img src="docs/assets/images/page_3/mc_rts.png"/>
</div>
<br>
Let's have a look at the <a href="https://github.com/StefansAI/MicroCodeGenerator">MicroCodeGenerator</a> application, where the sequences for all instructions are defined. Here are the screenshots for the two instructions JSR and RTS. It can be cross-referenced with the SimTTL screenshots. MC corresponds to T0, T1, T2,... and iPHI2 to the phase Tn_0 or Tn_1.
<br>
<h4 style="text-align: center;">Page 4: Address Register</h4>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_4/addr_reg.png"/>
</div>
<br>
The address register page looks similar to the program counter page. But there are no buffers to put the register parts on the internal data bus and there is the AL register instead. The AL register was already utilized in the program counter part above. In contrast to the mostly used edge-triggered 74x574 chips, the AR is based on the transparent latch chip 74x573. This way, a register behind the AL register can be loaded from iDB through the AL.
<br>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_4/brk.png"/>
</div>
<br>
Here is an example, where AR is used. RESET, NMI, INT and the BRK instruction have to read the address from the vector table. The instruction starts again with fetching the instruction at the red marker. After saving the PC and SR (status register) onto the stack, AL and ARH are loaded with the table address 0xFFFE before the first blue marker. Then AR is placed onto the address bus and AL reloaded with the low part of the interrupt service routine (ISR) address. AR is then incremented and the high part is loaded together with AL into the PC. The first instruction of the ISR is then fetched at the yellow marker.
<br>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_4/mc_brk.png"/>
</div>
<br>
Here is the BRK instruction definition in <a href="https://github.com/StefansAI/MicroCodeGenerator">MicroCodeGenerator</a>.
<br>
<br>
<h4 style="text-align: center;">Page 5: Stack Pointer</h4>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_5/stack_pointer.png"/>
</div>
<br>
The stack pointer is only 8-bit wide in the 6502 with a fixed high address part of 0x01. The SP could have been implemented using counters similar to the PC and AR (but up/down counters 74HCT193 instead). However, the 6502 uses the ALU to increment or decrement the SP and that's why SP is just a register. To save a buffer, two registers are used to be loaded in parallel. One of them is connected to the internal data bus and the other is connected to lower address bus.<br>
Similar to AL, there is an additional transparent register in front of the SP register. This way a new SP value can be captured while the stack address is still on the bus.
<br>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_5/pha_pla.png"/>
</div>
<br>
Here is an execution example of the instructions PHA (Push A to stack) and PLA (Pull A from stack). The PHA instruction is fetched at the red marker. Then /OE_SP is activated and 0xFF from SP is placed on iDB to be loaded into ALU_A (first blue marker). At the same time /SP_OUT_EN is activated and 0x01FF appears on the address bus to begin writing.<br>
Back to the ALU: ALU_S is 0xF, which means "A minus 1 plus Cin" with Cin inverted (<a href="https://www.righto.com/2017/03/inside-vintage-74181-alu-chip-how-it.html">Interactive 74181 viewer</a>, where you can change all inputs via mouse click). In this case 0xFF-1=0xFE appears at the ALU output, which is loaded into SP* from iDB (/OE_ALU=L, /LD_SP*).<br>
Between second and third blue marker, the write cycle is finalized by placing the contents of register A on the external data bus and activating the write signal (R/W=L). At the same time /LD_SP is actiavted to load SP with the contents from SP*, which is the decremented stack pointer value for any next stack operation.<br>
The reverse is now happening at the PLA instruction between the yellow marker and the lightblue marker. Again, first the contents of SP is loaded into ALU_A, with ALU_S=0x0 and ALU_CIN=0, which means "A plus Cin" (Cin is inverted at the ALU). The result is now 0xFE+1=0xFF, which is loaded into SP*. In the next phase, SP is loaded from SP* and put out on the address bus for reading from the stack.<br> The previously written register A value of 0xA5 appears on the external data bus and then on iDB. It is not loaded into A immediately, but into ALU_A instead. The reason is, that PLA like other load instructions set the N and the Z flag according to the loaded value. The flags are set when going through the ALU. WIth ALU_S=0 and ALU_CIN=1, ALU_A is passed to the output (ALU=ALU_A). /OE_ALU and /LD_A are both activated in the next phase while the next instruction fetch is started. 
<br>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_5/mc_pha.png"/>
</div>
<div style="text-align: center;">
  <img src="docs/assets/images/page_5/mc_pla.png"/>
</div>
<br>
Here are the micro code definitions for the two instructions shown above, PHA and PLA. It can be followed, what signals are activated when.
<br>
<br>
<h4 style="text-align: center;">Page 6: Registers and DB driver</h4>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_6/regs_db.png"/>
</div>
<br>
Page 6 contains the registers A, X and Y as well as the data bus driver between external and internal data bus. Again, the headers for the <a href="https://github.com/StefansAI/HexDisplayController">HexDisplayControllers</a> are added here. It looks straight forward.<br>
Just as a side note, the last page of the schematics has shadow registers just for SimTTL. They are loaded in parallel to the real registers here, but their outputs are always active to show the contents in simTTL. These shadow registers don't appear in the BOM nor do they exist on the board. They show up in SimTTL as REG_A, REG_X and REG_Y (see below)
<br>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_6/regs.png"/>
</div>
<br>
Here is the code for this example:
<div class="box"><pre>
FE02  A9 A5                      LDA #$A5
FE04  AA                         TAX
FE05  A0 00                      LDY #$00
FE07  98                         TYA
FE08  A0 33                      LDY #$33
FE0A  85 10                      STA $10
</pre></div>
The first instruction (LDA #$A5) starts at the red marker. As in the PLA example above, the immediate value of $A5 is read from memory, placed onto the internal data bus when /OE_iDB=L and loaded into ALU_A at /LD_ALU_A=L. In the next phase /OE_ALU is activated and then loaded into A. At the same moment the NZ and ZF flags are changed. In this case NF is set and ZF is reset.<br>
The next instruction TAX activates /OE_A and again the data is loaded into ALU_A again and then into X via /LD_X.<br>
In result of LDY #$00, the zero flag (ZF) is set and the negativae flag (NF) is reset. With the next LDY #$33, resets ZF, since the value is not zero.<br>
The last instruction is STA $10, storing A to the zero page location 0x0010. This address is put out onto the address bus, then /OUT_DIR and /OE_A are activated and then R/W to write the contents to the external memory.
<br>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_6/mc_lda_tax.png"/>
</div>
<div style="text-align: center;">
  <img src="docs/assets/images/page_6/mc_ldy.png"/>
</div>
<div style="text-align: center;">
  <img src="docs/assets/images/page_6/mc_tya.png"/>
</div>
<div style="text-align: center;">
  <img src="docs/assets/images/page_6/mc_sta.png"/>
</div>
<br>
Above are the screenshots of all instructions used above as reference.
<br>
<br>
<h4 style="text-align: center;">Page 7: Micro Code Sequencer</h4>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_7/micro_code_sequencer.png"/>
</div>
<br>
There are 2 ROMs in parallel to produce 16 bit wide micro codes. An instruction register provides 8 bit of the address. But the micro steps are created by a 4-bit counter and a clock phase. This together allow for 5-bit for the micro steps, giving 16 clocks or 32 phases for each instruction. The 6502 would normally only use up to 7 clocks per instruction. But since I decided to implement BCD calculations with more micro steps instead of more hardware, I gave one more bit to the micro steps.<br>
If you look closer, the instruction and the micro steps generate 13 bits of the address bus. That leaves 4 bits for additional conditions. The highest bit is reserved for exceptions, like interrupts and reset, because they will get the highest priority. Then there are 3 bits split into any special condition, then full carry and half carry which also can have double meaning. Those bits are generated for address calculations, branch conditions or BCD handling.<br>
The 16-bit outputs are then grouped into the following:<br>
- Address output<br>
- Internal databus enable<br>
- Load internal register<br>
- ALU-code<br>
Originally there were just 2 identical registers to capture the ROM outputs. But when I ran out out bits, I decided to replace one register with a GAL. This GAL not only acts as a register, it also expands the 4-bit ALU codes plus CF from the ROM into 5-bit ALU function bits plus CIN. This is possible since the ALU chips have more functions than actually needed for the 6510 computer.<br>
The GAL projects are made with <a href="https://www.microchip.com/en-us/development-tool/WinCUPL">WinCupl</a> and the source codes are part of the project in <a href="https://github.com/StefansAI/TTL-6510-Computer/tree/main/WinCupl/ALUDECODER">GitHub.</a>
<br>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_7/jump_ds.png"/>
</div>
<br>
In <a href="https://github.com/StefansAI/SimTTL">SimTTL</a> the execution of the JMP instruction looks like this. The Micro Code counter (MC) is the input into the ROMs. But there is a latency of one phase, I added "REG_MC" to sample MC with the same latency. "REG_MC" directly corresponds now to T1,T2,T3 etc. There is a low pulse of /LD_IR at the end of the "LSR A" loading the instruction register (IR) with "0x4C" and incrementing PC, thus starting the execution of the "JMP abs" instruction. <br> 
At the end of T1 "/LD_AL" loads 0x7C from the internal databus (iDB) into the AL register. AL is a transparent D-Latch and that's why the output changes shortly after activating the load signal. At T2_1, the signal "/LD_PC_H" is activated and 0xFE is loaded from iDB into PC_H with the rising edge. At the same edge PC_L is loaded from AL, so PC is now 0xFE7C. T3_1 now loads the next instruction (EOR abs).
<br>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_7/jmp_example.png"/>
</div>
<br>
Again, here is the micro code for JMP and EOR.
<br>
<br>
<h4 style="text-align: center;">Page 8: Instruction Decoder</h4>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_8/instruction_decoder.png"/>
</div>
<br>
The instruction decoder is simply passing the bit groups to cascaded decoder chips. But in addition, signal combinations are often needed in all groups and a couple of AND-gates become low-active if one of the inputs are low-active. The blue texts at decoder outputs correspond to the selector name in <a href="https://github.com/StefansAI/MicroCodeGenerator">MicroCodeGenerator</a>. The final signal label appears right to it or the additional gates.<br>
The first decoder is responsible for the address bus selection. One input to enable the decoder is /AEC. Another bus user (i.e. VIC) can steal the low phase of PHI2 to use the bus while the high phase is still available for the CPU.<br>
The next group creates the load signals. Here the low phase of /2CLK enables the decoders to create better defined timing. There are still delays through the AND-gates. The very first output is left blank to allow for a no-load-code.<br>
It's beneficial (and common) to load the ALU_A register directly from the ALU output again for next operations. To implement this, the signal /OE_ALU again combined with /2CLK feeds into the final signal /LD_ALU_A. But it can be prohibited via DEC_COND (decimal coondition) for BCD corrections.<br>
The largest group is the output enable signal group with 5-bit. But instead of the possible 32 decoded outputs, only 24 were needed here. Since there was still one code needed to not enable any output at all, the decoders are wired in a way that the 24 outputs start at input code 8, leaving 0..7 as no-output-enable.
<br>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_8/adc_abs_x.png"/>
</div>
<br>
The SimTTL screenshot shows the execution of two "ADC abs,X" instructions back-to-back (first starting at red marker, second at yellow marker ending at light blue). The second one calculates a data address with a carry from the low part of the address to the high part. According to the 6502 documentation an additional clock is inserted after the low part addition to increment the high part as correction. In the screenshot, the signal FCY_COND is activated to switch the micro code execution to the increment portion.<br>
Here is the code, including the preparation before ADC.
<div class="box"><pre>
FE1E  A2 66                      LDX #$66
FE20  A0 67                      LDY #$67
FE22  A9 12                      LDA #$12
                                 ; Testing Op Codes
FE24  7D 34 12                   ADC $1234,X
FE27  7D F4 12                   ADC $12F4,X
</pre></div>
<div style="text-align: center;">
  <img src="docs/assets/images/page_8/mc_adc.png"/>
</div>
<br>
The MicroCodeGenerator screenshot shows both micro code definitions. The execution starts with FCY_COND=L and switches over, when it transitions to high executing the increment in the ALU and ending a clock later.<br>
In T1_1 the code for /LD_ALU_AB actually activates /LD_ALU_A and /LD_ALU_B simultanously through the AND-gates. Since /OE_X is active, the contents of register X is loaded into ALU_A, while at the same time the low part of the absolute address ($34 or $F4) is loaded into ALU_B.<br>
In T2_0 the signal /OE_ALU_FDCY is activated, which results in capturing the CY flag and setting FCY_COND accordingly. Details follow later on page 12. <br>
It is important to keep in mind that there is always 1 phase latency between the iput of the ROM and the outut sampling. When the FCY_COND signal is changing, it can only assumed to switch over to the other code track one clock phase later. <br>
<br>
<br>
<h4 style="text-align: center;">Page 9: Exception Handling</h4>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_9/exception_handling.png"/>
</div>
<br>
This part takes care of handling the exceptions /RESET, /NMI and /INT. These signals interrupt the current execution in different ways. <br>
<b>Reset:</b><br>
Reset has the highest priority, interrupting everything immediately after activation. As long as /RESET is kept low, the whole processor will be held in reset until execution can start with the transition to high. That rising edge forces the exception handling immediately.<br>
<b>NMI:</b><br>
The non-maskable interrupt (NMI) is activated by the falling edge, which is detected in the first FF and the OR-gate to activate the /NMI_BIT. The JK-FF will deactivate /NMI_BIT with the next /LD_IR after the NMI exception code had been executed (/EXC=L, IR1=L, which is the sampled /NMI_BIT). This way an NMI can occur while INT-exception is handled, so that the NMI handling follows immediately.<br>
<b>INT:</b><br>
The maskable interrupt has the lowest priority and has to be held low until handled. This happens with the next falling edge of /LD_IR if the interrupt had been enabled. <br><br>
In any of these exception cases the signal /ACT_EXC will be activated and then sampled with /LD_IR to create the signal EXC indicating the exception execution. At the same moment all three exception bits (/RES_BIT, /NMI_BIT and /INT_BIT) will be sampled into an alternative instruction register to replace the main instruction register contents as long as EXC is active.<br>
To ensure getting back to the original PC for the next instruction after interrupt handling, a special signal INHIBIT_PC++ is generated to disable the increment when /LD_IR is active.<br>
As described in <a href="https://en.wikipedia.org/wiki/Interrupts_in_65xx_processors">Interrupts in 65xx processors</a> each of these exceptions has to fetch an address from the vector table:
<table class="center">
  <tr>
    <th>Exception</th>
    <th>Vector Low</th>
    <th>Vector High</th>
  </tr>
  <tr>
    <td>INT</td>
    <td>0xFFFE</td>
    <td>0xFFFF</td>
  </tr>
  <tr>
    <td>RESET</td>
    <td>0xFFFC</td>
    <td>0xFFFD</td>
  </tr>
  <tr>
    <td>NMI</td>
    <td>0xFFFA</td>
    <td>0xFFFB</td>
  </tr>
</table>
The hardware has to generate these addresses for the vector table. The pull-up resistor array will take care of any 0xFF on the internal databus. The lowest 4 bits are coming from the micro codes MC_OUT_12..15 latched into the HCT173 when /OE_BRKL is active. These bits are normally repsonsible for the ALU function codes, so the micro codes have to select some ALU selection that creates the correct 4 bits for the vector table address access.
<br>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_9/reset.png"/>
</div>
<div style="text-align: center;">
  <img src="docs/assets/images/page_9/mc_reset.png"/>
</div>
<br>
Here is the reset timing and the micro code from the rising edge of /RESET on (red marker). IR shows the value 0x06, which means /RES_BIT is low and /NMI_BIT and /INT_BIT are high. The disassembler interprets it wrongly as ASL zpg, so ignore it here.<br> At the first blue marker the low address part for the vector table (0xFC) is read from iDB and loaded into AL. It is followed by reading 0xFF and loading it together with AL into the address register AR. Next, the address 0xFFFC is appearing on the address bus and the value 0x02 is read from the vector table. AR is incremented and 0xFE is now read loaded into PC together with AL. The PC is set to the reset execution address of 0xFE02 in this case. <br>Before starting there, the stack pointer SP is loaded to 0xFF and the <a href="https://www.nesdev.org/wiki/Status_flags#:~:text=The%20flags%20register%2C%20also%20called%20processor%20status%20or,one%20or%20more%20bits%20and%20leave%20others%20unchanged.">status register</a>
 SR is loaded to 0x06. This sets the Interrupt Disable flag (IF) as the default after reset. To finish up, /ACK_EXC is activated to deactivate /ACT_EXC so that EXC is cleared with /LD_IR.
<br>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_9/nmi.png"/>
</div>
<div style="text-align: center;">
  <img src="docs/assets/images/page_9/mc_nmi.png"/>
</div>
<br>
The NMI can be triggered by the falling edge of the signal /NMI as shown here with a very short pulse at the red marker. Even though the /INT line was pulled low before /NMI, the next /LD_IR activation goes into the exception handling for the NMI (yellow marker). The IR code is 0x01, meaning /RES_BIT=H and /NMI_BIT=L and /INT_BIT=L, so the micro code for 0x01 and 0x05 (/INT_BIT=H) will have to be identical. Again, ignore the wrong interpretation of the disassembler mnemonic here.<br>
Together with /LD_IR the signal INHIBIT_PC++ is activated here and as a result the PC is not incremented to keep it as is. In contrast to reset, interrupts have to store the return address and the status register on the stack first, which can be seen after the first 3 blue markers where the stack pointer is on the address bus and the values 0xFE, 0x8D and 0x41 are written. Then a 0xFF is written to SR to disable the interrupt.<br>
Similar to reset the vector addresses are generated and the address loaded from there. In this case the vector address is 0xFFFA, 0xFFFB and the NMI address to call is 0xFE00 in this case. The EXC signal is cleared again and the execution continues at the NMI service routine.
<br>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_9/int.png"/>
</div>
<div style="text-align: center;">
  <img src="docs/assets/images/page_9/mc_int.png"/>
</div>
<br>
The interrupt here is just following the NMI service routine above. The yellow marker shows the RTI instruction before the red marker, where the still pending interrupt is now handled. Again, the return PC address is not incremented, but put onto the stack again together with the status register. It all looks very similar to the NMI handling and the micro code is also almost the same. The difference is in the vector address generated by different ALU codes. Other than that is all identical.
<br>
<br>
<h4 style="text-align: center;">Page 10: ALU</h4>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_10/alu.png"/>
</div>
<br>
The heart of the ALU are the two 4-bit ALU chips <a href="https://en.wikipedia.org/wiki/74181">74181</a>, which were once widely used in computers of that time. The 74LS181 is still available. There is an <a href="https://www.righto.com/2017/03/inside-vintage-74181-alu-chip-how-it.html">Interactive 74181 viewer</a> where all input bits can be changed via mouse click to see the results. <br>
The schematics shows the two chips cascaded to 8-bits, the two input registers ALU_A and ALU_B as well as the output buffer connecting back to iDB.<br>
These ALU chips contain more functions than needed here. But they are also missing few things. First, not a signle shift right function is integrated in the chips, so it had to be implemented externally. It is done simply by adding a secodn ALU_A register with shifted input assignments, so it can be fed into the ALU alternatively. It has to be complemented with an additional FF to include loading D0 into SH_CY and the multiplexer of the ALU_FCY signal to COMB_CY.<br>
Second, the overflow flag is not generated by these ALY chips and is ralized via the three XOR-gates to create ALU_OV.<br>
Third, the "A=B" output is not exactly usable as zero flag, so the NAND/AND-gate combination had to be added to create ALU_Z.
<br>
<br>
<div style="text-align: center;">
  <img src="docs/assets/images/page_10/ror.png"/>
</div>
<div style="text-align: center;">
  <img src="docs/assets/images/page_10/mc_ror.png"/>
</div>
<br>
This example demonstrates the use of the ALU in different ways. The instructions "ROR $1234$,X" and "ROR $12F4,X" are executed with X=0x66. The first one does not require an address correction, but the second one does.<br>  
First, the data address has to be calculated by adding the low part of the absolute address and X together. It is loaded to AL and then the address high part is read and both are loaded into AR. In the second ROR-execution the ALU_FCY causes FCY_COND to be set and the high address is loaded to ALU_A to be incremented in the ALU and then loaded to AR as corrected address.<br>
After reading the data from the memory address (0x65 in the first case and 0x1C in the second) the right shift occurs with /OE_ALU_SH activation. The result is directly written back to the memory address.



<br><br><br><br><br><br><br><br><br><br>
<br>
<h4 style="text-align: center;">Status</h4>
<br>
I made the first board revision and had to realize that I should have finished simulating all instructions and micro code conditions completely. Now I'm pretty much done with all simulations and changing the schematic. I'm working on the new layout for Rev B.<br>
I'm also still working on the documentation of the design.

