# Handbook

Personal documentation site built with [Zensical](https://zensical.org).

**Live site:** <https://kiyam.github.io/handbook/>

## Preview locally

Requires [uv](https://docs.astral.sh/uv/). Dependencies are pinned in `uv.lock`, and CI installs the same versions.

```bash
uv sync                  # install pinned deps into .venv
uv run zensical serve    # live preview at http://localhost:8000
uv run zensical build --strict   # what CI runs; fails on warnings (e.g. broken links)
```

## Add a page

1. Create the Markdown file in the right section, e.g. `docs/hpc/slurm.md`.
2. Register it in the `nav` in `zensical.toml` under its section:

    ```toml
    { "HPC" = [
      "hpc/index.md",
      { "Slurm" = "hpc/slurm.md" },
    ] },
    ```

3. Put images in `docs/assets/images/<section>/` and link them relatively,
   e.g. `![Alt text](../assets/images/hpc/queue.png)`.

A new section is the same: create `docs/<section>/index.md`, add a `{ "Section" = ["<section>/index.md"] }` entry to `nav`, and add an images folder.

## Deployment

Pushes to `main` build and deploy to GitHub Pages via `.github/workflows/docs.yml`. Pull requests build only (strict mode), so broken links or config fail the check before merging.

To bump Zensical: `uv add zensical==<version>` and commit the updated `pyproject.toml` and `uv.lock`.
