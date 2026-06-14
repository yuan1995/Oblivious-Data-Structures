# The Garbled-Circuit Execution Process, Run For Real: `push 5, push 7, pop→7, pop→5`

This document follows the structure of the zkPass tutorial
[*Yao's Classical Garbled Circuits*](https://medium.com/zkpass/yaos-classical-garbled-circuits-4cbdbadf288e)
— Abstract → the protocol → the garbling schema `f = d ∘ F ∘ e` → the classical worked
example (truth table → labels → encryption → permutation → transfer → evaluation →
decoding) — but instead of a toy 2-gate example, every step is shown **with the actual
values of a real execution** of the oblivious-stack test circuit from
*Circuit Structures for Improving Efficiency of Security and Privacy Tools*
(Zahur & Evans, IEEE S&P 2013).

The companion document `stack-gc-numerical-execution-trace.md` traces the *plaintext
semantics* of this circuit (what values the wires carry). This one traces the **protocol
itself**: wire labels, garbled tables, oblivious transfers, output decoding.

---

## 1. Abstract — who computes what

Two mutually distrusting parties evaluate one public function on private inputs:

| tutorial role | this run | GCParser process | private input |
|---|---|---|---|
| **Alice** — garbler/generator **G** | server, party 2 | `ProgServer` (picks `Wire.R`, garbles) | the pushed values **5, 7** |
| **Bob** — evaluator **E** | client, party 1 | `ProgClient` (holds labels, evaluates) | the expected pop results **7, 5** |

The function *f* is: *"drive the oblivious stack through `push t1, push t2, pop, pop`
and output one bit: did both pops return exactly what E expected?"* Neither party may
learn anything else — G must not learn E's expected values, E must not learn G's pushed
values, and neither may see any intermediate stack state.

**Provenance.** All numbers below come from one actual execution: a faithful FastGC-style
implementation of Yao's protocol (`/tmp/netlist-run/garble_run.py`) was run on the real
`stacktiny.cir` produced by the paper's Haskell compiler (`Circuit/Stack.hs` +
`Circuit/NetList/Gcil.hs`), with G's inputs (5, 7) and E's inputs (7, 5). G-side and
E-side state are strictly separated in the implementation. End-to-end checks: the run
produced exactly **297 garbled tables** (matching the framework's cost model), G decoded
the output as **t262 = 1**, and decoding *every* intermediate wire with G's secret labels
reproduced the plaintext trace for all 262 variables with **0 mismatches**.

```
OUTPUT decoded by G : {'t262': 1}
verified vars       : 262   mismatches: []
R                   : ca103c906a738e78c2ad
garbled tables      : 297   table bytes: 11,880
free XOR / free NOT : 744 / 65        (zero bytes, zero crypto)
E rows decrypted    : 297   E label XORs: 744
OTs executed        : 32
const labels sent   : 2     G-input labels sent: 32
```

---

## 2. The function as a Boolean circuit

The tutorial's first requirement: *the function must be defined as a Boolean circuit.*
Here the compiler did that for us — `stacktiny.cir` is 258 word-level GCIL instructions
over 64 private input bits (4 × 16) producing 1 output bit, which both parties expand
into the identical binary-gate netlist: **297 table gates (AND-type), 744 free XORs,
65 free NOTs**.

### 2.1 Circuit diagram — block level

```
        t1 = 5 [G]      t2 = 7 [G]      t3 = 7 [E]      t4 = 5 [E]
        16 wires        16 wires        16 wires        16 wires
            │               │               │               │
   ─────────┼───────────────┼───────────────┼───────────────┼──── 64 input wires
            │               │               │               │     (+2 constant wires)
            ▼               │               │               │
  stack₀ = empty            │               │               │
  (compile-time constant:   │               │               │
   all 5 slots Nothing,     │               │               │
   head ptr = 2)            │               │               │
      │                     │               │               │
      ▼                     │               │               │
 ┌─────────────────────┐    │               │               │
 │ condPush true t1    │◄───┘ (t2 waits)    │               │
 │ t5…t26              │                    │               │
 │ 10 garbled tables   │                    │               │
 └──────────┬──────────┘                    │               │
     stack₁ │ ≈88-wire state bus:           │               │
            │ 5 slots × (present + 16-bit payload) + 3-bit head ptr
            ▼                               │               │
 ┌─────────────────────┐                    │               │
 │ condPush true t2    │                    │               │
 │ t27…t101            │                    │               │
 │ 118 garbled tables  │                    │               │
 └──────────┬──────────┘                    │               │
     stack₂ │                               │               │
      ┌─────┴──────────────────┐            │               │
      ▼                        ▼            │               │
 ┌─────────────────────┐  ┌─────────────────────┐           │
 │ top → mb₁           │  │ condPop true        │           │
 │ t102…t129           │  │ t130…t166           │           │
 │ 35 garbled tables   │  │ 19 garbled tables   │           │
 │ present = t123      │  └──────────┬──────────┘           │
 │ payload = t128      │      stack₃ │                      │
 └──────┬──────────────┘             │                      │
        │ t123 (1w) + t128 (16w)     │                      │
        ▼                            │                      │
 ┌─────────────────────┐             │                      │
 │ check₁  t167…t171   │◄────────────┼───── t3              │
 │ c₁ = t168 ∧ (t128≟t3)             │                      │
 │   = valid₁ ∧ eq₁    │             │                      │
 │ 17 garbled tables   │             │                      │
 └──────┬──────────────┘             │                      │
        │ c₁ = t171            ┌─────┴──────────────────┐   │
        │                      ▼                        ▼   │
        │       ┌─────────────────────┐  ┌─────────────────────┐
        │       │ top → mb₂           │  │ condPop true        │
        │       │ t172…t199           │  │ t200…t257           │
        │       │ 35 garbled tables   │  │ 46 garbled tables   │
        │       │ present = t193      │  │ (stack₄ never used: │
        │       │ payload = t198      │  │  all 58 instrs are  │
        │       └──────┬──────────────┘  │  dead code)         │
        │              │ t193, t198      └─────────────────────┘
        ▼              ▼                                    │
 ┌─────────────────────────┐                                │
 │ check₂  t258…t262       │◄────────────────────────────── t4
 │ c₂ = c₁ ∧ t259 ∧ (t198≟t4)
 │ 17 garbled tables       │
 └──────────┬──────────────┘
            │ c₂ = t262 (1 wire)
            ▼
      .output t262     →  the ONLY bit either party ever learns:  1
```

The first and last lines of the actual circuit file:

```
.input t1 2 16          # party 2 = G (server):  5
.input t2 2 16          # party 2 = G (server):  7
.input t3 1 16          # party 1 = E (client):  7
.input t4 1 16          # party 1 = E (client):  5
t5 trunc 2:3 2          # head-pointer constant hp = 2 …
…
t260 equ t198 t4        # eq₂  : popped₂ ≟ 5
t261 and t171 t259      # c₁ ∧ valid₂
t262 and t261 t260      # c₂   : ∧ eq₂
.output t262
```

### 2.2 Circuit diagram — gate level of the verdict subcircuit

The part that turns stack outputs into the answer (everything here is *real wire ids*;
`mb₁`/`mb₂` are the `NetMaybe` results of `top` — a presence bit plus a 16-bit payload):

```
 mb₁.present                              popped₁ (16w)      E's expectation (16w)
    t123                                    t128 ═════╗      ╔═════ t3 = 7
     │                                                ║      ║
   NOT t167   (free)                            ┌─────╨──────╨─────┐
     │                                          │   EQU, 16 bits   │
   NOT t168   (free)                            │ 16 × XNOR  free  │
     │                                          │ 15 × AND  15 tbl │
 1 ──┴─► AND t170    1 tbl                      └────────┬─────────┘
         (const-true ∧ valid₁)                           │ t169 = eq₁
          └───────────────┐    ┌─────────────────────────┘
                          ▼    ▼
                       AND t171      1 tbl
                          │
                          │ c₁            mb₂.present
                          │                  t193
                          │                   │
                          │                 NOT t258   (free)
                          │                   │
                          │                 NOT t259   (free)
                          │                   │
                          └───────────┐   ┌───┘
                                      ▼   ▼
                                   AND t261     1 tbl     popped₂ (16w)  ╔══ t4 = 5
                                      │                   t198 ═════╗    ║
                                      │                       ┌─────╨────╨─────┐
                                      │                       │  EQU, 16 bits  │
                                      │                       │ 16 × XNOR free │
                                      │                       │ 15 × AND 15 tbl│
                                      │                       └───────┬────────┘
                                      │                               │ t260 = eq₂
                                      └────────────────┐   ┌──────────┘
                                                       ▼   ▼
                                                    AND t262    1 tbl
                                                       │
                                                  .output t262
```

The 16-bit equality comparator (`equ`) expands as: per-bit XNOR (= XOR + NOT, both
free), then a binary AND-reduce tree — 15 garbled tables for width 16. Note `t167/t168`
is a double negation (the API computes `valid = not (isNothing mb)` and `isNothing` is
itself a NOT of the presence bit): it costs literally nothing, so the compiler does not
bother removing it.

### 2.3 What each instruction costs (gate templates)

Per word-level instruction of width *w* (exactly `Gcil.hs`'s `opcodeAndCost`, including
its documented one-gate overcount on `add`/`sub`):

| GCIL op | binary expansion | garbled tables |
|---|---|---|
| `xor` | per-bit XOR | 0 (free-XOR) |
| `not` | label-pair swap by G | 0 (free) |
| `trunc/select/concat/zextend/sextend` | re-wiring only | 0 |
| `and`, `or` (binary) | per-bit table gate | w |
| `chose c u v` | per bit: `u ⊕ ((u⊕v) ∧ c)` | w |
| `add`/`sub` | ripple-carry, 1 AND per bit | w |
| `equ` | free XNORs + AND-tree | w−1 |
| `gtu` | LSB→MSB chain `gt' = a ⊕ ((a⊕gt)∧(b⊕gt))` | w |

Examples from this run: `t8 and`→1, `t14 add`(3-bit)→3, `t43 and`(16-bit)→16,
`t73 gtu`(3-bit)→3, `t76 sub`(3-bit)→3, `t131 equ`(3-bit)→2, `t169/t260 equ`(16-bit)→15,
`t262 and`→1.

### 2.4 Cost per stack operation (this circuit, as emitted)

| phase | wires | instrs | garbled tables | dead instrs |
|---|---|---:|---:|---:|
| condPush true 5 | t5…t26 | 22 | 10 | — |
| condPush true 7 | t27…t101 | 75 | 118 | — |
| top (1st pop) | t102…t129 | 28 | 35 | — |
| condPop (1st) | t130…t166 | 37 | 19 | some |
| check pop₁ ≟ 7 | t167…t171 | 5 | 17 | — |
| top (2nd pop) | t172…t199 | 28 | 35 | — |
| condPop (2nd) | t200…t257 | 58 | 46 | **all 58** |
| check pop₂ ≟ 5 | t258…t262 | 5 | 17 | — |
| **total** | | **258** | **297** | 101 |

---

## 3. The garbling schema: `f = d ∘ F ∘ e`

The tutorial's formalism maps onto this run exactly — with security parameter
**k = 80**, not 128, because the framework's test harness (`runtestgcparser`) passes
`-w 80` (80-bit wire labels):

- **e — encoding.** Maps the plain inputs x = (5, 7, 7, 5) to garbled inputs X = one
  80-bit label per input bit. Realized two ways: *direct transfer* for G's own 32 bits,
  *oblivious transfer* for E's 32 bits (§4, Step 5).
- **F — the garbled function.** The 297 garbled tables (plus the free-XOR/free-NOT
  bookkeeping). E computes Y = F(X) — one output label — without learning any plain
  value (§4, Step 6).
- **d — decoding.** The secret map {W⁰ ↦ 0, W¹ ↦ 1} for the output wire t262, known
  only to G. E sends Y to G, G applies d and shares y = 1 (§4, Step 7).

---

## 4. Classical Yao's seven steps — executed with real values

The tutorial walks a toy circuit through seven steps. Here is each step as it actually
happened for `stacktiny.cir`. The message flow, end to end:

```
        G — garbler (server)                      E — evaluator (client)
              │                                           │
              │  0. both parse stacktiny.cir (public)     │
              │◄─────────────────────────────────────────►│
              │                                           │
              │  labels for G's 32 input bits + 2 consts  │
              │  ────────────────────────────────────►    │   340 B
              │                                           │
              │  32 × 1-of-2 oblivious transfer           │
              │  C, gʳ, e₀, e₁  ───────────────────►      │
              │       ◄───────────────────  pk₀           │
              │                                           │
              │  297 garbled tables, STREAMED in          │
              │  instruction order  ────────────────►     │   11,880 B
              │                              (E evaluates each gate on arrival)
              │                                           │
              │  output label of t262                     │
              │       ◄───────────────────                │   10 B
              │                                           │
              │  decoded result: t262 = 1  ─────────►     │   1 bit
              ▼                                           ▼
```

### Step 1 — Create the truth table (tutorial step 1)

Every one of the 297 table gates has an ordinary 2-input truth table. The very first
one in the circuit, gate id 0, implements `t8 = and t6 1:1` — and since `t6` resolves
to the constant-1 wire, *both* gate inputs happen to be the same wire:

```
   a   b  │ a∧b                a = b = const-1 wire
   0   0  │  0
   0   1  │  0
   1   0  │  0
   1   1  │  1
```

There is nothing secret yet; both parties know every gate's truth table (the circuit is
public — only the *inputs* are private).

