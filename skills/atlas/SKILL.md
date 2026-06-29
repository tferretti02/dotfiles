---
name: atlas
description: Génère UNE page HTML autonome de documentation visuelle — diagrammes d'architecture, traces / timelines, schémas de données, topologies machines/VM, comparaisons, plans, simulateurs. À utiliser dès qu'une explication gagne à être VUE (architecture, flux, trace, structure, comparaison) plutôt que lue en prose.
argument-hint: <le sujet / problème à documenter visuellement>
author: Thibault Ferretti (tferretti02)
metadata:
  hermes:
    tags: [html, documentation, diagrams, visualization, architecture, trace]
    category: productivity
---

# Atlas

Atlas transforme un problème technique en **une seule page HTML autonome, soignée, hors-ligne** :
documentation + diagrammes. Tu génères le fichier, tu annonces son chemin, fin.

## Request

$ARGUMENTS

Si la requête ci-dessus est non vide, l'utilisateur a invoqué `/atlas` explicitement — produis la page
pour ce sujet maintenant, en suivant le workflow. Si elle est vide, déduis le sujet de la conversation.

## Quand l'utiliser

Dès que la réponse serait plus claire en image qu'en texte :
- **architecture / flux** d'un système (services, files, workers, réseau)
- **trace / timeline / Gantt** d'une exécution (ordonnancement, latences, ordre de push/pop)
- **structure de données** (schéma Redis, modèle de tables, format de message)
- **topologie machines / VM / conteneurs** (specs, rôles, liens réseau)
- **comparaison** d'options, avant/après, trade-offs
- **plan / rapport** technique à faire valider d'un coup d'œil
- **simulateur** d'un comportement paramétrable (faire varier un débit, un nb de workers, un TTL…)

## Sortie — le contrat

1. **UN seul fichier `.html` autonome** : CSS **inline** (et JS inline si besoin), en **français**.
2. **Emplacement** : par défaut `docs/<slug>.html` à la racine du projet concerné (pas forcément le cwd).
   Si le projet n'a pas de convention claire, écris à la racine et indique-le.
3. **Hors-ligne au maximum.** Seule dépendance externe tolérée : **Mermaid via CDN**, et uniquement si un
   diagramme de flux/archi/séquence/état l'exige. Tout le reste (SVG, canvas, JS, CSS) est local et inline.
4. **Aucun design system externe injecté** : la page rend à l'identique ouverte directement dans un navigateur.

## Workflow

1. **Cerner** le problème et choisir le(s) **playbook(s)** adapté(s) (souvent plusieurs).
2. **Choisir la direction design** : pars de la base ci-dessous, puis **étends la palette avec
   des accents métier** (ex. une couleur par type Redis, par tier, par rôle de VM) et **une légende**.
3. **Écrire** le HTML autonome (+ JS inline si interactivité utile).
4. **Auto-vérifier** avec la checklist finale (overflow, contraste, responsive, légendes, a11y).
5. **Annoncer le chemin** du fichier. L'utilisateur l'ouvre lui-même (`xdg-open <fichier>`).

## Design system (base à coller, puis adapter)

