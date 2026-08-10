# Example flight: star137

One complete, real flight — the best mission this stack has flown — with
**all three logs from the same run**, so the cross-checking workflow the
whole analysis suite is built on can be reproduced from this folder alone.

| file | source | what it is |
|---|---|---|
| `star137_topdown_altitude.png` | `tools/star_plot.py` | top-down planned-vs-flown + altitude profile |
| `star137_bridge.log` | the bridge process | **ground truth**: Gazebo pose supervision, harness decisions, pass/fail |
| `star137_onboard_flash.jsonl` | firmware DebugLog, decoded by `tools/decode_fcwd.py` | **what the FC believed and commanded**: PositionState, VelocityState/Desired, AttitudeState, PathDesired, ActuatorDesired, SystemStats… at up to 10 Hz |
| `star137_gazebo_server.log` | `gz sim -s` stderr/stdout | **simulator complaints** — deliberately 0 bytes: an empty server log is the passing state (any content is physics/plugin errors) |
| `star137_analysis.txt` | `tools/analyze_run.sh` | the three logs cross-examined into one verdict |

## Why three logs

Each log alone has produced confident wrong conclusions in this project
(documented in `docs/PROJECT-NOTES.md`): a phantom estimator bias, a
"control bug" that was harness starvation, a crash blamed on gains that was
corrupted decode data. Truth says where it flew; the board log says what
the FC believed; the server log says whether the simulator itself was
unhappy. **Disagreement between them is the diagnosis** — e.g. the wp3
flyaway signature is the board log showing a stable hover while the bridge
log shows the vehicle departing.

For this run they agree:

- bridge: `PASS - mission complete in 120s, landed at N=-0.75 E=0.02`
- board: `xtrack mean 0.04  p95 0.10  max 0.10 m`, closest approach 0.10 m
  to every vertex, zero overshoot
- estimator vs its own GPS input: 0.015 m mean — remaining error is
  controller, not sensing
- server: silent

## Re-run the analysis against these files

```bash
cd ../ground/gazebo_bridge
./venv/bin/python3 tools/score.py            star137 ../../examples/star137_onboard_flash.jsonl
./venv/bin/python3 tools/wp_arrival.py       ../../examples/star137_onboard_flash.jsonl
./venv/bin/python3 tools/corner_handedness.py ../../examples/star137_onboard_flash.jsonl
./venv/bin/python3 tools/plan_check.py       ../../examples/star137_onboard_flash.jsonl
./venv/bin/python3 tools/star_plot.py        ../../examples/star137_onboard_flash.jsonl /tmp/replot.png
```
