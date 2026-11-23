# Gestion des rames en gare (TUSP) : Branch-and-Price, ALNS et application de visualisation

Projet SNCF du cours PORT, Master Parisien de Recherche Opérationnelle (MPRO, CNAM), 2025–2026.
BACCAR Bedis · ESTEVE Erwann · VERBOIS Maël.

Des trains arrivent en gare, doivent recevoir des services (nettoyage, réparation, tour en fosse)
puis repartir à l'heure. Il faut décider, pour chaque train, sa voie de garage, ses manœuvres et
le moment de ses services, sans que deux trains n'occupent la même voie au même instant, ou bien
le refuser avec une pénalité. C'est le *Train Unit Shunting Problem* (TUSP).

Le dépôt contient :

| | |
|---|---|
| **Solveur C++** (`src/`) | Modèle sur graphe espace-temps. Deux méthodes : un **Branch-and-Price** exact (génération de colonnes, pricing par plus court chemin avec ressources, règles de branchement dédiées, heuristiques primales) et une **ALNS** (recherche adaptative à grands voisinages) dont les réparations réutilisent le pricing et la borne lagrangienne. |
| **Application web** (`app/`) | FastAPI + React/D3 : simulation animée des solutions sur le plan de la gare, courbes de convergence en direct, arbre de branchement, comparaison B&P / ALNS côte à côte, benchmark avec export CSV. Chaque solution est re-vérifiée à partir des données (conflits de voies, croisements). |
| **Instances** (`instances/`) | 4 gares (small, medium, large, variant) × 5 scénarios, de 8 à 105 trains. |
| **Résultats** (`results/`) | Benchmark B&P / ALNS sous Gurobi, runs d'exemple rejouables dans l'application, résultats du solveur d'origine. |
| **Documentation** (`docs/`) | Rapport, présentation, note technique sur le modèle et les méthodes. |

## Résultats en bref

Benchmark sous Gurobi sur 19 instances (`results/benchmark/bp_vs_alns_2026-11-18.csv`) :

- l'ALNS trouve une meilleure solution que le B&P sur **10 instances**, la même sur 3, et le B&P
  ne fait jamais mieux ;
- l'ALNS trouve sa première solution en **moins de 0,8 s** partout, contre 0,5 à 1 044 s pour le B&P ;
- sur **6 instances** (medium 3–5, large 2–4), seule l'ALNS a une solution dans le temps imparti ;
- le B&P prouve l'optimalité sur medium 1 et large 1, où l'ALNS atteint aussi l'optimum.

Les durées des runs vont de 400 à 1 250 s et ne sont pas toujours identiques entre les deux méthodes
(détail dans [`results/README.md`](results/README.md)).

