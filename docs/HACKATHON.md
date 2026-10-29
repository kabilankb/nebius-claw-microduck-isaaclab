# Nebius × NVIDIA Global AI Hackathon — submission

Copy-ready text for the Devpost form.

| Field | Value |
|---|---|
| Project | Nebius × NemoClaw: Autonomous MicroDuck Training and Sim-to-Real Deployment in Isaac Lab |
| Track | **Physical AI** |
| Repository | https://github.com/kabilankb/nebius-claw-microduck-isaaclab (Apache-2.0) |
| Project page | https://kabilankb.github.io/nebius-claw-microduck-isaaclab/ |
| Demo video | https://www.youtube.com/watch?v=O5wfg00z8pc |
| Runs on | Nebius Token Factory (runtime calls to `https://api.tokenfactory.nebius.com/v1`) |
| NVIDIA models | Nemotron 3 Ultra 550B, Nemotron 3 Super 120B, Nemotron 3.5 Super VL, Nemotron 3 Nano 30B |
| NVIDIA tools | Isaac Lab 3, NemoClaw, OpenShell, OpenClaw, Hermes Agent |

## What it is (short description)

Teach a robot to walk by asking for it. NemoClaw agents running NVIDIA Nemotron on
Nebius Token Factory train the MicroDuck bipedal robot in Isaac Lab, judge the result,
tune it, verify it and stage it for the real robot — from one sentence in a chat window
or one command in a terminal.

## Why we built it

Getting a walking policy from simulation onto a small robot is a day of specialist
work: launch training, read reward curves, decide whether the robot is really walking or
just farming reward, guess new hyperparameters, restart, export, convert the ONNX, check
it matches the robot's interface, copy it over, and hope it doesn't fall off the table.
Students, hackathon teams and robotics newcomers stall at every one of those steps. We
wanted that loop to be a conversation, with a human approving only the step that moves
real motors.

## How it works

- **Two agents, each in a NemoClaw/OpenShell sandbox** with deny-by-default egress:
  the **Claw** (OpenClaw) for free-form control — "train the G1 on rough terrain", "how's
  it going?", "tune it" — and the **Duck pilot** (Hermes Agent), which owns the whole
  sim-to-real pipeline. Both call Isaac tools over MCP through the OpenShell L7 proxy.
- **The pipeline runs on the host** (so it survives an hour of training):
  train (Isaac Lab 3, 4,096 parallel MicroDucks) → **gate** on fall rate, not reward alone →
  **tune**: Nemotron 3 Ultra reads the failed run and picks the retry's learning rate and
  entropy → resume → **play** (records a clip, exports ONNX) → **verify** (9 sim-to-real
  checks: single-file ONNX, 61 observations → 14 actions, ONNX ≡ PyTorch, bounded
  actions, < 64 MiB) → **deploy** (copies to the duck and waits; loads with
  `robotctl` only after a human says the duck is on its stand — enforced by the server).
- **Gait critique:** the console records the policy, samples six frames and asks
  Nemotron 3.5 Super VL what's wrong with the gait.
- **Honest replies:** the server renders every status card from real pipeline state and
  the agent relays it verbatim, so it cannot invent progress.
- **Interfaces:** a Nebius-styled web console (key verification, live model switcher,
  3D viewer, training stats, Tune it, Critique the gait, Replay) and a `./duck` CLI.

## How we used Nemotron and where Token Factory accelerated the work

| Job | Model on Token Factory |
|---|---|
| Tuning advisor when a policy fails the gate | Nemotron 3 Ultra 550B (~2 s) |
| Planning and tool calls for both agents | Nemotron 3 Super 120B |
| Gait critique from video frames | Nemotron 3.5 Super VL (~3 s) |
| Fast mode, switchable live | Nemotron 3 Nano 30B |

Token Factory is what makes this fit on one laptop: the GPU is fully used by a
4,096-robot simulation, so there is no room for a local 120B–550B model. With one
OpenAI-compatible endpoint we route each job to the right Nemotron — Ultra for the
expensive judgment call, Super for tool use, VL for vision, Nano for cheap turns — and
switch models live without redeploying anything.

