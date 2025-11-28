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

### 4.1. Overall extraction diagram

The operation can be represented succinctly as follows:

   



### 4.2. Native PDF extraction

When the document is in PDF format, the first try to get information is done using the pdfplumber tool. This tool helps to pull out the text from the file directly, if there is any, along with how the pages are arranged. At the same time, it finds and takes out any tables in the PDF, which is very handy for things like tax papers or financial records. 

Next, a score is given to check the quality of the text. This score shows if the text that was taken out can be used or if the document is probably a scanned copy that doesn’t have any digital text. This is an important step because handling a scanned document needs a completely different way to process it.

### 4.3. Automatic OCR processing

When the text that comes out is confusing or not good enough, the system automatically changes to a method called OCR extraction. This means it makes a picture of each page of the PDF and then uses something called an optical character recognition (OCR) model to read it. 

This OCR method uses EasyOCR and PyMuPDF. PyMuPDF helps create clear images of the pages, and EasyOCR looks at these images and gives back the text. After this reading step, the text is put together, fixed up, and made standard using the ftfy library, so it can be easy to read, even if the original documents aren’t very clear.

### 4.4. Extraction XML

Handling XML files is way easier than dealing with PDFs because XML documents already have a clear structure that makes them usable right off the bat. When we find an XML file, the system reads all of it into memory at once, without needing to make complicated changes or break it down. This simple reading keeps the original content intact as it is stored in MinIO, including all the tags, attributes, and how everything is organized inside. 

After the file is opened, the system decodes what’s inside based on the type of encoding mentioned in the XML header, which is usually UTF-8. If there's no encoding given, it uses a standard one and checks to make sure there are no mistakes. The text is then lightly cleaned, where only non-printable characters or any clearly broken ones are taken out to keep everything stable for future processing without changing how the document is set up. 

Different from a usual XML parser, this system doesn’t try to change or interpret the tags. Instead, it just gives back the text as it is, avoiding typical mistakes that can happen with messed-up documents and letting a language model handle the content analysis. The system also notes the file size in bytes so it’s easier to manage and troubleshoot. 

In the end, the outcome is a simple but complete structure that includes the MinIO location, the type of document, the raw XML text, and some helpful metadata. This method makes sure the original document is kept perfectly while also getting it ready for analysis by the language learning model and for logging in ClickHouse.

## 5. SQLite database backup and MinIO archiving

After the data is taken out, the system saves everything to a temporary SQLite database. This part is controlled by tasks/sqlite_saver. py and is very important for the process. 

The goal is to keep a complete record of the original data in a format that is universal, small, and easy to move around: SQLite. The system makes a database for each type of document and stores: 

- text taken from PDFs, 
- tables found in the PDFs, 
- XML files, 
- a group of information about the extraction. 

After the SQLite database is made, it gets saved as a temporary file, loaded into memory, and then transferred to the MinIO bucket that is used for backups. The name of the SQLite file automatically has a timestamp and the type of document. 

This method has a few benefits. First, it gives a dependable automatic backup option in case something goes wrong during later processing. Second, it lets the system restart from the gathered data without needing to reload or extract the original documents again.

## 6. Semantic processing and structuring via a language model (LLM)

Once the information is gathered and saved, the next thing to do is change this basic data into organized data that can be used by other systems. PDF and XML extractions create unorganized text that can sometimes be messy or confusing, and how useful it is relies a lot on being able to get a good understanding of it. This is where understanding the meaning of the text becomes important. 

This change is done by a smart language model, which can be used either on a local computer or through an outside service, like an API, depending on the setup. Its job is to make sense of the text, find important details such as numbers, dates, IDs, and other important fields, and rearrange this information into a common format. The aim is not just to pull out values, but also to grasp the logic of the document, how different elements are connected, and to spot any mistakes that might suggest that the initial data gathering was not done well. 

All of this logic takes place in the process. py file. This file acts as a link between the gathered data (which comes from extract. py) and the language model. It gets the data ready to send to the language model: it cleans up the text, sets up prompts, adds examples, normalizes certain data fields, and follows rules to check validity. Process. py also manages handling errors: if a call to the language model does not work, it can try again, change the prompt, or clearly note that the document should be considered invalid. 

After getting the response from the model, the module turns it into a neat dictionary that meets the format needed for the next steps in the process. It also adds important information such as the type of document, any error messages, and a detailed record of the model’s response, which can help in fixing problems. 

This step is very important in the process: it changes the data from “raw text that was automatically gathered” into a “structured record,” which is then ready to be stored in ClickHouse and used by other business software.

### 6.1. General scheme of LLM treatment

The overall functioning can be represented as follows:


### 6.2. Construction of the prompt

The message used in our system isn't just a bunch of words for the model; it's a full plan that clearly shows how to turn a raw document into organized data. It starts by telling the LLM what to do: read the content from a PDF or XML file and create properly formatted JSON. The format it needs to follow is provided right in the message, along with what each data field means and what type it is. This makes it clear for the model and ensures the output is always the same. 

The message also explains important business rules, like how to find a number, spot a date, or pull out official references. These instructions help the model understand the document better and focus on what's important. To help the LLM behave consistently, it gets examples to show what is needed, letting it copy the right format. 

In the end, the message sets strict rules for the final output: the model has to give valid JSON with no extra text. This strictness makes it easier to directly use the results in the next steps of the system, especially when saving them to ClickHouse.

### 6.3. Request to the language model

Once the instructions are ready, a request is sent to the model's API using HTTP. The model replies with information in JSON format, which is then understood. The system is designed to expect that the model may give back text that includes JSON, or even JSON that isn't complete. So, strong methods are used to pull out the right JSON structure no matter what the exact form of the reply is.

### 6.4 Cleaning and normalizing the response

The model output is then cleaned: numerical amounts are corrected, dates are standardized, and CPF, CNPJ, or other administrative identifiers are correctly interpreted. This step transforms imprecise textual values ​​into consistent formats usable in an analytical database.

### 6.5 Validation against the diagram

After the data is organized, it is checked against the original plan. Any differences or missing parts are noted in a collection of information that points out mistakes or issues. This helps to keep track of how well the data was gathered and sets up ways to fix problems later on.

## 7. Insertion into ClickHouse and deduplication

The last part of the process involves putting the final information into a ClickHouse database. The program for this is found in db_io/clickhouse_io. py. It starts by connecting to the ClickHouse server using the settings set in the environment variables. 

The first thing it does is check if the database and the specific table are there. If they are missing, it creates them automatically. The table uses a MergeTree engine, which is good for handling organized data sorted by time. 

Before adding any new information, the pipeline cleans up the data by getting rid of all the internal details, like the original model responses or checking flags. It keeps only the important business fields. 

To make sure there aren’t any repeats, the pipeline checks the new data against what is already in ClickHouse. It turns every entry into sorted JSON and then sees if that version is already in the table. This way, the pipeline can run again without accidentally adding the same documents more than once. 

After eliminating duplicates, the new information is added to ClickHouse, making it ready for more analysis.

## 8. Conclusion 

The pipeline that has been set up is a well-organized system that takes a document from when it first comes in to when it is finally added to an analytical database. The different steps—finding, pulling out information, using optical character recognition (OCR), saving, organizing with a language model, checking, removing duplicates, and putting it in the system—make a strong and clear process. 

This system is made to grow easily. You can add new types of documents just by including a JSON schema, or you can change the language model without needing to change the other parts. The pipeline can also handle problems: if there are network issues, if scanned PDFs cannot be read, or if the model gives wrong answers, there are backup systems in place to take care of that.

