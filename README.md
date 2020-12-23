# svpack
Tools for filtering, comparing, and annotating structural variant (SV) calls in VCF format.

## Filter a structural variant callset
Filter variants in `A.vcf`, writing variants that satisfy all specified filters to `stdout`.

```
-p, --pass-only       Only retain variants with FILTER of PASS or .
--require-svtype      Only retain variants with SVTYPE INFO field
--require-svlen       Only retain variants with SVLEN INFO field
-l N,--min-svlen N    Minimum |SVLEN| for variants with SVLEN INFO field
-v, --invert          Invert result. Return variants that do not satisfy the filters.
```

### Return PASS variants at least 50 bp long
```svpack filter --pass-only --min-svlen 50 A.vcf```

Variants like breakends that do not have an `SVLEN` INFO attribute are considered as satisfying
the `--min-svlen`.

### Invert to return variants shorter than 50 bp
```svpack filter --invert --min-svlen 50 A.vcf```

If `--pass-only` were added with `--invert`, then `svpack filter` would return variants that
fail either `--pass-only` or `--min-svlen 50`.  To filter for PASS variants shorter than 50 bp,
chain two `svpack filter` commands:

```svpack filter --invert --min-svlen 50 A.vcf | svpack filter --pass-only -```
