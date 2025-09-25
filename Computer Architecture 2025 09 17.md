### Computer Architecture — Lecture Notes (Sep 17, 2025)

These notes consolidate and correct the lecture content. They fix logical/factual errors from the live transcript and add structure and explanations for clarity.

#### Branch Target Buffer (BTB) vs Branch Predictor
- **What the BTB does**: Predicts the target address of a branch so the fetch unit knows where to fetch next. It is consulted very early (often in the fetch stage) using the current PC as an index/tag.
- **Conditional branches**:
  - If predicted not taken, the next PC is simply `PC + 4`; no BTB entry is needed.
  - If predicted taken, the BTB supplies the predicted target.
- **Unconditional branches and indirect branches**:
  - Unconditional direct jumps always use their target; a BTB entry avoids waiting for decode/execute to compute the target.
  - **Indirect branches** (e.g., `jmp r`, virtual dispatch, switch/jump tables) and **returns** are branches whose targets come from registers/memory. A return is an unconditional indirect branch whose target is the previously saved return address.

##### Returns: why BTB alone struggles
- A single BTB entry per static return instruction cannot capture multiple dynamic return targets (different call sites). This causes target aliasing and frequent mispredictions if we only use a BTB.
- Modern designs add a small **Return Address Stack (RAS)**: on call, push `PC+4`; on return, pop to predict the return target. A modest-size RAS (e.g., 8–32 entries) yields very high return prediction accuracy.
- For other indirect branches, processors often add specialized **indirect target predictors** (e.g., per-branch target caches keyed by global/local history) beyond a simple BTB.

##### What if the BTB target is wrong?
- The front-end fetches along the wrong path until the true target is known (in decode/execute). The pipeline then performs a misprediction recovery: **flush younger work** and redirect fetch to the correct target. The cost is the branch resolution latency times the front-end width.

#### Tomasulo’s Algorithm (IBM System/360 Model 91)
- Spelled “Tomasulo,” not “Thomas.” Introduced commercially on IBM System/360 Model 91 (~1967) for floating-point units.
- Core ideas:
  - **Reservation Stations (RS)**: per-FU buffers that hold operations and their operands/tags, decoupling issue from execution.
  - **Register Renaming** (rename table): map architectural destination registers to RS/physical tags, eliminating WAW and WAR hazards while preserving true RAW dependencies.
  - **Common Data Bus (CDB)**: when an FU finishes, it broadcasts `(tag, value)`; any RS waiting on that tag captures the value. The register file also captures the value only if its rename table still points to that tag (i.e., it is still the latest producer).

##### Pipeline in Tomasulo
- **Issue (in order)**:
  - Allocate an RS for the decoded instruction (stall if none available).
  - For each source operand: if the value is ready in the register file (no pending producer in rename table), copy it into the RS; otherwise record the producing tag (Qj/Qk).
  - Update the rename table to map the destination register to this RS’s tag.
- **Execute (out of order)**: When all operands in an RS are ready, the FU starts execution. Different op latencies are supported naturally.
- **Write Result (out of order)**: Broadcast `(tag, value)` on the CDB. All RSs matching the tag capture the value; the register file also captures it only if the rename table still maps that register to the same tag (ensuring that only the latest writer commits to the architectural register value). This preserves program-order semantics for reads without stalling for WAW/WAR.

##### Why WAR/WAW hazards disappear
- Because architectural names are renamed to unique tags, later writers do not clobber earlier readers (no WAR), and two writers target different tags (no WAW). Only true dependencies (RAW) remain and are enforced via tags.

#### Loads, Stores, and Memory Disambiguation
- Loads/stores use a **Load/Store Queue (LSQ)** that preserves program order across memory operations.
- A conservative baseline is to allow a memory op to execute only when it reaches the LSQ head and its address and dependencies are known; this avoids read-after-write violations through memory.
- To gain performance, modern CPUs deploy a **memory disambiguation predictor**: allow a younger load to speculatively bypass older unknown-address stores if predicted not-aliasing. On verification, if a conflict is detected (load read the wrong value), the machine squashes younger work and re-executes.
- Example hazard: A store to `[R1+1000]` preceded by a multiply producing `R1`, and a younger load from `[R0+1000]` where `R0=0`. If the load executes early and later `R1` resolves to `0`, then both refer to the same address; the load must get the store’s value. The LSQ/disambiguation logic prevents or repairs this.

#### Branch Prediction with Tomasulo: need in-order commit
- Pure Tomasulo (as taught in its simplest form) allows results to be broadcast as soon as they are ready. This makes speculative recovery difficult: a mis-speculated instruction might have already updated the architectural state via the CDB.
- Modern out-of-order processors therefore add a **Reorder Buffer (ROB)** and perform **in-order commit**:
  - Results are buffered after execution (and may update physical registers) but architectural state (register file/memory) is updated only when the instruction reaches the ROB head and is known to be non-speculative (e.g., branch resolved correctly, exceptions known).
  - On a branch misprediction, the machine discards all younger ROB entries and restores rename/ROB checkpoints.

#### Common Data Bus (CDB) notes
- A CDB transfer is designed to be very low latency (often a single cycle) so that dependent RSs can capture the value and begin execution in the next cycle.
- Multiple FUs may finish in the same cycle; an arbiter selects which can broadcast. Superscalar designs frequently provision multiple CDBs to maintain throughput.

#### Brief historical context (corrected)
- **IBM 7030 “Stretch” (1961)**: an earlier high-performance IBM system.
- **CDC 6600 (1964, Seymour Cray)**: introduced scoreboarding and multiple functional units; roughly 3× faster than Stretch and widely regarded as the first successful supercomputer.
- **IBM System/360 Model 91 (~1967)**: introduced Tomasulo’s algorithm (for FP) and very aggressive pipelining/out-of-order execution of FP ops.
- **CDC 7600 (1969)**: substantially faster successor to CDC 6600 with deeper pipelining; retook performance leadership.
- Later: vector machines, shared-memory multiprocessors, and then heterogeneous CPU+accelerator systems. Modern Top500 leaders pair many-core CPUs with GPUs/accelerators.

#### Worked example (floating-point latencies)
- Assumed execution latencies (execute stage only):
  - Load = 2 cycles; FP Add/Sub = 2; FP Mul = 10; FP Div = 40.
- Sequence and dependencies (abridged from lecture): two loads feeding a multiply and a subtract; divide depends on multiply; final add depends on subtract. With sufficient RSs/CDB bandwidth, Tomasulo schedules ops close to the ideal critical path.
- Rough timing: first two loads overlap; multiply starts once its load operand arrives; divide starts after multiply; subtract/add finish earlier and are off the critical path. Total completion around 56–57 cycles, matching the dependency-critical path under the stated latencies.

### Additional elaboration, segment-by-segment

