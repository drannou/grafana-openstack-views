# Nova - Usage théorique des hosts par aggregat

Dashboard : `nova-hosts-theoretical-usage.json`

## Prérequis

- Prometheus scrapant [`openstack-exporter`](https://github.com/openstack-exporter/openstack-exporter) (le projet actif, pas l'ancien `niedbalski/openstack-exporter`).
- Une datasource Prometheus configurée dans Grafana (le dashboard demande de la sélectionner via la variable `$datasource` à l'import).

## Import

Grafana → Dashboards → New → Import → coller le JSON (ou "Upload").

## Principe

- **`$aggregate`** : liste déroulante (single-select) peuplée depuis le label `aggregates` exposé par l'exporter. Choisir un aggregat filtre la liste de hosts affichés.
- **`$hostname`** : liste des hosts membres de l'aggregat sélectionné (chaînée sur `$aggregate`).
- Un panel **Stat carré est répété** (`repeat: hostname`) pour chaque host : titre = hostname, puis RAM % et CPU % empilés, avec fond coloré (vert/orange/rouge) selon des seuils.

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

## Seuils à ajuster

Les seuils actuels (vert < 80 %, orange 80-100 %, rouge ≥ 100 %) sont identiques pour RAM et CPU. Si votre `cpu_allocation_ratio` est > 1 (overcommit CPU volontaire, cas fréquent), un rouge à 100 % sur le CPU sera un faux signal en fonctionnement normal — indiquez-moi vos ratios configurés pour que j'ajuste les seuils CPU/RAM séparément (via `fieldConfig.overrides` par nom de série `RAM %` / `CPU %`).
