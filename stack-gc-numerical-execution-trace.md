# Complete Numerical Execution Trace: `push 5, push 7, pop→7, pop→5` on the Oblivious Stack (Garbled-Circuit Backend)

This document traces, at full numerical detail, how the circuit-structure stack from
*Circuit Structures for Improving Efficiency of Security and Privacy Tools* (Zahur & Evans,
IEEE S&P 2013) executes the four-operation test

```
push 5,  push 7,  pop (expect 7),  pop (expect 5)
```

as a Yao's garbled-circuit protocol between the **generator G** (server, party 2) and the
**evaluator E** (client, party 1).

**Provenance.** The circuit was produced by the paper's own compiler (`netlist-master`,
`Circuit/Stack.hs` + `Circuit/NetList/Gcil.hs`), driven by an instrumented replica of
`Test/StackGcil.hs`'s `gcRandomTest` that records which instruction ranges belong to which
stack operation. The instrumented circuit was verified **byte-identical** to the uninstrumented
emission (`stacktiny.cir`, 263 instructions, generator log line
`Created Gcil test case: stacktiny. ANDs: 263. successVar: t262`). Every wire value below comes
from a plaintext interpreter for the GCIL `.cir` format whose op semantics were taken from
`Circuit/NetList.hs`; the full machine-generated dump is embedded in the Appendix.
The protocol-role facts (party 2 = garbler, party 1 = evaluator, 80-bit labels, OT roles,
output decoding) were confirmed against the GCParser/FastGC backend sources
(`wrmelicher/GCParser`: `ProgServer.java` sets `Circuit.isForGarbling = true` and creates the
OT sender; `ProgClient.java` the reverse; `Input_Variable.java` defines `CLIENT = 1`,
`SERVER = 2`; `runtestgcparser` runs both parties with `-w 80`).

---

## 0. Setup: the agreed circuit and the two private input files

Both parties hold the same public circuit, generated from the *public* op schedule
"push, push, pop, pop". It declares:

```
.input t1 2 16     G (server, party 2):  t1 = 5  (0000000000000101₂)  1st pushed value
.input t2 2 16     G:                    t2 = 7  (0000000000000111₂)  2nd pushed value
.input t3 1 16     E (client, party 1):  t3 = 7   expected result of 1st pop
.input t4 1 16     E:                    t4 = 5   expected result of 2nd pop
```

The corresponding test-input files are `stacktiny-server.in` (`t1 5`, `t2 7`) and
`stacktiny-client.in` (`t3 7`, `t4 5`).

**State encoding.** Each level-0 buffer slot `b0..b4` is a **NetMaybe** =
(1-bit presence wire, 16-bit payload wire); the head pointer `hp` is a 3-bit wire.
Initial state is compile-time constant: all five slots "known Nothing", `hp = 2`.

Conventions used throughout:

- `hp` points at the next free slot; the top of the stack lives at slot `hp − 1`.
- `mux(c, a, b) = c ? b : a`; the GCIL instruction `chose c u v` means `c ? v : u`.
- Constants are written `value:width` (e.g. `2:3` = the 3-bit constant 2).
- The push decoder's minterm *i* enables a write into slot *i* (slot 4 ↔ minterm 0,
  because the slot list is rotated to `[b4,b1,b2,b3]`).
- The Maybe-mux is implemented via `condSwap`, i.e. `out = a ⊕ ((a⊕b) ∧ c)`; it computes
  *both* swap outputs and discards one, which is why dead wires appear next to every mux.

---

## 1. The data-structure trajectory (what the numbers do)

| after | logical stack | buffer `[b0,b1,b2,b3,b4]` as (presence, payload) | hp |
|---|---|---|---|
| init | — | [N, N, N, N, N] (all compile-time empty) | const 2 |
| condPush 5 | 5 | [N, (t19=**0**,5), (t22=**1**,**5**), (t25=**0**,5), (t16=**0**,5)] | t14=**3** |
| condPush 7 + slide | 7, 5 | [(t82=**1**,t62=**5**), (t86=**1**,t91=**7**), (t95=**0**,5), N, N] | t76=**2** |
| top→**7**, condPop | 5 | [(t160=**1**,**5**), (t162=**0**,7), (t164=**0**,5), (t166=0), N] | t138=**1** |
| top→**5**, condPop + slide | — | [(t243=**0**,5), (t247=**0**,·), (t251=**0**,5), N, N] | t240=**2** |

Two things to notice:

