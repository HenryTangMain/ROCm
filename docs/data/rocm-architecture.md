# ROCm/ROCm — internal architecture

This repo doesn't hold ROCm's source — it holds the documentation site, the
changelog/release generator, and legacy build scripts that operate on top of
the component repos (`rocm-systems`, `rocm-libraries`, `llvm-project`,
`TheRock`, and others). Three independent zones share a handful of
root-level files.

```mermaid
flowchart TB
    subgraph ROOT[" Repo root — shared inputs "]
        direction LR
        CMAKE["CMakeLists.txt<br/><i>BUILD_DOCS → add_subdirectory(docs)</i>"]
        RTD[".readthedocs.yaml<br/><i>Sphinx build config for RTD</i>"]
        REL["RELEASE.md · CHANGELOG.md<br/><i>Canonical release content</i>"]
        MANIFEST["default.xml<br/><i>ROCm component manifest</i>"]
    end

    subgraph ZONES[" Three independent zones "]
        direction LR
        DOCS["<b>docs/</b><br/>Sphinx + MyST site<br/>conf.py · _toc.yml.in<br/>extension/*.py · contribute/*.md<br/>data/reference/precision-support.yaml"]
        AUTOTAG["<b>tools/autotag/</b><br/>Changelog + release notes<br/>tag_script.py · templates/<br/>precision_fetch.py · README.md"]
        BUILD["<b>tools/rocm-build/</b><br/>Legacy build scripts<br/>envsetup.sh · build_*.sh (~90)<br/>rocm-*.xml · docker/"]
    end

    subgraph OUT[" Outputs "]
        direction LR
        SITE["rocm.docs.amd.com<br/><i>Built by Read the Docs</i>"]
        EXTREPO["External component repos<br/>rocm-systems · rocm-libraries<br/>llvm-project · TheRock"]
    end

    CMAKE --> DOCS
    RTD --> DOCS
    MANIFEST --> BUILD
    AUTOTAG -- writes --> REL
    REL -- consumed by build --> DOCS
    AUTOTAG -. writes precision-support.yaml .-> DOCS
    DOCS --> SITE
    AUTOTAG -- GitHub API --> EXTREPO
    BUILD -- clones & builds --> EXTREPO

    classDef zone fill:#fff4ec,stroke:#c8451f,stroke-width:1px,color:#221d16;
    classDef root fill:#f7f3ec,stroke:#756a58,stroke-width:1px,color:#221d16;
    classDef out fill:#eaf4f1,stroke:#1f6f63,stroke-width:1px,color:#221d16;
    class CMAKE,RTD,REL,MANIFEST root;
    class DOCS,AUTOTAG,BUILD zone;
    class SITE,EXTREPO out;
```

## Legend

- **Solid arrow** — generates / writes content
- **Solid arrow (config)** — configures / builds from
- **Dashed arrow** — cross-reference at the detail level

Component source (rocm-systems, rocm-libraries, llvm-project, TheRock, and
others) lives in separate repositories. This repo only documents and
packages releases for them — see `CLAUDE.md` for the full contributor-facing
guide.
