# VSDBabySoC Physical Design – Using OpenROAD

This README documents the complete, step‑by‑step flow I followed to complete **Task 13** of the VSD Physical Design workshop using **OpenROAD Flow Scripts (ORFS)**.

It includes **all explanations**, **all fixes**, and **all commands** needed to successfully run the VSDBabySoC design from RTL → GDS.


---

# 📘 Overview

The goal of this task is to take the **VSDBabySoC RTL**, integrate it inside **OpenROAD Flow Scripts**, and then run through:

* Synthesis (Yosys)
* Floorplan
* Placement
* Clock Tree Synthesis
* Global Routing
* Detailed Routing
* GDS Export

Because VSDBabySoC contains **custom analog blocks** (DAC, PLL), the `.lib` files are provided. However, they need **modifications** to be compatible with OpenROAD.

This README explains everything cleanly.

---

# 🗂 Repository Structure


```
OpenRoad-VSDBabySOC-Physical-Design/
│
├── designs/
│   └── sky130hd/
│       └── vsdbabysoc/
│           ├── config.mk
│           ├── src/
│           │   └── vsdbabysoc.v
│           └── lib/
│               ├── avsddac.lib
│               └── avsdpll.lib     ← (fixed version)
│
├── assets/      
└── README.md
```

---

# 🚀 Step 1 – Clone OpenROAD Flow Scripts

```bash
git clone https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts.git
cd OpenROAD-flow-scripts
```

Install dependencies:

```bash
sudo ./etc/DependencyInstaller.sh
```

Build ORFS tools:

```bash
./build_openroad.sh --local
```

This builds:

* Yosys
* OpenROAD
* TritonRoute
* KLayout export

---

# 🧩 Step 2 – Add VSDBabySoC to ORFS

Inside:

```
flow/designs/sky130hd/
```

create a folder:

```
vsdbabysoc/
```

### Add Files:

```
vsdbabysoc/
│── config.mk
│── src/
│   └── vsdbabysoc.v
└── lib/
    ├── avsddac.lib
    └── avsdpll.lib
```

### ⚠ IMPORTANT: Fixing the Liberty Files

The PLL library (`avsdpll.lib`) originally had **invalid `//` comments**, which Liberty format does NOT support.

Liberty supports only:

```c
/* comment */
```

So I fixed the file by converting all `//` lines to proper block comments.
This prevents the `STA-0164` syntax error during floorplan.

---

### Config file 
Now, create a config.mk file whose contents are shown below:

```shell
export DESIGN_NICKNAME = vsdbabysoc
export DESIGN_NAME = vsdbabysoc
export PLATFORM    = sky130hd

# export VERILOG_FILES_BLACKBOX = $(DESIGN_HOME)/src/$(DESIGN_NICKNAME)/IPs/*.v
# export VERILOG_FILES = $(sort $(wildcard $(DESIGN_HOME)/src/$(DESIGN_NICKNAME)/*.v))
# Explicitly list the Verilog files for synthesis
export VERILOG_FILES = $(DESIGN_HOME)/src/$(DESIGN_NICKNAME)/vsdbabysoc.v \
                       $(DESIGN_HOME)/src/$(DESIGN_NICKNAME)/rvmyth.v \
                       $(DESIGN_HOME)/src/$(DESIGN_NICKNAME)/clk_gate.v

export SDC_FILE      = $(DESIGN_HOME)/$(PLATFORM)/$(DESIGN_NICKNAME)/vsdbabysoc_synthesis.sdc

export vsdbabysoc_DIR = $(DESIGN_HOME)/$(PLATFORM)/$(DESIGN_NICKNAME)

export VERILOG_INCLUDE_DIRS = $(wildcard $(vsdbabysoc_DIR)/include/)
# export SDC_FILE      = $(wildcard $(vsdbabysoc_DIR)/sdc/*.sdc)
export ADDITIONAL_GDS  = $(wildcard $(vsdbabysoc_DIR)/gds/*.gds.gz)
export ADDITIONAL_LEFS  = $(wildcard $(vsdbabysoc_DIR)/lef/*.lef)
export ADDITIONAL_LIBS = $(wildcard $(vsdbabysoc_DIR)/lib/*.lib)
# export PDN_TCL = $(DESIGN_HOME)/$(PLATFORM)/$(DESIGN_NICKNAME)/pdn.tcl

# Clock Configuration (vsdbabysoc specific)
# export CLOCK_PERIOD = 20.0
export CLOCK_PORT = CLK
export CLOCK_NET = $(CLOCK_PORT)

# Floorplanning Configuration (vsdbabysoc specific)
export FP_PIN_ORDER_CFG = $(wildcard $(DESIGN_DIR)/pin_order.cfg)
# export FP_SIZING = absolute

export DIE_AREA   = 0 0 1600 1600
export CORE_AREA  = 20 20 1590 1590

# Placement Configuration (vsdbabysoc specific)
export MACRO_PLACEMENT_CFG = $(wildcard $(DESIGN_DIR)/macro.cfg)
export PLACE_PINS_ARGS = -exclude left:0-600 -exclude left:1000-1600: -exclude right:* -exclude top:* -exclude bottom:*
# export MACRO_PLACEMENT = $(DESIGN_HOME)/$(PLATFORM)/$(DESIGN_NICKNAME)/macro_placement.cfg

export TNS_END_PERCENT = 100
export REMOVE_ABC_BUFFERS = 1

# Magic Tool Configuration
export MAGIC_ZEROIZE_ORIGIN = 0
export MAGIC_EXT_USE_GDS = 1

# CTS tuning
export CTS_BUF_DISTANCE = 600
export SKIP_GATE_CLONING = 1

# export CORE_UTILIZATION=0.1  # Reduce this value to allow more whitespace for routing.
```

