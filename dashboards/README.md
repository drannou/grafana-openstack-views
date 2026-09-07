# Nova - Usage théorique des hosts par aggregat

Dashboard : `nova-hosts-theoretical-usage.json`

## Prérequis

- Prometheus scrapant [`openstack-exporter`](https://github.com/openstack-exporter/openstack-exporter) (le projet actif, pas l'ancien `niedbalski/openstack-exporter`).
- Une datasource Prometheus configurée dans Grafana (le dashboard demande de la sélectionner via la variable `$datasource` à l'import).

## Import

Grafana → Dashboards → New → Import → coller le JSON (ou "Upload").

## Principe

- **`$aggregate`** : liste déroulante multi-select, peuplée depuis le label `aggregates` exposé par l'exporter. Par défaut sur **All** → toutes les rows (une par aggregat) s'affichent. Sélectionner un ou plusieurs aggregats limite l'affichage à ceux-ci.
- Une **row Grafana est répétée par aggregat** (`repeat: aggregate` sur le panel de type `row`) : une section par aggregat, avec son titre (`Aggregat : <nom>`).
- **`$hostname`** : liste des hosts membres de l'aggregat courant (chaînée sur `$aggregate`).
- Dans chaque row, un panel **Bar gauge est répété par host** (`repeat: hostname`) : titre = hostname, barre RAM au-dessus, barre CPU en dessous, colorées (vert/orange/rouge) selon des seuils.

### ⚠️ À vérifier à l'import : repeat imbriqué (row + panel)

La combinaison "row répétée par aggregat" contenant "panel répété par host filtré sur cet aggregat" est un **repeat imbriqué**. Ce pattern est officiellement supporté depuis les versions récentes de Grafana (moteur "Scenes", Grafana ≥ 10.3), mais sur des versions plus anciennes le ré-scoping de `$hostname` par row peut ne pas fonctionner correctement (toutes les rows affichant alors les mêmes hosts). **À tester en premier après import** : sélectionnez au moins 2 aggregats différents et vérifiez que chaque row affiche bien des hosts différents et cohérents avec son propre aggregat. Si ce n'est pas le cas, dites-le moi (avec votre version de Grafana) — je basculerai sur une variante sans repeat imbriqué (une row par aggregat définie explicitement, ou un dashboard généré par script à partir de la liste réelle d'aggregats).

## Tri par RAM/CPU

Grafana ne permet pas de trier des panels répétés (les carrés) par une valeur de métrique — seul l'ordre du label (ici `hostname`, alphabétique) est utilisable pour l'ordre du repeat. Le tri par valeur RAM/CPU n'existe nativement que sur un panel **Table** (tri au clic sur la colonne) ; c'est un compromis assumé en gardant le rendu "carrés".

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

## Rendu visuel

Panel type **Bar gauge**, orienté horizontal : une barre RAM au-dessus d'une barre CPU, par host. `max` est fixé à 100 — si un host est en overcommit (> 100 %), la barre se remplit entièrement mais le texte affiché reste la vraie valeur (ex. `320 %`).

## Synthèse par aggregat

En tête de chaque section aggregat, 4 tuiles :
- **Hosts** : `count(openstack_nova_vcpus_available{aggregates=~".*$aggregate.*"})`
- **RAM moyenne** / **CPU moyen** : moyenne des % par host (mêmes seuils vert/orange/rouge que les carrés). Les hosts avec une capacité à 0 (ex. bug de pinning en cours d'investigation) sont **exclus** du calcul via un filtre `and ... > 0`, pour ne pas polluer la moyenne avec un `+Inf`.
- **VMs** : `sum(openstack_nova_running_vms{aggregates=~".*$aggregate.*"})` — somme toutes tenants confondus (le label `tenant_id` de cette métrique est agrégé par le `sum()`).

## Indicateur "nova-compute disabled"

Une 3ᵉ barre apparaît **uniquement** sur les hosts dont le service `nova-compute` est administrativement désactivé (`openstack compute service set --disable`) : bande rouge pleine avec le texte `⚠ DISABLED`. Basée sur `openstack_nova_agent_state{service="nova-compute", adminState="disabled"}`.

- Un host activé n'affiche **aucune** 3ᵉ barre (requête sans résultat) — seuls RAM/CPU restent visibles, panel légèrement plus compact.
- ⚠️ Cet indicateur suppose que le label `hostname` de `openstack_nova_agent_state` (dérivé de `service.Host`) correspond au `hostname` des métriques hyperviseur (dérivé de `hypervisor.HypervisorHostname`). C'est le cas standard, mais vérifiez dans Explore : `openstack_nova_agent_state{service="nova-compute"}` — si les valeurs `hostname` ne matchent pas celles de `openstack_nova_vcpus_available`, la barre ne s'affichera jamais et il faudra adapter le label utilisé.
- Ne reflète que l'état **admin** (enabled/disabled), pas le heartbeat up/down du service (un service down mais toujours enabled n'affichera pas cette barre).

## Seuils à ajuster

Les seuils actuels (vert < 80 %, orange 80-100 %, rouge ≥ 100 %) sont identiques pour RAM et CPU. Si votre `cpu_allocation_ratio` est > 1 (overcommit CPU volontaire, cas fréquent), un rouge à 100 % sur le CPU sera un faux signal en fonctionnement normal — indiquez-moi vos ratios configurés pour que j'ajuste les seuils CPU/RAM séparément (via `fieldConfig.overrides` par nom de série `RAM %` / `CPU %`).