### Step 2 — Assign random labels to each wire (tutorial step 2)

The tutorial assigns each wire a random label pair (A⁰, A¹). This run does the same with
two modern refinements:

- **free-XOR**: G samples a single global secret offset, this run
  `R = ca103c906a738e78c2ad` (low bit forced to 1), and only W⁰ is random per wire;
  always `W¹ = W⁰ ⊕ R`.
- **point-and-permute**: a label's low bit is its public **select bit**; since
  lsb(R)=1, the two labels of every wire have opposite select bits. Which select bit
  means "0" is random per wire (the *permute bit*) — that randomness replaces the
  tutorial's row shuffle (Step 4).

Real labels generated in this run (select bit in parentheses):

| wire | W⁰ ("means 0") | W¹ = W⁰⊕R ("means 1") |
|---|---|---|
| t1 bit 0 | `0f8e9ef0c232d06a553e` (0) | `c59ea260a8415e129793` (1) |
| t1 bit 1 | `9adcbf585f803a80a61d` (1) | `50cc83c835f3b4f864b0` (0) |
| t3 bit 0 | `be76e464b056da35508c` (0) | `7466d8f4da25544d9221` (1) |
| const-1 wire | `6af536200e0909daa804` (0) | `a0e50ab0647a87a26aa9` (1) |
| t8 (gate-0 output) | `c975a136a7366b13e1b7` (1) | `03659da6cd45e56b231a` (0) |
| t262 (output wire) | `bb5c407d6ffe3aa0ca6f` (1) | `714c7ced058db4d808c2` (0) |

