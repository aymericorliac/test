

## 1. Introduction générale

Le pipeline présenté dans ce document est conçu pour automatiser l’ingestion, l’extraction et l’analyse sémantique de documents administratifs variés (PDF, XML). Ce système a été pensé pour traiter des volumes importants, de manière robuste, tout en garantissant la traçabilité et la structuration des données.

Le point de départ du pipeline est un environnement MinIO qui joue le rôle de stockage objet et contient des dossiers organisés par type de document. À partir de cet environnement, l’ensemble du processus se déroule sans intervention humaine : découverte des fichiers, téléchargement, extraction du contenu, restructuration grâce à un modèle de langage, sauvegarde intermédiaire dans une base SQLite, et enfin ingestion dans une base analytique ClickHouse.

Le pipeline est entièrement orchestré par le fichier main.py, qui coordonne successivement toutes les étapes. Chaque composant du système est isolé dans un module dédié, ce qui favorise non seulement la lisibilité du code, mais aussi sa maintenabilité et sa capacité à évoluer dans le futur.

## 2. Vue d’ensemble de l’architecture

Voici une première représentation globale du fonctionnement du pipeline, depuis la découverte des fichiers jusqu’à l’ingestion dans ClickHouse :

                                 ┌──────────────────────────┐
                                 │        MinIO (input)      │
                                 │   Dossiers + fichiers     │
                                 └───────────────┬──────────┘
                                                 │
                                                 ▼
                                 ┌──────────────────────────┐
                                 │   Listing des dossiers   │
                                 │  et récupération des PDF │
                                 └───────────────┬──────────┘
                                                 │
                                                 ▼
                      ┌─────────────────────────────────────────────────────────────┐
                      │                 Extraction PDF / XML                        │
                      │  - Texte natif (pdfplumber)                                 │
                      │  - Tables                                                   │
                      │  - OCR automatique (EasyOCR + PyMuPDF)                      │
                      │  - Extraction XML                                           │
                      └───────────────┬─────────────────────────────────────────────┘
                                      │
                                      ▼
                      ┌─────────────────────────────────────────────────────────────┐
                      │     Sauvegarde SQLite + Upload sur MinIO (backups)          │
                      │        Base regroupant extraction PDF + tables + XML        │
                      └───────────────┬─────────────────────────────────────────────┘
                                      │
                                      ▼
                      ┌─────────────────────────────────────────────────────────────┐
                      │     Traitement sémantique via LLM                           │
                      │   (prompt → réponse → parsing → validation schema)          │
                      └───────────────┬─────────────────────────────────────────────┘
                                      │
                                      ▼
                      ┌─────────────────────────────────────────────────────────────┐
                      │               Ingestion dans ClickHouse                     │
                      │      Nettoyage → déduplication → insertion MergeTree        │
                      └─────────────────────────────────────────────────────────────┘


L’architecture globale suit une logique linéaire, mais chaque étape peut gérer ses propres erreurs ou corrections automatiques. On peut la décrire comme une succession de transformations du document initial jusqu’à sa version finale prête pour l’analyse dans ClickHouse.

Le pipeline commence par explorer le stockage MinIO afin de déterminer les dossiers et les fichiers pertinents à traiter. Une fois les documents identifiés, chaque fichier est téléchargé en mémoire puis transmis au module d’extraction. Ce module traite les PDF en extrayant d’abord le texte natif, puis bascule automatiquement vers une extraction OCR si le texte est jugé insuffisant ou illisible. Les fichiers XML sont interprétés différemment : ils sont chargés tels quels, nettoyés et convertis en une version texte prête à être traitée par la suite.

Après cette phase d’extraction, toutes les informations obtenues sont stockées dans une base SQLite temporaire, qui est ensuite sauvegardée dans un bucket MinIO séparé. Cette sauvegarde joue un rôle essentiel dans la traçabilité du pipeline : elle permet de garder une archive complète de l’extraction brute, avant toute transformation sémantique par un modèle de langage.

L’étape suivante constitue le cœur du traitement métier. Les extraits textuels et XML sont soumis à un modèle de langage (LLM) via une API. Le modèle reçoit un prompt construit automatiquement à partir d’un schéma JSON qui définit les champs attendus pour ce type de document. La sortie du modèle est ensuite analysée, nettoyée, corrigée et validée. Enfin, les données validées sont envoyées dans ClickHouse, où elles sont insérées de manière dédupliquée.  

## 3. Récupération et découverte des documents dans MinIO

Le pipeline débute par l’exploration du bucket MinIO d’entrée, via le module db_io/data_io.py. Ce module initialise un client MinIO configuré à partir des variables d’environnement. Il est entièrement responsable de la découverte de l’arborescence de documents.

La fonction utilisée pour cela inspecte un chemin racine défini (BASE_DOCS_PATH) et identifie tous les sous-dossiers présents. Parmi ceux-ci, seuls certains types de documents sont considérés comme pertinents. Ils sont définis dans une liste interne selon les besoins du projet. Le code analyse ensuite le contenu de ces dossiers et en extrait la liste exacte des fichiers à traiter. Le résultat est un dictionnaire qui associe chaque type de document à une liste de fichiers à ingérer.

