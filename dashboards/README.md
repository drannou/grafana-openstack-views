# Nova - Usage théorique des hosts par aggregat

Dashboard : `nova-hosts-theoretical-usage.json`

## Prérequis

- Prometheus scrapant [`openstack-exporter`](https://github.com/openstack-exporter/openstack-exporter) (le projet actif, pas l'ancien `niedbalski/openstack-exporter`).
- Une datasource Prometheus configurée dans Grafana (le dashboard demande de la sélectionner via la variable `$datasource` à l'import).

## Import

Grafana → Dashboards → New → Import → coller le JSON (ou "Upload").

## Structure du dashboard

1. **Vue d'ensemble** (en haut) : un rectangle par aggregat, un seul niveau de repeat (`repeat: aggregate` sur un panel Stat, pas de row imbriquée) — donc fiable sur toutes les versions de Grafana. Chaque rectangle empile 5 sous-blocs colorés : **Hosts** (total, toujours vert), **Disabled** (vert si 0, rouge si ≥1), RAM % moyenne, CPU % moyen, VMs.

   - `Hosts` = `count(openstack_nova_vcpus_available{aggregates=~".*$aggregate.*"})` — tous les hosts de l'aggregat, activés ou non.
   - `Disabled` = jointure PromQL entre `openstack_nova_agent_state{service="nova-compute", adminState="disabled"}` et `openstack_nova_vcpus_available{aggregates=~...}` via `and on(hostname)`, pour ne compter que les hosts désactivés **appartenant à cet aggregat** (le label `aggregates` n'existe que sur les métriques hyperviseur, pas sur `agent_state`, d'où la jointure par `hostname`).
   - ⚠️ Les seuils par défaut de Grafana (vert <80 / rouge ≥80) auraient fait passer `Hosts` et `VMs` en rouge dès que l'aggregat dépasse 80 hosts/VMs — un override force ces deux champs en vert fixe, indépendamment de la valeur.
2. **Détail par aggregat** (en dessous) : une section par aggregat, chacune avec un tableau des hosts — pour drill-down.

## Principe (section détail)

- **`$aggregate`** : liste déroulante multi-select, peuplée depuis le label `aggregates` exposé par l'exporter. Par défaut sur **All** → toutes les rows (une par aggregat) s'affichent. Sélectionner un ou plusieurs aggregats limite l'affichage à ceux-ci.
- Une **row Grafana est répétée par aggregat** (`repeat: aggregate` sur le panel de type `row`) : une section par aggregat, avec son titre (`Aggregat : <nom>`).
- Dans chaque row, un panel **Table** interroge directement Prometheus filtré sur `$aggregate` (pas de variable `$hostname` chaînée) : une ligne par host, colonnes `RAM %` / `CPU %` (cellules en jauge colorée) / `Statut` (vert "OK" ou rouge avec la raison si désactivé). Tri par défaut : RAM % décroissant, cliquable sur n'importe quelle colonne.

### Correctif appliqué : plus de repeat imbriqué

La version précédente utilisait un panel "carrés" répété par host (`repeat: hostname`), une variable elle-même chaînée sur `$aggregate` — un **repeat imbriqué** (row répétée par aggregat contenant un panel répété par host). Ce pattern s'est révélé cassé en pratique : chaque row affichait la liste globale de hosts au lieu de celle filtrée par son propre aggregat (Grafana ne ré-évalue pas la liste de valeurs d'une variable chaînée dans le contexte scopé d'une row répétée). Le panel Table actuel interroge directement `{aggregates=~".*$aggregate.*"}` dans sa propre requête (pas de variable intermédiaire), donc chaque row scope correctement sa requête — un seul niveau de repeat (la row), fiable.

⚠️ Les noms de colonnes après fusion (`Value #A`, `Value #B`, `disabledReason`, etc., renommés en `RAM %` / `CPU %` / `Statut` via la transformation "Organize fields") peuvent différer légèrement selon votre version de Grafana. Si les colonnes semblent vides ou mal nommées à l'import, ouvrez l'éditeur du panel → onglet **Transform** → vérifiez les noms produits par "Merge" et ajustez `renameByName`/`excludeByName` dans "Organize fields" (modifiable directement dans l'UI, pas besoin de retoucher le JSON).

## Calcul du "théorique"

```
RAM % = 100 * openstack_nova_memory_used_bytes / openstack_nova_memory_available_bytes
CPU % = 100 * openstack_nova_vcpus_used        / openstack_nova_vcpus_available
```

`*_available` reflète la capacité **physique** du hyperviseur (pas multipliée par les ratios d'overcommit), `*_used` la somme allouée aux instances (flavors). Le ratio peut donc dépasser 100 % **volontairement** si `cpu_allocation_ratio` / `ram_allocation_ratio` sont configurés > 1 dans `nova.conf` — c'est la définition même de l'usage "théorique"/alloué, par opposition à l'usage réel mesuré sur l'hyperviseur.

## ⚠️ Limite connue : label `aggregates`

L'exporter expose `aggregates` comme **une seule chaîne, aggregats séparés par des virgules** (ex. `"rack-a,gpu"`) quand un host appartient à plusieurs aggregats non-AZ. Conséquences :

- Le filtrage par panel (`aggregates=~".*$aggregate.*"`) fonctionne correctement même pour un host multi-aggregat.
- Mais le menu déroulant `$aggregate` peut lister des entrées composites (`"rack-a,gpu"`) en plus des noms simples, si des hosts cumulent plusieurs aggregats — Prometheus/Grafana ne peuvent pas éclater une valeur de label en plusieurs entrées de variable côté requête.
- Si vos aggregats sont mutuellement exclusifs (cas le plus courant), ce n'est pas un problème. Sinon, dites-le moi : on peut générer des *recording rules* Prometheus (une règle par aggregat connu) pour produire un label propre par aggregat.

## Synthèse par aggregat

En tête de chaque section aggregat, 4 tuiles :
- **Hosts** : `count(openstack_nova_vcpus_available{aggregates=~".*$aggregate.*"})`
- **RAM moyenne** / **CPU moyen** : moyenne des % par host (mêmes seuils vert/orange/rouge que les carrés). Les hosts avec une capacité à 0 (ex. bug de pinning en cours d'investigation) sont **exclus** du calcul via un filtre `and ... > 0`, pour ne pas polluer la moyenne avec un `+Inf`.
- **VMs** : `sum(openstack_nova_running_vms{aggregates=~".*$aggregate.*"})` — somme toutes tenants confondus (le label `tenant_id` de cette métrique est agrégé par le `sum()`).

## Colonne "Statut" (nova-compute disabled)

La colonne `Statut` du tableau reflète l'état admin de `nova-compute` pour chaque host : `openstack_nova_agent_state{service="nova-compute", adminState="disabled"}`, jointe par `hostname` à `openstack_nova_vcpus_available{aggregates=~...}` (`and on(hostname)`) pour ne garder que les hosts désactivés de cet aggregat.

- Cellule **verte "OK"** si le host est activé (pas de ligne dans la requête `C`, donc valeur nulle après la fusion → mappée sur "OK").
- Cellule **rouge** avec le texte de la `disabledReason` réelle si elle a été renseignée via `openstack compute service set --disable-reason "..."`, sinon `DISABLED` par défaut (substitution PromQL via `label_replace(..., "^$")`).
- ⚠️ Suppose que le label `hostname` de `openstack_nova_agent_state` (dérivé de `service.Host`) correspond au `hostname` des métriques hyperviseur (dérivé de `hypervisor.HypervisorHostname`) — cas standard, à vérifier dans Explore si la colonne reste vide pour des hosts que vous savez désactivés.
- Ne reflète que l'état **admin** (enabled/disabled), pas le heartbeat up/down du service.

## Seuils à ajuster

Les seuils actuels (vert < 80 %, orange 80-100 %, rouge ≥ 100 %) sont identiques pour RAM et CPU. Si votre `cpu_allocation_ratio` est > 1 (overcommit CPU volontaire, cas fréquent), un rouge à 100 % sur le CPU sera un faux signal en fonctionnement normal — indiquez-moi vos ratios configurés pour que j'ajuste les seuils CPU/RAM séparément (via `fieldConfig.overrides` par nom de série `RAM %` / `CPU %`).
