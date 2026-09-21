# Haltere: Project Plan for a Fast Fruit Fly Brain Simulator

## Idea

In September 2026, Google and HHMI Janelia released a full wiring map of a male fruit fly's brain and nerve cord. It shows about 166,700 nerve cells (neurons) and about 125 million connections between them. A map like this is called a **connectome**. A map is not a working brain, though. To see what the brain *does*, you have to turn the map into a simulation and run it. **Haltere** is a tool that does that quickly and reliably: you give it the map, and you get back a brain simulation you can poke, measure, and connect to other programs.

## About the name

Halteres are two tiny knobs behind a fly's wings. They work like a built-in gyroscope and keep the fly steady and balanced in flight. The name fits the project's main goal: keeping fly brain research steady and honest, with results that can be checked.

The name is free on both crates.io (Rust) and PyPI (Python). Still to check: GitHub and a domain name.

## Why this is worth doing

- **Speed.** A fast simulator lets you run many experiments. If it runs in real time (one second of simulated brain activity takes one second or less of computer time), you can later connect it to a virtual fly body and watch it react live.
- **Honest testing.** Many viral "fly brain plays a game" demos never checked whether the real wiring matters. One team that did check, Oruk Labs, found that randomly shuffled wiring worked just as well on their task. This tool will make that kind of check a one-line command.
- **It fits my skills.** The work is close to compiler and systems programming: read a big input, transform it step by step, and produce fast code.

## Be honest about what already exists

Other tools already turn network descriptions into fast code:

- **Brian2**, a popular brain-simulation tool that can generate its own code (including for GPUs).
- **GeNN**, a tool that generates fast GPU code for brain simulations.

If this project only makes things fast, it is just a copy of GeNN. What makes it different:

1. It goes straight from an official connectome release to a working simulation in one step.
2. It has built-in "shuffled wiring" tests.
3. It has a simple live connection (a socket) so other programs, like a virtual body, can talk to the brain while it runs.

## What "done" looks like

The first version is finished when:

1. **It gives the right answers.** It matches the results of the 2024 Berkeley study, which simulated the female fly brain (139,255 neurons, 50 million connections) and correctly predicted which neurons fire for taste and touch. We check this first on the female map, because that is where known answers exist.
2. **It is fast.** It runs the whole male brain and nerve cord at real time or faster on one GPU, and it has been compared fairly against Brian2 and GeNN on the same machine.
3. **It supports honest tests.** A real-versus-shuffled comparison can be run with one short script.

## How it works (the big picture)

Think of it as a pipeline with five parts:

1. **Download.** Fetch the connectome data using the official tools (neuPrint and its Python library). Save it locally so every run uses the exact same version.
2. **Translate.** Turn the raw data into a simple internal format: groups of neurons, and the connections between them. Each connection gets a strength and a sign (does it excite or calm down the next neuron?).
3. **Optimize.** Clean up and rearrange the data so the computer can process it quickly. For example: drop very weak connections, and place neurons that talk to each other close together in memory.
4. **Run.** Simulate the brain step by step. First on the CPU (written in Rust), later on a GPU.
5. **Connect.** Provide ways to control and read the simulation: a Python interface, and a live socket connection for other programs.

A note to myself: most of the speed will probably come from smart memory layout, not fancy code generation. Only add code generation if measurements show it helps.

## Software stack

The simple rule: **Rust for the fast engine, Python for everything around it.** This is how most serious simulators are built: a fast core that people control with easy scripts.

| Part | Tool | Why |
|---|---|---|
| Simulation engine (CPU) | **Rust** | As fast as C++, catches memory bugs in multithreaded code, and I already use it for Delta. |
| Simulation engine (GPU) | **CUDA**, called from Rust with the `cudarc` crate | Works on NVIDIA GPUs, which is what university clusters and cloud providers have. Also makes the comparison with GeNN fair, since GeNN uses CUDA too. |
| Link between Rust and Python | **PyO3** (built with **maturin**) | Lets Python use the Rust engine like a normal Python library. |
| Downloading the brain map | **neuprint-python** | The official tool for getting connectome data. |
| Saving data locally | **Parquet** files (read and written with **Polars** or **Arrow**) | Fast, compact, and easy to use from both Rust and Python. |
| Tests, experiments, plots | **Python**, with **NumPy** and **Matplotlib** | This is what neuroscientists already use, so results are easy to share. |
| Quick first prototype | **PyTorch** | A roughly 100-line GPU version to measure speed before writing any Rust (see step 0). |
| Checking correctness | **Brian2**, plus the Berkeley team's model code | The "known good" answers my simulator must match. |
| Speed comparison | **GeNN** | The main existing fast simulator to compare against. |
| Automatic testing | **GitHub Actions** | Runs the correctness tests every time I change the code. |

