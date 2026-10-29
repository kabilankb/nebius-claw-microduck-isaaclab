# nebius-claw-microduck-isaaclab

**Teach a robot to walk by asking for it.** NemoClaw agents running NVIDIA Nemotron
on **[Nebius Token Factory](https://tokenfactory.nebius.com)** train the
MicroDuck robot in NVIDIA Isaac Lab, judge the result, tune it, check it, and stage it
for the real robot — from one sentence in a chat window or one command in a terminal.

```
"train the duck and get it ready for the robot"

  train ✓ ─ gate ✓ ─ play ✓ ─ export ✓ ─ verify ✓ ─ deploy ✓

  MicroDuck · flat — Done (dry run)

  Policy     2026-10-29_16-00-25 · iteration 750
  Verify     ✅ 9/9 checks · 786 KB · 61→14 · Δ onnx/torch 3e-06
  Bundle     isaac_sim/_deploy/IsaacContrib-Velocity-Flat-MicroDuck_20261029-183351
  Robot      not configured (dry run)
```

## Why this is useful

Getting a walking policy from simulation onto a small robot is normally a day of
specialist work: launch training, watch reward curves, decide whether the robot is
really walking or just collecting reward, tweak hyperparameters, restart, export,
convert the ONNX, check it matches the robot's interface, copy it over, and hope it
doesn't fall off the table. This project turns that loop into a conversation.

- **Anyone can run it.** A student, a hackathon team or a robotics newcomer types what
  they want; the agent picks the task, the settings and the tools.
- **It judges, not just runs.** The pipeline gates on the *fall rate*, not reward
  alone, automatically resumes with gentler learning when a policy falls too often,
  and a vision model watches the rollout video and critiques the gait.
- **It is safe to hand to a real robot.** Every exported policy passes 9 sim-to-real
  checks (single-file ONNX, 61 observations → 14 actions, ONNX ≡ PyTorch, bounded
  actions, under the robot's 64 MiB limit). Loading it on the robot always waits for a
  human to say the duck is on its stand — enforced by the server, not just the prompt.
- **The answers are honest.** Status replies are rendered from the real pipeline state
  and relayed verbatim, so the agent can't invent progress.
- **One laptop GPU is enough.** Simulation and training run locally (tested on an RTX
  PRO 5000 Blackwell laptop GPU); the thinking happens in the cloud on Token Factory,
  so no GPU memory goes to the language model.

## Why Nebius Token Factory

The agents need large, tool-calling, reasoning models — far bigger than fit next to a
4,096-robot simulation on one GPU. Token Factory serves the NVIDIA Nemotron family
behind one OpenAI-compatible endpoint, so the agents get frontier-class models with
no local inference at all.

| Role in this project | Token Factory model | Why this one |
|---|---|---|
| Claw agent + Duck pilot (planning, tool calls) | `nvidia/nemotron-3-super-120b-a12b` | reliable multi-step tool calling, 262K context |
| Fast/cheap mode | `nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B` | ~5× cheaper ($0.06 / $0.24 per 1M tokens) |
| Gait critique from video frames | `nvidia/Nemotron-3.5-Super-VL` | vision: watches the rollout, answers in ~3 s |
| Also switchable | `Nemotron-3-Ultra-550b-a55b`, `Nemotron-3_5-Lightning` | live model switcher in the console |

- **One key, one endpoint:** `https://api.tokenfactory.nebius.com/v1`. Paste the key
  in the web console; it is verified against `/v1/models`, stored in
  `~/.openclaw/.env` (mode 600), pushed into the sandboxes, and never shown again.
- **Switch models live** from the console — the agent keeps working on the new model.
- **Sandboxed by NemoClaw:** the agents run in OpenShell sandboxes with deny-by-default
  egress; they are allowed to reach Token Factory and the local Isaac control server.
- **Pay per token:** the GPU work runs on your own machine, so a whole train → verify
  run costs only the handful of agent turns that drive it.

## What's inside

```
 Web console (:8800)  ─┐      ┌─ Claw agent  (OpenClaw in NemoClaw sandbox "isaacsim")
 ./duck (terminal)    ─┼──────┤                                      │
                       │      └─ Duck pilot  (Hermes in NemoClaw sandbox "duckpilot")
                       │                    │  Nemotron on Nebius Token Factory
                       ▼                    ▼
              Isaac control server (:5561)  ── one GPU job at a time
                       │
                       ▼
       Isaac Lab 3: MicroDuck · G1 · H1 · Go2 · Cassie · ANYmal-D
       train → gate → tune → play (mp4 + ONNX) → verify → deploy to the duck
```

| Piece | What it does |
|---|---|
| **Web console** (`web/`) | Nebius-styled page: key box, active model + switcher, live 3D viewer, training stats, chat with live tool steps, *Tune it*, *Critique the gait*, *Replay* |
| **Claw agent** (`agents/nemoclaw/`) | free-form control: "train the G1 on rough terrain", "how's it going?", "tune it" |
| **Duck pilot** (`agents/hermes/`) | self-running Hermes agent that owns the whole sim-to-real pipeline for the MicroDuck |
| **`./duck`** | the same pipeline from a terminal, with live status cards |
| **Control server** (`isaac_sim/scripts/isaac_control_server.py`) | runs Isaac Lab jobs, the pipeline, the gate, the tuning loop, the deploy |
| **`verify_policy.py`** (`isaac_lab/scripts/`) | the 9 sim-to-real ONNX checks |

## Quick start

Requirements: an NVIDIA GPU, [Isaac Lab](https://github.com/isaac-sim/IsaacLab)
(`~/IsaacLab-develop`, with the MicroDuck task), NemoClaw, and a
[Token Factory API key](https://tokenfactory.nebius.com).

```bash
echo "NEBIUS_API_KEY=<your key>" >> ~/.openclaw/.env && chmod 600 ~/.openclaw/.env

./demo.sh check      # key, model, GPU memory, ports, checkpoints
./demo.sh up         # control server + agents + console → http://localhost:8800
./demo.sh down       # stop everything
```

First time only, create the duck pilot: `agents/hermes/install_duckpilot.sh`.

### In the browser

Open http://localhost:8800, check the active Nemotron model in the header, then ask
the **Claw** or the **Duck pilot**:

```
train the duck on flat terrain, gui, 64 envs, 300 iterations
how is the training going?            tune it
critique the gait                     ship the best duck policy we already have
run the full duck pipeline            the duck is on its stand, load it
```

### In the terminal

```bash
./duck run -n 1500 --tries 2      # train → gate → tune → play → verify → deploy, live card
./duck ship                       # best checkpoint → play → verify → deploy (~30 s)
./duck status | watch | history
./duck deploy                     # asks "Is the duck on its stand?" first
./duck chat                       # talk to the duck pilot on Nemotron
```

### On the real MicroDuck

Add `DUCK_HOST=microduck@<ip>` to `~/.openclaw/.env` and restart. The pipeline copies
the verified policy to the robot and waits; after you confirm the duck is on its stand
it runs `sudo robotctl policy load walk …`. Roll back any time with
`sudo robotctl policy reset walk`. Without `DUCK_HOST` you get a ready-to-copy bundle
with `DEPLOY.md`.

### Training on a Nebius cloud GPU

```bash
./demo.sh up --gpu-host user@<nebius-vm>    # Isaac Lab runs on the VM, console stays local
```

## Results so far

| | |
|---|---|
| MicroDuck flat, 750 iterations | mean reward ~113, falls in ~7.5% of episodes, passes the gate |
| Exported policy | 786 KB single-file ONNX, 9/9 checks, ONNX vs PyTorch Δ 3e-06 |
| "Ship best" (play → verify → bundle) | ~30 s end to end |
| Agent turn on Nemotron 3 Super | ~5–10 s |
| Gait critique (record → frames → Nemotron 3.5 VL) | ~1.5 min, VL answer in ~3 s |
| Unitree G1 humanoid, Isaac Lab 3 | 0.34 s per iteration at 1,024 envs |

## More

- [`agents/nemoclaw/RUNBOOK.md`](agents/nemoclaw/RUNBOOK.md) — setup, before-demo checklist, troubleshooting
- [`agents/hermes/README.md`](agents/hermes/README.md) — the duck pilot and the pipeline stages
- [`docs/NEBIUS_TEST_REPORT.md`](docs/NEBIUS_TEST_REPORT.md) — Token Factory model tests
- [`docs/ISAAC_CLAW.md`](docs/ISAAC_CLAW.md) — the underlying isaac-claw stack (Isaac Sim scenes, skills, tuning panel)

Built on NVIDIA Isaac Lab, NVIDIA NemoClaw / OpenClaw / Hermes, the
MicroDuck open-source robot, and Nebius Token Factory.
