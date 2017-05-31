# OpenMRS Initializer module
The Initializer module is an API-only module that processes the content of the **configuration** folder when it is found inside OpenMRS' application data directory:
<pre>
  .
   \_ modules/
   \_ openmrs.war
   \_ openmrs-runtime.properties
   \_ ...
   \_ <b>configuration/</b>
</pre>
The configuration folder is subdivided into 'domain specific' subfolders:
```
  configuration/
    \_ addresshierarchy/
    \_ concepts/
    \_ globalproperties/
    \_ idgen/
    \_ metadatasharing/ 
    \_ personattributetypes/
```  
Each domain-specific subfolder contains the metadata and configuration information that is relevant to the subfolder's domain. Although several file types are supported for providing metadata, CSV files are the preferred format and all domain should aim at being covered through parsing CSV files.
### Conventions for CSV files
There are a number of conventions that apply to CSV files across all domains such as the meaning of certain columns (or headers). Headers are in general optional, it is up to the implementor to provide or not the fields that are required to create or edit the OpenMRS instance. Mandatory headers will be specifically indicated.
<br/>Each CSV line is meant to provide enough data to create, edit or void/retire an OpenMRS instance of the domain.

###### Header `Uuid` (mandatory)
* If the value under this header is missing the OpenMRS instance will be created with a new random UUID.
* If the value under this header is provided the initializer will attempt to retrieve the OpenMRS instance that may already exist with this UUID.
  * If the OpenMRS instance already exists it will be modified and saved again according to the CSV line.
  * If the OpenMRS instance doesn't exist yet it will be created with the UUID specified on the CSV line.

###### Header `Void/Retire`
Set this **true** to indicate that the OpenMRS instance with the provided UUID should be voided or retired.
<br/>When `Void/Retire` is set to true, the parsing of the remaining of the CSV line is interrupted since the only objective is to retire the concept. And to this end, only the UUID and the retire flag are needed.

###### CSV metadata headers
Special headers are used to provide metadata information about the CSV file itself.
<br/>All metadata headers start with an underscore, eg. `_version:1`, `_order:1000`, ... etc.

###### Header `_version:*`
Versions are primarily introduced to allow for evolutions in the implementation of CSV parsers. Those evolutions may require to modify the CSV headers which will likely lead to backward compatibility issues. Using versions works around this by ensuring that specific parsers are used based on the version specified via the CSV file header.
<br/>When no version header is provided the initializer will fallback on a default one or may complain in its log messages that no version was specified.

###### Header `_order:*`
This metadata header specifies the order of loading of the CSV file _within the domain_. In many cases the creation of OpenMRS instances relies on the existence of other OpenMRS instances that are referred to. This use case is covered by the order header that allows to control the order of loading of files in a given domain.
<br/>For example `_order:1000` indicates that all CSV files with an order smaller than 1,000 will be processed _before_ this file within the domain.
<br/> If the order metadata cannot be parsed or is missing, then the file will be processed _after_ all the ordered CSV files of the domain. However if several CSV files have no order defined, then the loading order between them is undefined.

### Supported domains and loading order
This is the list of supported domain respective of their loading order:
1. Metadatasharing packages (ZIP files)
1. Global properties (XML files)
1. Concepts (CSV files)
1. Person attribute types (CSV files)
1. Identifier sources (XML files)

Let's review each domain in details below:

---

#### Domain 'addresshierarchy'
The **addresshierarchy** subfolder contains all the address hierarchy metadata. This is a possible example of its content:
```
  addresshierarchy/
    \_ addressConfiguration.xml
    \_ addresshierarchy.csv
    \_ addresshierarchy_en.properties
    \_ addresshierarchy_km_KH.properties
```
This is a mixed scenario since the Address Hierarchy module's activator itself can handle most of the provided configuration and metadata: **addressConfiguration.xml** (the actual configuration file) and **addresshierarchy.csv** (the CSV import file containing all address hierarchy geographies.)

The Initializer module will take care of loading the address hierarchy entries translations for use cases where the Address Hierarchy module must support i18n.

---

#### Domain 'concepts'
The **concepts** subfolder contains CSV import files for saving concepts in bulk. This is a possible example of its content:
```
  concepts/
    \_ diagnoses.csv
    \_ findings.csv
    \_ misc.csv
    \_ ...
```
The way those CSV files are processed is controlled by a reserved part of the CSV file header line that holds metadata about the CSV file itself. Here is an example of a header line:

| <sub>Uuid</sub> | <sub>Void/Retire</sub> | <sub>Fully specified name:en</sub> | <sub>Short name:en</sub> | <sub>Description:en</sub> | ... | <sub>_version:1</sub> | <sub>_order:1000</sub> |
| - | - | - | - | - | - | - | - |

<br/>Let's review some important headers.