**Why the GPU choice matters most.** Rust's GPU support is weaker than C++'s. CUDA only works on NVIDIA GPUs, but gives the best speed and tooling. The other option, **wgpu**, works on any GPU (including Macs) but is slower for this kind of heavy math. Running on any GPU is not a goal for version 1, so CUDA wins.

**What I'm skipping:**
- **C++**, because it only makes sense if I were adding to GeNN directly.
- **Julia**, because it's a good language, but has a small community for this work and no clear benefit for me.

## The plan, step by step

This assumes about 10 hours a week alongside my first semester at HKU, over roughly 16 weeks.

| Step | Weeks | What I do | I move on when... |
|---|---|---|---|
| 0. Measure the starting point | 1–2 | Run the Berkeley team's public code on the female map. Download the male map and count its connections. Time how fast the existing tool runs. Write a quick PyTorch version on a GPU and time it too. | I know the current speed, how big the problem is, and how fast a simple GPU version already is. |
| 1. Simple, correct version | 3–5 | Write a basic simulator in Rust that uses the same neuron model as the Berkeley study. | Its results match the existing tool, checked by automatic tests. |
| 2. Fast CPU version | 6–9 | Build the translate and optimize steps. Use all CPU cores and vectorized math. | It is at least 10× faster than Brian2, and the tests still pass. |
| 3. GPU version | 10–12 | Move the main simulation loop onto a GPU. | The full male brain runs at real time or faster. Compared against GeNN. |
| 4. Testing tools and interfaces | 13–14 | Add the shuffled-wiring generators, the Python interface, and the live socket. | A real-versus-shuffled test runs in one script. |
| 5. Check, write up, release | 15–16 | Repeat the Berkeley taste and grooming tests on the male map. Write up the results and publish the code. | Public code, a short report, and benchmarks anyone can rerun. |

## What could go wrong

- **Someone already solved the speed problem.** If GeNN already runs the male brain in real time on my hardware, stop racing it. Use GeNN as the engine underneath, and focus on the parts that are new: easy setup, shuffled-wiring tests, and the live connection. Make this decision at step 3, on purpose.
- **We don't know all the brain's settings.** The map shows who connects to whom, but not exact connection strengths or how fast each neuron reacts. Every simulation depends on guesses. To handle this, test on the female map first (where known answers exist) and write down every guess.
- **No GPU at home.** Steps 0–2 only need a normal computer. Before step 3, find GPU access through HKU or cloud credits.
- **Doing too much.** Connecting a virtual fly body is not part of version 1. The live socket makes it possible later.
- **A busy semester.** Steps 0 and 1 are the must-haves. A correct Rust simulator with published speed results is useful even if the rest waits.

## Getting help

Find a computational neuroscience professor or PhD student at HKU. Ask them to review my neuron settings before step 5. Their input is what turns this from a code project into something publishable.

## Current To-Do List

- [ ] Reserve the name: create the `haltere` GitHub repo and publish placeholder packages on crates.io and PyPI.
- [ ] Create a neuPrint account and get an access token.
- [ ] Download and run the Berkeley team's fly brain model code.
- [ ] Time how long one second of simulated brain activity takes. This sets my real speed target.
- [ ] Write the quick PyTorch GPU version and time it. If it already runs in real time, focus the project on setup, shuffle tests, and the live connection instead of raw speed.
- [ ] Start looking for a neuroscience contact at HKU.

## Simple glossary

- **Neuron:** a nerve cell. It sends signals to other neurons.
- **Synapse / connection:** the point where one neuron passes a signal to another.
- **Connectome:** a complete map of all neurons and their connections.
- **Spiking simulation:** a model where each neuron builds up charge and "fires" a spike when it reaches a limit, like a real neuron.
- **Real time:** the simulation runs at least as fast as the real brain would.
- **Excitatory / inhibitory:** a connection that makes the next neuron more likely to fire, or less likely.
- **Shuffled wiring:** the same neurons with randomly swapped connections. Used to test whether the real wiring actually matters.
- **SIMD / vectorized math:** a CPU feature that does the same calculation on many numbers at once.
- **Socket:** a connection that lets two programs send messages to each other while running.
- **CUDA:** NVIDIA's system for running code on its graphics cards (GPUs).
- **PyO3:** a tool that lets Rust code be used from Python.
- **PyTorch:** a popular Python library for fast math on GPUs, mostly used for machine learning.
- **Parquet:** a file format for storing large tables of data compactly.