# TP4 : Stratégies d'héritage JPA (SINGLE_TABLE, JOINED, TABLE_PER_CLASS)

Projet Maven de démonstration JPA/Hibernate avec base H2 en mémoire.
Ce TP explore les 3 stratégies que JPA propose pour mapper l'héritage Java vers des tables SQL, en implémentant chacune avec un exemple métier différent.

## Objectifs du TP

- Comprendre les 3 stratégies d'héritage JPA : `SINGLE_TABLE`, `JOINED`, `TABLE_PER_CLASS`
- Implémenter chaque stratégie avec un modèle métier concret
- Tester les requêtes polymorphiques (interroger la classe parente pour récupérer toutes les sous-classes)
- Comparer les avantages/inconvénients de chaque approche

## Prérequis

- JDK 8 ou supérieur
- Maven 3.6+
- Un IDE (IntelliJ IDEA, Eclipse, NetBeans...)

## Structure du projet

```
hibernate-inheritance/
├── pom.xml
└── src/main/
    ├── java/com/example/
    │   ├── App.java                       # Classe principale (teste les 3 stratégies)
    │   └── model/
    │       ├── singletable/
    │       │   ├── Vehicule.java          # Classe abstraite parente
    │       │   ├── Voiture.java           # Sous-classe
    │       │   └── Moto.java              # Sous-classe
    │       ├── joined/
    │       │   ├── Employe.java           # Classe abstraite parente
    │       │   ├── Developpeur.java       # Sous-classe
    │       │   └── Manager.java           # Sous-classe
    │       └── tableperclass/
    │           ├── Produit.java           # Classe abstraite parente
    │           ├── Livre.java             # Sous-classe
    │           └── Electronique.java      # Sous-classe
    └── resources/META-INF/
        └── persistence.xml                # Configuration JPA / Hibernate / H2
```

## Stack technique

| Composant | Version |
|---|---|
| JPA API (javax.persistence) | 2.2 |
| Hibernate Core | 5.6.5.Final |
| Hibernate Validator | 6.2.0.Final |
| Jakarta EL (requis par Hibernate Validator) | 3.0.4 |
| Base de données H2 (en mémoire) | 2.1.214 |
| SLF4J (logs) | 1.7.36 |
| JUnit | 4.13.2 |

## Configuration (persistence.xml)

- **Unité de persistance** : `hibernate-inheritance`
- **Base** : H2 en mémoire (`jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1`)
- **hibernate.hbm2ddl.auto = create-drop** : le schéma est créé au démarrage et supprimé à l'arrêt
- **hibernate.show_sql = true** : affiche les requêtes SQL générées (essentiel pour voir la différence entre les 3 stratégies)
- Les 9 entités (3 hiérarchies × 3 classes) sont déclarées explicitement via `<class>`

## Les 3 stratégies d'héritage expliquées

### 1. `SINGLE_TABLE` — une seule table pour toute la hiérarchie

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "type_vehicule", discriminatorType = DiscriminatorType.STRING)
public abstract class Vehicule { ... }

@Entity
@DiscriminatorValue("VOITURE")
public class Voiture extends Vehicule { ... }

@Entity
@DiscriminatorValue("MOTO")
public class Moto extends Vehicule { ... }
```

Une seule table `vehicules` contient toutes les colonnes de `Vehicule`, `Voiture` et `Moto`. Une colonne technique `type_vehicule` (discriminateur) indique le type réel de chaque ligne.

| Avantage | Inconvénient |
|---|---|
| Requêtes rapides (jamais de jointure) | Beaucoup de colonnes NULL (une moto n'a pas de `nombrePortes`) |

### 2. `JOINED` — une table par classe, reliées par jointure

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Employe { ... }

@Entity
@Table(name = "developpeurs")
public class Developpeur extends Employe { ... }

@Entity
@Table(name = "managers")
public class Manager extends Employe { ... }
```

Chaque classe a sa propre table (`employes`, `developpeurs`, `managers`). Les tables filles ne contiennent que leurs colonnes spécifiques + une clé étrangère vers `employes`. Hibernate fait une jointure SQL pour reconstituer l'objet complet.

| Avantage | Inconvénient |
|---|---|
| Structure normalisée, aucune colonne NULL | Jointures = requêtes plus lentes |

### 3. `TABLE_PER_CLASS` — une table complète par classe concrète

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Produit { ... }

@Entity
@Table(name = "livres")
public class Livre extends Produit { ... }

@Entity
@Table(name = "electroniques")
public class Electronique extends Produit { ... }
```

Chaque sous-classe a sa propre table complète, qui **duplique** les colonnes de la classe parente (`nom`, `prix`, `description`...). Utilise `GenerationType.AUTO` (et non `IDENTITY`) pour garantir des ids uniques entre toutes les tables.

| Avantage | Inconvénient |
|---|---|
| Aucune duplication inutile, tables autonomes | Requêtes polymorphiques lentes (UNION SQL entre toutes les tables) |

## Tableau récapitulatif

| Stratégie | Nombre de tables | Colonnes NULL | Perf. lecture directe | Perf. requête polymorphique |
|---|---|---|---|---|
| `SINGLE_TABLE` | 1 | Beaucoup | Rapide | Rapide |
| `JOINED` | 1 par classe | Aucune | Moyenne (jointures) | Moyenne |
| `TABLE_PER_CLASS` | 1 par classe concrète | Aucune | Rapide | Lente (UNION) |

## Modèles de données

### SINGLE_TABLE : Vehicule → Voiture / Moto
| Classe | Champs spécifiques |
|---|---|
| Vehicule (abstraite) | id, marque, modele, anneeFabrication, prix |
| Voiture | + nombrePortes, climatisation, typeCarburant |
| Moto | + cylindree, typeTransmission |

### JOINED : Employe → Developpeur / Manager
| Classe | Champs spécifiques |
|---|---|
| Employe (abstraite) | id, nom, prenom, email, dateEmbauche |
| Developpeur | + langage, specialite, anneeExperience |
| Manager | + service, nombreSubordonnes, bonus |

### TABLE_PER_CLASS : Produit → Livre / Electronique
| Classe | Champs spécifiques |
|---|---|
| Produit (abstraite) | id, nom, prix, description, dateCreation |
| Livre | + auteur, isbn, nombrePages, editeur |
| Electronique | + marque, modele, garantieMois, caracteristiques |

## Installation et exécution

### 1. Récupérer les dépendances

```bash
mvn clean install
```

Dans IntelliJ : clic droit sur `pom.xml` → **Maven → Reload project**.

### 2. Lancer l'application

```bash
mvn clean compile exec:java -Dexec.mainClass="com.example.App"
```

Ou directement dans l'IDE : clic droit sur `App.java` → **Run**.

## Ce que fait `App.java`

<img width="1270" height="674" alt="1" src="image/Capture d'écran 2026-09-30 001740.png" />

<img width="1270" height="674" alt="1" src="image/Capture d'écran 2026-09-30 001747.png" />

<img width="1270" height="674" alt="1" src="image/Capture d'écran 2026-09-30 001754.png" />
