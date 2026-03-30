# Build Diagnostic Tools Instead of Guessing

## The Rule

When iterating on visual or numerical accuracy and you've made 3+ adjustment attempts without converging, STOP adjusting and build a diagnostic tool instead.

## Why This Matters

On PPT2VXP's canvas text rendering, we spent multiple rounds adjusting margins, font scale factors, and line spacing by small increments, overlaying screenshots, and re-uploading test decks. Each cycle took 5-10 minutes and the results were ambiguous — "closer but not right."

**The fix**: Building a comparison tool with diagnostic sliders (Y offset, X offset, spacing multiplier) took 20 minutes but gave us exact pixel values in one pass. The sliders revealed:
- Y offset: 0.5px (sub-pixel rounding, not worth fixing)
- X offset: 1px (same)
- Spacing: 96% (CSS vs PowerPoint rendering difference — the actual root cause)
- The PT_TO_PX conversion we'd been chasing was algebraically irrelevant

The tool also accidentally revealed that browser font hinting breaks below 720px — something we never would have found by manual overlay comparison.

## When to Apply

- Pixel-level visual alignment (CSS positioning, font rendering)
- Numerical calibration (conversion factors, scaling)
- Any problem where the feedback loop is: change → rebuild → screenshot → compare → "hmm, maybe a little more"

## How to Apply

1. Build a side-by-side or overlay comparison view
2. Add sliders or inputs for the variables you're adjusting
3. Show the exact values being used (scale factors, pixel sizes)
4. Let the human dial in the exact numbers and report them back
5. Apply the numbers as constants

The diagnostic tool becomes a permanent asset — future changes can be re-calibrated in seconds instead of hours.

## The Anti-Pattern

Repeatedly adjusting a constant by 0.5, re-deploying, and asking "does this look right?" This is the guess-and-check loop. It feels productive but converges slowly, and you can't distinguish between "close but wrong" and "correct but different problem."