Check the free-XOR invariant on the first row:
`0f8e9ef0c232d06a553e ⊕ ca103c906a738e78c2ad = c59ea260a8415e129793` ✓.
Note t1-bit-1 and the t8/t262 wires: their select-bit↔value association is flipped —
that is the random permute bit at work.

Only table-gate outputs and input wires get fresh randomness. XOR outputs are *derived*
(`W⁰_out = W⁰_a ⊕ W⁰_b` — the entire free-XOR trick), and a NOT is just G swapping its
own interpretation of the pair (E does nothing at all). The two constants `0`/`1`
appearing in the circuit become two extra wires whose correct label G simply sends.

### Step 3 — Encrypt the truth table (tutorial step 3)

The tutorial encrypts each output label under the two input labels of its row,
`Enc_{(A,B)}(E)`. Here the cipher is one hash call:

```
row = H(W_a ‖ W_b ‖ gateId) ⊕ W_out         H = SHA-1, truncated to 10 bytes
```

G's actual garbling of gate 0 (`t8`), using the labels from Step 2 — all four rows as
G computed them (reconstructed from the run's recorded labels and verified to match the
recorded table byte-for-byte):

```
(a=0,b=0)→0 :  H(6af5…04, 6af5…04, 0) ⊕ W⁰ₜ₈ = cdb6418e18c9d7ae527e
(a=0,b=1)→0 :  H(6af5…04, a0e5…a9, 0) ⊕ W⁰ₜ₈ = c9bafcbaeef4d8ec686a
(a=1,b=0)→0 :  H(a0e5…a9, 6af5…04, 0) ⊕ W⁰ₜ₈ = 3ce0de0494c43c3ede62
(a=1,b=1)→1 :  H(a0e5…a9, a0e5…a9, 0) ⊕ W¹ₜ₈ = bd41caef43d65cd87eec
```

