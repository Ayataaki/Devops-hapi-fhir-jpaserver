# HAPI FHIR Observabilite — Projet DevOps ENSIAS 2026

Observabilite d'un serveur de sante FHIR avec Prometheus, Grafana, Loki et Tempo.
Soutenance : 5 Mars 2026

---

## Equipe

Membre A — Infrastructure & Docker
Membre B — Observabilite & Dashboards
Membre C — Chaos Engineering & Tests
Membre D — Analyse & Redaction

---

## Description du projet

Ce projet mesure l'impact de l'observabilite sur la detection et resolution d'incidents dans un systeme de sante reel (HAPI FHIR).

On compare 4 niveaux d'observabilite. Le niveau 0 n'a aucun outil de monitoring. Le niveau 1 ajoute les logs avec Loki, Promtail et Grafana. Le niveau 2 ajoute en plus les metriques avec Prometheus. Le niveau 3 est l'observabilite complete avec Tempo pour les traces.

On simule 6 incidents et on mesure le MTTD (temps de detection) et le MTTR (temps de resolution) pour chaque niveau.

---

## Lancement rapide

Prerequis : Docker Desktop installe et Git.

    git clone https://github.com/hapifhir/hapi-fhir-jpaserver-starter.git
    cd hapi-fhir-jpaserver-starter

Pour lancer le niveau 0 sans monitoring :

    docker compose up hapi-fhir hapi-fhir-postgres -d

Pour lancer le niveau 1 avec logs :

    docker compose --profile logs up -d

Pour lancer le niveau 2 avec logs et metriques :

    docker compose --profile logs --profile metrics up -d

Pour lancer le niveau 3 complet :

    docker compose --profile logs --profile metrics --profile traces up -d

---

## Acces aux interfaces

HAPI FHIR est accessible sur http://localhost:8080/fhir/metadata
Grafana est accessible sur http://localhost:3000 avec les identifiants admin / admin
Prometheus est accessible sur http://localhost:9090
Loki est accessible sur http://localhost:3100

---

## Scenarios d'incidents

6 incidents sont simules durant le projet. Le premier est l'arret de la base de donnees avec la commande docker compose stop hapi-fhir-postgres. Le deuxieme est une surcharge CPU en envoyant 1000 requetes par seconde avec k6. Le troisieme est une fuite memoire en limitant le conteneur a 256 Mo. Le quatrieme est une latence reseau artificielle de 2 secondes via tc netem. Le cinquieme est un envoi massif de requetes FHIR invalides pour generer des erreurs 500. Le sixieme est le remplissage du volume disque avec des fichiers temporaires.

Pour chaque experience, le protocole est le suivant : lancer le niveau, attendre 1 minute de stabilisation, noter l'heure T0 puis injecter l'incident, noter T1 quand le probleme est detecte, noter T2 quand il est resolu. Le MTTD est T1 moins T0 et le MTTR est T2 moins T0. On relance ensuite avec docker compose down.

---

## Resultats

Les resultats sont stockes dans results/experiments.xlsx avec 24 lignes au total (4 niveaux x 6 incidents). Les graphiques sont generes automatiquement avec la commande python analyze_results.py.

---

## Structure du projet

Le projet contient un fichier docker-compose.yml principal, un dossier grafana avec les dashboards et datasources, un dossier prometheus avec sa configuration, un dossier loki, un dossier tempo, un dossier incidents avec les 6 scripts shell, un dossier results avec le fichier Excel et les graphiques, et le script analyze_results.py.

---

## Concepts cles

Le MTTD est le Mean Time To Detect, c'est-a-dire le temps moyen pour detecter un incident. Le MTTR est le Mean Time To Resolve, le temps moyen pour le resoudre. FHIR est le standard international d'echange de donnees de sante. L'observabilite est la capacite a comprendre un systeme via ses logs, metriques et traces. Le Chaos Engineering consiste a injecter des pannes volontaires pour tester la resilience du systeme.

---

ENSIAS — Projet Fiche 3 — Fevrier 2026
