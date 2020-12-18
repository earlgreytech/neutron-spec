The gas interface into and out of Neutron is specified as being "x 100". Thus, 100 gas is the minimum amount that can be specified. This greatly increases the granularity available in the gas model. When outputing gas from Neutron, it should thus be divided by 100, rounding up, to get the actual gas cost on the blockchain.

The Neutron gas model will involve two separate metering points:

1. VM execution metering
2. Element execution metering

These will be controlled by a two tables of "gas schedules". The element gas schedule is as so:

    (element) -> (operation) -> (cost_parameter, Optional<computation_function>)

Due to the intense computational workload that can come from checking for optional functions within a VM loop, the VM gas metering is greatly simplified:

    (operation) -> (cost_parameter)

Operations cover shared costs between VMs, though it is expected that some operations could be specific to a particular VM type. It is expected that each VM will copy relevant costs into their own local memory upon initialization, to avoid table lookups in the middle of VM loops. 

Standard VM operation costs:

* GAS_ALGORITHM_VERSION -- note: not an actual cost, used to easily include fork behaviors
* INSTRUCTION_BASE_COST
* VERY_LOW_COST_OPCODE -- note: may be 0 cost
* LOW_COST_OPCODE
* MEDIUM_COST_OPCODE
* HIGH_COST_OPCODE
* VERY_HIGH_COST_OPCODE
* PREDICTABLE_BRANCH_OPCODE
* UNPREDICTABLE_BRANCH_OPCODE
* COPY_INTO_VM_MEMORY -- charge per byte copied into VM memory
* COPY_FROM_VM_MEMORY -- charge per byte copied from VM memory
* MEMORY_READ_COST -- charge per byte of memory read within the VM
* MEMORY_WRITE_COST -- charge per byte of memory written within the VM
* VM_MUTABLE_MEMORY_ADDED -- charge per byte of VM memory allocated which can not be optimized away easily
* VM_READABLE_MEMORY_ADDED -- charge per byte of VM memory added which can potentially be optimized away due to being read-only
* VM_MEMORY_PRESSURE -- Neutron supplied function. Applies gas pressure to prevent too much allocation
* VM_MEMORY_PRESSURE_THRESHOLD -- Neutron supplied function. Specifies when pressure should start to be applied to an allocation
* COMSTACK_PUSH_BASE_COST -- base cost per push. Note this is not used by VMs directly, but rather only by hypervisors
* COMSTACK_PRESSURE_THRESHOLD -- determination of when the comstack should begin exerting "pressure" costs
* COMSTACK_PRESSURE -- amount of gas pressure to exert (could potentially represent a curve parameter etc

Additional costs per VM could be for things like complex Mod R/M decoding in x86, or using conditional execution in ARM.

## Communication of Gas

Element gas is controlled and updated through the holder of the Comstack








