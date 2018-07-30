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
* Run some iterations of HWPL-driven QP
* Optionally run RippleFPGA partitioning (e.g. this doesn't work on MAX V because it's way too small. It's unclear for iCE40 which doesn't have as much directional routing bias.)
* Run some more? iterations of HWPL-driven QP.
* Run some iterations of congestion-driven QP.
* Fully place RAM/DSP now (UTPlaceF)
* Finish congestion-driven global placement (skip RippleFPGA soft BLE packing -- intuition tells me this probably doesn't do much)
* CLB packing
* Detail placement
* Final assignment

Need to further research (directly-relevant algorithm details):
* How does QP rough legalization work? Something about bins and density...
* How does detail placement work? (bipartite matching, independent set matching, etc)
* What does the RippleFPGA deferred slot assignment actually gain us?
* Can MAX V or iCE40 or other "simple LUT4" FPGAs skip complexity in CLB packing?
* How do UTPlaceF/RippleFPGA CLB packing differ?
* Carry chains? HeAP talks about them, RippleFPGA/UTPlaceF don't

Need to further research (FPGA architectures):
* Investigate global net structures in real FPGAs
* How will other "special" blocks affect our PAR algorithm? (e.g. PLLs, ADCs, user flash, etc.)

Need to further research (general abstract CS topics):
* Efficient cache-optimized data structures, especially for multithreading
* General research on "modern" approaches to multithreaded algorithms
* Brief research on numerical stability (can we just use integers/fixed?)

Features for minimum viable product:
* Quadratic programming core engine, HWPL only
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