Dans sa version actuelle, le pipeline est configuré pour ne traiter qu’un seul type de document afin de faciliter les tests et la validation. Cependant, cette restriction n’est qu’un choix temporaire et peut être retirée pour étendre le traitement à tous les dossiers détectés.

Une fois cette étape de découverte réalisée, le pipeline télécharge chaque fichier brut depuis MinIO sous forme binaire. Ces fichiers téléchargés sont ensuite transmis à la phase d’extraction.

## 4. Extraction du contenu (PDF, OCR et XML)

L’extraction des fichiers constitue la partie la plus technique du pipeline. Elle est assurée par la classe DataExtractor du module extract.py, qui combine plusieurs stratégies selon la nature du document et sa qualité.

### 4.1. Schéma global de l’extraction

Le fonctionnement peut être représenté de manière synthétique ainsi :

                ┌────────────────────────────────┐
                │           Fichier PDF           │
                └───────────────┬────────────────┘
                                │
                                ▼
                ┌────────────────────────────────┐
                │  Extraction texte natif (PDF)   │
                │           + tables              │
                └───────────────┬────────────────┘
                        Est-ce lisible ? (score)
                                │
             ┌──────────────────┴──────────────────┐
             │                                     │
             ▼                                     ▼
   Résultat natif conservé            OCR automatique déclenché
             │                                     │
             ▼                                     ▼
   Nettoyage / normalisation            EasyOCR + PyMuPDF → Texte OCR
             │                                     │
             └──────────────────┬──────────────────┘
                                ▼
                ┌────────────────────────────────┐
                │  Structure unifiée d’extraction │
                └────────────────────────────────┘


                ┌────────────────────────────────┐
                │            Fichier XML          │
                └────────────────────────────────┘
                                │
                                ▼
                       Extraction brute XML

### 4.2. Extraction PDF native

Lorsque le fichier est un PDF, la première tentative d’extraction est réalisée via la bibliothèque pdfplumber. Cette méthode permet d’obtenir directement le texte numérique du document, lorsqu’il existe, ainsi que la structure des pages. En parallèle, les éventuels tableaux contenus dans le PDF sont détectés et extraits, ce qui est particulièrement utile pour les documents comptables ou déclaratifs.

Un score de qualité du texte est ensuite calculé. Ce score permet de déterminer si le contenu extrait est exploitable ou si le document est probablement une version scannée dépourvue de texte numérique. Cette étape est essentielle, car un document scanné nécessitera un traitement complètement différent.

### 4.3. Passage automatique par l’OCR

Lorsque le texte obtenu est trop bruité ou insuffisant, le pipeline bascule automatiquement vers une extraction OCR, qui consiste à produire une image de chaque page PDF puis à appliquer un modèle de reconnaissance optique de caractères.

Cette version OCR repose sur EasyOCR et PyMuPDF. PyMuPDF est utilisé pour rendre les pages en images haute résolution, tandis qu’EasyOCR analyse ces images et restitue un texte brut. À l’issue de cette reconnaissance, le texte est consolidé, nettoyé et normalisé avec la bibliothèque ftfy, ce qui permet d’obtenir un rendu lisible même pour des documents scannés de qualité médiocre.

### 4.4. Extraction XML

La gestion des fichiers XML est plus directe. Le pipeline lit le contenu complet du fichier, vérifie la taille du document, nettoie les éventuels caractères problématiques, puis retourne une structure simple contenant le texte XML. Cette approche garantit une parfaite conservation du contenu original, ce qui est essentiel pour les étapes suivantes.

## 5. Sauvegarde en base SQLite et archivage MinIO

Après l’extraction, le pipeline sauvegarde l’ensemble des données dans une base SQLite temporaire. Cette étape est gérée par tasks/sqlite_saver.py et constitue une caractéristique importante du pipeline.

L’idée consiste à conserver un instantané complet de l’extraction brute sous un format universel, compact et facilement transportable : SQLite. Le système crée ainsi une base par type de document et y enregistre :

les extraits textuels PDF,

les tableaux identifiés dans les PDF,

les contenus XML,

un ensemble de métadonnées liées à l’extraction.

Une fois la base SQLite constituée, elle est stockée sous forme d’un fichier temporaire, chargé en mémoire puis envoyé dans le bucket MinIO dédié aux backups. Le nom du fichier SQLite inclut automatiquement un horodatage ainsi que le type de document correspondant.

Cette stratégie présente plusieurs avantages. D’abord, elle offre un mécanisme de sauvegarde automatique fiable, en cas d’erreur au cours des traitements ultérieurs. Ensuite, elle permet de rerun le pipeline à partir des données extraites sans avoir à recharger ou réextraire les documents sources.

## 6. Traitement sémantique et structuration via un modèle de langage (LLM)

