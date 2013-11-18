# Running a R benchmark

## The Basic Command

The driver is [rbench.py](../utility/rbench.py) under the utility directory. You can use "-h" to get the help.
```bash
$ rbench.py -h
usage: rbench.py [-h] [--meter {time,perf}]
                 [--rvm {R,R-bytecode,rbase2.4,...}]
                 [--warmup_rep WARMUP_REP] [--bench_rep BENCH_REP]
                 source [args [args ...]]
...
```

Do a simple benchmark
```bash
$ cd examples
$ ../utility/rbench.py hello_rbenchmark.R
```

It will use the default R VM (R-bytecode) and the default meter to benchmark the application. 

The basic benchmark method has two phases
- pure warmup: run run() 2 times
- warmup + benchmark: run run() 2 + 5 times

Then the post processing will diff the two phases, and reports the average value for the 5 benchmark iterations.

## The Command Line Arguments

- --meter: Choose a meter for benchmarking. there are two meters right now
  + time: use the default timer of the os. Precision is platform depenent. The reported value's unit is ms.
  + perf: Linux perf, only available in Linux platform. The reported value is from "perf stat"
- --rvm: Choose a R VM for running the benchmark. All RVMs are defined in the [rbench.cfg](../utility/rbench.cfg) under the utility directory.
  + R: the default R in your enviroment, with R byte-code compiler disabled
  + R-bytecode: the default R in your enviroment, with R byte-code compiler enabled. By default, we use this R VM for benchmarking
  + ...: other R VMs listed in the [rbench.cfg](../utility/rbench.cfg)
- --warmup_rep: The number of repetition to execute run() in warmup. Default is 2.
- --bench_rep: The number of repetition to execute run() in Benchmark. At least 1. Default is 5.
- source: The source file for benchmarking
- args: the arguments that will be parsed into the source file.

## The Configuration Files

The configuration file [rbench.cfg](../utility/rbench.cfg) defines the general configurations as well as all the R VMs that could be used for benchmarking.

There are several sections in the configuration file. The first section _GENERAL_ defines the basic information.
```
[GENERAL]
#Warmup iterations for run()
WARMUP_REP=2
#Benchmark iterations for run()
BENCH_REP=5
#Used for Linux perf temporary storage
PERF_TMP=_perf.tmp
#Linux perf measurement iterations
PERF_REP=1
#Linux perf command - default one
PERF_CMD=perf stat -r %(PERF_REP)s -x, -o %(PERF_TMP)s --append
```

Each of the following sections defines a new RVM installed in your environment. For example, a rbytecode2.4 VM
```
[rbytecode2.4]
ENV=R_HOME=%(HOME)s R_COMPILE_PKGS=1 R_ENABLE_JIT=2
HOME=/home/hwang154/workspace/R-2.14.1.base
CMD=bin/Rscript
ARGS=--vanilla
HARNESS=r_harness.R
HARNESS_ARGS=TRUE
```

These configurations will be used to generate the command line issued by the driver.

## The Shell Commands Generated by the driver
The driver [rbench.py](../utility/rbench.py) will use the configuration files and the command line input to generate the command for benchmarking.

For example, if we use the default meter, the command line (shell script) structure is
```
#pure warmup
${ENV} ${CMD} ${ARGS} ${HARNESS} ${HARNESS_ARGS} ${WARMUP_REP} source args ...
#warmup + benchmark
${ENV} ${CMD} ${ARGS} ${HARNESS} ${HARNESS_ARGS} ${WARMUP_REP+BENCH_REP} source args ...
```

Let's use the hello_rbenchmark.R and all the default configuration as example, the final two commands are
```
#the command line, with the benchmark input is 1234
../utility/rbench.py hello_rbenchmark.R 1234
#pure warmup
R_COMPILE_PKGS=1 R_ENABLE_JIT=2 Rscript --vanilla /cygdrive/c/workspace/project/rbenchmark/benchmarks/utility/r_harness.R TRUE 2 hello_rbenchmark.R 1234
#warmup + benchmark
R_COMPILE_PKGS=1 R_ENABLE_JIT=2 Rscript --vanilla /cygdrive/c/workspace/project/rbenchmark/benchmarks/utility/r_harness.R TRUE 7 hello_rbenchmark.R 1234
```

## Add R VM for Benchmarking

In order to add a new RVM for benchmarking test, the VM developer needs to add a section in the configuration file [rbench.py](../utility/rbench.py).
And provide the required harness file if necessary.

For example, there is a pqR config in the configuration file. It reuses the r_harness.R
```
[pqRbase2.5]
ENV=R_HOME=%(HOME)s
HOME=/home/hwang154/workspace/pqR-2013-07-22
CMD=bin/Rscript
ARGS=--vanilla
HARNESS=r_harness.R
HARNESS_ARGS=FALSE
```

And there is a TERR config in the configuration file. Because TERR has no Rscript like command, we had to pass all into the TERR command.
```
[TERR]
ENV=TERR_HOME=%(HOME)s
HOME=/home/hwang154/tools/TIBCO/terr15
CMD=bin/TERR
ARGS=-f
HARNESS=r_harness.R
HARNESS_ARGS=--args FALSE
```