Decrypting any row requires holding *both* matching input labels — and E will only ever
hold one label per wire.

### Step 4 — Permute the rows (tutorial step 4)

Classical Yao shuffles the four ciphertexts randomly (this is where the name *garbled*
comes from), and Bob must trial-decrypt all four. This run uses the modern equivalent,
**point-and-permute**: the row index *is* the pair of public select bits,

```
table[ 2·select(W_a) + select(W_b) ] = H(W_a‖W_b‖gid) ⊕ W_out
```

Because each wire's permute bit is random (Step 2), the rows still land in an order
that is uncorrelated with plaintext values — but E knows exactly *which* row to open
(one decryption instead of four trial decryptions), while still learning nothing about
what that row means. For t8 the four rows above happen to land in truth-table order
(the const-1 wire's permute bit was 0); for a gate fed by t1-bit-1 they would land
shuffled.

### Step 5 — Transfer: input labels move to E (tutorial step 5)

E needs exactly one label per input wire. Two mechanisms:

**G's own inputs (t1 = 5, t2 = 7; 32 bits): direct transfer.** G knows its bits and
sends the matching label — like the tutorial's Alice sending (A¹, C⁰). For t1 = 5 =
…0101₂, from this run:

```
bit0 = 1 :  G sends W¹ = c59ea260a8415e129793
bit1 = 0 :  G sends W⁰ = 9adcbf585f803a80a61d
bit2 = 1 :  G sends W¹ = 8a17b2e09793e91d6eef
bit3 = 0 :  G sends W⁰ = 885362172fa0c93b0aa8
```

These are uniformly random strings to E — it never sees the counterpart label, so they
carry no information about 5.

**E's inputs (t3 = 7, t4 = 5; 32 bits): 1-of-2 oblivious transfer.** Like the
tutorial's Bob obtaining B⁰ "through a 1-out-of-2 oblivious transfer". For each bit, G
offers (W⁰, W¹); E's private bit σ selects one. The actual Diffie-Hellman OT (768-bit
Oakley group) for t3's bit 0 — σ = 1, since 7 is odd:

