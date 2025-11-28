# Pipeline explannation

## 1. Introduction 

The pipeline explained in this paper is meant to make it easier to take in, pull out, and understand different types of administrative documents like PDFs and XML files. This system is built to manage a lot of information effectively, while also keeping track of the data and its organization. 

The process starts with a MinIO environment that serves as storage for files, neatly sorted by the type of document. From this storage area, everything happens automatically without needing anyone to do it manually. This includes finding the files, downloading them, pulling out the content, reorganizing it with a language model, saving it temporarily in an SQLite database, and finally adding it to a ClickHouse database for analysis. 

The whole pipeline is run by a main. py file, which makes sure each part happens in the right order. Each part of the system is separated into its own module. This separation helps make the code easier to read, as well as easier to update and expand in the future.

## 2. diagram and overview

Voici une première représentation globale du fonctionnement du pipeline, depuis la découverte des fichiers jusqu’à l’ingestion dans ClickHouse :

<figure>
  <img src="images/pipeline.png" alt="Pipeline" width="500">
  <figcaption>Figure 1 — architecture </figcaption>
</figure>



The whole setup works in a straight line, but every part is capable of fixing its own mistakes. Think of it as a series of adjustments that converts the initial document into its ultimate form, which is ready to be examined in ClickHouse. 

The process starts by checking the MinIO storage to find the right folders and files to work on. After figuring out which documents to use, each one is downloaded to the computer’s memory and then sent to the extraction section. This part deals with PDFs by first getting the original text from them, and if that text isn’t good enough or hard to read, it automatically uses a method called OCR to extract it instead. XML files are handled differently; they are directly loaded, cleaned up, and turned into a text format to be ready for the next steps. 

Once this extraction is done, all the collected data is kept in a temporary SQLite database, and then it is backed up to a different MinIO bucket. This backup is really important for keeping track of the process, as it stores a complete record of the original data before any changes are made by a language model. 

The following phase is where the main business actions happen. Text and XML pieces are sent to a Language Learning Model (LLM) through an API. The model gets a request that is automatically created from a JSON format that describes the fields needed for that type of document. The results from the model are then cleaned up, corrected, and checked. In the end, the confirmed data gets sent to ClickHouse, where it is added in a way that prevents duplicates.

## 3. Retrieving and discovering documents in MinIO

The pipeline starts by looking into the input MinIO bucket using a module called db_io/data_io. py. This module sets up a MinIO client that uses settings from the environment. Its main job is to find the structure of the documents. 

The function used for this checks a specific starting point (BASE_DOCS_PATH) and finds all the subfolders that are there. Out of these, only some types of documents are seen as important. These types are listed in an internal list based on what the project needs. The code then looks through the folders and lists the exact files that need to be processed. The outcome is a dictionary that connects each document type with a list of files to handle. 

After finishing this discovery step, the pipeline downloads each raw file from MinIO in a binary format. Then, these files are moved on to the extraction phase.

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