##### Localized headers: `Fully specified name`, `Short name` and `Description`
Those are locale specific headers, they are never used as such because they always need to indicate the locale for the values of their column.
<br/>For example for a column to contain short names in English (locale 'en') simply name the header `Short name:en`. The same logic applies for the other locale specific headers.

###### Header `Fully specified name` (localized)
###### Header `Short name` (localized)
###### Header `Description` (localized)
###### Header `Data class`
###### Header `Data type`

Here is an example of valid CSV to define basic concepts:

| <sub>Uuid</sub>  | <sub>Fully specified name:en</sub> | <sub>Short name:en</sub> | <sub>Description:en</sub> | <sub>Data class</sub>  | <sub>Data type</sub> |
| - | - | - | - | - | - |
| | <sub>Nationality</sub> | <sub>Nat.</sub> | <sub>The status of belonging to a particular nation.</sub> | <sub>Question</sub> | <sub>Text</sub> |
| <sub>db2f4fc4-..</sub>| <sub>Language</sub> | <sub>Lang.</sub> | <sub>The method of human communication.</sub> | <sub>Question</sub> | <sub>Text</sub> |

###### Header `Answers`
To provide a semicolumn-separated list of answer concepts to the concept to be created or edited.
###### Header `Members`
To provide a semicolumn-separated list of concepts to be members of the concept set to be created or edited. Note that the concept will be marked a being a set as soon as set members are provided.

Here is an example of the 'nested' columns:

| ... | <sub>Answers</sub> | <sub>Members</sub> | ... |
| - | - | - | - |
| ... | <sub>CONCEPT_NAME; source:134; db2f4fc4-..</sub> | | ... |
| ... | | <sub>CONCEPT_NAME; source:134; db2f4fc4-..</sub> | ... |

As the example suggests, it is possible to provide lists of concepts identifiers to fill the values of the columns 'answers' or 'members' under the form of concept names (eg. "CONCEPT_NAME"), concept mappings (eg. "source:134") and concept UUIDs (eg. "db2f4fc4-.."). The concepts that could not be fetched through their provided identifier will fail the creation of the concept from the CSV line altogether, and the parser will continue to the next CSV line.

**NOTE** In the current implementation the listing order of the concepts in the CSV file does matter since unexisting concepts will fail the CSV line processing. It is recommended to take this into account and to insert CSV lines for concepts with nested lists low enough in the CSV file so that all nested concepts are found when the CSV line is being processed.

###### Header `Same as mappings`
To provide a semicolumn-separated list of concept mappings for the concept to be created or edited.

Here is an example of the 'mappings' columns:

| ... | <sub>Same as concept mappings</sub> | ... |
| - | - | - |
| ... | <sub>ICD-10-WHO:T45.9; CIEL:122226; Cambodia:115</sub> | ... |

Below are the headers specific to concept numerics:
###### Header `Absolute low`
###### Header `Critical low`
###### Header `Normal low`
###### Header `Normal high`
###### Header `Critical high`
###### Header `Absolute high`
###### Header `Units`
###### Header `Allow decimals`
###### Header `Display precision`

**NOTE** The concept will be considered as a candidate to be a concept numeric if and only if its data type is set to `Numeric`.

---

#### Domain 'globalproperties'
The **globalproperties** subfolder contains XML configuration files that specify which global properties to override. Note that existing global properties will be overridden and missing ones will be created.
This is a possible example of how the configuration subfolder may look like:
```
  globalproperties/
    \_ gp_core.xml
    \_ gp_coreapps.xml
    \_ ...
```
There can be as many XML files as desired. One may be enough in most cases, but providing multiples files is also a possibility if the implementation requires to manage them by modules, areas or categories. Beware that the behaviour will be undefined iif a global property is overridden in several places. 

###### Global properties XML configuration file example:
```xml
<config>
  <globalProperties>
    <globalProperty>
      <property>addresshierarchy.i18nSupport</property>
      <value>true</value>
    </globalProperty>
    <globalProperty>
      <property>locale.allowed.list</property>
      <value>en, km_KH</value>
    </globalProperty>
  </globalProperties>
</config>
```
The above XML configuration will set **addresshierarchy.i18nSupport** to `true` and **locale.allowed.list** to `"en, km_KH"`.

---

#### Domain 'idgen'
The **idgen** subfolder contains XML configuration files that help modify and create identifier sources. It should be possible in most cases to configure them via a single XML configuration file, however there can be as many XML files as desired.
This is a possible example of how the configuration subfolder may look like:
```
  idgen/
    \_ idgen.xml
```
The XML configuration allows to either modify exisiting identifier sources or to create new identifier sources. At the moment the only possible modification allowed is to _retire_ exisiting sources.

