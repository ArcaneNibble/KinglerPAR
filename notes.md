The overall approach will probably use a hybrid of the [RippleFPGA](https://pdfs.semanticscholar.org/c1c9/43c047e9d834c8ac487ec6d5292485546744.pdf)
and [UTPlaceF](http://wuxili.net/pdf/TCAD17_UTPlaceF_Li.pdf) algorithms for combined packing and placing. Early experiments
with VPR demonstrated to me that an artificial barrier between these stages is not helpful. These algorithms all use
quadratic programming. These two algorithms are fairly similar to each other.

For now, routing will just use whatever algorithm is "typical" (Pathfinder?). I may end up with egg on my face, but it
seems that this is "just" an informed search problem. Some ideas that _may_ end up enhancing this stage:
* A* or other heuristics (XXX can we use inadmissible heuristics?)
* Beam search

Miscellaneous papers on how we "got to" analytic placers from SA placers:
* http://janders.eecg.toronto.edu/1387/readings/marcelfpl12.pdf
* http://www.ece.umich.edu/cse/awards/pdfs/iccad10-simpl.pdf

Proposed overall flow for KinglerPAR:
* Read yosys netlist
* Cell munging (general purpose and arch-specific)
    * Types: IO, LUT, FF, RAM/DSP/large-in-fabric-movable-blocks, fixed specials (blocks like PLLs, flash, ADC), movable specials?
* Arch-specific logic propagates around some constraints now (e.g. for special blocks or for promoting globals (XXX should this be generic?))
    * XXX should we have some logic for handling clock domains? Is this ever useful?
* Arch-specific greedy packing (e.g. for RAM/DSP/special)
* Arch-specific greedy placement (e.g. for specials)
* Build HeAP-style superblocks for carry chains
    * Warn if carry chains are huge?
    * XXX can we auto-split them? This is hard; involves injecting feedthrough cells, which affects legalization
    * XXX UNTESTEED IDEA: Can we twiddle net weights to encourage packing of carry chains and BLEs and other similar inside-CLB features? Do we need to?
* XXX can QP work with _zero_ constraints? If not, assign non-user-constrained IOs uniformly so as to avoid getting a giant blob of 100% overlapped cells (XXX what about clocks and stuff?).
* Distribute all cells uniformly in fabric (required to seed B2B).
* Run some iterations of HPWL-driven QP (3 iterations? just to get something that vaguely looks right. We need this for the next step)
* Optionally run RippleFPGA partitioning (e.g. this doesn't work on MAX V because it's way too small. It's unclear for iCE40 which doesn't have as much directional routing bias.)
* Run HPWL-driven QP to a convergence criteria.
    * QP legalization will probably be UTPlaceF/POLAR-like for the spreading phase
* Arch-specific hook point (e.g. for clocks). This hook point has enough information to e.g. assign quadrants but is also early enough that there is room to "recover from" constraints such as quadrants.
* Run some iterations of congestion-driven QP.
    * Fully legalize RAM/DSP now, but don't lock it in yet (UTPlaceF)
    * Will probably use RippleFPGA-style area inflating and congestion estimation. Definitely won't import NTUPlace.
* Finish congestion-driven global placement (skip RippleFPGA soft BLE packing -- intuition tells me this probably doesn't do much)
* CLB packing with full legalization (XXX if it fails do we run QP again?)
* Arch-specific hook point (e.g. for clocks). This is an "extra" hook point that I'm not currently sure how to use. It is after CLB packing, so that might be useful.
* Save a design checkpoint now; possibly rewrite data structures ((somewhat) packed netlist)
* Detail placement (RippleFPGA-style with alignment)
* Final slot assignment
* Save a design checkpoint now; possibly rewrite data structures (packed+placed netlist)
* Arch-specific hook point (e.g. for clocks). This is intended for code to actually implement clock routing.
* XXX research how to implement routing algorithms
    * It seems we should be able to reenter pack+place and adjust cell density if we save enough state
* Save a design checkpoint now; possibly rewrite data structures (fully-PARed result)
    * XXX we probably want a way to back-annotate this
* Convert to arch-specific data structures and write bitstream

Need to further research (directly-relevant algorithm details):
* What does the RippleFPGA deferred slot assignment actually gain us?
    * Simplicity, but it seems we need to do per-arch manual simplifying of legalization rules to take advantage of this
* Can MAX V or iCE40 or other "simple LUT4" FPGAs skip complexity in CLB packing?
    * See above
* How do UTPlaceF/RippleFPGA CLB packing differ?

Need to further research (FPGA architectures):
* Investigate global net structures in real FPGAs
* How will other "special" blocks affect our PAR algorithm? (e.g. PLLs, ADCs, user flash, etc.)
* Does any FPGA have "fracturable" RAM/DSP blocks? (Answer: YES) What to do about those? (Probably just treat them similarly and apply legalization etc. except with fixing their locations earlier)

Need to further research (general abstract CS topics):
* Efficient cache-optimized data structures, especially for multithreading
* General research on "modern" approaches to multithreaded algorithms
* Brief research on numerical stability (can we just use integers/fixed?)

Features for minimum viable product:
* Quadratic programming core engine, HPWL only
* CLB packing
* Detail placement
* Final assignment
* MVP will target iCE40 + MAX V

Post-MVP top priorities:
* Carry chains (can probably ignore for MVP as long as code gets plumbed properly for them)
* RAM/DSP (iCE40 has, MAX V doesn't)
* LUT6/ALM (question: can we understand X-ray by then? If not then we can consider working on S6 or Cyc10GX)
* Congestion-driven placement (does this need to be higher priority?)
* Partitioning

XXX additional notes:
* Need to ensure we don't pessimize LUT6/ALM architectures (everything I know well are "simple" LUT4)
* VPR XML packing descriptions for AAPack are ridiculously overengineered -- KinglerPAR will just use code instead of declarative data
    * How to make this code sufficiently reusable? MAX V is probably simpler than iCE40 is simpler than ECP5 is simpler than 7
* G-cell congestion estimates seem to be designed around "Xilinx-style" big switch box architectures, what happens on Altera-style architectures?
* Can we "dump" data back into VPR for routing initially?

Architectures:
* MAX V -- "Altera-style", LUT4
* iCE40 -- "Altera-style", LUT4, RAM
* 7 -- "Xilinx-style", LUT6/ALM, RAM+DSP
* ECP5 -- "Xilinx-style", LUT4, RAM+DSP
* MAX10 (future) -- "Altera-style", LUT4, RAM+mult
* Cyc10LP (future) -- "Altera-style", LUT4, RAM+mult
* Cyc10GX (future) -- "Altera-style", LUT6/ALM, RAM+DSP

Unsorted references:
* http://www.cse.cuhk.edu.hk/~byu/papers/C54-ICCAD2016-RippleFPGA-slides.pdf
* https://chengengjie.github.io/papers/C2-ICCAD16-RippleFPGA.pdf
* https://pdfs.semanticscholar.org/c1c9/43c047e9d834c8ac487ec6d5292485546744.pdf
* http://wuxili.net/pdf/TCAD17_UTPlaceF_Li.pdf
* http://sci-hub.tw/https://ieeexplore.ieee.org/document/6691143/
* http://sci-hub.tw/https://ieeexplore.ieee.org/document/1560039/
* http://www.cse.cuhk.edu.hk/~fyyoung/paper/iccad11_ripple.pdf
* http://appsrv.cse.cuhk.edu.hk/~jkuang/pdf/ripple2.pdf
* http://janders.eecg.toronto.edu/1387/readings/marcelfpl12.pdf
* http://www.ece.umich.edu/cse/awards/pdfs/iccad10-simpl.pdf
* https://atrium.lib.uoguelph.ca/xmlui/bitstream/handle/10214/12985/Abuowaimer_Ziad_201805_PhD.pdf?sequence=5&isAllowed=y
