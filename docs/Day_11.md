[Back to TOC](../README.md)  
[Prev: Day 10](Day_10.md)$~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~$[Next: Day 12](Day_12.md)  
_________________________________________________________________________________________________________  
# Days 11-13: Advanced Synthesis and STA

# Day 11: Introduction to STA
Static Timing Analysis (STA) is a method of validating the timing performance of a design by checking all possible paths for timing violations.
  * The design is broken down into timing paths having start and endpoints, the signal propagation delay along each path is calculated, and checked for violations of timing constraints inside the design and at the input/output interfaces.
  * The STA analysis is the static type - i.e., the timing analysis is carried out statically and does not depend upon the data values being applied at the input pins.  

As the naming suggests, there is another method - Dynamic Timing Analysis (DTA) where the timing behaviour is analyzed by applying a stimulus/ test vector at the input signals, the resulting behaviour of the design is captured in the simulation and functionality is verified using the testbenches.
  * This approach verifies the design meets the timing requirements while simultaneously ensuring the functionality.
  * However, this is a very expensive method in terms of both simuation run time and resources as we cannot verify the entire space of test vectors exhaustively.
  * A trade-off could be to select the test vectors that exercise majority of the circuits, but this is usually not sufficient for sign-off as the coverage/ confidence obtained is only as exhaustive as the test vectors used and no 100% guarantee that the design will work across conditions.
## STA Basics
STA can performed at various stages of the ASIC design flow, with the emphasis being on different aspects.

| ![D11_STA_in_Design_Flow_resized](/docs/images/D11_STA_in_Design_Flow_resized.png) |
|:---|

### Timing Arcs
  1. Combinational arcs: Between input and output pin of a combinational block/ cell.
  2. Sequential arcs: Between the clock pin and either the input or output
     * Timing check arc: Between the clock pin and the input. (For example, the setup and hold timing arcs between the clock pin and input data pin of a Flip Flop)
     * Delay arc: Between the clock pin and the output.
  3. Net arcs: Between the output pin of a cell to the input pin of another cell. (i.e., between the driver pin of a net and the load pin of that net)

### Unateness
Defines how the output changes for different types of transitions on the input.
  1. Positive unate: Rising input transition causes rising output transition and falling input transition causes falling output transition.
     * Examples: Buffer, AND Gate, OR Gate
  2. Negative unate: Rising input transition causes falling output transition and falling input transition causes rising output transition.
     * Examples: Inverter, NAND Gate, NOR Gate
  3. Non-unate: The output transition is determined not only by the direction of an input but also on the state of the other inputs.
     * Examples: XOR Gate

## Advanced Synthesis and STA with DC
## To Do: Complete the documentation of the theoretical aspects


## STA of the RISC-V CPU core using OpenSTA
  * [**Design Constraints**](../code/riscv/sdc/riscv_core_synthesis.sdc)
    ```
    set_units -time ns
    
    set period 10.000
    create_clock -name clk -period $period [get_ports {clk}]
    
    set_clock_latency -source -min 1 clk
    set_clock_latency -source -max 4 clk
    
    set clk_uncertainty_factor_setup 0.05
    set clk_uncertainty_setup [expr $period * $clk_uncertainty_factor_setup]
    set clk_uncertainty_factor_hold 0.02
    set clk_uncertainty_hold [expr $period * $clk_uncertainty_factor_hold]
    set_clock_uncertainty -setup $clk_uncertainty_setup [get_clock clk]
    set_clock_uncertainty -hold $clk_uncertainty_hold [get_clock clk]
    
    set min_input_dly_factor 0.1
    set max_input_dly_factor 0.3
    set min_input_dly [expr $period * $min_input_dly_factor]
    set max_input_dly [expr $period * $max_input_dly_factor]
    set_input_delay -clock clk -min $min_input_dly [get_ports reset]
    set_input_delay -clock clk -max $max_input_dly [get_ports reset]
    
    
    set min_tran_factor 0.01
    set max_tran_factor 0.05
    set min_tran [expr $period * $min_tran_factor]
    set max_tran [expr $period * $max_tran_factor]
    set_input_transition -max $min_tran [get_ports reset]
    set_input_transition -min $max_tran [get_ports reset] 
    
    set min_ouput_dly_factor 0.2
    set max_ouput_dly_factor 0.5
    set min_ouput_dly [expr $period * $min_ouput_dly_factor]
    set max_ouput_dly [expr $period * $max_ouput_dly_factor]
    set_output_delay -clock clk -min $min_ouput_dly [get_ports out]
    set_output_delay -clock clk -min $max_ouput_dly [get_ports out]
    ```
  * [**SDC File written from OpenSTA using write_sdc command**](../code/riscv/sdc/riscv_core_sdc.sdc)

  * **OpenSTA shell showing snapshots**
    The following snapshots show the commands to:
      * `read_liberty`: Read the Liberty format library file
      * `read_verilog`: Read the gate level verilog netlist file
      * `link_design`: Link (elaborate, flatten) the the top level cell
      * `read_sdc`: Read SDC commands from the given constraints file
      * `check_setup`: Perform sanity checks on the design
      * `report_checks`: Report paths in the desing based on the additional arguments/ filters provided
    
  | **OpenSTA commands to do STA** <br>  ![D11_OpenSTA_sta_commands](/docs/images/D11_OpenSTA_sta_commands.png) |
  |:---|

  | **(OLD) Min path**<br>  ![D11_OpenSTA_min_delay](/docs/images/D11_OpenSTA_min_delay.png) | (OLD) **Max path**<br>  ![D11_OpenSTA_max_delay](/docs/images/D11_OpenSTA_max_delay.png) |
  |:---:|:---:|


  * As can be seen from the OpenSTA log, the design has 3 unconstrained endpoints.
    ```
    _15766_/D
    _15768_/D
    _15777_/D
    ```
    * On inspecting the gate level netlist, these endpoints are the D input pins of DFF (sky130_fd_sc_hd__dfxtp_1) tied to a constant "0".
    * Yosys should have reduced them to constant drivers but somehow they still remain.
    * A Yosys GitHub issue has been raised to check if there is a way to avoid this.  
      Details of the issue can be read here: [https://github.com/YosysHQ/yosys/issues/4266](https://github.com/YosysHQ/yosys/issues/4266)

  * The design also fails to meet the target timing requirements:
    * Min path: most probably due to the overly conservative/ pessimistic values of the clock source latencies used.
    * Max path: due to the conservative values of clock source latencies used and undefined fanout constraints. (Inverter instance `_10539_` of type **sky130_fd_sc_hd__clkinv_1** is having a fanout of 932)

**<ins>UPDATE:</ins>**
  * The synthesis script was updated based on inputs from [**Yosys issue #4266**](https://github.com/YosysHQ/yosys/issues/4266).
    * The DFF with constant inputs have been taken care of.
    * The STA analysis of the new netlist is given below:

  | **Min path**<br>  ![D11_OpenSTA_min_delay](/docs/images/D11_OpenSTA_min_delay_New.png) | **Max path**<br>  ![D11_OpenSTA_max_delay](/docs/images/D11_OpenSTA_max_delay_New.png) |
  |:---:|:---:|

<br>

_________________________________________________________________________________________________________  
[Prev: Day 10](Day_10.md)$~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~$[Next: Day 12](Day_12.md)  
