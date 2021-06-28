## Speeding up Python on the Elbrus architecture

1. Using `--with-computed-gotos=no` is essential, `configure` detects its availability, but `computed-gotos` slows down Python interpreter by 4 times!
2. Use the patch below to speed up the interpreter by about 10-15%. 
3. Use `--enable-optimizations` to get around 10% speedup.

- Patching with `sed` and `awk` gives more resilience against changes in Python sources. 
- Relevant for LCC 1.25.15

### Python 2

- tested for Python 2.7.18

```sh
# unsupported profiling option
sed -i "s| -fprofile-correction||" configure*
# need this option for LCC because tests use threads
sed -i "s|-fprofile-generate|-fprofile-generate-parallel|" configure*
# LCC profiling bug workaround
sed -i "/^Modules\\/_math.o:/{n;s|\$(CCSHARED) \$(PY_CFLAGS)|\$(filter-out -fprofile-generate-parallel,\$(CCSHARED) \$(PY_CFLAGS))|}" Makefile.pre.in

# faster interpreter on Elbrus
sed -i "/#if USE_COMPUTED_GOTOS/{:b;g;N;/DISPATCH/!bb;s|^|#undef USE_COMPUTED_GOTOS\n#define USE_COMPUTED_GOTOS 0\n#if 1\n#define TARGET(op) case op:\n#define TARGET_WITH_IMPL(op, impl) if(0)goto impl;case op:\n#define TARGET_NOARG TARGET\n#define TARGET_WITH_IMPL_NOARG TARGET_WITH_IMPL|;:a;n;ba}" Python/ceval.c
sed -i "s|goto \\*opcode_targets\\[\\*next_instr++\\]|opcode=NEXTOP();oparg=0;if(HAS_ARG(opcode))oparg=NEXTARG();goto switch_loop|" Python/ceval.c
sed -i "/switch (opcode) {/{s|^|switch_loop:|;:a;n;ba}" Python/ceval.c
sed -i "/_unknown_opcode:/{n;n;s|$|__builtin_unreachable();\n#include \"opcode_unknown.h\"|}" Python/ceval.c
awk '/_unknown_opcode/{print "case " NR-2 ":"}' Python/opcode_targets.h > Python/opcode_unknown.h
```

### Python 3

- tested for Python 3.9.5

```sh
# add e2k arch
sed -i "/elif defined(__hppa__)/i\\\n# elif defined (__e2k__)\n        e2k-linux-gnu" configure*
# unsupported profiling option
sed -i "s| -fprofile-correction||" configure*
# LCC profiling bug workaround
sed -i "/^Modules\\/_math.o:/{n;s|\$(CCSHARED) \$(PY_CORE_CFLAGS)|\$(filter-out -fprofile-generate,\$(CCSHARED) \$(PY_CORE_CFLAGS))|}" Makefile.pre.in

# faster interpreter on Elbrus
sed -i "/#if USE_COMPUTED_GOTOS/{:b;g;N;/LLTRACE/!bb;s|^|#undef USE_COMPUTED_GOTOS\n#define USE_COMPUTED_GOTOS 0\n#if 1\n#define TARGET(op) op|;:a;n;ba}" Python/ceval.c
sed -i "s|\\*opcode_targets\\[opcode\\]|switch_loop|" Python/ceval.c
sed -i "/switch (opcode) {/{s|^|switch_loop:|;:a;n;ba}" Python/ceval.c
sed -i "/_unknown_opcode:/{n;n;s|$|Py_UNREACHABLE();\n#include \"opcode_unknown.h\"|}" Python/ceval.c
awk '/_unknown_opcode/{print "case " NR-2 ":"}' Python/opcode_targets.h > Python/opcode_unknown.h
```