## What was significantly updated during the Submission Period

isaac-claw existed in June–July 2026 as a way to drive Isaac Sim from **local** models
on a DGX box. During the Submission Period we rebuilt it around Nebius Token Factory and
the MicroDuck sim-to-real loop (35+ files, ~4,300 lines):

- all agent reasoning moved to Nemotron on Token Factory (custom NemoClaw provider)
- new web console with key verification, live model switcher, Tune it, gait critique
  (Nemotron 3.5 VL) and Replay
- Isaac Lab 3 tasks for the MicroDuck and humanoids; fixed the humanoid training errors
- the Hermes Duck pilot agent and the train → gate → Ultra tune → play → verify → deploy
  pipeline
- the 9-check sim-to-real ONNX verifier, the `./duck` CLI, the project page

## Testing instructions

Needs an NVIDIA GPU, Isaac Lab (with the MicroDuck task), NemoClaw and a Token Factory
key. Setup is in the README:

```bash
echo "NEBIUS_API_KEY=<key>" >> ~/.openclaw/.env && chmod 600 ~/.openclaw/.env
./demo.sh check && ./demo.sh up          # → http://localhost:8800
./duck ship                              # best checkpoint → play → verify → bundle
```

Without the hardware, the video and project page show every module working.

## Feedback on Nebius Token Factory and NVIDIA tools

1. **Nemotron 3.5 Super VL returns `content: null`** when reasoning is on and
   `max_tokens` is small — all tokens go to reasoning. We had to send
   `chat_template_kwargs: {"enable_thinking": false}`. The model page should say so, or
   the API should warn when output is truncated inside reasoning.
2. **Nemotron 3 Super needs `max_tokens` ≥ ~1024** before any visible content appears;
   lower values return empty content with no error.
3. **`/v1/models` lists Ultra and Lightning without context length or pricing** —
   metadata for every listed model would let apps pick a model programmatically.
4. **Ultra is excellent for judgment calls**: 1.9–2.1 s for a grounded, bounded JSON
   answer — faster than Nano (6 s) on the same prompt in our tests. Worth documenting
   that Ultra isn't only for long tasks.
5. **NemoClaw `inference set`** often reports "health timeout" even though the new
   route is committed and works — misleading in scripts.
6. **Hermes `mcp add`** prompts interactively to enable tools; a `--yes` flag would make
   provisioning scriptable (we pipe `Y`).
7. **`nemoclaw stop`** keeps the gateway running while sandboxes are registered and
   warns about Ollama cleanup when Ollama isn't used.
8. **OpenShell sandbox egress policies** were easy to reason about and made it
   comfortable to give an agent tools that start GPU jobs.

## Demo video script (≤ 3:00, upload to YouTube as Public)

| Time | Shot |
|---|---|
| 0:00–0:15 | Title + the problem: sim-to-real is a day of specialist work |
| 0:15–0:35 | Console: Token Factory panel, key verified, active Nemotron model, model switcher |
| 0:35–1:05 | Ask the Claw "train the microduck on flat ground, gui, 64 envs" → live 3D view, reward curve |
| 1:05–1:25 | "Tune it" → the agent reads the curve, restarts with new settings |
| 1:25–1:45 | "Critique the gait" → clip + Nemotron 3.5 VL answer |
| 1:45–2:20 | Duck pilot: "ship the best policy" → train/gate/play/verify card; show the Ultra `Tune` line from a failed-gate run |
| 2:20–2:45 | Terminal: `./duck run`, `./duck watch`, the 9/9 verify card |
| 2:45–3:00 | Architecture diagram; "Nemotron on Nebius Token Factory" close |

Physical AI needs ≥ 1 minute of the robot (or, without hardware, the key modules)
operating — the 0:35–2:45 segment covers the modules. No copyrighted music.