```
G → E :  C    = e624b98ff602016acb3c5722…     (random group element)
E → G :  pk0  = 2761c5aea496f9ccd75d3c56…     (E knows the dlog of pk_σ only;
                                               the constraint pk0·pk1 = C forces
                                               ignorance of the other key)
G → E :  gʳ   = a39b25fd413f297aaf016eae…
         e0   = 9bf07ff4015563acd434   =  W⁰ ⊕ H(pk0ʳ)
         e1   = e05c67e0aa3eeee8a955   =  W¹ ⊕ H(pk1ʳ)
E      : decrypts e1 with (gʳ)^k  →  7466d8f4da25544d9221  =  W¹  ✓
```

G never learns σ = 1; E never learns W⁰. After 32 such OTs (the real FastGC runs 80
Naor-Pinkas base OTs once and IKNP-extends them — same interface, cheaper at scale),
E holds exactly one opaque 80-bit label for each of the 64 input wires + 2 constants.

### Step 6 — Evaluation (tutorial step 6)

The tutorial's Bob "goes through all gates one by one and tries to decrypt the rows in
their garbled tables; he is able to open one row for each table." Here, both parties
walk the 258 instructions *in lockstep* — this is FastGC's pipelining: G garbles a gate,
streams its 40 bytes, E opens its one row immediately, both free dead wires (`.remove`)
and move on. Neither side ever stores the whole garbled circuit.

