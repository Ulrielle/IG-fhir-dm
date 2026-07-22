{% include markdown-link-references.md %}

La Share Layer constitue la couche de transformation permettant de projeter les ressources FHIR vers des structures analytiques destinées à l'usage secondaire des données. Elle s'appuie sur Pathling et les ViewDefinitions SQL-on-FHIR v2 pour produire des tables exploitables dans un modèle analytique tel qu'OMOP CDM v5.4.

>* API REST (HAPI)
>* Pathling
>* Bucket 
>* OMOP (StructureDefinition logic)
>* ViewDefinition / per OMOP Table
>* Parquet
>* DuckDB

{% include data-export-process.mermaid %}

Vous pouvez trouver l'alignement formel entre la couche sémantique FHIR et le modèle physique OMOP : 

* Patient : [FHIR Patient vers Modèle physique OMOP](StructureMap-CoreFHIRPatient2OMOP.html)
* Claim : [FHIR Claim vers Modèle physique OMOP]()

Mise en œuvre de la projection de FHIR → OMOP CDM v5.4

L'architecture ci-dessus a été mise en œuvre et validée sur un corpus couvrant les 51 variables du socle EDSH, à travers 10 cas d'usage.

### Éléments d'entrée

Le travail part de ressources FHIR au format JSON, conformes aux profils APHP/DMPatient (Patient, Encounter, Condition, Procedure, Observation, MedicationRequest, MedicationAdministration), produites et vérifiées à partir des QuestionnaireResponse. Le corpus couvre l'ensemble des 51 variables du socle EDSH : 10 patients, 10 séjours, 23 diagnostics, 27 observations (biologie, signes vitaux, pression artérielle), 13 actes, 32 prescriptions et 34 administrations de médicaments avec pour cible le schéma OMOP CDM v5.4.

### Etapes de réalisation

#### Get Asynchronous $export

Les données sources sont représentées sous forme de ressources FHIR au format JSON, produites et vérifiées à partir des `QuestionnaireResponse` des 10 cas d'usage de l'IG EDSH socle commun. Elles couvrent notamment :

- Patient
- Encounter
- Condition
- Procedure
- Observation
- MedicationAdministration
- MedicationRequest

Ces ressources ont été converties du format JSON vers NDJSON et déposées dans le stockage objet MinIO, qui sert de support d'ingestion pour Pathling via l'opération `$import`.

#### Pathling + ViewDefinition Transform

Des ViewDefinitions SQL-on-FHIR v2 dont les colonnes sont nommées d'après le schéma OMOP CDM v5.4 projettent directement les ressources FHIR brutes vers les tables OMOP, sans étape intermédiaire. Cette approche réduit le nombre d'étapes du pipeline et permet d'identifier rapidement les écarts structurels entre les ressources FHIR et les tables OMOP.

Les projections ont été définies pour les tables :

- Person
- Location
- Visit_Occurrence
- Condition_Occurrence
- Procedure_Occurrence
- Observation
- Measurement
- Drug_Exposure
- Death
- Visit_detail

Pour les `concept_id` dont les valeurs sont connues et stables (codes FHIR standard universellement reconnus, modalités restraintes et fixes), la technique `count() × concept_id` contourne l'absence de `iif()` dans Pathling : chaque terme vaut 0 ou 1 selon la condition, un seul terme non nul étant sommé au résultat final.

```
gender.where($this = 'female').count() * 8532 + gender.where($this = 'male').count() * 8507
```

```json
{
  "resourceType": "ViewDefinition",
  "name": "person",
  "resource": "Patient",
  "select": [
    { "column": [
      {"path": "id", "name": "person_id"},
      {"path": "gender.where($this = 'female').count() * 8532 + gender.where($this = 'male').count() * 8507",
       "name": "gender_concept_id"},
      {"path": "birthDate", "name": "birth_date"},
      {"path": "id", "name": "location_id"},
      {"path": "gender", "name": "gender_source_value"}
    ]}
  ]
}
```

Les champs obligatoires de type `*_type_concept_id` sont renseignés à partir du type et de la provenance des ressources FHIR. Pour les codes médicaux non encore mappés vers les vocabulaires OMOP standard (SNOMED, LOINC, RxNorm), la valeur `0` est utilisée dans les champs `concept_id`, et une colonne supplémentaire `coding_system` conserve le système de codage source en vue des traitements ultérieurs.

