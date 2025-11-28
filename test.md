Documentation complète du pipeline d’ingestion, d’extraction et de traitement de documents

(Version longue, détaillée, avec schémas)

1. Introduction générale

Le pipeline décrit dans ce document permet de réaliser une ingestion automatisée et complète de documents administratifs ou comptables, principalement des fichiers PDF et XML déposés dans un environnement MinIO. L’objectif de ce système est de transformer des documents bruts en données structurées, validées et exploitables au sein d’une base analytique ClickHouse.

Le fonctionnement repose sur une succession d’étapes, chacune prenant en entrée le résultat de la précédente. Le processus inclut notamment la détection des documents, leur téléchargement, l’extraction du texte ou des tables, l’application automatique d’un OCR lorsque nécessaire, la sauvegarde complète en bases SQLite temporaires, puis une phase de traitement sémantique via un modèle de langage. Finalement, les données sont validées, dédupliquées et insérées dans une base ClickHouse.

L’ensemble de ce pipeline est orchestré par un script central, main.py. Chaque composant du système est développé dans un module distinct, de manière à obtenir une architecture propre, lisible et facile à faire évoluer.

2. Vue d’ensemble de l’architecture

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


Cette vision d’ensemble montre chaque étape comme un bloc indépendant. Les sections suivantes détaillent leur fonctionnement interne.

3. Découverte et téléchargement des documents depuis MinIO

Le pipeline commence par analyser le contenu du bucket MinIO. Tout le travail de découverte est réalisé dans db_io/data_io.py. Le module initialise un client MinIO configuré via les variables d’environnement.

Le rôle de ce composant est de traverser le préfixe défini (généralement un dossier racine tel que “SKZ Oberle - Amostra DOCS/”), d’identifier les sous-dossiers correspondant à des types de documents, puis de lister les fichiers présents dans chacun de ces dossiers. Les données sont ensuite renvoyées sous forme d’un dictionnaire Python où chaque clé correspond à un type de document et chaque valeur à la liste des fichiers.

Le pipeline utilise ensuite une méthode dédiée pour télécharger chaque fichier identifié. Les fichiers ne sont pas stockés localement mais seulement chargés en mémoire, ce qui permet de traiter des volumes importants sans encombrer le disque.

Cette première étape est essentielle car elle détermine l’ensemble des documents qui seront transmis aux phases suivantes.

4. Extraction du contenu (PDF, OCR et XML)

L’extraction des fichiers constitue la partie la plus technique du pipeline. Elle est assurée par la classe DataExtractor du module extract.py, qui combine plusieurs stratégies selon la nature du document et sa qualité.

4.1. Schéma global de l’extraction

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

4.2. Extraction PDF native

Lorsque le fichier est un PDF, le pipeline commence par utiliser la bibliothèque pdfplumber. Celle-ci permet de récupérer le texte exact contenu dans le fichier sans passer par une reconnaissance optique. Elle permet également d'extraire automatiquement les tableaux, ce qui s’avère indispensable pour les documents financiers.

Cependant, certains PDF ne contiennent aucun texte numérique. C’est le cas par exemple des documents scannés. Afin de déterminer si l’extraction PDF native est exploitable, un score de qualité est calculé en mesurant la proportion de caractères incohérents ou non lisibles. Si ce score dépasse un certain seuil, le pipeline décide que le document doit passer par une étape OCR.

4.3. Passage automatique par l’OCR

Lorsque le texte natif est considéré comme inutilisable, le pipeline effectue une extraction OCR qui implique deux étapes : d’abord la transformation des pages PDF en images haute résolution via PyMuPDF, puis la reconnaissance optique du texte avec EasyOCR. Les résultats sont ensuite assemblés et nettoyés afin de reconstituer un texte propre et uniforme.

Ce mécanisme de fallback rend le pipeline robuste face à des documents scannés, mal numérisés ou très hétérogènes.

4.4. Extraction XML

Les fichiers XML sont traités différemment : ils ne nécessitent ni parsing complexe ni OCR. Le pipeline lit directement le contenu du fichier, compile une version textuelle utilisable pour le modèle de langage, et renvoie également des métadonnées comme la taille du document ou son chemin d’origine.

5. Sauvegarde en base SQLite et archivage MinIO

