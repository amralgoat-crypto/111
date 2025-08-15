#!/usr/bin/env python3
"""
PromptSmith: an AI prompt generator for coding tasks.

Purpose
-------
Given a concise spec, emit production-grade prompts that direct a code LLM to produce correct, testable output. The generator assembles System, Developer, and User sections; injects constraints; and requests artifacts in a reproducible layout.

Usage
-----
python promptsmith.py --preset react_component --title "Todo list" --spec "Add, toggle, filter" --lang ts --framework react --out prompt.txt

Or provide a JSON/YAML brief:
python promptsmith.py --brief brief.json --out prompt.txt

No external dependencies.
"""
from __future__ import annotations
import argparse
import json
import sys
from pathlib import Path
from textwrap import dedent

# ----------------------------
# Core templates
# ----------------------------
SYSTEM_BASE = dedent(
    """
    You are an expert software engineer and architect. You write minimal, correct, well-structured code. 
    You think before you code, but you only expose conclusions, not chain-of-thought. 
    You follow specifications precisely. You return only the requested artifacts and nothing else.
    """
).strip()

DEV_BASE = dedent(
    """
    Quality bar:
    - Deterministic builds; zero ambiguous placeholders.
    - Clear file tree. One root folder. Include all files referenced by imports.
    - Add minimal docs: README with run steps.
    - Include unit tests where applicable.
    - Avoid external services unless explicitly allowed.
    - Prefer standard library. If libraries are required, justify in comments at the top of each file.
    Output format contract:
    - Return a single fenced code block containing a tar-like directory listing followed by file contents.
    - Directory header format:
      ```
      <project_name>/
      ├─ README.md
      ├─ <other files>
      ```
    - Then, for each file, emit:
      """
      // path: <project_name>/<path/to/file>
      <file contents>
      """
    - Do not emit explanations outside files. No chatty text.
    """
).strip()

# Task presets define targeted scaffolds
PRESETS = {
    "react_component": {
        "user_template": dedent(
            """
            Build a self-contained {lang} React {kind} using {framework} + Vite.
            Feature set: {spec}.
            Constraints:
            - One file for the component, one for styles, one for tests.
            - Accessibility: keyboard and ARIA where relevant.
            - State management: local hooks only unless justified.
            - No UI kits unless allowed.
            Deliverables:
            - <project>/src/{component_name}.{ext}
            - <project>/src/{component_name}.test.{test_ext}
            - <project>/src/{component_name}.css
            - README with install/run instructions.
            """
        ).strip(),
        "defaults": {
            "kind": "component",
            "framework": "react",
        },
    },
    "express_api": {
        "user_template": dedent(
            """
            Build a minimal Node.js Express REST API.
            Endpoints: {spec}.
            Constraints:
            - Node LTS, ESM modules.
            - Validation with zod or express-validator; pick one and include.
            - Error handling middleware.
            - In-memory store only.
            - Include Jest tests and a Postman collection.
            Deliverables: package.json, src/index.ts, src/routes/*.ts, tests, README.
            """
        ).strip(),
        "defaults": {},
    },
    "python_script": {
        "user_template": dedent(
            """
            Write a Python {kind}.
            Task: {spec}.
            Constraints:
            - Python 3.11.
            - Type hints; mypy-clean.
            - CLI via argparse.
            - Unit tests with pytest.
            Deliverables: src/main.py, tests/test_main.py, README.
            """
        ).strip(),
        "defaults": {"kind": "script"},
    },
    "unity_minigame": {
        "user_template": dedent(
            """
            Produce C# scripts and asset layout for a Unity 2D minigame.
            Core loop: {spec}.
            Constraints:
            - Unity 2022 LTS.
            - Single scene.
            - Input System package.
            - Prefabs for all runtime-spawned objects.
            - Include brief setup steps in README.
            Deliverables: Assets/Scripts/*.cs, Assets/Prefabs/*.prefab (text YAML), README.
            """
        ).strip(),
        "defaults": {},
    },
    "godot_minigame": {
        "user_template": dedent(
            """
            Produce GDScript and scene files for a Godot 4 minigame.
            Core loop: {spec}.
            Constraints:
            - Single main scene.
            - Signals for gameplay events.
            - Exported variables for tuning.
            - Minimal assets as placeholders.
            Deliverables: *.tscn, *.gd, README with run steps.
            """
        ).strip(),
        "defaults": {},
    },
}

