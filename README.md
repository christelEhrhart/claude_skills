# claude_skills

Il s'agit de skills pour Claude Code selon les bonnes pratiques et les conventions de nommage de [Christel Ehrhart](https://christel-ehrhart.info/).
Elles ont été générées par Claude.ai suite à un prompt et peuvent contenir des incohérences qui seront corrigées au fur et à mesure.

---

## Qu'est-ce qu'un Skill ?

Les Skills sont des capacités modulaires qui étendent les fonctionnalités de Claude. Chaque Skill regroupe des instructions, des métadonnées, et des ressources optionnelles (scripts, templates) que Claude utilise automatiquement quand c'est pertinent. 
L'idée est de penser les Skills comme des supports d'onboarding personnalisés : tu packages ton expertise pour en faire un spécialiste sur ce qui compte pour toi.

---

## Structure d'un Skill

Un Skill est un dossier contenant un fichier SKILL.md et des scripts ou ressources optionnels. Les sous-dossiers sont autorisés (et encouragés) pour organiser les scripts d'aide, templates et fichiers de données.

```
.claude/skills/
├── pdf/
│   ├── SKILL.md
│   ├── extract_text.py
│   └── templates/
└── csv/
    ├── SKILL.md
    └── analyze.py
```
Chaque **SKILL.md** contient du frontmatter YAML (entre **---**) qui indique à Claude quand utiliser le skill, suivi du contenu Markdown avec les instructions à suivre :

```yaml
---
name: nom-du-skill
description: "Ce que fait ce skill et quand l'utiliser"
---
# Mon Skill
## Instructions
[Étapes claires pour Claude]
```

<!-- pagebreak -->

## Skills intégrés (bundled) dans Claude Code
Les Skills bundlés sont livrés avec Claude Code et disponibles dans chaque session. 
Ce sont des Skills basés sur des prompts : ils donnent à Claude un playbook détaillé et lui permettent d'orchestrer le travail avec ses outils.

- **/simplify** : revoit les fichiers récemment modifiés pour la réutilisation de code, la qualité et l'efficacité, puis les corrige. Il lance trois agents de review en parallèle et applique les correctifs.
- **/batch < instruction >** : orchestre des changements à grande échelle dans une codebase en parallèle — décompose le travail en 5 à 30 unités indépendantes, présente un plan pour approbation, puis lance un agent par unité dans un worktree git isolé.
- **/debug [description]** : diagnostique la session Claude Code courante en lisant le log de debug.

---

En résumé, les Skills sont une façon puissante d'automatiser des règles de codage récurrentes.


