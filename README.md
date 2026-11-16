# Tâche 3 - IFT3913: Automatisation des tests de sécurité

---

## 1. Modification du workflow Github Actions

Nous avons modifié le fichier YAML du workflow Github Actions [main-jdk17-build](/.github/workflows/main-jdk17-build.yml) puisqu'il contenait toutes les étapes nécéssaires à la réalistation de ce travail (build maven), mais également puisqu'il utilise Java 17, qui est largement utilisé.

### Modification #1
```yml
steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0
```
Cette modification permet à Github d'aller chercher non seulement le commit qui vient d'être ajouté, mais toute l'historique des commits nécéssaires pour Semgrep. On veut que Semgrep puisse comparer le commit présent à une ancienne version, le paramètre `--baseline-commit`, pour pouvoir détecter de nouvelles vulnérabilité (en ignorant celles déjà présentes avant l'implémentation) Par défaut, Github utilise `fetch-depth: 1`. Cependant, utiliser cette valeur ne permet pas à Github de récupérér `--baseline-commit`, donc Semgrep est incapable de faire la comparaison correctement. 

### Modification #2
```yml
- name: Set up Python
    uses: actions/setup-python@v6
    with:
        python-version: '3.14'

- name: Install Semgrep
    run: pip install semgrep
```
Ces lignes permettent d'installer python dans l'environnement, qui sera utilisé pour installer Semgrep avec pip.

### Modification #3
```yml
- name: Run Semgrep
    env:
        BASELINE: ${{ github.event_name == 'pull_request' && github.event.pull_request.base.sha || github.event.before }} 
    run: | 
        semgrep scan \
        --config auto \
        --baseline-commit $BASELINE  \
        --error
```
Ces lignes configurent et exécutent l'analyse avec Semgrep. Premièrement, la section `env` définie une variable d'environnement nommée `BASELINE`. La valeur de cette variable va dépendre du type d'évènement qui a déclenché l'exécution dur workflow, soit une pull-request ou un push. La variable prendra la valeur du SHA qu'il faudra utiliser pour identifer le commit de base auquel Semgrep va comparer le nouveau commit. Semgrep sera ensuite exécuté avec la commande `semgrep scan`, avec les paramètres suivant:

- `--config-auto`: Permet à Semgrep de choisir automatiquement la configuration selon le projet.
- `--baseline commit $BASELINE`: Compare les résultats de l'analyse actuelle avec l'analyse de base.
- `--error`: Échoue si de nouvelles vulnérabilités sont détectées dans le nouveau commit.