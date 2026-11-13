# Operator guide: the DGX Spark

Every command, why it is there, and what to expect. Run them in order. Nothing here needs editing a file.

**What is different about this machine.** The DGX Spark has an ARM processor (aarch64), a Blackwell GB10 GPU, and
one pool of memory shared by the CPU and the GPU. The project was built and verified on x86 machines. Every pinned
package has an aarch64 build at the same version (checked against `uv.lock` on 2026-11-13), so the install is the
same. What only this machine can answer is whether JAX 0.11.0's CUDA 12.9 build runs on the GB10, and how fast. Steps
2 and 4 answer both before anything long is started.

## 1. Get the code

The repository is private, so the machine needs your GitHub login once.

If `gh` is installed (`gh --version` prints a version):

    gh auth login
    cd ~
    gh repo clone Dxv-404/g-one-spark
    cd g-one-spark

If it is not, clone over HTTPS. Git asks for a username and a password; give your GitHub username and, as the
password, a personal access token (github.com, Settings, Developer settings, Personal access tokens), not your
account password:

    cd ~
    git clone https://github.com/Dxv-404/g-one-spark.git
    cd g-one-spark

Keep the folder in your home folder on the Spark's own disk, not on a network share.

## 2. First contact: the probe

    bash scripts/a6000/probe.sh

Read-only, about two minutes. It prints and saves `results/probe.txt`: the GPU, the driver and the CUDA version it
supports, the CPU, memory, disk, network and tools.

Two lines may look wrong and are expected here: the CPU model can be blank (ARM processors do not report one the
way x86 does), and GPU memory can read `[N/A]` or `Not Supported`, because the GPU has no memory of its own.

**Send `results/probe.txt` and wait for the reply before going on.**

## 3. Set up

**uv**, which builds the Python environment, if `uv --version` says it is missing:

    curl -LsSf https://astral.sh/uv/install.sh | sh

then close and reopen the terminal. uv fetches the ARM builds by itself.

**tmux**, so that work survives a closed window or a dropped connection (`sudo apt install tmux` if it is missing):

    tmux new -s spark

Everything from here runs inside it. To leave it running, press Ctrl+B, then D. To return: `tmux attach -t spark`.

## 4. Session checks

    bash scripts/a6000/session2.sh

About 2 to 3 hours, five parts in order, each reported as it finishes:

| Part | What it checks | Why |
|---|---|---|
| check | the full machine check: installs the environment, runs the test gates | the run refuses to start on an unchecked machine |
| determinism | one seed of the final program in 6 fresh processes must agree bit for bit | different compilations of one program once disagreed (DEV-038) |
| concurrency | how many long-run processes at once, results compared bit for bit; the costliest conditions timed; a projected duration with the in-run S4 included | sets how the run is split and how long it takes |
| recheck | the S4 learnability check on the frozen world, on this GPU | the calibration ran on a laptop |
| batch | several seeds inside one process, compared bit for bit with one at a time, and timed (D-094) | the other way to use a GPU one seed does not fill |

**The first thing to watch is the check's line "the GPU is used, not the CPU (gate E0)".** If it fails, JAX cannot
use the GB10 with the project's pinned CUDA 12.9. Stop there and send `results/session2.tar`: fixing it means a
newer JAX or CUDA, which is a project decision and never a local change.

If GPU memory reads `[N/A]`, the concurrency part may report that no round was clean and recommend nothing. That is
a limit of what this GPU reports, not a failure of the run. Send the file anyway.

At the end it says which file to send: `results/session2.tar`. **Send it and wait**: the launch command depends on what
it measured.

## 5. The long run

The exact command comes back after step 4, in this form:

    bash scripts/a6000/run.sh longrun --shards=N

The run is resumable: after a crash, a reboot or a power cut, the same command continues where it stopped, and
finished work is never repeated. It prints its progress and its own time estimate. When it finishes it says which
folders to send.

## If something stops

Every stop prints a line beginning `STOPPED:` that names the fix. Send that line if the fix is not obvious.