Direction : **clair et épuré**, fond blanc franc, encre quasi-noire, **dégradés « aube »**
(bleu → violet → corail, avec une touche d'or en décoratif), bleu primaire, type **géométrique**,
accents en **traits fins**, coins arrondis, ombres douces. Neutres froids, pas de dark-glow.

```html
<style>
  :root{
    --bg:#ffffff; --bg-soft:#f5f6f9; --surface:#ffffff; --surface-2:#eef1ff;
    --ink:#1a1a1a; --ink-2:#5c5c62; --ink-3:#8a8d96;
    --blue:#0028ff; --blue-deep:#001fcc; --blue-bright:#2b66fe; --blue-soft:#eef1ff;
    /* accents : violet, corail, or */
    --violet:#8a38f5; --violet-soft:#f0e8ff; --coral:#ff7575; --coral-soft:#ffecec; --gold:#ffb545; --gold-soft:#fff3df;
    --grad:linear-gradient(120deg,#0028ff 0%,#7a2fe8 52%,#ff7575 100%);      /* hero : bleu→violet→corail, texte blanc OK */
    --grad-aurora:linear-gradient(120deg,#0028ff 0%,#8a38f5 34%,#ff7575 68%,#ffb545 100%); /* décoratif (barres, filets), pas de texte par-dessus */
    --border:#e4e6ec; --border-strong:#d2d4dc;
    --ok:#15a05a; --warn:#d9870b; --danger:#e5484d;
    --radius:16px; --radius-sm:10px;
    --shadow:0 1px 2px rgba(16,24,57,.05),0 10px 30px rgba(16,24,57,.06);
    --mono:ui-monospace,"SF Mono","JetBrains Mono",Menlo,Consolas,monospace;
    --sans:system-ui,-apple-system,"Segoe UI",Roboto,"Helvetica Neue",Arial,sans-serif;
    /* étends ici avec tes accents métier + ajoute une .legend qui les explique */
  }
  *{box-sizing:border-box;}
  body{margin:0;font-family:var(--sans);background:var(--bg);color:var(--ink);line-height:1.6;-webkit-font-smoothing:antialiased;}
  .wrap{max-width:1120px;margin:0 auto;padding:0 24px;}
  nav.top{position:sticky;top:0;z-index:50;background:rgba(255,255,255,.85);backdrop-filter:blur(10px);border-bottom:1px solid var(--border);}
  nav.top .wrap{display:flex;align-items:center;gap:18px;height:58px;overflow-x:auto;}
  nav.top .brand{font-weight:800;color:var(--ink);letter-spacing:-.03em;font-size:16px;white-space:nowrap;}
  nav.top .brand .dot{background:var(--grad);-webkit-background-clip:text;background-clip:text;color:transparent;}
  nav.top a{color:var(--ink-2);text-decoration:none;font-size:13.5px;font-weight:500;white-space:nowrap;}
  nav.top a:hover{color:var(--blue);}
  /* hero : bandeau dégradé bleu, texte blanc */
  .hero{background:var(--grad);color:#fff;padding:66px 0 58px;}
  .hero .eyebrow{color:rgba(255,255,255,.85);}
  .eyebrow{font-size:12px;letter-spacing:.18em;text-transform:uppercase;color:var(--blue);font-weight:700;margin:0 0 14px;}
  .hero h1{font-size:clamp(30px,5vw,50px);line-height:1.06;margin:0 0 16px;letter-spacing:-.035em;font-weight:800;}
  .hero .lede{font-size:clamp(16px,2.2vw,19px);color:rgba(255,255,255,.92);max-width:760px;margin:0;}
  /* sections : blanc + .soft (gris bleuté très clair) pour le rythme */
  section{padding:56px 0;border-bottom:1px solid var(--border);}
  section.soft{background:var(--bg-soft);}
  .sec-eyebrow{font-size:11.5px;letter-spacing:.16em;text-transform:uppercase;color:var(--blue);font-weight:700;margin:0 0 10px;}
  h2{font-size:clamp(24px,3.4vw,33px);margin:0 0 16px;letter-spacing:-.025em;font-weight:800;line-height:1.14;}
  h3{font-size:19px;margin:30px 0 12px;font-weight:700;letter-spacing:-.01em;}
  p{margin:0 0 14px;max-width:860px;} .muted{color:var(--ink-2);}
  code{font-family:var(--mono);font-size:.88em;background:var(--blue-soft);color:var(--blue-deep);padding:1.5px 6px;border-radius:6px;}
  /* cards : blanches, arrondies, ombre douce, enfants min-width:0 (anti-overflow) */
  .grid{display:grid;gap:18px;} .g2{grid-template-columns:repeat(auto-fit,minmax(min(100%,320px),1fr));} .g3{grid-template-columns:repeat(auto-fit,minmax(min(100%,260px),1fr));}
  .card{background:var(--surface);border:1px solid var(--border);border-radius:var(--radius);padding:22px;box-shadow:var(--shadow);min-width:0;}
  .card h4{margin:0 0 8px;font-size:16px;font-weight:700;display:flex;align-items:center;gap:9px;flex-wrap:wrap;}
  /* légende + badges typés (duplique .ty.<type> par couleur métier) */
  .legend{display:flex;flex-wrap:wrap;gap:10px;margin:22px 0 8px;}
  .ty{font-family:var(--mono);font-size:11.5px;font-weight:700;padding:3px 10px;border-radius:999px;color:#fff;white-space:nowrap;}
  /* mermaid : carte claire zoomable + caption */
  .diagram{position:relative;background:var(--surface);border:1px solid var(--border);border-radius:var(--radius);padding:24px;overflow:auto;max-height:80vh;margin:18px 0;box-shadow:var(--shadow);}
  .mermaid svg{display:block;margin:0 auto;max-width:none;}
  .cap{font-size:12.5px;color:var(--ink-3);margin:10px 2px 0;font-style:italic;}
  /* code : fond sombre conservé pour le contraste, coloration via spans */
  pre{background:#0d1220;color:#e8ecf6;border:1px solid var(--border-strong);border-radius:var(--radius);padding:18px 20px;overflow-x:auto;font-family:var(--mono);font-size:13px;line-height:1.65;margin:16px 0;}
  pre .cm{color:#7c869b;} pre .kw{color:var(--blue-bright);} pre .key{color:#8fb4ff;} pre .str{color:#7fd1a6;} pre .num{color:#ffb27a;}
  /* tables : wrapper scrollable, header clair sticky */
  .tbl-wrap{overflow-x:auto;border:1px solid var(--border);border-radius:var(--radius);margin:18px 0;}
  table{border-collapse:collapse;width:100%;min-width:680px;font-size:13.5px;}
  th,td{text-align:left;padding:12px 14px;border-bottom:1px solid var(--border);vertical-align:top;}
  thead th{background:var(--surface-2);color:var(--ink);font-weight:700;font-size:12px;letter-spacing:.04em;text-transform:uppercase;position:sticky;top:0;}
  tbody tr:hover{background:var(--bg-soft);}
  /* callouts info / warn */
  .callout{border-left:3px solid var(--blue);background:var(--blue-soft);border-radius:0 12px 12px 0;padding:16px 18px;margin:18px 0;font-size:14px;}
  .callout .tag{font-weight:800;color:var(--blue-deep);font-size:12px;letter-spacing:.06em;text-transform:uppercase;display:block;margin-bottom:6px;}
  .callout.warn{border-left-color:var(--warn);background:rgba(217,135,11,.08);} .callout.warn .tag{color:var(--warn);}
  /* contrôles interactifs accordés à la marque */
  input[type=range]{accent-color:var(--blue);}
  .btn{font:600 13px var(--sans);color:#fff;background:var(--grad);border:0;border-radius:999px;padding:9px 16px;cursor:pointer;}
  .btn.ghost{background:none;color:var(--blue-deep);border:1px solid var(--border-strong);}
  .accentbar{height:4px;background:var(--grad-aurora);border:0;border-radius:999px;} /* filet « aurore » décoratif */
  :focus-visible{outline:2px solid var(--blue);outline-offset:2px;}
</style>
```

## Playbooks (choisis selon le problème — souvent plusieurs)

- **architecture / flux / séquence / état** → **Mermaid**, dans un `.diagram` zoomable + `.cap`. Jamais de
  boîtes-et-flèches faites main en div/flex.
  ```html
  <script src="https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.min.js"></script>
  <script>
    mermaid.initialize({
      startOnLoad:true, securityLevel:'loose', theme:'base',
      themeVariables:{ fontFamily:'system-ui, sans-serif', background:'#ffffff',
        primaryColor:'#eaf1ff', primaryTextColor:'#0d1220', primaryBorderColor:'#2f6bff',
        lineColor:'#7c869b', secondaryColor:'#f5f7fb', tertiaryColor:'#ffffff' },
      flowchart:{ htmlLabels:true, curve:'basis' }
    });
  </script>
  <div class="diagram"><pre class="mermaid">flowchart LR
    P[Producteur] --> Q[(file:chunks)] --> W[Worker] --> M[Merge]
    Q -.retry.-> DLQ[(dead-letter)]
  </pre></div>
  <p class="cap">Fig. 1 — flux des jobs.</p>
  ```
- **trace / timeline / Gantt** → **SVG dessiné en JS** dans un `.panel`, tooltip au survol. (lanes, barres
  positionnées au temps, axe en haut). Garde-le inline et sans dépendance.
- **structure de données (Redis, schéma, message)** → **tables** à color-coding métier (`.ty.<type>` +
  `.legend`), noms de clés en monospace, colonnes TTL/commande. Cards pour les patterns d'accès.
- **topologie machines / VM / conteneurs** → **Mermaid** (`flowchart`/`graph`) pour les liens réseau, ou une
  grille de **cards « nœud »** (hostname, rôle, vCPU/RAM/disque, IP) ; couleur par rôle + légende.
- **comparaison / avant-après** → **table** ou **deux colonnes côte à côte** ; pills d'état (✅/❌, ok/risque).
- **plan / rapport** → `hero` dégradé + sections alternées blanc/`.soft` + `callout` (décision, risque, next step).
- **simulateur / paramétrage** → **sliders + recalcul live** (sortie en chiffres, SVG ou canvas). Exemple :
  ```html
  <label>Workers <input id="w" type="range" min="1" max="16" value="4"></label>
  <output id="out"></output>
  <script>
    const w=document.getElementById('w'), out=document.getElementById('out');
    const render=()=>out.textContent = `${w.value} workers → ${(1000/ w.value).toFixed(0)} ms/job`;
    w.addEventListener('input', render); render();
  </script>
  ```

## Règles visuelles (anti-slop)

- **Hiérarchie** : rends évidents au premier coup d'œil les décisions, risques et chemins critiques.
- **Anti-overflow horizontal à TOUS les niveaux** : enfants de grid/flex en `min-width:0` (et pistes
  `minmax(0,1fr)`) ; tables dans `.tbl-wrap` (`overflow-x:auto`) avec `min-width` ; `pre` en `overflow-x:auto` ;
  `word-break` sur les longues clés/identifiants monospace.
- **Color-coding métier toujours accompagné d'une légende.**
- **Blanc franc** en fond, encre quasi-noire, neutres froids ; accents tirés de la gamme aube (violet / corail / or) ; sur fond coloré, le texte est une nuance du fond, pas un gris délavé.
- **Mermaid** pour nœuds+liens ; **SVG** pour le temporel ; **canvas** pour courbe/calcul dense.
- **Interactivité au service du sens** (régler un paramètre, filtrer, comparer), jamais décorative ;
  toujours accessible (labels, `:focus-visible`) et `prefers-reduced-motion`.
- Left-aligned, rythme d'espacement varié, casse la grille pour l'emphase. Pas de murs de cards identiques.

## Checklist avant de livrer

- [ ] Une seule page `.html` autonome, CSS (et JS) **inline**, en **français**.
- [ ] **Zéro overflow horizontal** (vérifie mentalement mobile ET desktop).
- [ ] Tout color-coding a une **légende**.
- [ ] Diagrammes : Mermaid (archi/flux/séquence/état) ou SVG (trace/timeline) — jamais de boîtes div/flex.
- [ ] Si interactif : **JS vanilla inline**, accessible (labels, focus), sans réseau, `prefers-reduced-motion`.
- [ ] **Chemin du fichier annoncé** à l'utilisateur pour qu'il l'ouvre (`xdg-open …`).
