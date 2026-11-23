# Résultats

```text
results/
  benchmark/        benchmark B&P / ALNS (Gurobi) et sorties de benchmark_tusp_solver
  runs/             un dossier par run : solution.json, trace.jsonl, dynamics.csv, run.json
  original_solver/  résultats du solveur d'origine (avant ce projet), modèle sans contrainte de lien
```

L'application lit ces trois dossiers : tout ce qui s'y trouve peut être rejoué (simulation, convergence, comparaison).

## `benchmark/bp_vs_alns_2026-11-18.csv`

Export de la page Benchmark de l'application, runs sous Gurobi, modèle corrigé (contrainte de lien).
Une ligne par instance : meilleur run B&P et meilleur run ALNS, coût, écart, durée, temps avant la
première solution (`ttfs`). L'écart est mesuré par rapport à la **meilleure borne inférieure connue**
pour l'instance, toutes méthodes confondues (0 % = optimum prouvé).

| Instance | B&P : coût | B&P : écart | B&P : durée | ALNS : coût | ALNS : écart | ALNS : durée | Meilleure |
|---|---:|---:|---:|---:|---:|---:|---|
| small 1 | 1 974 | 0.5 % | 300 s | 1 974 | 0.5 % | 124 s | égalité |
| small 2 | 12 457 | 115.6 % | 551 s | 9 803 | 69.7 % | 550 s | ALNS |
| small 3 | 35 647 | 73.6 % | 1253 s | 28 662 | 39.5 % | 1250 s | ALNS |
| small 4 | 43 416 | 56.3 % | 400 s | 36 413 | 31.1 % | 1250 s | ALNS |
| small 5 | 80 500 | 26.1 % | 1251 s | 76 255 | 19.5 % | 550 s | ALNS |
| medium 1 | 3 974 | 0.0 % | 23 s | 3 974 | 0.0 % | 5.78 s | égalité |
| medium 2 | 18 715 | 75.0 % | 1250 s | 16 053 | 50.1 % | 1250 s | ALNS |
| medium 3 | aucune solution | – | – | 35 852 | 55.4 % | 550 s | ALNS seule |
| medium 4 | aucune solution | – | – | 101 814 | 26.3 % | 1250 s | ALNS seule |
| medium 5 | aucune solution | – | – | 182 983 | 58.3 % | 550 s | ALNS seule |
| large 1 | 7 264 | 0.0 % | 116 s | 7 264 | 0.0 % | 31 s | égalité |
| large 2 | aucune solution | – | – | 26 139 | 28.7 % | 1250 s | ALNS seule |
| large 3 | aucune solution | – | – | 79 313 | 150.2 % | 1250 s | ALNS seule |
| large 4 | aucune solution | – | – | 220 636 | 225.1 % | 1250 s | ALNS seule |
| variant 1 | 2 475 | 3.8 % | 1250 s | 2 471 | 3.7 % | 550 s | ALNS |
| variant 2 | 12 078 | 101.6 % | 1254 s | 11 338 | 89.2 % | 550 s | ALNS |
| variant 3 | 28 787 | 21.7 % | 551 s | 28 624 | 21.1 % | 1250 s | ALNS |
| variant 4 | 53 119 | 12.9 % | 1250 s | 52 998 | 12.6 % | 400 s | ALNS |
| variant 5 | 84 923 | 15.7 % | 550 s | 79 501 | 8.3 % | 550 s | ALNS |

À lire avec deux précautions :
- les durées ne sont pas toujours identiques entre les deux méthodes ; l'ALNS avait autant ou moins
  de temps que le B&P pour 8 de ses 10 victoires ;
- « aucune solution » signifie qu'aucun run B&P avec solution n'existe pour l'instance dans cet export.

`benchmark_tusp_solver` écrit son propre résumé dans `benchmark/summary_benchmark.csv`.

## `runs/` : runs d'exemple

| Run | Méthode | Temps limite | Coût | Intérêt |
|---|---|---:|---:|---|
| `variant_station_sce_3_bp_20261104-133353` | B&P | 350 s | 32 348 | run B&P complet, avec la sortie console (`stdout.txt`) et `dynamics.csv` |
| `medium_station_sce_5_lns_20261105-091813` | ALNS | 550 s | 182 983 | même instance que le suivant, avec moins de temps et un meilleur résultat |
| `medium_station_sce_5_lns_20261106-130557` | ALNS | 1 250 s | 187 448 | phase racine plus longue (meilleure borne, 89 935), refroidissement plus lent, un opérateur dominant |

Pour ces trois runs, `solution.json` est reconstruit à partir de la dernière solution enregistrée
dans `trace.jsonl` ; l'application la re-vérifie (aucun conflit, aucun croisement, coût identique).

Les runs lancés depuis l'application ou le CLI s'ajoutent dans ce dossier ; ils sont ignorés par git
(voir `.gitignore`), sauf ces exemples.

## `original_solver/`

Résultats fournis avec le solveur d'origine (Gurobi, 1 h par instance), **modèle sans contrainte de lien** :
- `summary_benchmark.csv` : bornes, gap, temps, nombre de nœuds par instance ;
- `solutions/solution_<instance>.json` : chemins des trains ;
- `dynamics/dynamics_<instance>.csv` : évolution des bornes.

Ces solutions contiennent 89 croisements de trains en sens inverse sur un même lien, autorisés par
ce modèle. Elles ne sont donc pas comparables aux résultats du modèle corrigé. Les bornes inférieures
de l'ancien solveur ne sont pas garanties (défauts corrigés depuis, voir `docs/MODEL_AND_METHODS.md`).
