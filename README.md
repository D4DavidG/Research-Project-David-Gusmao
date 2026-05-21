# Research-Project-David-Gusmao

LLM agent that, given an arbitrary source tarball, infers and executes a reproducible build recipe: installing dependencies, selecting the toolchain, and running build phases all inside an isolated container.

Undergraduate research project in Paul Gazzillo's lab at the University of Central Florida.

## Status

Pre-implementation. Repository scaffolding only.

## Setup

Requires Python 3.12 and Docker.

```sh
python -m venv .venv
.venv\Scripts\activate          # Windows PowerShell: .venv\Scripts\Activate.ps1
pip install -r requirements.txt
copy .env.example .env          # then fill in ANTHROPIC_API_KEY
```

## Running the executor

Not yet implemented. The sandbox executor will live under `runner/` and will take `(image, mount_dir, command, timeout)` and return `(exit_code, stdout, stderr)` via the Docker Python SDK. See the planned repo layout below.

## Planned layout

```
build_agent/
├── baseline/   # Agentless-style three-phase pipeline (detect → infer → execute)
├── tools/      # Domain-specific tool wrappers (apt, Maven, Gradle, Make, ...)
├── runner/     # Container management, log capture
├── eval/       # Pass@k harness, metrics, cost tracking
└── data/       # Dataset subset (JSON)
```

## License

Pending to be confirmed with the advisor before adding.