1) BTB Q&A (around lines 113–149)
- **Update policy**: A basic BTB typically overwrites an entry with the most recent target for that branch PC. Real designs use set-associative BTBs with LRU/PLRU replacement to reduce conflicts; entries are usually filled/updated on branch resolution.
- **Single target per entry**: A standard BTB stores one target per static branch. This is insufficient for returns and other indirects that have many dynamic targets; use a **RAS** for returns and specialized **indirect target predictors** for indirect branches.
- **Misprediction handling**: Fetch goes to the wrong target until the branch resolves; then the pipeline flushes younger work and redirects to the correct PC.

2) Fetch queue concept (around lines 521–538)
- **Fetch/issue decoupling**: The front end can fetch ahead and place decoded instructions in an instruction/issue queue even if backend issue/execution is temporarily stalled by dependencies or RS/LSQ pressure.
- This smoothing improves utilization and hides front-end latency.

3) Issue and rename walk-through (around lines 545–657)
- On issue, each source operand either copies a ready value (Vj/Vk) or records a producing tag (Qj/Qk) from the rename table.
- The destination architectural register is mapped to the RS/FU tag in the rename table. Later CDB broadcasts deliver `(tag, value)` that RSs and the register file capture if still mapped to that tag.

4) CDB timing model (around lines 1532–1545)
- The lecture’s “first half write, second half read” is a helpful abstraction. Implementations may pipeline/segment the CDB and wakeup-select network; the key property is that dependents can typically begin in the next cycle after a producer’s broadcast.

5) Loads/stores and LSQ (around lines 739–916)
- **Baseline**: Execute memory ops conservatively in program order at the LSQ head once their addresses and data are ready. This avoids memory RAW violations.
- **Speculation for performance**: A memory disambiguation predictor may let younger loads bypass unknown-address older stores when predicted non-aliasing. Detection of a conflict triggers squash and re-execution.

6) Branch prediction with Tomasulo (around lines 1031–1110)
- Pure Tomasulo broadcasts directly to architectural state, which complicates recovery. Modern OoO adds a **ROB** and performs **in-order commit**, with rename-map checkpoints on branches to enable precise recovery on misprediction or exceptions.

7) CDB contention and throughput (around lines 1193–1210)
- Multiple FUs can finish concurrently. Designs arbitrate access per cycle; superscalar cores often provision multiple CDBs (or segmented result networks) to sustain width.

8) Area vs. complexity (around lines 1211–1244)
- Area for more RS entries, rename tables, and FUs is usually manageable. The challenging scaling is the wakeup/select CAMs and broadcast networks, where dependency checks and tag broadcasts grow superlinearly with width and window size.

9) Worked timing sketch (around lines 1254–1637)
- Cycles 1–4: Two loads overlap; first load broadcasts, second completes next.
- Cycle 5: Second load broadcasts; multiply and subtract receive operands and start (Mul needs 10, Sub needs 2).
- Cycles 6–11: Sub finishes; add issues and completes; results broadcast via CDB.
- Cycles 6–16: Mul completes and broadcasts; divide receives its input and starts (40 cycles).
- Cycle ~56–57: Divide completes; the program’s critical path matches the dependency chain under given latencies.