1. A "pop" never erases the payload (the 7 sits on wire t91 forever) — `condZap` only clears
   the *presence bit*, making the value unreachable. Existence rides entirely on presence bits.
2. After the 2nd push the `adjustPush` slide fires (every 2nd push), and after the 2nd pop the
   `adjustPop` slide fires (every 2nd pop), restoring the invariant 1 ≤ hp ≤ 4. The level-1
   parent stack is **never materialized**: at slide time `b0` is still compile-time-Nothing, so
   the generator statically elides the entire parent (`adjustPush`'s `knownNothing` branch —
   the paper's "known blank block" optimization, Figure 13).

---

## 2. Wire-by-wire numerical trace

### Phase 1 — condPush 5 (t5–t26, 22 instructions, 10 garbled tables)

`hp` is still the constant 2, but constants only fold through mux/extend ops at the netlist
layer, so the pointer logic is emitted anyway (gcparser's optimizer folds it later):

```
t5  trunc 2:3 2   = 2    p = hp mod 4          t14 add 2:3 1:3 = 3   hp := hp+1
t6  select t5 1 2 = 1    p₁ (high bit)
t7  trunc t5 1    = 0    p₀ (low bit)
--- decoder tree (decoderUnit: and + xor per node), enable c = true ---
t8 =1 t9 =0              en∧p₁ , en∧¬p₁
t10=0 t11=0 t12=0 t13=1  write₁, write₀, write₃, write₂   → only slot 2 enabled
--- slot writes: mux(writeᵢ, Nothing, Just 5) — blank-slot cheap form:
    presence' = writeᵢ (1 AND); payload becomes wire t1=5 with NO gates ---
slot4 (t15–t17): t15=0, presence t16 = 0   [t17=1 dead]
slot1 (t18–t20): t18=0, presence t19 = 0   [t20=1 dead]
slot2 (t21–t23): t21=1, presence t22 = 1   [t23=0 dead]  ← 5 lands here
slot3 (t24–t26): t24=0, presence t25 = 0   [t26=1 dead]
```

### Phase 2 — condPush 7 (t27–t101, 75 instructions, 118 tables)

Now the slots are runtime Maybes, so each write is a full mux: presence = 1-bit AND,
payload = 16-bit AND, via `out = a ⊕ ((a⊕b) ∧ c)`:

```
t27=3 t28=1 t29=1                      p = hp mod 4 = 3, its bits
t30=1 t31=0 t32=0 t33=0 t34=1 t35=0    decoder → write₃ = t34 = 1 (slot 3)
t36 add t14 1:3 = 4                    hp := 4
slot4 (t37–t45): presence t39=0; payload: t41=5⊕7=2, t42=0x0000, t43=0, t44=5
slot1 (t46–t54): presence t48=0; payload: t50=2, t51=0x0000, t52=0, t53=5
slot2 (t55–t63): presence t57=1; payload: t59=2, t60=0x0000, t61=0, t62=5
slot3 (t64–t72): presence t66=1; payload: t68=2, t69=0xFFFF, t70=2, t71=5⊕2=7  ← the 7
--- adjustPush (2nd push ⇒ slide check) ---
t73 gtu t36 3:3 = 1                    slide condition: hp=4 > 3 → yes
t74=2 t75=2 t76 sub t36 t75 = 2        hp := 4−2 = 2
t77=1 t78=0 t79=0                      pushC to parent = 0 [dead; parent elided]
new b0 = mux(t73, b0, b2): t80=1 t81=1 → presence t82=1, payload = t62=5 (shared wire)
new b1 = mux(t73, b1, b3): t84=1 t85=1 → presence t86=1
                           t88=5⊕7=2 t89=0xFFFF t90=2 → payload t91=7
new b2 = mux(t73, b2, b4): t93=1 t94=1 → presence t95=0
                           t97=0 t98=0xFFFF t99=0 → payload t100=5
b3, b4 := compile-time Nothing (zero wires)      [t83,t87,t92,t96,t101 dead]
```

### Phase 3 — top for 1st pop (t102–t129, 28 instructions, 35 tables)

A Maybe-mux tree over `[b0,b1,b3,b2]` indexed by hp−1 (the `muxListOffset … 1` trick),
selector bits of hp=2: t102=0 (bit0), t104=t110=1 (bit1):

```
re = mux(bit1, b3, b1):  t106=1 t107=1 → presence t108=1; payload = t91 (b3 blank ⇒ free)
ro = mux(bit1, b0, b2):  t112=1 t113=1 → presence t114=0
                         t116=0 t117=0xFFFF t118=0 → payload t119=5
top1 = mux(bit0, re, ro): t121=1 t122=0 → presence t123 = 1
                          t125=2 t126=0x0000 t127=0 → payload t128 = 7   ← top = Just 7 ✓
```

### Phase 4 — condPop #1 (t130–t166, 37 instructions, 19 live tables)

```
t130 = ¬p_b1 = 0;  t131 = (hp==2) = 1;  t132 = null = 0     empty-stack guard
t133 = 1;  t134 = c = true ∧ ¬null = 1                       pop is enabled
t135 = 2  [dead]
t136=1 t137=1; t138 = sub t76 t137 = 1                       hp := 2−1 = 1
t139–t150: 3-bit decoder of new hp → erase₀=t148=0, erase₁=t147=1, erase₂=t150=0, erase₃=t149=0
t151–t158: minterms 4–7 of the decoder — all dead (only 4 slots zipped)
condZap (presence ∧ ¬eraseᵢ):
  b0: t159=1, t160 = 1        survives
  b1: t161=0, t162 = 0        ← presence cleared: the 7 is popped
  b2: t163=1, t164 = 0
  b3: t165=1, t166 = 0
(popAdjusted was fresh ⇒ no refill circuit this time)
```

### Phase 5 — check pop1 == 7 (t167–t171, 17 tables)

The only place E's `t3` is touched:

```
t167 = ¬t123 = 0          isNothing(top1)?
t168 = 1                  valid: stack wasn't empty
t169 equ t128 t3 = 1      7 == 7  (15 tables: 16-bit equality)
t170 = true ∧ t168 = 1;  t171 = t170 ∧ t169 = 1      running success flag c₁
```

### Phase 6 — top for 2nd pop (t172–t199, 28 instructions, 35 tables)

Same tree, hp=1 → bits t172=1, t174=t180=0 → selects b0:

```
re: t176=0 t177=0 → presence t178=0
ro: t182=1 t183=0 → presence t184=1;  t186=0 t187=0x0000 t188=0 → payload t189=5
top2: t191=1 t192=1 → presence t193 = 1
      t195=2 t196=0xFFFF t197=2 → payload t198 = 7⊕2 = 5     ← top = Just 5 ✓
```

### Phase 7 — condPop #2 + adjustPop (t200–t257, 58 instructions, 46 tables)

**Every single instruction is dead code** — the final stack state feeds nothing downstream.
It still computes faithfully:

```
null = 0 (t202);  c = 1 (t204);  hp := 0 (t208)
erase₀ = 1 → b0 presence cleared (t230 = 0)
adjustPop fires: t237 = (2 > 0) = 1 → slide right
  hp := 0 + 2 = 2 (t240)
  slots mux back to all-empty: presences t243 = t247 = t251 = 0
```

The stack ends in its canonical empty state — and gcparser's optimizer deletes all 58
instructions before anything is garbled.

### Phase 8 — check pop2 == 5 and output (t258–t262, 17 tables)

```
t258 = ¬t193 = 0;  t259 = valid = 1
t260 equ t198 t4 = 1      5 == 5
t261 = t171 ∧ t259 = 1
t262 = t261 ∧ t260 = 1
.output t262              ← the circuit's single output bit: SUCCESS
```

---

## 3. What G and E each do with these numbers (the protocol pass)

Per-phase cost of the emitted circuit (garbled tables = non-free gates; everything else is
free under free-XOR):

| phase | instrs | garbled tables | dead |
|---|---|---|---|
| condPush 5 | 22 | 10 | 4 |
| condPush 7 (+slide) | 75 | 118 | 16 |
| top #1 | 28 | 35 | 7 |
| condPop #1 | 37 | 19 | 9 |
| check ==7 | 5 | 17 | 0 |
| top #2 | 28 | 35 | 7 |
| condPop #2 (+slide) | 58 | 46 | 58 (all) |
| check ==5 | 5 | 17 | 0 |
| **total** | **258** | **297** | **101** |

The generator's log line `ANDs: 263` is exactly 297 minus the two 17-table check blocks,
which `ignoreAndsUsed` excludes from the statistic as test scaffolding (they are still in the
circuit and still executed).

Concretely, after `optimize.sh` prunes dead code, the run (`runtestgcparser`) looks like:

1. **G** picks a global 79-bit free-XOR offset Δ and, for every wire, a random 80-bit label
   pair (W⁰, W¹ = W⁰ ⊕ Δ) with permute bits (`-w 80`).
2. **Input labels — 64 bits total.** G knows t1=5 and t2=7, so for each of those 32
   bit-positions it just *sends* the matching label (bits of 5: positions 0 and 2 get W¹, the
   other 14 get W⁰; bits of 7: positions 0,1,2 get W¹). For E's 32 bits (t3=7, t4=5), the
   parties run 32 oblivious transfers (Naor–Pinkas base OTs + IKNP extension): E uses each of
   its private bits as the choice bit and receives exactly one label per bit; G never learns
   the choices, E never sees the opposite labels.
3. **Streamed garble/evaluate.** Walking the instruction list in order, G garbles the ~247
   live AND-type table gates (4 SHA-1-encrypted rows each, ≈40 bytes/gate ≈ 10 KB total) and
   streams them; E decrypts exactly one row per table, selected by the permute bits of the two
   labels it holds. All the `xor`/`not`/`trunc`/`select`/`sextend`/`concat` lines cost
   nothing: both sides just XOR or rewire labels locally. Every value in Section 2 — t22=1,
   t91=7, t128=7, t169=1, … — exists during the protocol only as "which of the two labels is
   E holding", which neither party can read.
4. **Output.** For `.output t262`, E ends up holding one 80-bit label and sends it back to G;
   G compares it against the pair it generated for t262, decodes **1**, prints `t262 = 1` into
   `results/siserverout` (and returns the plaintext so E prints it too). The test harness
   (`makeutils/GcilTest`) greps for `t262 = 1` → test passed.

**One gate, fully worked (illustrative labels).** Take the last gate `t262 and t261 t260`,
with toy 8-bit labels (real ones are random 80-bit): Δ=0x9e; t261 labels A⁰=0x3a / A¹=0xa4;
t260 labels B⁰=0x51 / B¹=0xcf; t262 labels C⁰=0x77 / C¹=0xe9. G sends the four rows
Enc(A^a, B^b → C^{a∧b}), permuted by select bits. E holds A¹ and B¹ (both inputs are 1,
though E doesn't know that), decrypts the one row those select bits point at, and obtains
0xe9 — which only G can recognize as "C¹, i.e. the bit 1".

**Footnote on constant folding.** In *this* regression test the push/pop conditions are the
constant `true` and the schedule is public, so a sufficiently aggressive optimizer could
constant-fold most of the pointer logic. The construction earns its keep in real uses (the
200-op random regression test, the DBSCAN benchmark) where `condPush`/`condPop` conditions —
and therefore every presence bit and pointer — are secret wires: exactly the values flowing
through t73, t134, t237 above, just no longer known to anyone.

---

## Reproducing this trace

All artifacts live in a scratch copy at `/tmp/netlist-run` (the repository itself was left
unmodified, apart from this document):

- `MainTrace.hs` — instrumented replica of the test that records op boundaries
  (compile: `ghc --make -outputdirbin -O1 MainTrace.hs -o MainTrace`; needs two small
  modern-GHC compatibility patches already applied in the scratch copy: an `Applicative`
  instance for `Control.Monad.StreamWriter` and removal of the `modifyIORef'` backport in
  `Circuit/NetList/Gcil.hs`).
- `simulate_cir.py` — plaintext interpreter for the GCIL `.cir` format.
- `annotate.py` — merges the boundary table with the value trace, computes garbled-table
  costs (per `Circuit/NetList/Gcil.hs`'s `opcodeAndCost`) and liveness.
- `tmp/stacktiny.cir`, `tmp/stacktiny-server.in`, `tmp/stacktiny-client.in` — the circuit and
  per-party inputs; `tmp/stacktiny-trace.txt` — the raw annotated dump (embedded below).

---

## Appendix: full annotated instruction trace

Format: `instruction = value (width) [garbled-table count | free] [DEAD if pruned by DCE]`.

```text
.input t1 2 16               =     5 (w16) [free] 
.input t2 2 16               =     7 (w16) [free] 
.input t3 1 16               =     7 (w16) [free] 
.input t4 1 16               =     5 (w16) [free] 

##### inputs

##### condPush 5
t5 trunc 2:3 2               =     2 (w2 ) [free] 
t6 select t5 1 2             =     1 (w1 ) [free] 
t7 trunc t5 1                =     0 (w1 ) [free] 
t8 and t6 1:1                =     1 (w1 ) [1 tbl]
t9 xor t8 1:1                =     0 (w1 ) [free] 
t10 and t7 t9                =     0 (w1 ) [1 tbl]
t11 xor t10 t9               =     0 (w1 ) [free] 
t12 and t7 t8                =     0 (w1 ) [1 tbl]
t13 xor t12 t8               =     1 (w1 ) [free] 
t14 add 2:3 1:3              =     3 (w3 ) [3 tbl]
t15 and 1:1 t11              =     0 (w1 ) [1 tbl]
t16 xor 0:1 t15              =     0 (w1 ) [free] 
t17 xor 1:1 t15              =     1 (w1 ) [free]   DEAD
t18 and 1:1 t10              =     0 (w1 ) [1 tbl]
t19 xor 0:1 t18              =     0 (w1 ) [free] 
t20 xor 1:1 t18              =     1 (w1 ) [free]   DEAD
t21 and 1:1 t13              =     1 (w1 ) [1 tbl]
t22 xor 0:1 t21              =     1 (w1 ) [free] 
t23 xor 1:1 t21              =     0 (w1 ) [free]   DEAD
t24 and 1:1 t12              =     0 (w1 ) [1 tbl]
t25 xor 0:1 t24              =     0 (w1 ) [free] 
t26 xor 1:1 t24              =     1 (w1 ) [free]   DEAD

##### condPush 7
t27 trunc t14 2              =     3 (w2 ) [free] 
t28 select t27 1 2           =     1 (w1 ) [free] 
t29 trunc t27 1              =     1 (w1 ) [free] 
t30 and t28 1:1              =     1 (w1 ) [1 tbl]
t31 xor t30 1:1              =     0 (w1 ) [free] 
t32 and t29 t31              =     0 (w1 ) [1 tbl]
t33 xor t32 t31              =     0 (w1 ) [free] 
t34 and t29 t30              =     1 (w1 ) [1 tbl]
t35 xor t34 t30              =     0 (w1 ) [free] 
t36 add t14 1:3              =     4 (w3 ) [3 tbl]
t37 xor t16 1:1              =     1 (w1 ) [free] 
t38 and t37 t33              =     0 (w1 ) [1 tbl]
t39 xor t16 t38              =     0 (w1 ) [free] 
t40 xor 1:1 t38              =     1 (w1 ) [free]   DEAD
t41 xor t1 t2                =     2 (w16) [free] 
t42 sextend t33 16           =     0 (w16) [free] 
t43 and t41 t42              =     0 (w16) [16 tbl]
t44 xor t1 t43               =     5 (w16) [free] 
t45 xor t2 t43               =     7 (w16) [free]   DEAD
t46 xor t19 1:1              =     1 (w1 ) [free] 
t47 and t46 t32              =     0 (w1 ) [1 tbl]
t48 xor t19 t47              =     0 (w1 ) [free] 
t49 xor 1:1 t47              =     1 (w1 ) [free]   DEAD
t50 xor t1 t2                =     2 (w16) [free] 
t51 sextend t32 16           =     0 (w16) [free] 
t52 and t50 t51              =     0 (w16) [16 tbl]
t53 xor t1 t52               =     5 (w16) [free] 
t54 xor t2 t52               =     7 (w16) [free]   DEAD
t55 xor t22 1:1              =     0 (w1 ) [free] 
t56 and t55 t35              =     0 (w1 ) [1 tbl]
t57 xor t22 t56              =     1 (w1 ) [free] 
t58 xor 1:1 t56              =     1 (w1 ) [free]   DEAD
t59 xor t1 t2                =     2 (w16) [free] 
t60 sextend t35 16           =     0 (w16) [free] 
t61 and t59 t60              =     0 (w16) [16 tbl]
t62 xor t1 t61               =     5 (w16) [free] 
t63 xor t2 t61               =     7 (w16) [free]   DEAD
t64 xor t25 1:1              =     1 (w1 ) [free] 
t65 and t64 t34              =     1 (w1 ) [1 tbl]
t66 xor t25 t65              =     1 (w1 ) [free] 
t67 xor 1:1 t65              =     0 (w1 ) [free]   DEAD
t68 xor t1 t2                =     2 (w16) [free] 
t69 sextend t34 16           = 65535 (w16) [free] 
t70 and t68 t69              =     2 (w16) [16 tbl]
t71 xor t1 t70               =     7 (w16) [free] 
t72 xor t2 t70               =     5 (w16) [free]   DEAD
t73 gtu t36 3:3              =     1 (w1 ) [3 tbl]
t74 chose t73 0:2 2:2        =     2 (w2 ) [2 tbl]
t75 zextend t74 3            =     2 (w3 ) [free] 
t76 sub t36 t75              =     2 (w3 ) [3 tbl]
t77 not 0:1                  =     1 (w1 ) [free]   DEAD
t78 not t77                  =     0 (w1 ) [free]   DEAD
t79 and t73 t78              =     0 (w1 ) [1 tbl]  DEAD
t80 xor 0:1 t57              =     1 (w1 ) [free] 
t81 and t80 t73              =     1 (w1 ) [1 tbl]
t82 xor 0:1 t81              =     1 (w1 ) [free] 
t83 xor t57 t81              =     0 (w1 ) [free]   DEAD
t84 xor t48 t66              =     1 (w1 ) [free] 
t85 and t84 t73              =     1 (w1 ) [1 tbl]
t86 xor t48 t85              =     1 (w1 ) [free] 
t87 xor t66 t85              =     0 (w1 ) [free]   DEAD
t88 xor t53 t71              =     2 (w16) [free] 
t89 sextend t73 16           = 65535 (w16) [free] 
t90 and t88 t89              =     2 (w16) [16 tbl]
t91 xor t53 t90              =     7 (w16) [free] 
t92 xor t71 t90              =     5 (w16) [free]   DEAD
t93 xor t57 t39              =     1 (w1 ) [free] 
t94 and t93 t73              =     1 (w1 ) [1 tbl]
t95 xor t57 t94              =     0 (w1 ) [free] 
t96 xor t39 t94              =     1 (w1 ) [free]   DEAD
t97 xor t62 t44              =     0 (w16) [free] 
t98 sextend t73 16           = 65535 (w16) [free] 
t99 and t97 t98              =     0 (w16) [16 tbl]
t100 xor t62 t99             =     5 (w16) [free] 
t101 xor t44 t99             =     5 (w16) [free]   DEAD

##### top (1st pop)
t102 trunc t76 1             =     0 (w1 ) [free] 
t103 select t76 1 3          =     1 (w2 ) [free] 
t104 trunc t103 1            =     1 (w1 ) [free] 
t105 select t103 1 2         =     0 (w1 ) [free]   DEAD
t106 xor 0:1 t86             =     1 (w1 ) [free] 
t107 and t106 t104           =     1 (w1 ) [1 tbl]
t108 xor 0:1 t107            =     1 (w1 ) [free] 
t109 xor t86 t107            =     0 (w1 ) [free]   DEAD
t110 trunc t103 1            =     1 (w1 ) [free] 
t111 select t103 1 2         =     0 (w1 ) [free]   DEAD
t112 xor t82 t95             =     1 (w1 ) [free] 
t113 and t112 t110           =     1 (w1 ) [1 tbl]
t114 xor t82 t113            =     0 (w1 ) [free] 
t115 xor t95 t113            =     1 (w1 ) [free]   DEAD
t116 xor t62 t100            =     0 (w16) [free] 
t117 sextend t110 16         = 65535 (w16) [free] 
t118 and t116 t117           =     0 (w16) [16 tbl]
t119 xor t62 t118            =     5 (w16) [free] 
t120 xor t100 t118           =     5 (w16) [free]   DEAD
t121 xor t108 t114           =     1 (w1 ) [free] 
t122 and t121 t102           =     0 (w1 ) [1 tbl]
t123 xor t108 t122           =     1 (w1 ) [free] 
t124 xor t114 t122           =     0 (w1 ) [free]   DEAD
t125 xor t91 t119            =     2 (w16) [free] 
t126 sextend t102 16         =     0 (w16) [free] 
t127 and t125 t126           =     0 (w16) [16 tbl]
t128 xor t91 t127            =     7 (w16) [free] 
t129 xor t119 t127           =     5 (w16) [free]   DEAD

##### condPop (1st)
t130 not t86                 =     0 (w1 ) [free] 
t131 equ 2:3 t76             =     1 (w1 ) [2 tbl]
t132 and t130 t131           =     0 (w1 ) [1 tbl]
t133 not t132                =     1 (w1 ) [free] 
t134 and 1:1 t133            =     1 (w1 ) [1 tbl]
t135 trunc t76 2             =     2 (w2 ) [free]   DEAD
t136 chose t134 0:1 1:1      =     1 (w1 ) [1 tbl]
t137 zextend t136 3          =     1 (w3 ) [free] 
t138 sub t76 t137            =     1 (w3 ) [3 tbl]
t139 select t138 2 3         =     0 (w1 ) [free] 
t140 trunc t138 2            =     1 (w2 ) [free] 
t141 and t139 t134           =     0 (w1 ) [1 tbl]
t142 xor t141 t134           =     1 (w1 ) [free] 
t143 select t140 1 2         =     0 (w1 ) [free] 
t144 trunc t140 1            =     1 (w1 ) [free] 
t145 and t143 t142           =     0 (w1 ) [1 tbl]
t146 xor t145 t142           =     1 (w1 ) [free] 
t147 and t144 t146           =     1 (w1 ) [1 tbl]
t148 xor t147 t146           =     0 (w1 ) [free] 
t149 and t144 t145           =     0 (w1 ) [1 tbl]
t150 xor t149 t145           =     0 (w1 ) [free] 
t151 select t140 1 2         =     0 (w1 ) [free]   DEAD
t152 trunc t140 1            =     1 (w1 ) [free]   DEAD
t153 and t151 t141           =     0 (w1 ) [1 tbl]  DEAD
t154 xor t153 t141           =     0 (w1 ) [free]   DEAD
t155 and t152 t154           =     0 (w1 ) [1 tbl]  DEAD
t156 xor t155 t154           =     0 (w1 ) [free]   DEAD
t157 and t152 t153           =     0 (w1 ) [1 tbl]  DEAD
t158 xor t157 t153           =     0 (w1 ) [free]   DEAD
t159 not t148                =     1 (w1 ) [free] 
t160 and t82 t159            =     1 (w1 ) [1 tbl]
t161 not t147                =     0 (w1 ) [free] 
t162 and t86 t161            =     0 (w1 ) [1 tbl]
t163 not t150                =     1 (w1 ) [free] 
t164 and t95 t163            =     0 (w1 ) [1 tbl]
t165 not t149                =     1 (w1 ) [free] 
t166 and 0:1 t165            =     0 (w1 ) [1 tbl]

##### check pop1==7
t167 not t123                =     0 (w1 ) [free] 
t168 not t167                =     1 (w1 ) [free] 
t169 equ t128 t3             =     1 (w1 ) [15 tbl]
t170 and 1:1 t168            =     1 (w1 ) [1 tbl]
t171 and t170 t169           =     1 (w1 ) [1 tbl]

##### top (2nd pop)
t172 trunc t138 1            =     1 (w1 ) [free] 
t173 select t138 1 3         =     0 (w2 ) [free] 
t174 trunc t173 1            =     0 (w1 ) [free] 
t175 select t173 1 2         =     0 (w1 ) [free]   DEAD
t176 xor t166 t162           =     0 (w1 ) [free] 
t177 and t176 t174           =     0 (w1 ) [1 tbl]
t178 xor t166 t177           =     0 (w1 ) [free] 
t179 xor t162 t177           =     0 (w1 ) [free]   DEAD
t180 trunc t173 1            =     0 (w1 ) [free] 
t181 select t173 1 2         =     0 (w1 ) [free]   DEAD
t182 xor t160 t164           =     1 (w1 ) [free] 
t183 and t182 t180           =     0 (w1 ) [1 tbl]
t184 xor t160 t183           =     1 (w1 ) [free] 
t185 xor t164 t183           =     0 (w1 ) [free]   DEAD
t186 xor t62 t100            =     0 (w16) [free] 
t187 sextend t180 16         =     0 (w16) [free] 
t188 and t186 t187           =     0 (w16) [16 tbl]
t189 xor t62 t188            =     5 (w16) [free] 
t190 xor t100 t188           =     5 (w16) [free]   DEAD
t191 xor t178 t184           =     1 (w1 ) [free] 
t192 and t191 t172           =     1 (w1 ) [1 tbl]
t193 xor t178 t192           =     1 (w1 ) [free] 
t194 xor t184 t192           =     0 (w1 ) [free]   DEAD
t195 xor t91 t189            =     2 (w16) [free] 
t196 sextend t172 16         = 65535 (w16) [free] 
t197 and t195 t196           =     2 (w16) [16 tbl]
t198 xor t91 t197            =     5 (w16) [free] 
t199 xor t189 t197           =     7 (w16) [free]   DEAD

##### condPop (2nd)
t200 not t162                =     1 (w1 ) [free]   DEAD
t201 equ 2:3 t138            =     0 (w1 ) [2 tbl]  DEAD
t202 and t200 t201           =     0 (w1 ) [1 tbl]  DEAD
t203 not t202                =     1 (w1 ) [free]   DEAD
t204 and 1:1 t203            =     1 (w1 ) [1 tbl]  DEAD
t205 trunc t138 2            =     1 (w2 ) [free]   DEAD
t206 chose t204 0:1 1:1      =     1 (w1 ) [1 tbl]  DEAD
t207 zextend t206 3          =     1 (w3 ) [free]   DEAD
t208 sub t138 t207           =     0 (w3 ) [3 tbl]  DEAD
t209 select t208 2 3         =     0 (w1 ) [free]   DEAD
t210 trunc t208 2            =     0 (w2 ) [free]   DEAD
t211 and t209 t204           =     0 (w1 ) [1 tbl]  DEAD
t212 xor t211 t204           =     1 (w1 ) [free]   DEAD
t213 select t210 1 2         =     0 (w1 ) [free]   DEAD
t214 trunc t210 1            =     0 (w1 ) [free]   DEAD
t215 and t213 t212           =     0 (w1 ) [1 tbl]  DEAD
t216 xor t215 t212           =     1 (w1 ) [free]   DEAD
t217 and t214 t216           =     0 (w1 ) [1 tbl]  DEAD
t218 xor t217 t216           =     1 (w1 ) [free]   DEAD
t219 and t214 t215           =     0 (w1 ) [1 tbl]  DEAD
t220 xor t219 t215           =     0 (w1 ) [free]   DEAD
t221 select t210 1 2         =     0 (w1 ) [free]   DEAD
t222 trunc t210 1            =     0 (w1 ) [free]   DEAD
t223 and t221 t211           =     0 (w1 ) [1 tbl]  DEAD
t224 xor t223 t211           =     0 (w1 ) [free]   DEAD
t225 and t222 t224           =     0 (w1 ) [1 tbl]  DEAD
t226 xor t225 t224           =     0 (w1 ) [free]   DEAD
t227 and t222 t223           =     0 (w1 ) [1 tbl]  DEAD
t228 xor t227 t223           =     0 (w1 ) [free]   DEAD
t229 not t218                =     0 (w1 ) [free]   DEAD
t230 and t160 t229           =     0 (w1 ) [1 tbl]  DEAD
t231 not t217                =     1 (w1 ) [free]   DEAD
t232 and t162 t231           =     0 (w1 ) [1 tbl]  DEAD
t233 not t220                =     1 (w1 ) [free]   DEAD
t234 and t164 t233           =     0 (w1 ) [1 tbl]  DEAD
t235 not t219                =     1 (w1 ) [free]   DEAD
t236 and t166 t235           =     0 (w1 ) [1 tbl]  DEAD
t237 gtu 2:3 t208            =     1 (w1 ) [3 tbl]  DEAD
t238 chose t237 0:2 2:2      =     2 (w2 ) [2 tbl]  DEAD
t239 zextend t238 3          =     2 (w3 ) [free]   DEAD
t240 add t208 t239           =     2 (w3 ) [3 tbl]  DEAD
t241 xor t230 0:1            =     0 (w1 ) [free]   DEAD
t242 and t241 t237           =     0 (w1 ) [1 tbl]  DEAD
t243 xor t230 t242           =     0 (w1 ) [free]   DEAD
t244 xor 0:1 t242            =     0 (w1 ) [free]   DEAD
t245 xor t232 0:1            =     0 (w1 ) [free]   DEAD
t246 and t245 t237           =     0 (w1 ) [1 tbl]  DEAD
t247 xor t232 t246           =     0 (w1 ) [free]   DEAD
t248 xor 0:1 t246            =     0 (w1 ) [free]   DEAD
t249 xor t234 t230           =     0 (w1 ) [free]   DEAD
t250 and t249 t237           =     0 (w1 ) [1 tbl]  DEAD
t251 xor t234 t250           =     0 (w1 ) [free]   DEAD
t252 xor t230 t250           =     0 (w1 ) [free]   DEAD
t253 xor t100 t62            =     0 (w16) [free]   DEAD
t254 sextend t237 16         = 65535 (w16) [free]   DEAD
t255 and t253 t254           =     0 (w16) [16 tbl]  DEAD
t256 xor t100 t255           =     5 (w16) [free]   DEAD
t257 xor t62 t255            =     5 (w16) [free]   DEAD

##### check pop2==5
t258 not t193                =     0 (w1 ) [free] 
t259 not t258                =     1 (w1 ) [free] 
t260 equ t198 t4             =     1 (w1 ) [15 tbl]
t261 and t171 t259           =     1 (w1 ) [1 tbl]
t262 and t261 t260           =     1 (w1 ) [1 tbl]

##### output
.output t262                 =     1 (w1 ) [free] 
```