#### Post-process Data

Certaines transformations ne peuvent pas être réalisées directement dans les ViewDefinitions et nécessitent un traitement complémentaire après projection :

| Cas nécessitant un post-traitement | Traitement appliqué |
|---|---|
| Identifiants `string` (UUID) vs `integer NOT NULL` attendu par OMOP | UUID conservés en `VARCHAR`, génération d'entiers reportée en aval |
| `birthDate` unique vs `year_of_birth`/`month_of_birth`/`day_of_birth` | Pas de `substring()` dans Pathling → export en `birth_date` puis décomposition SQL (`SUBSTRING`) |
| Types temporels polymorphes (`date`/`dateTime`/`Period`) vs deux colonnes séparées | Même valeur `dateTime` injectée dans les deux colonnes ; séparation par `CAST()`/`DATE()` |
| `coding[]` (codages multiples) vs un seul `concept_id` par enregistrement | `coding.first()` dans la ViewDefinition ; codage alternatif conservé en `_source_value` pour traçabilité |
| Mesures composites (`component[]`, ex. pression artérielle) | ViewDefinitions distinctes `fhir_measurement_simple` / `fhir_measurement_composite`, fusionnées par `UNION ALL` |
| Cause du décès portée par une `Condition` séparée du patient | Deux ViewDefinitions (`death` + Condition-cause) jointes en SQL sur `person_id` |
| `Condition` non pathologiques (allergies, antécédents, statut fonctionnel) | Redirection vers `observation` plutôt que `condition_occurrence`, selon le domaine associé au code CIM |
| Tables nécessitant une agrégation ou une relation entre ressources (`observation_period`, `condition_era`, `drug_era`, `fact_relationship`) | Hors périmètre de SQL-on-FHIR, traitement SQL/Python en aval |
{: .grid}

Pathling joue ainsi le rôle d'un pré-mapping ETL efficace, mais ne remplace pas un ETL SQL complet.

#### Write to Analytic Format

Les tables produites sont exportées en CSV depuis Pathling. WhiteRabbit produit un scan report par champ sur ces exports, ingéré dans Rabbit-in-a-Hat pour visualiser les correspondances champ par champ entre tables sources et tables OMOP cibles. Le Data Catalogue du Domaine MSD de la DSN documente et visualise ensuite les correspondances établies entre ressources FHIR et tables OMOP, représentant graphiquement les flux de transformation et améliorant la traçabilité des règles de mapping.

Pathling permet également l'export des résultats de ViewDefinitions vers des formats analytiques tels que Parquet, ouvrant la voie à une exploitation directe via DuckDB en aval du Data Catalogue.

### Compétences impliquées

Ce travail mobilise la connaissance du standard HL7 FHIR R4, ainsi que de FHIRPath et du standard SQL-on-FHIR v2. Il suppose une bonne connaissance du modèle OMOP CDM v5.4 et de l'écosystème OHDSI, une capacité à concevoir un pipeline ETL déclaratif sur Pathling/Apache Spark, et une analyse fine des écarts structurels et sémantiques entre standards d'interopérabilité. S'y ajoutent la prise en main des outils de profilage et de documentation ETL (WhiteRabbit, Rabbit-in-a-Hat), la prise en compte des exigences RGPD et d'anonymisation des données de santé.

### Éléments de sortie

Le travail produit des ViewDefinitions OMOP validées pour `Person`, `Visit_Occurrence`, `Condition_Occurrence`, `Procedure_Occurrence`, `Observation`, `Measurement`, `Drug_Exposure`, `Location`, `Visit_detail` et `Death`, ainsi que les tables CSV correspondantes, prêtes pour le profilage et le chargement OMOP. À cela s'ajoutent le scan report WhiteRabbit, le document de mapping Rabbit-in-a-Hat et la documentation des écarts structurels identifiés avec les solutions retenues.

### Informations complémentaires

Certaines tables restent hors périmètre de SQL-on-FHIR et nécessitent un traitement SQL/Python en aval : `observation_period` (agrégation `MIN`/`MAX`), `condition_era` et `drug_era` (algorithmes ERA OHDSI), `fact_relationship` (jointures inter-ressources).
