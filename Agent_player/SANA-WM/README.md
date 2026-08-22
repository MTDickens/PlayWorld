# SANA-WM Agent Player Adapter

The SANA-WM `player.py` implements the same Agent Player entry-point pattern as
the bundled `genie3`, `happyoyster`, and `hyworld2` adapters. Unlike those
browser-controlled models, it invokes a local SANA-WM checkout directly. The
adapter performs chunked closed-loop prediction:

1. the agent observes the initial image, four sampled frames from every older
   chunk, and eight sampled frames from the latest chunk, all chronologically;
2. it selects `W`, `S`, `A`, `D`, `LEFT`, `RIGHT`, `UP`, `DOWN`, `WAIT`, or
   `STOP`;
3. the decision is translated to SANA's native action string;
4. SANA generates one chunk and its last frame conditions the next chunk;
5. chunks are joined into the task MP4.

`agent` and `refined` query the agent at every chunk. In
`dataset_then_agent`, every original action segment is split independently:
the first `ceil(count * 0.6)` action tokens are a mandatory dataset minimum.
After those chunks, the agent may add any number of independent chunks for the
current segment; `NEXT` advances to the next original action segment. Thus 40%
is not an agent maximum. The scheduler reserves enough slots to preserve the
60% dataset minimum of all later segments. After all segments, the agent fills
the remaining slots until the fixed 30-chunk rollout is complete. `dataset`
retains the original non-agent baseline.

Agent extension of any one original action segment is capped at five chunks.
After five additions the scheduler forces `NEXT`, even if the agent keeps
requesting the same action.

The prompt does not expose the dataset phase recommendation. The agent must
judge progress independently from the complete visual history and choose the
action that best completes the task objective.

The current prompt starts directly with the exact task goal from the dataset
JSON. It reports the current step, completed chunk count, action history, and a
timestamp label for every attached frame. The agent selects one of
`W/S/A/D/LEFT/RIGHT/UP/DOWN/WAIT/NEXT`; `NEXT` advances the current action
segment, while `END` terminates the rollout when the full task goal is visually
complete. Therefore 30 chunks is now a maximum rather than a mandatory length.

OE `insight evolution` tasks whose dataset action is wait-only use a dedicated
stationary policy. A 30-chunk rollout uses 30 fixed `WAIT -> none-33` chunks;
the agent is not called, and camera movement actions cannot be introduced.

Dataset actions are grouped in threes and converted to one 33-frame native
command: for example `W,W,W -> w-33`, `D,D,D -> d-33`; a final group of one or
two identical actions also becomes one `*-33` chunk. The adapter preserves the
requested character mapping `W/S/A/D -> w/s/a/d`, arrows to `j/l/i/k`, and
`WAIT -> none`. Agent decisions likewise execute as exactly one `*-33` chunk;
they may add new chunks but cannot lengthen one command to `*-66` or similar.

## Prerequisites

```bash
export SANA_WM_RUNNER="$PWD/Agent_player/SANA-WM/run_agent.py" # optional override
export SANA_ROOT=/m2v_intern/public_datasets/worldmodel_benchmark/experiment/SANA_WM/Sana
export SANA_CONFIG=/path/to/config.yaml       # when required by the checkpoint
export SANA_CHECKPOINT=/path/to/checkpoint    # when not resolved by the config
cp Agent_player/api_keys.example.sh Agent_player/api_keys.sh
# Fill PLAYER_API_URL, PLAYER_API_KEY, and PLAYER_MODEL.
```

For a nonstandard benchmark-data location, set
`WORLDMODEL_BENCHMARK_DIR`; for category runs, `WORLDPLAY_DATA_ROOT` can point
directly to the directory containing `GC.json`, `IF.json`, and `OE.json`.

Install the native SANA-WM environment and this adapter's small dependencies.
Model weights are not downloaded automatically.

The adapter follows the model-directory layout used by the other players:

- `player.py`: stable CLI entry point and Agent API configuration;
- `run_task.sh`: positional task-ID launcher and advanced flag passthrough;
- `run_agent.py`: scheduling, task prompt, per-segment 60% dataset minimum,
  dynamic agent chunks, 30-chunk cap, `NEXT`/`END`, OE wait-only handling, the
  repository-standard OpenAI-compatible Agent API, and timestamped video
  observations;
- `requirements.txt`: adapter-only Python dependencies.

## Run

Like the other adapters, `run_task.sh` accepts task IDs as its first argument:

```bash
SANA_WM_MAPPING_JSON="$PWD/Agent_player/example/GC/001.json" \
SANA_WM_DATA_ROOT="$PWD/Agent_player/example" \
Agent_player/SANA-WM/run_task.sh GC002 \
  --action-source agent \
  --num-chunks 17 \
  --chunk-frames 33
```

Multiple task IDs can be comma-separated. When `SANA_WM_MAPPING_JSON` is not
set, the launcher infers the category from the first task ID and reads
`$SANA_WM_DATA_ROOT/<CATEGORY>.json`. Outputs default to `outputs/sana_wm/`;
override them with `SANA_WM_OUT_ROOT`.

The explicit CLI remains available for custom datasets:

```bash
Agent_player/SANA-WM/run_task.sh \
  --mapping-json Agent_player/example/GC/001.json \
  --images-dir Agent_player/example \
  --tasks GC002 \
  --action-source agent
```

Mixed mode:

```bash
Agent_player/SANA-WM/run_task.sh \
  --mapping-json /path/to/GC.json --tasks GC001 \
  --action-source dataset_then_agent --num-chunks 30 \
  --dataset-actions-per-chunk 3 --chunk-frames 33 \
  --output-dir outputs/sana_wm_mixed
```

For the local full benchmark:

```bash
Agent_player/SANA-WM/run_task.sh \
  --mapping-json /m2v_intern/public_datasets/worldmodel_benchmark/worldplay_0622/GC.json \
  --tasks GC007 \
  --action-source agent \
  --output-dir outputs/sana_wm
```

Each task directory contains chunk MP4s, `last_frame_chunkXX.jpg`, the final
`<task_id>.mp4`, action/source metadata, and `<task_id>_result.json`.