###### Idgen XML configuration file example:
```xml
<config>
  <identifierSources>
    <identifierSource>
      <uuid>c1d8a345-3f10-11e4-adec-0800271c1b75</uuid>
      <retired>true</retired>
    </identifierSource>
    <identifierSource>
      <uuid>c1d90956-3f10-11e4-adec-0800271c1b75</uuid>
      <retired>true</retired>
    </identifierSource>
    <sequentialIdentifierGenerator>
      <name>Autogenerated IDs source</name>
      <description>Source used for the autogeneration of OpenMRS IDs.</description>
      <identifierType>
        <name>PATIENTIDENTIFIERTYPE_1_OPENMRS_ID</name>
      </identifierType>
      <firstIdentifierBase>001000</firstIdentifierBase>
      <minLength>7</minLength>
      <maxLength>7</maxLength>
      <baseCharacterSet>0123456789</baseCharacterSet>
    </sequentialIdentifierGenerator>
  </identifierSources>
</config>
```
The above XML configuration will retire the identifier sources whose UUIDs are `c1d8a345-3f10-11e4-adec-0800271c1b75` and `c1d90956-3f10-11e4-adec-0800271c1b75` ; and will create a new `SequentialIdentifierGenerator` with the specified properties. When creating a new identifier source, pay special attention to the way it is linked to its `PatientIdentifierType`. This is done through the _name_ of the patient identifier type (that must be unique in OpenMRS.)

---

#### Domain 'personattributetypes'
The **personattributetypes** subfolder contains CSV import files for saving person attribute types in bulk. This is a possible example of its content:
```
  personattributetypes/
    \_ registration_pat.csv
    \_ ...
```
There is currently only one format for the person attribute type CSV line, here are the possible headers:

| <sub>Uuid</sub> | <sub>Void/Retire</sub> | <sub>Name</sub> | <sub>Description</sub> | <sub>Format</sub> | <sub>Foreign uuid</sub> | <sub>Searchable</sub> | <sub>_order:1000</sub> |
| - | - | - | - | - | - | - | - |

Headers that start with an underscore such as `_order:1000` are metadata headers. The values in the columns under those headers are never read by the CSV parser.
<br/>Let's review some important headers.

###### Header `Name`
This is _not_ a localized header.

###### Header `Format`
Here are the possible values for this column: `java.lang.Boolean`, `java.lang.Character`, `java.lang.Float`, `java.lang.Integer`, `java.lang.String`, `org.openmrs.Concept`, `org.openmrs.Drug`, `org.openmrs.Encounter`, `org.openmrs.Location`, `org.openmrs.Patient`, `org.openmrs.Person`, `org.openmrs.ProgramWorkflow`, `org.openmrs.Provider`, `org.openmrs.User`, `org.openmrs.util.AttributableDate`.

###### Header `Foreign uuid`
When the header `Format` refers to an OpenMRS class (such as `org.openmrs.Concept` for example), `Foreign uuid` should point to the UUID of an existing instance of that class. 

---

#### Domain 'metadatasharing'
The **metadatasharing** subfolder contains all the Metadata Sharing (MDS) packages as .zip files to be imported. This is a possible example of its content:
```
  metadatasharing/
    \_ PatientIdentifierType.zip
    \_ PersonAttributeType.zip
    \_ ...
```
There can be as many MDS packages as desired. Providing multiples .zip files allows to split the metadata to be imported by areas, categories or any other segmentation that the implementors deem relevant.
<br/>They will all be imported following the 'prefer theirs' rule, meaning that the the metadata shipped with the packages is considered being the master metadata. Existing objects will be overwritten, missing objects will be created... etc.
<br/>MDS packages are a convenient way to bring in metadata, especially while other methods have not yet been implemented. However when otherwise possible, other ways should be preferred.

### How to try it out?
Build the master branch and install the built OMOD to your OpenMRS instance:
```
git clone https://github.com/mekomsolutions/openmrs-module-initializer/tree/master
cd openmrs-module-initializer
mvn clean install
```
##### Runtime requirements & compatibility
* Core 1.11.8

### Quick facts
Initializer enables to achieve the OpenMRS backend equivalent of Bahmni Config for Bahmni Apps. It facilitates the deployment of implementation-specific configurations without writing any code, by just filling the **configuration** folder with the needed metadata and in accordance to Initializer's implementation.

### Get in touch
Find us on [OpenMRS Talk](https://talk.openmrs.org/): sign up, start a conversation and ping us with the mentions starting with @mks.. in your message.

----

### Releases notes

#### Version 1.0
##### New features
* Loads i18n messages files from **configuration/addresshierarchy**.
* Bulk creation and saving of concepts provided through CSV files in  **configuration/concepts**.<br/>This covers: basic concepts, concepts with nested members or answers and concepts with multiple mappings.
* Overrides global properties provided through XML configuration files in **configuration/globalproperties**.
* Modifies (retire) or create identifier sources as specified in  **configuration/idgen**.
* Bulk creation and saving of person attribute types provided through CSV files in  **configuration/personattributetypes**.
* Imports MDS packages provided as .zip files in **configuration/metadatasharing**.