# Output style enforcers
CONSTRAINTS_BASE = dedent(
    """
    Global constraints:
    - Keep it minimal; remove dead code.
    - Deterministic randomness: seed where used.
    - Security hygiene: no eval, sanitize inputs, avoid secret leakage.
    - Performance: O(n) unless higher complexity is justified.
    - Lint: provide config if linting is referenced.
    - Tests must run with a single command.
    """
).strip()

ARTIFACT_CONTRACT = dedent(
    """
    Required final response structure:
    1) One fenced code block with everything.
    2) Project tree, then per-file sections marked with path headers.
    3) No prose outside code fences.
    """
).strip()

EXEMPLAR = dedent(
    """
    Example of the directory header style to imitate:
    ```
    my-project/
    ├─ README.md
    ├─ src/index.ts
    ├─ tests/index.test.ts
    ```
    """
).strip()

# ----------------------------
# Generator
# ----------------------------

def assemble_prompt(
    title: str,
    user_body: str,
    system_base: str = SYSTEM_BASE,
    dev_base: str = DEV_BASE,
    constraints: str = CONSTRAINTS_BASE,
    artifact_contract: str = ARTIFACT_CONTRACT,
    exemplar: str = EXEMPLAR,
) -> str:
    return dedent(
        f"""
        ### SYSTEM
        {system_base}

        ### DEVELOPER
        {dev_base}

        ### USER
        Title: {title}
        {user_body}

        ### CONSTRAINTS
        {constraints}

        ### OUTPUT CONTRACT
        {artifact_contract}

        ### EXEMPLAR
        {exemplar}
        """
    ).strip()


def render_user_template(preset: str, **kwargs) -> str:
    if preset not in PRESETS:
        raise ValueError(f"Unknown preset: {preset}")
    tpl = PRESETS[preset]["user_template"]
    defaults = PRESETS[preset].get("defaults", {})
    data = {**defaults, **kwargs}
    return tpl.format(**data)


# ----------------------------
# CLI
# ----------------------------

def main(argv=None):
    p = argparse.ArgumentParser(description="Generate coding prompts for LLMs")
    p.add_argument("--preset", choices=sorted(PRESETS.keys()), required=True)
    p.add_argument("--title", required=True)
    p.add_argument("--spec", required=False, default="")
    p.add_argument("--kind", required=False, default="component")
    p.add_argument("--framework", required=False, default="react")
    p.add_argument("--lang", required=False, default="ts")
    p.add_argument("--component_name", required=False, default="App")
    p.add_argument("--out", required=False)

    args = p.parse_args(argv)

    # Derive language-specific extensions
    lang = args.lang.lower()
    if args.preset == "react_component":
        if lang in ("ts", "typescript"): 
            ext = "tsx"; test_ext = "test.tsx"
        else:
            ext = "jsx"; test_ext = "test.jsx"
    else:
        ext = "txt"; test_ext = "test.txt"

    user_body = render_user_template(
        args.preset,
        spec=args.spec,
        kind=args.kind,
        framework=args.framework,
        lang=args.lang,
        component_name=args.component_name,
        ext=ext,
        test_ext=test_ext,
    )

    prompt = assemble_prompt(title=args.title, user_body=user_body)

    if args.out:
        Path(args.out).write_text(prompt, encoding="utf-8")
    else:
        sys.stdout.write(prompt)


if __name__ == "__main__":
    main()