---
Raw transcript (unedited) below:
ment is due today, right? So, it's like
the latest thing is due tomorrow. Okay.
So, so there's been a bunch of questions
about uh the branch target buffer. So,
uh so let me just clarify that a little
bit. Okay. So,
so a branch target buffer and a branch
predictor are logically different
structures like in today's processors in
fact one is accessed much earlier than
the other right so these are not like I
showed in one of the slides that because
they use the same PC I I ch I used for
one of the advanced predictors them
being logically the same but They need
not be logically the same entity.
They're different entities, right? So
what does a branch target buffer do? For
any type of branch, whether it is
conditional branch or an unconditional
branch, you need to be able to predict
where the branch actually needs to go.
Right? So if it's a if it's a
conditional branch and if it's not
taken, then it's simple. It's PC++ 4.
You don't need a branch target buffer
for that. Everybody agree that?
>> But if it's a conditional branch and if
it's taken, you need to know the target.
Everybody agree with me here?
And we've been talking about conditional
branches. But there are also
unconditional branches, right? Do you
see what an unconditional branches? It's
like a jump. Jump is an unconditional
branch. Uh but you can also have
uh unconditional branch which is
indirect right value where value jump is
a function of a register value
right agree with me here everybody and
what after all is the return instruction
it's a form of an indirect branch again
you agree with me here so the question
what it really asks you is consider the
the semantics of a return instruction.
At the heart of a return instruction,
what is it doing? It's an indirect
branch. It's an unconditional indirect
branch, isn't it? So,
one thing about a return instruction is
uh is interesting, right? And I'm going
to give you a hint and actually the hint
is given in part of the question. The
the fact that multiple a function can be
called at multiple call sites, isn't it?
So if you have a library function that
is called like say million times in your
program, potentially there are more than
one call side, isn't it? So you see what
I mean here? So you can have a let's say
print f is such a function you can call
it within the main function. You can
call it within any other function,
right? Where you're calling
so affects where the return address
actually jumps back, isn't it? That is
really the heart of what that question
is about. If you have multiple call
sites, it means by definition the return
which is an indirect branch instruction
has multiple targets. Right? So that's
the heart of the question. How well will
a small cache
which is essentially storing for every
instruction address what the target
address is going to be how well is it
going to be for performed for a
particular indirect branch instruction
that is returned which has multiple
targets that's the heart of the question
okay does that make sense yes
>> if I remember correctly on that question
on the homework did mention that the
call function will actually put the
return address in memory. Yes. So
we might also need to consider that.
>> Yeah. Yeah.
>> Okay.
>> Yes.
>> Sorry.
So how many addresses you
like?
>> So the way in which you can only store
one,
>> right? So that is the way in which
typically
caches function right. So, so you cannot
have something like a oh by the way you
can be creative and think about one such
structure but in the way that I
explained in the class
it's just a simple structure you can
only score only one
so so so and then the next part really
asks you to think about carefully think
about
ideas for improving predictability for
instructions.
>> Okay, everybody in the same page here?
>> Any followup questions on this?
>> Yes.
>> Yeah, go ahead.
>> Um, just with branch target buffers,
does it overwrite the table every time
with like just the most recently done
one?
>> Yeah, that would be the obvious way to
implement it unless you come up with a
different way.
Yeah, you and then
>> and making sure I didn't misunderstand
anything. We're only storing a single
return address at the time inside of the
branch private buffer.
>> Yeah, in the way that I introduced
branch targets and the most logical
implementation, that's how it would
perform.
>> That's how it would function.
>> Okay.
>> If the BTB yields an incorrect value,
what happens?
uh it'll be similar to a branch in this
prediction.
>> Okay.
So it would just flush the address sorry
>> uh so it's like for example if it's a
branch right so uh you predict so I'm
considering a jump instruction let's say
or an indirect branch or a return okay
you predict something you see that uh uh
you predict that it needs to go back to
some location 100 but in actual fact it
needs to go to th00and okay so you'll
assume it's 100 and then fetch from
there and later realize that oh actually
you should have fetched from thousand
and you just need to flash the pipeline.
>> All right,
>> guys. Okay. Okay. Any other questions?
Cool. Let's now uh go back to Thomas's
algorithm. Uh and actually I want to you
asked a question last time, right? I
introduced uh
I introduced uh that in 1964
um
CDC 6,600
uh was invented and at that time 1964 it
was IBM uh which which had the fastest
computer and CDC 6600 which came out in
uh 1964 by Cray
used this idea called scoreboarding
which is a form of out of order
execution and it turned out to be uh
actually
[Music]
just a second I have not refreshed the
slides just a second let me let me
Sorry about that. Just a second
because I have some interesting
information. I thought I'd share that.
Just a second.
Right. So, um it turns out that yeah,
CTC 6600 was invented in 1964. Right. At
that point, uh IBM had a processor
called IBM 7030
and that was the fastest computer out
there then. Okay. So and CDC 6,600 it's
like a startup imagine it's a small
startup and IBM is this behemoth company
and then it turn turned out to be
actually three times faster than the
fastest computer until that point and
then the uh there was this guy uh called
Thomas J. Watson um and he
um yeah sent out a memo saying uh
they've announced a 6,600 system. I
understand that in the laboratory
developing that system there are only 34
people contrasting this modest effort
with a vast development activities. I
failed to understand why we have lost
our industry leadership uh by letting
someone else offer the world's most
powerful computer. So this is a memo
that this guy sent to all of the
employees in IBM and that's when I in 3
years they came out with this uh uh 30
IBM
360. Okay. So to answer your question so
um this IBM that was the earlier fastest
computer CDC 6600 was three times
faster. Okay. So and then I might have
actually said this wrong. IBM came up
with 36091. Yeah. So it turned out to be
about um
two to three times faster than CDC
6,600. So with tomosulos algorithm they
managed to beat CDC 6,600 and also they
gained from a little bit about with
denard scaling a faster clock. So they
did manage to beat 6,600 but then then I
actually looked up right. So very soon
uh after they came up with CD6 uh IBM
360 in 1967 CDC 7,600
so Cray came up with the next version of
their processor and they totally beat
360 again. Okay. So it was a kind of a
race between them. So I I thought I
could actually get this information from
then on. So uh and then on kind of list
the fastest computers at different
points in time right so then we actually
started producing this as the top 500
list right currently we have a top 500
list where it's a ranking of the
supercomputers so so interest so if you
see some of the ideas 730
actually came up with pipelining okay so
so what we've covered and then uh 667
came up with scoreboarding and multiple
functional units, right? That we just
covered the last lecture. 360 came up
with Tomasulus algorithm and uh the
Tomasulus algorithm did out of
orderuling and pipelining but only for
floatingoint instructions and then CC
7,600 actually came up with a more
aggressive version of pipelining which
included not only floatingoint
operations but then integer operations
also. Okay. So and then one of you
mentioned vector processing. That's when
again Cray came up with uh the company
that produced CDC 6600 came up with
vector processing and that really uh and
then shar memory multipprocessing which
we are going to study later in the in
the course. Uh and then um
there was a point in time where you had
heterogeneous architectures and
IBM was the first one there.
They basically combine the CPU with
another architecture like a GPU but not
exactly. They called it the cell
multipprocessor but from then on it is
the age of GPUs. Every
most powerful supercomputer has GPUs and
CPUs working in concept. So so it's yeah
it's very interesting.
Okay coming back right. So we saw that
uh the key idea of tomosulos algorithm
um is really
three things reservation stations
register renaming and a common datab
bus. Okay. So,
we already saw what a reservation
station is.
Um, right. So, so it's essentially that
you don't need to when an instruction
needs to be issued after it's decoded,
you don't need to have a free function
unit available necessarily. All you need
is a buffer, set of buffers associated
with every function unit. It's called a
reservation station. All you need is a
free buffer entry so that it can be
issued to that entry and maybe it can
wait there. Okay, so that's what a
reservation station is. And the next
idea is called register renaming. And
the main idea here is that uh you're not
necessarily waiting for
the registers themselves but you are
really waiting for the reservation
stations. Right? So reservation stations
are effectively a way in which
architectural res registers are renamed
by the hardware into physical registers
and the reservation station's ID is one
way to represent a physical register.
Okay, that's what we mean by registering
and we saw an example for that. So in
this case if you see there's a load
instruction here which is loading a
value from memory into R1 and here is a
multiply instruction which is using the
value of R1. Okay but with register
renaming what are we going to do? This
load instruction is going to be issued
to a reservation station. And because
it's a load instruction, it's going to
be issued to a buffer associated with a
load store unit. There are going to be
multiple buffers. And one of these
buffers is a reservation station for
this load instruction. And let's say the
identity of that reservation station is
RS2. Okay. In allocating this load
instruction into that particular buffer
which is called RS2, you are effectively
renaming
the target register R1 with this
reservation station ID RS2.
Okay. So when this multiply instruction
it uses the value of R1
but it's effectively waiting for this
reservation station RS2 instead of R R1
because we have renamed R1 into this
reservation station RS2. So what this
gives you is the ability to not stall
for right after write hazards and write
after read hazards. You always need to
make read after write hazards work
because those are the two dependencies.
So you recall everything that I'm saying
here.
So and that's what this example
demonstrates, right? So the first load
instruction loads into uh register R0.
But suppose the load instruction goes
and lives in a reservation station
called RS1. So you are effectively
renaming R0 to RS1. Okay. So the next
load instruction uh likewise loads into
register R1. Suppose it goes and lives
in reservation station RS2. You're
effectively renaming R1 to RS2. You're
kind of mapping R1 to RS2. And where are
these mappings actually held in metadata
structures associated with the register
file called the rename table, right?
It's essentially a mapping between the
architectural register and the renamed
reservation station ID. Okay? Now you
have this multiply instruction which
computes R4 by performing R0 * R1. So
it's dependent on R0 and R1. But in this
case we have completely renamed it. So
it's like RS3 the reservation station
where the multiply instruction is issued
to depends on RS1 and RS2. Yeah,
everybody following me here? So again
what this gives you is suppose uh you
have a here we have a read after write
dependency and that read after write
dependency is reflected in the
reservation station dependency as well
right so you have multiply instruction
dependent on R0 and R1 here after
renaming we are enforcing these
dependencies
you're relying on RS1 and RS2 okay
Yes.
>> At what stage does the renaming happen?
>> When at the point in which an
instruction is issued, that's when the
renaming happen. So we are going to
discuss this thoroughly. Okay. The
algorithm itself.
So the main advantage is that when you
have a subsequent instruction
also writing to R1. Okay. And so
intuitively you have this add
instruction the fourth instruction which
can actually perform out of order and
complete before the multiply
instruction. Right? So so um that is the
main uh thing here. Even if the add
instruction completes there is no
problem here. Why?
Because
so here if you see there is a right
after write dependency between the load
instruction and the add instruction.
Why? Because both of them are writing to
the register R1. Right? So now the
advantage here is that with register
renaming we would have renamed this AD
instructions target as RS4 the
reservation station where the ad issued.
So when even if the add instruction
completes before the node, there is no
risk of the multiply instruction
reading the value produced here because
it is still going to be dependent on
RS2.
That is what register renaming gives us.
The fact that we have renamed these
registers means that even when there is
a potential right after right hazard and
the addition completes out of order the
actual read after write dependencies
that we care about from the load to the
multiply remains unaffected. Okay,
does this make sense? This is the
critical point that I want all of us to
appreciate. Have any questions about
this? You understand how even though
there's a right after write hazard
between the load instruction and the add
instruction and let's say even if the
add instruction completes before the
load instruction the actual value that
the multiply instruction reads is still
going to read the value produced by the
load instruction because we have renamed
the registers. Okay, so everybody
appreciates this.
Okay,
so that's the core of register renaming.
Now the final idea with uh Tomasuro's
algorithm. Um, oh yeah, we also
discussed how remember here that we have
a load instruction and an add
instruction. Uh, and we I asked this
question, what instruction should
finally update the value to the register
file? And you all agreed that it should
be the add instruction. Why? because
that is the one that is later in the
program order. Okay. So, so the one even
if the add instruction completes before
the load instruction, the add
instruction will actually write its
value into the register file, the load
instruction which potentially completes
later. It will supply the value for the
multiply instruction through the common
data bus, but it will not go ahead and
write its contents to the register file.
Okay? And it should not, isn't it? And
how is this ensured? Using the rename
table because
when this add instruction is issued,
register R1's rename table will say
actually I'm waiting for RS4.
That is the reservation station which is
going to provide the value of R1 that I
need to update when I see the value in
the common data bus. So later when the
load instruction publishes the value R1
like the reservation station RS2 through
the common data bus only the multiply
instruction will use its value because
it's dependent on the node instruction
but the register file will totally
ignore it. Why? Because the register
file knows that it's really RS4 that
it's waiting for. Right? And that's
already happened.
The load instruction is RS2.
It's not waiting for that. Okay.
You see how the rename table is how we
actually implement the invariant that
only the latest instruction should
update the register file. Okay.
Okay. Cool.
Okay. So we now come to the algorithm
itself. Okay. So u we are again ignoring
the fetch stage for now. Okay. That is
step zero. I'm assuming that the fetch
stage is already completed.
So Thomas algorithm handles read after
write with proper stalls for satisfying
true dependencies.
uh but completely eliminates stalling
for right after right hazards and right
after read hazards uh through register
renaming and it also minimizes stalls
through structural hazards through
reservation stations. Okay, of course if
you're out of reservation stations you
have to stop. No, no, no penia there.
Anyway, so what we do, this is after the
fetch stage. Uh we get the next
instruction from the fetch cube. Imagine
there's a fetch cube containing the list
of instructions that have been fetched
already and in program order. So you
fetch the next instruction from the
fetch Q. What I mean as a fetch Q is
after the fetch has happened, they're
all ordered in one queue. This is the
order in which the instructions need to
appear. And that's what I mean by the
fetch Q. Okay, so fetch Q is populated
after the fed stage. Now recall that in
the fetch stage, you actually need to go
to the instruction memory or the
instruction cache and load the
instruction and these instructions are
populated into a fetch tube. That's what
I mean by the fetch here. Okay. Okay. So
we're getting the next instruction from
the fetch tube.
And then uh depending on the type of
instruction, if it's an ad instruction,
obviously it needs to go to the ad unit.
So we we are going to send an
instruction to the right reservation
station. Okay. So of course issue is
only possible if you have a free
reservation station entry. There is no
reservation station entries then you
cannot issue and that stalls the
pipeline. It also blocks future
instructions. if you're not able to find
a reservation station to go and put the
instruction into. Yes.
>> Um so by a fetch Q um so do you mean
that like maybe multiple instructions
might be fetched from memory at once and
stored in some sort of like buffer or
something like that and that's the
queue. Okay. potentially
or even if there's only one instruction
fetch, maybe there are lots of read
after write dependencies and then you're
not able to execute them at a very fast
rate. In which case, it's also useful to
have a queue, right? That's what the
fetch is.
>> Oh, so as in if the issuing is piling
up, you can still be fetching things
from memory. Oh, I see. Yeah, that makes
sense.
>> Okay. So, uh
yeah. So, so this step is clear, right?
So if you have a reservation station 3,
you go ahead and put this instruction
into the reservation station. That
process is called issue. What do we do
as part of the issue stage? Suppose you
have an instruction R3 = R1 + R2. Okay.
So you go and see R1 and R2 which are
the source registers. You go and check
the metadata associated with R1 and R2.
What are you going to see? We're going
to check actually is the reservation
station currently producing a value of
R1 and R2. How would you know that? We
can know that with this metadata entry,
right? So you go and see for R1, does it
have a reservation station currently? It
means that reservation station is
responsible for producing R1. Maybe
there was a previous instruction which
did R1= R5 + R6. Okay. So if there is no
such instruction then where would the
value of R1 be?
What's the correct value to actually
load for R1? Load is the wrong word.
Read from R1 the one in the register
file right because the the value in the
register file R1 would be the right
value. If there is an instruction it's
outstanding which is supposed to be
producing R1 then the value in the
register file would be stayed. you
shouldn't read that value. Okay. So
there there are two possibilities,
right? One, there is no reservation
stations currently producing a source
register value. There is a reservation
station. It's currently producing a
source register value. If it's the
former, then you simply read the value
from the register file and buffer in the
reservation station associated with the
current instruction. We have R3= R1 +
R2. And no instructions are currently
producing a value of R1. You simply go
and read the register file. Read the
value of R1. Put it in the reservation
station associated with the current
instruction R3= R1 + R2. Does it make
sense? Okay. If there is in fact an
instruction which is currently producing
the value of R1, then you need to copy
that reservation station ID.
Right? Because why don't you need that?
Because you need to know what
reservation station are you waiting for.
Okay? Let me make it clear with this
example.
So you have
R1 = R4
* R5. Okay. Now you have a multiply unit
here and you'll have lots of reservation
stations associated with it. And this
instruction currently is in let's say
reservation station Mull one. Okay,
let's call it mull one. And that is
really the one which is in charge of
this instruction. Okay. Uh so with R1
you have this metadata for R1, right?
What will it store? It will store m one
because you know at this point the
reservation station m one is the one
that will actually provide the value for
R1 once it finishes computing the
multiply. Right? Everybody with me here?
Now we have R3 = R1 + R2. Right? So this
is an add instruction. You're going to
the add unit
and you're going to have again a bunch
of reservation stations. Let's say this
is going to a unit here
and let's call it add one. Okay, add one
is the ID of this reservation station.
Now what are you going to do? This is
the process that we just described here.
Now R3 equals R1 + R2.
So you go to R2.
There is no reservation station
currently producing R2 in which case you
simply read the value whatever the value
of R2 is. Maybe it's zero. You copy the
value here.
Okay. What about R1 though? You can't
simply read the value from the register
file. Why? Because there's an
outstanding instruction
represented by mull one which is
currently computing the value of R1. How
would you know this? you'll go to the
rename table and see R1 is currently
mapped to mull one. So and so
essentially you're waiting for mull one
here.
Okay. So when mull one completes
completes its uh multiply operation is
going to broadcast the value that it
computes in the common data bus. It's a
set of wires connecting all of the
reservation stations.
Crucially along with the value is going
to also have a tag what tag the tag mull
one. Okay. So and which means this unit
this reservation station can check the
bus and see oh I see a value here and
that value is tag mull one. By the way
I'm waiting for mull one. So whatever
the value is there in the bus it can
copy it. Let's say it's 10. Okay. So
then it goes ahead and that's 10 * 0
which is zero. Okay, you see how
everything works now.
Okay, so that this is precisely how
issue works, right? So you read operands
from the register file if available or
rename operands uh if pending and you
resolve read after write dependencies
and then in the execute stage
you can go ahead and execute an
instruction when both operants are ready
in the reservation station. Okay, if not
you will actually have to monitor the
common data bus. Okay, you have to
monitor the common data bus for
operands. So in this case for example,
we had to monitor the common data bus uh
for this operand which is mal one.
So when the operants are available, you
snoop the value from the common data bus
and load it into the reservation
station. And then when the values are
available, you go ahead and execute
instructions when both of the operants
are available.
Uh and then finally when the result is
available, you put the result in the
common data bus and uh and the common
data bus is not only connected to the
reservation stations but also to the
register file. So for example when mull
one broadcasts the value that it
computes it also is sent to the register
file and this register file R1 will know
to snoop the value in the common data
bus and here it says mull one okay R1 is
actually waiting for mull one and so
when it sees the value will go and write
it in the register file as well so that
future instructions which read the value
of R1 will see the correct value okay so
you understand the process here at a
high
any questions on how to solulo's
algorithm works in at a high level?
Yeah.
>> Um
>> I noticed um just like information from
different places. I heard the mention of
a rename table but then also the
register file having like pending bits.
So I'm just um I don't know. I guess the
question is the register file contains
like pending bits, right?
>> Uh so the register
>> is it all contained in like this rename
table? I so so the point here is that so
the tomosol it's an algorithm there are
several different implementations right
but the core part is this that every
register file has to have know what
reservation station it is waiting for so
that when it sees the value it can not
copy it otherwise so so of course there
are lots of variants of this algorithm
so in fact like I said today's process
implement a variant of stomos algorithm,
but it's not the exact algorithm. It's
it's a variant of that. And you can
think of several different possible
variants. For example, you can have
multiple different buses, common data
buses. You can have um for example, h
how you as we're going to
uh study now how loads and stores are
actually how do they work. So there are
lots of variants of this core idea. So I
don't know exactly what pending bits
mean here but maybe you can discuss this
often. Okay.
Okay. Any questions on the core
algorithm here?
Okay. One thing I actually ignore are
loads and stores. Now loads and stores
also have reservation stations. Right?
So they are the buffers associated with
the load store unit.
Now
loads and stores are maintained in a
separate queue. The queue of reservation
stations and you actually enforce the
queue uh Q structure and it actually
preserves the program order. Uh what I
mean by this is that suppose you have a
bunch of loads and stores. Okay, in
program order. So maybe you have store
one, load two, load three, store four,
load five, some bunch of stores and
loads in that order. Okay. So they're
all issued to the reservation stations
associated with the load store unit also
called the load store queue. Okay. Now
the way the load store Q works it works
quite conservatively in the sense that
only when a load or a store instruction
gets to the head of the queue it
performs. So there is no concurrency
there right. So, so here we have lots of
out of order processing with register uh
like with things like adds and
multiplies and subtracts right as as as
soon as one of the operants are ready we
want to go ahead and aggressively try
and execute these instructions but load
store instructions in the most
conservative case I'll come back to that
is a bit special we don't have
concurrency there right so so we our
more accurate statement is we need to be
very careful when we concurrently
perform loads and stores. Right? So, so
in the most simplest implementation with
the tomosulus algorithm, you have a load
store q things are ordered. You do it
one by one in program order. Okay. Why
might this be? Why do you think I'm
making this restriction?
Yes.
professions take the same amount of time
compared to add and multiply and lower
take.
>> Yes. So you have the chance of
overriting things.
>> Overriting uh yeah I think you are
thinking along the right direction here
right? So overriting is the key word
here. You need to be careful about the
order in which loads and stores have
actually performed and let me explain
that with an example.
Actually,
do you have a different answer? You you
also raised your hand, right?
>> I was going to say that maybe the reason
why we preserve program order and don't
run them immediately is because we want
to preserve whatever memory addresses
we're dealing with.
>> Yeah.
>> Yes.
>> I'm going to explain.
This is something that I relate very
closely to question 2.3.
Okay.
>> Can you use blackh?
>> Yeah.
So suppose you have
this instruction which is a store
instruction. Okay, I'm just not I'm just
going to use some notation here instead
of the store instruction. So let's say
it computes it needs to store to an
effective address, isn't it? So let's
say that is R1 +,000. Okay, it needs to
store to this address R1 +,000. Okay.
And then
um the value that is stored is what? R
E.
Let's say earlier you have a multiply
instruction
which is actually computing the value of
R1. R1 equals let's say R R 5 * R six
and let's say this takes maybe 27 it
takes some time because it's a multiply
instruction. Okay. So this is a store
instruction. Then you have a load
instruction. Okay. This load instruction
is going to write read into register uh
R8
and it's going to
use R0
plus,000. Typically in all of these
process R0 has a value zero. It's a zero
register. So
if you think about performing these
instructions out of order, you know the
address of this load instruction very
fast because it's zero and it's th000.
You know that this instruction is going
to load from address. Everybody with me
here? Whereas this instruction is going
to need the value of R1 to compute its
address. But R1 is a prior multiply is
produced by a prior multiply
instruction. That's going to take some
time. Okay. So, it is tempting to go
ahead and perform this load out of order
so to speak. Right? That's what we've
been doing with to algorithm. Right? If
you see at the heart of it for register
operations, we've been trying to
aggressively perform instructions out of
order. Right? You see the problem here
that can happen
if you perform this load out of order.
You see the problem that can happen.
What's the problem that can happen?
>> Right after right
>> right after right
uh can we be more specific?
>> Yes.
>> The value of R1 ends up being zero.
>> If it ends up being zero, what will
happen?
>> It'll be the same memory. So you might
end up reading from that address before
you wrote to it.
>> Exactly. So let's say that the
computation here actually turns out to
be R1 equals Z. Okay, we don't we will
only know that after 20 cycles but if
you went ahead and aggressively loaded
from this address 1 th00and maybe you
read the initial value that is there in
memory. But what value should this load
actually return? It should return the
value written by the store instruction,
isn't it?
So really this load instruction should
happen after the store instruction but
because of the way the addresses are
computed we wouldn't know this right so
that's why we have to be conservative
conservative here and perform these
instructions in programmable
the key part here is that we've been
able to analyze read after write right
after right read after
critically read after write dependencies
using just the identities of the
registers that is how we are able to
execute Thomas's algorithm out of order
right but if things
the loads and store instructions yes we
also need to honor read after write
dependencies but the dependencies are
through memory addresses in this case
there is a read after write dependency
because it turns out that R1 + 1,000 is
equal to R0 +,000. But we would not know
this until we know the value of R1. We
wouldn't know this without executing
this instruction.
You see the problem here? Whereas with
just registers by simply looking at the
the instruction itself, we can figure
out what are the important read after
write dependencies that we need to
honor. Right? So everybody grock this
situation here.
>> Yes.
>> Is it possible for like the algorithm to
determine that like the addresses don't
alias?
>> Yes.
>> That's called memory disambiguation
unit. Right. So, so in today's process
that's what they do. Uh just like a
branch predictor the memory
disambiguation unit. What it does is
given for example this uh load
instruction
address is available it's ready to go.
So it will just predict is this load
actually probably going to read from a
outstanding store which has not yet
executed. So it's going to say yes or no
just like a branch predictor. If the
answer is no go ahead and perform the
door. Okay. Later it's you going to have
to check it's true. Yeah.
Okay. So we understand this. Now what
about this? So for in the way that
Thomas's algorithm I've introduced
actually it permits no branch prediction
in the way I've introduced this
algorithm right in reality of course
Tommos algorithm of course needs branch
prediction but the way I introduced the
algorithm test until this point
tomos algorithm requires that preceding
branches must resolve before an
instruction can execute. So it permits
no branch prediction. So let's take a
small break. Why don't you think about
this? Why is this true? Okay. So let's
assume
at uh 250
[Music]
questions 2.3 on the home. So we're
pretty restricted on like how we can
alter the loop. Like we have
instructions in the top.
>> Yes. Um the instructions specifically
say that you can't change the number of
>> instructions
instruction
[Music]
I wanted
[Music]
[Music]
I mean I'm
[Music]
Just a thought. But but in general
[Music]
>> are we
[Music]
registering?
I was really
into my
last
[Music]
like
this stage to LU and this stage to LU
and from B2 ID by radio file. Um but
in dis
to
um to pull out some information from any
to the ID.
>> Yeah, we can do that. Um autism
um why we um sometimes
>> if you want to do branches for example
you might want yeah
>> you can do that
>> I just find that try to think about how
>> yeah
and um
this way for example for one
um we can we may also move the branch
operation to the United States.
>> Oh
uh resolving the branch
>> yeah it's possible it's possible
>> it's possible to resolve a branch at the
exe stage.
>> Yeah
>> how can
it's all about uh the pipeline balance
right? Yeah.
>> So how much hardware are you when you
resolve something stage?
>> So it's just tradeoffs.
>> So but for the questions in the homework
you need to adhere to exactly what the
assumption asks you.
>> So if we say that the the branch is
resolved in the memory stage then that's
what you need to use and if the branches
are resolved in the decode stage. Yeah.
But you're absolutely right that you can
think about a pipeline design where
you're resolving branches in the AC.
Yes.
>> Actually
stored to um manage the IT state and the
state.
>> Okay.
>> Just checking for
[Music]
>> Yeah. So can I take other question after
the class?
>> Yeah. Okay.
[Music]
Okay.
>> Right. So, yeah. Why what do you think?
Why do you think we need this assumption
that we can't really permit a branch
predictor with Tom Sulos algorithm as
I've discussed it?
Yes.
>> Uh it'd be really hard to flush
everything.
>> Yes. So imagine that in pipelining like
even in the s simple pipeline what
you're working on with assignments,
right? When you predict branch not
taken. Okay. So but you eventually find
at the end of the memory stage that the
branch has to be taken. Okay. It means
that
some bad stuff happened, right? You need
to un all of that. You need to nullify
all of that and you need to know what to
nullify.
And if you think about it with
pipelining,
one of the things that we relied on
there was we exactly knew what to
nullify
everything after the branch instruction
because things were executed in order.
We knew exactly what to nullify. But
with Thomas algorithm like things are
executed out of order. Maybe
you manage to perform something
in the not taken path and that is
published in the common data bus
already. It's done. Right? So that would
be bad, right? How would you nullify
that? It's very hard to nullify an
instruction which executed in the wrong
part and finished executing published it
results in the common database.
So intuitively what do you think is the
answer? How do we actually make tomosulo
agree with prediction?
We want to right and we're going to soon
discuss a technique for that in the next
lecture. But intuitively, what do you
think? What we what do we what what
should we do to make Thomas work with
transition? Yes.
>> Would we use even more buffers and put a
buffer before something gets posted to
the common data bus?
>> Yes, absolutely.
>> More buffers.
So uh what we need to do is once an
instruction is computed we need to be
able to buffer the resource in the
reservation stations or something like
that until we know that actually that
instruction is not speculated right it's
in fact the right instruction to to have
actually fetched until we know that fact
we need to be able to buffer the results
and not publish it in the common
database isn't So in other words, what
we really need is that's why we say in
order issue out of order execute in
order commit right we need that in order
commit which came for free with
pipelining that let us exactly squash
the instructions in the wrong path here
without the in order commit as in the
way that I've explained tomosulos
instructions actually complete out of
order potentially Right? So without that
in order completion it's extremely hard
to support branch prediction. So we're
going to discuss that in the next
lecture but for now like assume that we
are not supporting branch prediction.
Okay. So so uh this is just uh uh the
way all of these uh work. So we talked
about the instruction Q which comes
after the fetch unit. So you have uh
like if you have floating point add a
unit you have reservation stations
associated with it. You have a floating
point multiplying it you have
reservation stations associated with it.
You have a memory unit you have
reservation stations also known as load
store buffers associated with it. Okay.
All of these units are connected uh by
the common data bus. Okay.
Let's look at an example.
We already covered this. Oh, one more
thing. Common data bus. There was a
question also of the common data bus.
How does the bus work? As you as we have
just described, it's a set of wires that
all of the uh reservation stations have
access to. They can insert things into
the bus. They can look at the bus to see
what is there currently, whether it's
relevant to me. Okay. So every
transaction in the bus is identified by
the reservation station tag. Okay. So
that other reservation stations know
whether what is there in the bus is
relevant to them. Okay. So great. One
other thing is yes the bus can become a
bottleneck here right when an
instruction is ready which is finished
executing it needs to put its result in
the bus. That's why it's very important
that the whole bus transaction is kept
extremely low latency like one cycle.
It's also important that when an
instruction
uh in a reservation station is waiting
on let's say a value that's being
computed maybe R0 okay multiple
instructions could be waiting on R0. In
other words, multiple reservation
stations could be waiting on one
reservation station. So one of the nice
things with Thomas algorithm is when you
see the value in the bus multiple
reservation stations can potentially
snoop on the the value in the bus and
away and go ahead and do the execution.
Okay. So again the common data bus is
the avenue through which we do
forwarding right things are forwarded
between uh reference stations and that's
how we achieve forwarding right recall
that what is essentially forwarding is a
way that we can transfer values from an
instruction that is computing to its
other instruction which needs that value
without going through the register file.
That's what forwarding is right and that
is what we have accomplished using the
common data bus. Oh one other thing as
soon as an instruction
puts the reservation station puts let's
say a value in the common data bus
uh
there might be some other reservation
station waiting on it. So typically you
can both write and read in the same
clock cycle. Okay, just like how we saw
in one of the forwarding cases, right?
The write and the read can happen in the
same clock cycle so that you avoid the
case where you see the value in the bus.
Okay, you then take it and then start
execution in the next cycle. Typically,
you can actually read the value uh in
the same cycle so that you can start the
execution in the next one. You don't
need an extra cycle. That's all I'm
trying to say. Yes. So if you have
multiple functional units that finish
executing at the same time, how do you
decide which one gets to control the
bus?
>> Good point. So in that case, uh it
depends on uh if there's only one bus,
>> yeah,
>> you have to compete for the bus.
>> Only one of them gets access to the bus.
But you one might imagine right in let's
say supercalar processors, you might
have let's say four instructions
completing at the same time. Typically
you have four buses right so so so that
all of these instructions can like get
access to the bus so in today's process
you have multiple common data buses
>> so uh tennos seems like just from a high
level perspective and especially from
somebody who doesn't have as much
practical hardware experience that um
employing this sort of architectural
technique takes up a lot of surface area
on the chip Does that mean that we're
typically sacrificing
for example adding another alou for
something like a supercalar technique?
>> Yeah, great question.
>> Right. So, so typically
uh
and we're going to look at this in
detail in the next lecture. So one thing
that the the short story is here is that
adding function units, adding these uh
these metadata units like a rename
table,
>> these reservation stations,
>> they are not a big problem.
>> Oh,
>> right. So the problem is where when you
have like multiple instructions trying
to issue at the same time and need to
check all of the dependencies that is
what gets quadratic and that is what
limits things. So usually like because
of most loss bounty the area overhead of
like adding more reservation stations
adding more buses adding more uh rename
tables they are not
>> that that big of a problem
>> I'll come back to this
>> okay
>> okay so let's now look at the u oh by
the way so there is an algorithm here
explaining exactly what we've discussed
I'm just going to skip that right So, so
hopefully you can just read the
algorithm and you can like make sense of
it. If not, please ask in PJ and I'll
answer questions. Okay, so because I
want to go ahead with an example. Okay,
so let's say uh for this example, let's
say we have every load instruction is
going to take two cycles, right? Meaning
we talking about the execute stage of
the load instruction where we go and
fetch the value from memory. That's
going to take two cycles. The execute
stage of uh add and subtract are going
to take two cycles. Again, these are
floatingoint adds and floating point
subtract instructions. We're going to
assume it's going to take two cycles,
right? This is for actually performing
the addition and the subtraction. The
execute stage of the multiply
instruction is going to take 10 cycles
and execute stage of the divide is going
to take 40 cycles. Okay, these are the
assumptions
>> and all of those are also floating point
instruction.
>> Yes, all of these are protein points and
then uh let's look at a code sequence
here. Okay. So if you quickly look at
the code sequence, you have two loads.
Uh any dependencies between the first
two loads, read after right, right,
right after right, right after read. Any
dependencies between the first two
loads?
>> None. Okay.
>> There's
>> Yes.
>> F2.
>> Sorry.
>> Load into F2.
>> Yes.
>> And multiply.
Yeah.
>> So there's one.
>> Yeah. So I'm Yeah, exactly. So I'm just
I was just asking about the first two
loads. Oh okay. So when we get to the
multiply instruction we can see that
there's a read after write dependency
right. So it's the value f2 that is
required by the multiply instruction is
actually produced by the the second load
instruction. Okay. So the third
instruction
yes again there's a read after write
dependency. The subtract instruction
reads the value produced by uh by the
first two loads. Okay. And then we have
the divide instruction. The divide
instruction
is actually dependent on the multiply
instruction. Okay. Okay. So, and then
finally the add instruction is dependent
on the the subtract instruction. Okay.
So, we're going to roughly see how this
is going to work out with the tomosulos,
right? Like we're going to cycle by
cycle think about how each of these
instructions are going to execute with
tomosulo. as a rough indication what do
we think everything should work out to
the first two load instructions can be
performed in parallel say it's going to
take two two so each of them is going to
take two cycles because of the right
back and all of that it's going to take
four five six cycles the first two loads
okay the multiply instruction is going
to be dependent on the load so after
five cycles the multiply instruction can
start it's going to take about 10 cycles
so it takes rough roughly 16 cycles to
complete. Okay. Mhm.
>> So the subtract instruction
um is going to be a function of the
first two loads. It's actually not in
the critical path, right? So so the
multiply instruction is going to take
longer. So if you think about the divide
instruction, it's going to wait for the
multiply instruction. Okay. So so it's
multiply instruction took around 16
cycles. So divide is going to take 40
more. So 56 cycles
and then that is going to determine the
critical path right the add is not
dependent on the the the the divide
instruction. So roughly we expect this
to take about 56 cycles in the best case
right 56 57 cycles. Let's see if
tomosulo can actually deliver that right
this is what an ideal thing can do right
let's see if tomosuler can achieve that.
So here we are assuming that there are
uh like we have three load store
reservation stations for the for the
loads. So because we don't have stores
here, we have three and then we have
three reservation stations for
performing addition and subtractions. We
have two reservation stations for
performing multiplication and division.
Both divide and multiply are going to
use that reservation station. Okay. So
one thing is from just the look of
things we have enough reservation
stations for these instructions. There
not going to be any stops, right?
because we have enough reservation
stations. Right? Everybody with me here?
>> So, and finally uh
with every reservation station we have
uh this op means the operation performed
here. Is it for this reservation station
is it an add or a subtract? Then we have
uh VJ and VK. So because there are two
operants for every uh like an
instruction VJ and VK represent the
values right. So remember that in the
reservation station we can also buffer
values okay qj and qk refer to the
reservation stations that they may
potentially be dependent on. Okay like
we saw in the example there right so and
and this is the rename table. So for
every floating point register we are
going to store the reservation station
that produces that val that value. Okay,
everybody understand the whole structure
here? Any questions on the structure?
Okay, so now let's go and simulate each
of these operations.
Um and by the way for the load unit and
for all of the reservation stations we
have a bit called busy whether that
reservation station is actually being
used now and for the load unit we also
have the address which is computed.
Okay.
Okay. So the first load instruction is
issued in cycle one. Okay. We going to
issue it cycle one. So it's a load
instruction. So it has to go to one of
the reservation stations that load one.
It's the reservation station associated
with the load instruction. So its
operants are 34 and R2. Nobody's
producing R2. So we can compute the
value and buffer it as part of the
address there. Keep it ready there.
Okay.
And then the second load instruction in
cycle two. What happens? Uh yes, the
first load actually performs starts
performing the load but it takes two
cycles, right? That's what we've been
assuming. So it would have completed the
load in cycle three. Okay. So so in
cycle two it it's not finished
completion. But what can happen is in
the cycle two the second load can be
issued. The second load is issued it
occupies this reservation station load
two and it gets the value of the
register R3 and buffers it as part of
its address. Okay. What happens in cycle
three?
load one would have finished executing.
Okay. So uh and so it would have uh uh
completed loading from memory but it has
still not published it in the common
data bus. That's what happens in the
last stage write result stage. Okay. So
meanwhile the second load has started
executing but it has not completed it
yet. It would have it will complete in
cycle four.
uh but the next instruction multiply is
issued in cycle three. So it's a
multiply instruction. So we go ahead and
occupy this reservation station here.
Okay, this mult one reservation station.
What are the operants source operants of
this multiply instruction f_sub_2 and
f_sub_4. Oh, f_sub_2 is actually
currently being computed by load two.
So what we do is go and write load two
here because this reservation station
mult one is dependent on this
reservation station load two right
everything make sense so the mult
multiply instruction also has another
operand f4 you go to f4 and see actually
nobody is currently producing f4 which
means you can go ahead and read the
value from the f4 register so in fact we
go and read the value from the f4
register and buffer for it here. Okay,
make sense. So everybody understand what
entails issuing this multiply
instruction. This all happens in cycle
three. Okay,
what do we do in cycle four? So in cycle
four
um
[Music]
in cycle four we go ahead and write the
results of the first instruction.
Um actually
is anybody waiting on load one? Nobody
is waiting on load one except this
register itself. That's all the register
F6 is waiting for the unit. So
when load completes
the load actually completes the value is
obtained from memory and published in
the common data bus and it's written to
F6 m of A1 is imagine that's the value
that's loaded it's memory of some
address A1 okay so and then what do we
do in cycle 4
the second load actually completes its
execution it loads from memory but it is
not yet published in the common datab
bus. So that's why we put four in the
execute phase, execute completion phase
of the second row. And then we also can
issue the subtract instruction, isn't
it? So what do we do? Subtract
instruction has source oper.
F6
uh
oh by the way, so so here's the thing.
So uh
yeah so this F6 in cycle 4 is published
in the common data bus.
It's written to the register file. At
the same time in cycle 4 subtract
instruction is also issued. Okay.
You find out in cycle 4 that subtract
instruction has the source operands f6
and f_sub_2.
F_sub_2 is currently being produced by
load two. So we go ahead and keep track
of load two.
But what about F6? This is where the
forwarding in the same cycle comes into
play. Right? It is as if
both the subtract instruction meets the
value of F6. And in cycle four, the
first load instruction produces the
value of X F6 via the common data bus.
Both of this can happen simultaneously
right. So that is why in cycle four f6
is published the value produced by the
load instruction is published in the
common data bus and is also loaded by
the subtract instruction so that it
loads the value and buffers it as one of
the operants. The value that's written
to f6 is also available here in the same
cycle. Do you see this point? So this is
what I mean by the common data bus
actually being like permitting the
writing and the reading of the registers
in the same cycle very similar to what
we discussed in the pipeline part.
Right? So in cycle four the value
produced by the load instruction F6 is
written as well as the same values
allowed to be read by the subtract
instruction. Okay.
>> Question
is the same cycle. Yes.
>> Does it divide the cycle into
>> That's one way to implement this. Yes.
>> Is it possible to do it?
Let's say not possible for now. Okay. So
because logically I think that's what it
means it'll end up meaning. So imagine
so for now we can assume that it's like
how we discussed pipelining. The first
half of the cycle where is where the
rights happen. The sec second half of
the cycle is where the reads happen.
Good question.
Okay. So this is cycle four. Uh in cycle
five uh what happens? The load
instruction, the second load instruction
is able to forward its results. Is
anybody waiting for the the second load
instruction? The second load instruction
is load two. Oh, there are two
instructions waiting for load two. The
multiply instruction as well as the
subtract instruction. Okay. So which
means in cycle five
value is sent in the common data bus and
also buffer
in these two places. Right?
So at this point we have a count on this
multiply instruction can start executing
from the next cycle. So its result will
be available. It will finish computing
in cycle 15. That's what this 10 means.
In 10 more cycles, the result it'll
finish completion. Likewise, this
subtract instruction will finish uh
computing in two more cycles.
And by the way, in cycle five, we are
also able to issue the divide
instruction. Uh the divide instruction
has is a function of f0 and f_sub_6. F0
is currently being produced by the
multiply instruction. We wait for that.
F6
this value is available. We copy and
buffer that. Okay. So multiply
instruction has to uh so the divide
instruction has to wait for the
multiply.
In cycle six the add instruction issues.
Okay. So we go and uh uh we go and issue
it here. F8 and F_sub_2. Uh F8 is
currently being produced by the the
subtract instruction. We copy it here.
F_sub_2 is available. So we copy the
value here. Okay. So meanwhile in cycle
six, we have one more cycle to go for
the subtract instruction to complete.
Nine more cycles to go for the multiply
instruction to complete. Okay.
So cycle seven the subtract instruction
actually completes right. So we had one
more cycle it completes. Uh and then for
the multiply instruction we have eight
more cycles to go to complete.
In cycle 9 the subtract instruction is
able to send its value in the common
data bus. Uh who is actually waiting for
the subtract instruction currently? is
anything waiting for the subtract
instruction.
>> The add instruction.
>> The add instruction. So in cycle 8, the
add instruction is able to get the value
and buffer it here. N minus m is the
value produced by the subtract
instruction. Okay. Okay. Meanwhile there
are at this point though you can start
performing the add instruction. Two more
cycles to go and the multiply has seven
more cycles to go. We are counting down.
So cycle 9, we have one more cycle to go
for the add instruction. Six more cycles
to go. Cycle 10, the add instruction
completes, right? That's why we write it
here.
And then in cycle 11,
it publishes it result in the common
data bus. It's also reflected in the
register file F6. Right? So that's why
Mus whatever m it's the actual value
that it computes. Uh so uh meanwhile uh
the division instruction still has uh
sorry the multiply instruction still has
four cycles to go.
So
in cycle 15 it completes. In cycle 16 it
publishes its result in the common data
bus. At this point, the divide
instruction which is waiting for the
multiply instruction gets its value into
the buffer and so it can start executing
in the next cycle. So that's why it has
40 more cycles to go and so in cycle 56
it's able to complete which is 40 cycles
after this and cycle 57 everything
completes. Right? Just like as we
thought this is kind of close to ideal
right because there are no stalls that
are superfluous here. We are only
stalling for the right things. So
assuming we have enough reservation
stations, we'd be fine with Tomudo,
right? Model of branches and memory,
which we'll come back to in the next
lecture. Okay, thank you.