Une fois l’extraction terminée, toutes les données sont stockées dans une base SQLite temporaire avant d’être envoyées dans un bucket MinIO dédié aux sauvegardes. Cette étape est entièrement gérée par tasks/sqlite_saver.py.

La sauvegarde regroupe, dans une même base :

les extractions textuelles issues des PDF (qu’elles proviennent de l’OCR ou non),

les tableaux détectés,

les contenus XML bruts,

un ensemble de métadonnées décrivant le type de document et les résultats d’extraction.

Chaque type de document génère sa propre base SQLite. Celle-ci est ensuite envoyée en tant que fichier binaire dans un répertoire spécifique du bucket MinIO.

Ce mécanisme d’archivage possède plusieurs objectifs : il fournit une trace complète de l’état du document au moment de son extraction, il permet de rejouer le pipeline sans avoir à re-télécharger les documents bruts, et il constitue un outil utile pour les audits ou les analyses manuelles.

6. Traitement sémantique et structuration via un modèle de langage (LLM)

Après la sauvegarde, les données extraites sont transmises au module process.py. Celui-ci représente la partie “intelligence” du pipeline : il permet de transformer du texte libre en structure JSON cohérente, prête à être utilisée pour de l’analyse ou de l’intégration dans une base.

6.1. Schéma général du traitement LLM

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

6.2. Construction du prompt

Le pipeline commence par charger un schéma JSON définissant les champs attendus pour le type de document en question. Le texte extrait est ensuite intégré dans un prompt riche décrivant les règles que doit suivre le modèle : structure de sortie, types de champs, formats à respecter, erreurs à éviter.

Ce prompt est crucial pour obtenir une sortie la plus propre possible.

6.3. Requête vers le modèle de langage

Le pipeline envoie ensuite le prompt à l’API LLM via une requête HTTP. Le modèle renvoie du texte contenant un JSON ou un quasi-JSON. Cette sortie doit généralement être corrigée, car le modèle peut introduire du texte additionnel, des remarques, ou des erreurs de syntaxe JSON.

6.4. Parsing, normalisation et validation

Une étape de parsing robuste est alors appliquée pour extraire le JSON réel.
Ensuite, le pipeline nettoie les données : conversion cohérente des montants, interprétation correcte des dates (différents formats possibles), détection précise des identifiants numériques.

Enfin, le JSON nettoyé est comparé au schéma d’origine pour déterminer si la sortie du modèle est valide. Les éventuels écarts sont enregistrés dans un champ de métadonnées, mais n’empêchent pas nécessairement l’ingestion.

7. Insertion dans ClickHouse et déduplication

La dernière étape du pipeline consiste à envoyer les données structurées dans ClickHouse, via le module db_io/clickhouse_io.py. Celui-ci vérifie d’abord l’existence de la base et de la table utilisées pour l’ingestion. Si celles-ci n’existent pas, elles sont créées automatiquement.

Avant l’insertion, le pipeline supprime tous les champs non pertinents tels que les réponses brutes du modèle, les métadonnées ou les chemins de fichiers. Les objets JSON sont ensuite triés et convertis en chaîne afin de faciliter la déduplication.

La phase de déduplication repose sur une comparaison des JSON présents dans la table à ceux que le pipeline souhaite insérer. Cela garantit que la même extraction ne sera jamais insérée deux fois, même si l’on relance le pipeline plusieurs fois.

Une fois les données filtrées, elles sont insérées dans la table ClickHouse, prête à être utilisée dans des analyses ultérieures.

8. Conclusion générale

L’architecture présentée dans ce document constitue un pipeline complet et robuste permettant d’automatiser l’ingestion et le traitement de documents hétérogènes. Le pipeline combine plusieurs approches complémentaires : extraction directe, analyse OCR, sauvegarde systématique, traitement sémantique via modèle de langage et ingestion analytique.

Grâce à une séparation claire des responsabilités entre modules, le système est facilement maintenable et extensible. On peut y ajouter de nouveaux types de documents ou adapter la logique d’extraction et de validation sans remettre en cause l’ensemble du pipeline.

Cette architecture répond aux besoins d’un système moderne d’ingestion documentaire, capable de traiter de grands volumes, d’assurer une qualité contrôlée et d’intégrer des données structurées dans un environnement analytique avancé.