Per binary gate:

- **XOR** (744×): E XORs the two labels it holds. No table, no crypto, no bytes.
- **NOT** (65×): E does nothing (G swapped the pair's meaning on its side).
- **table gate** (297×): E reads the select bits (low bits) of its two held labels,
  opens exactly that row, and XORs out `H(held_a ‖ held_b ‖ gid)`.

**Worked gate — the first table gate, t8 (gid 0).** E holds the const-1 wire's label
twice (it was sent in Step 5):

```
held_a = a0e50ab0647a87a26aa9   select bit 1
held_b = a0e50ab0647a87a26aa9   select bit 1     → open row[2·1+1] = row 3

row 3  = bd41caef43d65cd87eec
output = bd41caef43d65cd87eec ⊕ H(a0e5…a9, a0e5…a9, 0) = 03659da6cd45e56b231a
```

E now holds `03659da6cd45e56b231a` for wire t8 — to E an opaque string; G knows it is
W¹ (plaintext 1, matching the plaintext trace).

**A 16-bit mux, t43 = and t41 t42** (part of the second push's payload routing):
expands to 16 independent table gates. Bit 0 from the run: E's select bits chose row 3
of `[b8066f11…, 7b4a3c04…, 8f572a41…, 7a147827…]` and decryption yielded that wire's
W⁰ (plaintext 0 — correct). Sixteen such tables moved 640 bytes.

**The comparisons that implement the test:** `t169 equ t128 t3` — *is the first popped
value equal to E's expected 7?* — cost 16 free XNORs + 15 AND-tree tables; likewise
`t260` for the second pop. The comparison *results* are labels like everything else:
at no point does either party see "equal/not equal". Every push, slide, presence bit,
head-pointer add/sub of the oblivious stack travels through Step 6 the same way —
encrypted as labels, costing tables only at AND-type gates.

### Step 7 — Output decoding (tutorial step 7)

After the last gate E holds one label for the single `.output` wire, t262. The final
gate of the run, `t262 = and t261 t260` (gid 296) — G's four rows (reconstructed and
verified, as in Step 3) and E's one decryption:

```
G garbles:                                          E evaluates:
(0,0)→0 : c005d1f50089affdb7f5                      held_a = 8a8786bc5b09152742e1 (s1)
(0,1)→0 : 41e6d32b60e6c5ba4fa8                      held_b = cc3a90e85f126b8ed969 (s1)
(1,0)→0 : 1ac8013f80b1758eca12                      → opens row[2·1+1] = row 3
(1,1)→1 : f03d7a159ff8693afcd5                      → f03d…d5 ⊕ H(a,b,296)
                                                      = 714c7ced058db4d808c2
```

E cannot interpret `714c7ced058db4d808c2`. Exactly like the tutorial's Bob holding I⁰,
it sends the label to the party that owns the decoding map d (in GCParser:
`GCParserClient.getOutputValues()` writes the label, `GCParserServer` interprets it).
G compares it with the pair it generated for t262 — it equals **W¹** — so the output
bit is **1**. G prints `t262 = 1` (this is the line in `results/siserverout` that the
framework's `makeutils/GcilTest` greps for), and returns the plaintext to E so both
parties learn the result:

> **1 — both pops returned exactly what E expected; the oblivious stack behaved
> exactly like a real stack.**

That single bit is the only semantic information either party gains all protocol.

---

## 5. The complete conversation, by the numbers

| # | direction | content | size (this run) |
|---|---|---|---|
| 1 | handshake | label length (80), public circuit | — |
| 2 | G → E | labels for G's 32 input bits + 2 constant wires | 340 B |
| 3 | G ⇄ E | 32 DH oblivious transfers for E's input bits | ~32 × (2 group elts + 20 B) |
| 4 | G → E | 297 garbled tables, streamed in instruction order | 11,880 B |
| 5 | E → G | 1 output-wire label | 10 B |
| 6 | G → E | decoded plaintext result `t262 = 1` | 1 bit |

Work done: G computed 297 × 4 = 1,188 H-evaluations to garble (plus label bookkeeping);
E computed 297 single-row decryptions and 744 label XORs.

**What each party never learns.** G: E's expected values (7, 5), E's OT choice bits,
any intermediate wire value. E: G's pushed values (5, 7), the meaning of any label it
held, any non-selected label, R. Both learn only: the circuit (public), the operation
schedule (public — that's why the *oblivious* stack exists at all: the schedule may be
public while the data and effective head position stay secret), and the final bit 1.

---

## 6. At last — classical Yao vs. this execution

| zkPass tutorial (classical Yao) | this run (FastGC-style) |
|---|---|
| security parameter k = 128 suggested | k = 80 (`runtestgcparser -w 80`) |
| both labels of a wire independent | free-XOR: `W¹ = W⁰ ⊕ R`, one global secret R |
| every gate gets a garbled table | only AND-type gates (297); XOR (744) and NOT (65) are free |
| rows shuffled randomly; Bob trial-decrypts up to 4 rows | point-and-permute: random permute bits order the rows; E opens exactly 1 |
| `Enc_{(A,B)}(·)` symmetric encryption + redundancy to detect success | one SHA-1 call: `row = H(W_a‖W_b‖gid) ⊕ W_out` |
| whole garbled circuit sent, then evaluated | pipelined: garble → stream → evaluate → free, gate by gate |
| OT per evaluator input bit | here literally 32 DH OTs; production FastGC: 80 base OTs + IKNP extension |
| Alice shares the output mapping | E returns the output label, G decodes and announces |

Two fidelity notes on the circuit itself: the 297-table count is the circuit *as
emitted*; the framework's `optimize.sh` (constant folding + dead-code elimination)
would shrink this regression test substantially — 101 of 258 instructions are dead,
including the entire second `condPop`, because all push/pop conditions are the constant
`true` here. In the real benchmarks (200 random ops, DBSCAN) the conditions are secret
wires and the tables remain. And the hierarchical-buffer machinery (the paper's Fig. 13
`knownNothing` elision) already fired at *compile* time — that is why a 4-element stack
costs only ~75 instructions per operation.

**Reproduce:** `cd /tmp/netlist-run && python3 garble_run.py`
(artifacts: `tmp/garble-showcase.json`; labels are fresh randomness each run —
the values above are from the recorded run, reconstructible via `reconstruct.py`).
