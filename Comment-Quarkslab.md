The reported observations will be added into the documentation. 
 
Only a few comments:

## 6.5.1 Instruction semantics consistency
The tests cover the interpreter implementation and then check the correctness of the compiled VMs transitively by comparing its outputs to the output of the interpreter.

## 6.5.3 Hard-coded constants
Fixed in commit c433f6d.

## 6.5.4 CPU utilization in SuperscalarHash Programs
Yes, we used the VTune Amplifier during the development of Superscalar hash.

## 6.5.6 Superscalar instruction creation
The value of opGroup_ is checked by a condition in selectDestination. Specifically this condition:
```
(registers[i].lastOpGroup != opGroup_ || registers[i].lastOpPar != opGroupPar_)
```
The purpose of this condition is to make sure that two operations that can cancel out are not sequentially applied with the same source register. ADD and SUB are such operations, so ISUB_R cannot immediately follow IADD_RS with the same source register (opGroupPar_).

## 6.5.9 Check size of Key
Keys longer than 60 bytes are truncated when initializing SuperscalarHash (see blake2_generator.cpp:40).

## 6.5.11 Code quality
NULL checks were added in commit aaa6e4e.
