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
2. **Hosts** (en dessous) : une seule vue à plat, un carré par host, filtrée par `$aggregate` — voir ci-dessous.

## Principe (section Hosts)

- **`$aggregate`** : liste déroulante multi-select. Par défaut sur **All** → tous les hosts (toutes aggregats confondus) s'affichent dans la grille. Sélectionner un seul aggregat → seuls ses hosts s'affichent. C'est un simple filtre, pas une répétition par aggregat.
- **`$hostname`** : liste des hosts, chaînée sur `$aggregate` (`label_values(openstack_nova_vcpus_available{aggregates=~".*$aggregate.*"}, hostname)`).
- Un seul panel **Bar gauge**, répété par `$hostname` (**un seul niveau de repeat**, pas de row imbriquée) : un carré par host, avec `R` (RAM %) au-dessus de `C` (CPU %), labels raccourcis et panel compact (`h: 3`).
- Une 3ᵉ ligne apparaît **uniquement** sur les hosts dont `nova-compute` est désactivé (`openstack_nova_agent_state{adminState="disabled"}`) : bande rouge pleine avec la `disabledReason` réelle si renseignée, sinon `DISABLED` par défaut.

### Pourquoi un seul niveau de repeat cette fois

Une version précédente imbriquait une row répétée par aggregat contenant un panel répété par host (deux niveaux) — Grafana ne ré-évaluait pas correctement la liste de hosts par row, et chaque section affichait tous les hosts au lieu des siens. En repassant à une seule vue (pas de row par aggregat, juste `$aggregate` comme filtre du panel unique), il n'y a plus qu'un seul niveau de repeat (`$hostname`), qui est le pattern fiable déjà utilisé pour la section "Vue d'ensemble" plus haut.

- `max` est fixé à 100 sur RAM/CPU — si un host est en overcommit (> 100 %), la barre se remplit entièrement mais le texte affiché reste la vraie valeur (ex. `320 %`).
- Si les barres paraissent trop tassées, réduisez `options.minVizHeight` (actuellement `10`) dans le JSON, ou remontez légèrement `gridPos.h`.

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

## Indicateur "nova-compute disabled"

- ⚠️ Suppose que le label `hostname` de `openstack_nova_agent_state` (dérivé de `service.Host`) correspond au `hostname` des métriques hyperviseur (dérivé de `hypervisor.HypervisorHostname`) — cas standard, à vérifier dans Explore si la bande rouge n'apparaît jamais pour des hosts que vous savez désactivés.
- Ne reflète que l'état **admin** (enabled/disabled), pas le heartbeat up/down du service.
- Note technique déjà signalée : un panel Bar Gauge ne peut pas teinter tout son contour depuis une métrique différente de celle affichée dans chaque barre — la bande rouge pleine (3ᵉ ligne) est le compromis le plus fiable pour signaler l'état sans repasser par un type de panel totalement différent.

## Seuils à ajuster

Les seuils actuels (vert < 80 %, orange 80-100 %, rouge ≥ 100 %) sont identiques pour RAM et CPU. Si votre `cpu_allocation_ratio` est > 1 (overcommit CPU volontaire, cas fréquent), un rouge à 100 % sur le CPU sera un faux signal en fonctionnement normal — indiquez-moi vos ratios configurés pour que j'ajuste les seuils CPU/RAM séparément (via `fieldConfig.overrides` par nom de série `RAM %` / `CPU %`).
