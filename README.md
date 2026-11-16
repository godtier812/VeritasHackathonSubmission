# FoldFlux: a spell checker for proteins, graded against the lab

**Track:** Health
**Team:** Ashrith Repaka, Shaunak Acharya

*This project is a prototype and is not medical advice.*

**Try it:** [landing page](https://brca-spellcheck.onrender.com/) · [the tool](https://brca-spellcheck.onrender.com/tool/) · [3D explorer](https://brca-spellcheck.onrender.com/explorer/) · [notebook](notebook.ipynb). The free server can take a minute to wake up.

## The problem

A genetic counselor sits with a patient whose BRCA1 test came back as a "variant of uncertain significance": a protein change nobody can yet call harmful or harmless. Most genes have no lab map, so counselors lean on computer predictions and need to know how far to trust them.

## What we built

Type a single-letter protein change such as `C61G` and FoldFlux answers **Looks harmful** (score −10.50), shows the lab's answer beside it (**loss of function**, −1.738), and gives that call's record on held-out data: right on 196 of 288 clear cases. <!-- app/app_data.csv C61G; band_counts.csv looks_harmful --> A 3D explorer places all 1,837 lab-tested changes on BRCA1's structure, showing where the model and the lab agree.

BRCA1 is the exam, not the use case: at these positions the lab has measured 96.5% of possible single-letter DNA changes, so a counselor would use the lab result there. <!-- BUILD_SPEC.md section 1 --> The exam shows how far to trust this kind of model on genes with no lab map.

## What we added

Our own idea. Grading a protein language model on this lab data has been done before (ProteinGym; the Evo 2 BRCA1 notebook), and stronger tools such as AlphaMissense exist. We added four things: a whole held-out region with every cutoff frozen in advance; a test of whether lab labels from one end of the protein help at the other; missed cases and false alarms counted separately; and a worked example of where it fails.

## How it works

**Data.** 1,837 single-letter BRCA1 changes tested with CRISPR in living cells (Findlay et al. 2018, via ProteinGym), labelled harmful or harmless by the study's own cutoffs; the 155 in-between results are left out. <!-- int_rows.csv 59 + 96 --> We rebuilt the normal sequence and checked every row against it.

**Model.** ESM-2 (650M), a protein language model, used zero-shot: hide one letter, then compare how likely the new letter is with the normal one. It read the BRCA1 sequence in training, but never a lab label. Baselines: always "harmless", and the BLOSUM62 substitution table. We also tried ESM-2 plus start-region labels (a logistic regression with six chemistry features).

**The split.** Cutoffs were set on positions 1–101, frozen, then tested once on positions 1631–1855. We never split at random, because changes at the same position behave alike.

## Results

Held-out end region, tested once (`run=1` in `outputs/results/run_log.txt`): 267 harmful and 899 harmless changes.

| Method | Harmful caught | Missed | False alarms | Catch rate | AUROC |
| --- | --- | --- | --- | --- | --- |
| Always harmless | 0 | 267 | 0 | 0.0% | 0.5000 |
| BLOSUM62 table | 232 | 35 | 571 | 86.9% | 0.6681 |
| **ESM-2 zero-shot** | **252** | **15** | **437** | **94.4%** | **0.8995** |
| ESM-2 plus labels | 245 | 22 | 405 | 91.8% | 0.8662 |

<!-- headline.csv; auroc.csv -->

A model that only read protein sequences ranked harmful BRCA1 mutations better than a classic substitution table. (AUROC gain +0.231, 95% interval +0.188 to +0.272.) <!-- result_sentences.txt S1; bootstrap.json esm_minus_blosum --> To catch 94.4% of harmful mutations, the model also flagged 437 of 899 harmless ones. <!-- result_sentences.txt S6 --> Its "Looks harmless" calls matched the lab on 462 of 477 clear cases. <!-- band_counts.csv looks_harmless -->

**What didn't work.** We could not detect a benefit from adding labels from the other end of the protein. <!-- result_sentences.txt S3 --> AUROC fell from 0.8995 to 0.8662 (95% interval for the change −0.054 to −0.014). <!-- bootstrap.json arm3_minus_esm -->

**Where it fails.** Our worst miss is A1823S: the lab found it broken (−3.643), but the model scored it +0.75, "Looks harmless", and all three methods missed it. <!-- failures.csv miss rank 1 --> Our guess, not tested: the DNA edit disrupts how the gene's message is spliced, which a protein model cannot see.

**Groups check.** By kind of swap, ESM-2 caught 93.5–100% of harmful changes in every group, but false alarms ranged from 42.8% to 75.0%, highest when a cysteine is lost. <!-- by_swap.csv arm2 --> Our data are mutations, not people, so we cannot check performance by age, sex or ancestry.

**Checked after the test.** On the same held-out rows, AlphaMissense's published predictions scored AUROC 0.9155 against 0.8995; the 95% interval for the gap (−0.003 to +0.038) includes zero. <!-- reference_alphamissense.json --> We do not claim to be better.

**Limits.** Blind to splicing (we could not test this). Cells in a dish, not people. Stronger tools exist, and zero-shot scoring is not a new method.

## What we'd do next

Add a DNA splicing predictor for the blind spot above, and repeat the frozen-cutoff exam on a second gene that has a lab map.

## Sources

- **Data:** Findlay GM et al. (2018), *Nature* 562:217–222, https://doi.org/10.1038/s41586-018-0461-z; UW BRCA1 saturation genome editing site, https://sge.gs.washington.edu/BRCA1/; ProteinGym (Notin P et al., 2023), https://github.com/OATML-Markslab/ProteinGym (also prior work).
- **Models and methods:** ESM-2 `facebook/esm2_t33_650M_UR50D` (Lin Z et al., 2023, *Science* 379:1123–1130), https://huggingface.co/facebook/esm2_t33_650M_UR50D; masked-marginal scoring (Meier J et al., 2021, NeurIPS), https://doi.org/10.1101/2021.07.09.450648; BLOSUM62 (Henikoff & Henikoff, 1992) via Biopython, https://biopython.org; Kyte & Doolittle (1982); Zamyatnin (1972); structures PDB 1JM7 and 1T15, https://www.rcsb.org.
- **Prior work and limits:** Evo 2 BRCA1 notebook, https://github.com/ArcInstitute/evo2; AlphaMissense (Cheng J et al., 2023, *Science* 381:eadg7492), published predictions from https://alphafold.ebi.ac.uk/files/AF-P38398-F1-aa-substitutions.csv; 2025 cell-type preprint, https://www.medrxiv.org/content/10.1101/2025.08.11.25333423.full.pdf.
- **Libraries:** PyTorch, Transformers, NumPy, pandas, SciPy, scikit-learn, Biopython, Plotly, Matplotlib, Gradio, 3Dmol.js, pytest, Jupyter.
- **AI tools:** **Claude Code (Anthropic)**: helped with planning, the build specification, some of the code (Arm 4, `src/arm4.py`), reviews and audits, the post-test reference comparison, the demo and page design, and drafting the write-up and the video script. **Gemini in Google Antigravity**: helped with writing and running the project code from the specification.

## License

Our code is released under the MIT License (see [`LICENSE`](LICENSE)). The lab data, pretrained models and vendored libraries keep their own licenses and terms; see the sources above and [`explorer/vendor/3Dmol-LICENSE.txt`](explorer/vendor/3Dmol-LICENSE.txt).