# 🔨 Step 3 – Run Synthesis

```bash
cd flow
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk synth
```

### Common Error I Faced

```
No rule to make target '.../vsdbabysoc.v'
```

This means the design RTL wasn’t in the correct path.

✔ FIX:
Place the RTL at:

```
flow/designs/src/vsdbabysoc/vsdbabysoc.v
```

<div align="center" >
  <img src="./assets/make_synth_1.png" alt="make_synth_1" width="80%">
</div>


<div align="center" >
  <img src="./assets/make_synth_2.png" alt="make_synth_2" width="80%">
</div>

### Synth check txt 

<div align="center" >
  <img src="./assets/synth_check_txt.png" alt="synth_check_txt" width="80%">
</div>

### Synth statistics

<div align="center" >
  <img src="./assets/synth_stats.png" alt="synth_stats" width="80%">
</div>

---

# 📐 Step 4 – Floorplan

```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk floorplan
```

### Error I Encountered

```
[ERROR STA-0164] syntax error in avsdpll.lib
```

✔ FIX: Replace all `//` with `/* ... */` in `avsdpll.lib`.

<div align="center" >
  <img src="./assets/make_floorplan_1.png" alt="make_floorplan_1" width="80%">
</div>


<div align="center" >
  <img src="./assets/make_floorplan_2.png" alt="make_floorplan_2" width="80%">
</div>


---

# 🖥 Step 5 – Floorplan GUI

Launch GUI mode:

```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk gui_floorplan
```

<div align="center" >
  <img src="./assets/make_gui_floorplan_1.png" alt="make_gui_floorplan_1" width="80%">
</div>

<div align="center" >
  <img src="./assets/make_gui_floorplan_2.png" alt="make_gui_floorplan_2" width="80%">
</div>

---

# 🧱 Step 6 – Placement

```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk place
```

<div align="center" >
  <img src="./assets/make_place_1.png" alt="make_place_1" width="80%">
</div>

<div align="center" >
  <img src="./assets/make_place_2.png" alt="make_place_2" width="80%">
</div>

<div align="center" >
  <img src="./assets/make_place_3.png" alt="make_place_3" width="80%">
</div>

<div align="center" >
  <img src="./assets/make_place_4.png" alt="make_place_4" width="80%">
</div>

<div align="center" >
  <img src="./assets/make_place_5.png" alt="make_place_5" width="80%">
</div>

```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk gui_place
```

<div align="center" >
  <img src="./assets/make_gui_place.png" alt="make_gui_place" width="80%">
</div>

### Heat Map

<div align="center" >
  <img src="./assets/heatmap_1.png" alt="heatmap_1" width="80%">
</div>

<div align="center" >
  <img src="./assets/heatmap_2.png" alt="heatmap_2" width="80%">
</div>


---

# ⏱ Step 7 – Clock Tree Synthesis (CTS)

```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk cts
```

<div align="center" >
  <img src="./assets/make_cts_1.png" alt="make_cts_1" width="80%">
</div>

<div align="center" >
  <img src="./assets/make_cts_2.png" alt="make_cts_2" width="80%">
</div>

<div align="center" >
  <img src="./assets/cts_final_report.png" alt="cts_final_report" width="80%">
</div>

---

# 🚦 Step 8 – Global Routing 

```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk route
```

<div align="center" >
  <img src="./assets/make_route_1.png" alt="make_route_1" width="80%">
</div>

<div align="center" >
  <img src="./assets/make_route_2.png" alt="make_route_2" width="80%">
</div>

### Error I Encountered

```
[ERROR GRT-0116] Global routing finished with congestion.
```

This means routing resources were insufficient.

### ✔ FIX 1 — Inspect Congestion

Open GUI:

```
make ... gui_floorplan
```

Enable congestion map:

```
Route → Congestion Map
```

### ✔ FIX 2 — Reduce Core Utilization

In `config.mk`, set:

```makefile
CORE_UTILIZATION = 0.60
```

This spreads cells further apart → lowers routing congestion.

### Re-run Routing

```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk route
```

---

# ✨ Step 9 – Detailed Routing

Once congestion is solved, TritonRoute completes successfully.


---

# 🏁 Step 10 – GDSII Export

```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk finish
```

Output GDS is located at:

```
results/sky130hd/vsdbabysoc/base/vsdbabysoc.gds
```


---

# 📦 Summary of All Fixes

| Error                 | Cause                           | Fix                                 |
| --------------------- | ------------------------------- | ----------------------------------- |
| RTL not found         | Wrong path                      | Place RTL in `flow/designs/src/...` |
| `STA-0164`            | Liberty file used `//` comments | Convert to `/* ... */`              |
| GUI heatmap issues    | Old OpenROAD version            | Use GUI menus, not Tcl              |
| `GRT-0116` congestion | High core utilization           | Lower to 0.60 and inspect heatmap   |
| GUI warnings          | Missing Wayland plugin          | Safe to ignore                      |

---

# 📚 References

* OpenROAD Project: [https://github.com/The-OpenROAD-Project](https://github.com/The-OpenROAD-Project)
* Sky130 PDK Documentation
* VSD Physical Design Course

---

# 🎯 Final Note

This walkthrough documents a **complete, real-world Physical Design flow** using the VSDBabySoC design. It includes every debugging step I faced so others can easily reproduce the setup.

Feel free to fork this repo or adapt the steps for your own custom designs.
