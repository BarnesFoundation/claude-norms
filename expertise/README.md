# Expertise Registry

A cross-project record of **who knows what**, **what's been built**, and **what was learned building it**. This is not a resume — it's a lookup table for future Claude sessions and developers who need to find prior art or the right person to consult.

## How This Works

### For People (`people/`)
Each file tracks a person's proven expertise across projects. "Proven" means they've shipped it, not just read about it. These files are updated by that person's Claude sessions as work is completed — not self-reported aspirationally.

### For Implementations (`implementations/`)
Each file documents a significant technical implementation that was completed across any Barnes project. These serve as reference material when a future project needs similar functionality. The file points to the source code, explains the approach, captures what worked and what didn't, and notes reuse potential.

### For Libraries (`libraries/`)
When an implementation is extracted into a reusable library or package, it gets its own entry here. This is the catalog of "things we've already built that you can use."

## When to Update

- **After completing a significant implementation**: The Claude session that built it adds an implementation entry and updates the relevant person's expertise file
- **After extracting a reusable library**: Add a library entry
- **After discovering a non-obvious lesson**: Add it to the relevant implementation's "Lessons Learned" section
- **After a person demonstrates new expertise**: Their Claude updates their people file

## File Formats

See the templates below. Every entry MUST include the project name, date, and a concrete description of what was done — not vague claims.
