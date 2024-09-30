# Introduction

*Gobra* is an automated, modular verifier for heap-manipulating, concurrent Go programs. It supports a large subset of Go, including Go’s interfaces and many primitive data structures. Gobra verifies memory safety, crash safety, data-race freedom, and partial correctness based on user-provided specifications. Gobra takes as input a `.gobra` file containing a Go program annotated with assertions such as method pre and postconditions and loop invariants. In case verification fails, Gobra reports at the level of the Go program which assertions it could not verify.

Verification proceeds by translating an annotated program into the Viper intermediate verification language and then applying an existing SMT-based verifier. 

This tutorial provides a practical introduction to Gobra and showcases Gobra's annotation syntax and main features. First, the tutorial covers the high-level structure of an annotated Go program. Afterwards, the tutorial gives a tour through the different features of Gobra  on several examples. Then, we demonstrate how to execute Gobra from the command line to verify `.gobra` files.