Une erreur de modèle a aussi été corrigée : le modèle d'origine laissait deux trains se croiser en
sens inverse sur la même voie (89 croisements dans les 10 solutions Gurobi d'origine). Chaque lien
est désormais une ressource de capacité 1 par pas de temps, dans les deux sens. L'erreur a été
repérée en regardant une simulation dans l'application.

## Arborescence

```text
src/                 solveur C++
  builder/           construction du graphe espace-temps (GraphBuilder)
  data/ graph/ parser/
  solver/            maître, pricing, Branch-and-Price, ALNS (LnsSolver, SubMaster), trace
  cli_B&P.cpp        exécutable cli_tusp_solver
  benchmark_B&P.cpp  exécutable benchmark_tusp_solver
app/                 application web (backend FastAPI, frontend React) — voir app/README.md
instances/           données des 20 instances — voir instances/README.md
results/             résultats — voir results/README.md
docs/                rapport, présentation, note technique
python/              ancien script de visualisation (matplotlib), remplacé par l'application
cmake/               recherche de Gurobi pour CMake
```

## Compilation (Gurobi requis)

Prérequis : `g++` (C++23), `cmake`, et Gurobi avec `GUROBI_HOME` défini.

```bash
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
cmake --build . -j
```

- `Release` (par défaut) : `-O3 -march=native` et optimisation à l'édition de liens.
- `Debug` : `-O0 -g` et AddressSanitizer.

On obtient `build/cli_tusp_solver` (une instance) et `build/benchmark_tusp_solver` (une série d'instances).

La licence Gurobi (par exemple une licence WLS dans `~/gurobi.lic`) ne doit **jamais** être placée
dans le dépôt : `*.lic` est ignoré par git.

## Lancer le solveur

```bash
./cli_tusp_solver -f <dossier du scénario> [-n <nom>] [-t <secondes>] [-o <dossier de sortie>] [options]
```

Exemples, depuis `build/` :

```bash
./cli_tusp_solver -f "../instances/small station/scenario 1" -n small_station_sce_1 -t 300
```

```bash
./cli_tusp_solver -f "../instances/large station/scenario 2" -n large_station_sce_2 -t 600 --method lns
```

`Ctrl-C` arrête proprement le solveur : la meilleure solution trouvée est quand même exportée.

**Options générales**
- `-t, --time` : temps limite en secondes (3600 par défaut)
- `-n, --name` : nom de l'instance dans les sorties
- `-o, --out` : dossier de sortie (`../results/runs/<nom>` par défaut)
- `--method bp|lns` : Branch-and-Price (par défaut) ou ALNS
- `--no-link-conflicts` : modèle d'origine, sans la contrainte anti-croisement sur les liens
- `--no-trace` : ne pas écrire `trace.jsonl`
- `-h, --help`

**Branch-and-Price**
- `--cg-gaps a,b,c,d` : tolérance d'arrêt de la génération de colonnes à la racine, jusqu'à la profondeur 5, jusqu'à 10, au-delà (`0,0.001,0.005,0.01`)
- `--heur-every n` : heuristiques primales tous les n nœuds, 0 = jamais (5)
- `--no-diving`, `--dive-gap g` (0.05) : heuristique de plongée
- `--no-mip`, `--mip-time s` (15) : MIP sur le maître restreint
- `--branching auto|quantum` : date des services puis divergence (par défaut), ou divergence seule
- `--dfs` : exploration en profondeur d'abord (par défaut : meilleure borne d'abord)
- `--threads n` : threads Gurobi (0 = choix de Gurobi)

**ALNS**
- `--lns-frac f` (0.3), `--lns-max-frac f` (0.7) : part initiale et maximale des trains libérés
- `--lns-ops random,time,related,dual,rejected,bottleneck` : opérateurs de destruction
- `--lns-repairs greedy,regret,dive,dive_mip` : opérateurs de réparation
- `--lns-accept sa|improve` : recuit simulé (par défaut) ou améliorations seulement ; `--lns-sa-start p` (0.02), `--lns-sa-end p` (0.0005)
- `--lns-segment n` (10), `--lns-reaction r` (0.3) : mise à jour des poids des opérateurs
- `--lns-iter-frac f` (0.03) : budget d'une itération, en part du temps limite
- `--lns-root-frac f` (0.3), `--lns-root-mip-time s` (10) : phase racine
- `--lns-cg-gap g` (0.001), `--lns-mip-time s` (5), `--lns-no-prune`, `--lns-max-iter n`, `--seed n`

## Application web

```bash
./app/run.sh
```

Puis ouvrir <http://localhost:8000>. Le script installe les dépendances, construit le frontend et
démarre le serveur. Sans exécutable du solveur, l'application fonctionne en mode relecture
(résultats existants seulement). Détails : [`app/README.md`](app/README.md).

## Benchmark en ligne de commande

`benchmark_tusp_solver` lance une série d'instances définie dans `src/benchmark_B&P.cpp`
(listes `instance_paths` et `instance_name`, méthode et modèle dans `base_params`, temps limite
`time_limit_s`). Chaque run est écrit dans `results/runs/<nom>/`, et un résumé dans
`results/benchmark/summary_benchmark.csv`. Le plus simple reste la page **Benchmark** de
l'application, qui lance les runs en file d'attente et exporte les résultats en CSV.

## Sorties d'un run

Chaque run écrit dans `results/runs/<nom>/` :
- `solution.json` : statut, statistiques et, pour chaque train, son chemin (`arc_sequence`) dans le graphe espace-temps ;
- `trace.jsonl` : le déroulé de la résolution (itérations de génération de colonnes, bornes, nœuds, branchements, solutions, itérations ALNS), format décrit dans `app/README.md` ;
- `dynamics.csv` : l'évolution des bornes inférieure et supérieure.

## Documentation

- [`docs/Rapport_PORT.pdf`](docs/Rapport_PORT.pdf) : rapport du projet.
- [`docs/presentations/presentation_BP_ALNS_IA_agentique.pdf`](docs/presentations/presentation_BP_ALNS_IA_agentique.pdf) : présentation (problème, modèle, B&P, ALNS, interface, résultats, correction du modèle).
- [`docs/MODEL_AND_METHODS.md`](docs/MODEL_AND_METHODS.md) : note technique (en anglais) sur la contrainte de lien et l'ALNS.
- [`app/README.md`](app/README.md) : application, format des traces.
- [`instances/README.md`](instances/README.md) : format des données.