Une fois les données extraites et sauvegardées, la prochaine étape consiste à transformer ces informations brutes en données structurées exploitables. Ce traitement sémantique est réalisé via un modèle de langage, appelé à travers une API locale ou externe. Cette étape est implémentée dans process.py.

### 6.1. Schéma général du traitement LLM

Le fonctionnement global peut être représenté ainsi :

                     ┌───────────────────────────────────────┐
                     │      Texte extrait (PDF/XML)           │
                     └──────────────────────┬────────────────┘
                                            │
                                            ▼
                     ┌───────────────────────────────────────┐
                     │ Construction du prompt à partir        │
                     │ d’un schema JSON + texte extrait      │
                     └──────────────────────┬────────────────┘
                                            │
                                            ▼
                     ┌───────────────────────────────────────┐
                     │      Appel API vers le modèle LLM      │
                     └──────────────────────┬────────────────┘
                                            │
                                            ▼
                     ┌───────────────────────────────────────┐
                     │ Parsing, nettoyage des valeurs         │
                     │ (dates, montants, CPF/CNPJ, etc.)      │
                     └──────────────────────┬────────────────┘
                                            │
                                            ▼
                     ┌───────────────────────────────────────┐
                     │ Validation stricte via schema JSON     │
                     └──────────────────────┬────────────────┘
                                            │
                                            ▼
                     ┌───────────────────────────────────────┐
                     │ Donnée structurée prête pour ingestion │
                     └───────────────────────────────────────┘

### 6.2. Construction du prompt

Le code commence par charger un schéma JSON spécifique à chaque type de document. Ce schéma définit les champs attendus ainsi que leurs types. Le texte extrait précédemment est ensuite injecté dans un prompt qui décrit précisément au modèle de langage les informations qu’il doit identifier.

La construction du prompt est particulièrement importante. Il doit être suffisamment explicite pour guider correctement le modèle, tout en évitant les ambiguïtés. Il contient donc la liste des champs exigés, des exemples de formats attendus, ainsi que l’extrait textuel complet du document.

### 6.3. Requête vers le modèle de langage

Une fois le prompt préparé, un appel HTTP est effectué vers l’API du modèle. Le modèle renvoie une réponse contenant du JSON, qui est ensuite interprété. Le pipeline prévoit la possibilité que le modèle fournisse un texte contenant du JSON, ou bien un JSON incomplet. De ce fait, un parsing robuste est implémenté pour extraire la structure JSON correcte quel que soit le format exact renvoyé.

### 6.4 Nettoyage et normalisation de la réponse

La sortie du modèle est ensuite nettoyée : correction des montants numériques, formatage normalisé des dates, interprétation correcte des CPF, CNPJ, ou autres identifiants administratifs. Cette étape permet de transformer des valeurs textuelles imprécises en formats cohérents utilisables dans une base analytique.

### 6.5 Validation par rapport au schéma

Une fois les données structurées, elles sont comparées au schéma initial. Toute divergence ou absence de champ est consignée dans un ensemble de métadonnées qui identifie les erreurs ou inconsistances. Cela permet à la fois de tracer la qualité de l’extraction et de définir des mécanismes de correction dans le futur.

## 7. Insertion dans ClickHouse et déduplication

La dernière étape du pipeline consiste à envoyer les données finales dans une base ClickHouse. L’implémentation se trouve dans db_io/clickhouse_io.py. Le code commence par établir une connexion au serveur ClickHouse grâce aux paramètres définis dans les variables d’environnement.

La première opération consiste à vérifier l’existence de la base de données et de la table cible. Si elles ne sont pas présentes, elles sont créées automatiquement. La table utilise un moteur MergeTree, adapté à l’ingestion de données structurées avec tri par timestamp.

Avant de procéder à l’insertion, le pipeline réalise un nettoyage des données afin d’en retirer toutes les métadonnées internes, comme la réponse brute du modèle ou les indicateurs de validation. Seuls les champs métier finaux sont conservés.

Pour garantir l’absence de doublons, le pipeline compare les données à insérer aux données déjà présentes dans ClickHouse. Il sérialise chaque entrée en JSON trié, puis vérifie si cette représentation existe déjà dans la table. Ce mécanisme permet au pipeline d’être réexécuté sans risquer de réinsérer plusieurs fois les mêmes documents.

Une fois la déduplication effectuée, les nouvelles données sont insérées dans ClickHouse, prêtes pour les analyses ultérieures.

## 8. Conclusion générale

Le pipeline mis en place constitue un système complet et structuré, allant de l’ingestion brute d’un document jusqu’à son intégration finale dans une base analytique. Les différentes étapes — découverte, extraction, OCR, sauvegarde, structuration via LLM, validation, déduplication et ingestion — créent un flux cohérent et robuste.

Cette architecture est pensée pour être évolutive. Il est possible d’ajouter de nouveaux types de documents en ajoutant simplement un schéma JSON, ou encore de remplacer le modèle de langage par un autre sans altérer les modules existants. Le pipeline est également résilient : les erreurs réseau, les PDFs scannés illisibles, ou les retours erronés du modèle sont pris en charge par différents mécanismes de fallback.

