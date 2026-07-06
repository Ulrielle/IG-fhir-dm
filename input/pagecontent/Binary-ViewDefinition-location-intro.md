<style>
table { border-collapse: collapse; width: 100%; }
th, td { border: 1px solid #cccccc; padding: 8px; text-align: left; }
th { background-color: #f2f2f2; font-weight: bold; }
</style>

Cette vue s'appuie également sur la ressource Patient, mais pour alimenter la table OMOP `location`. 
Elle contient la latitude et la longitude du patient, lorsque ces coordonnées sont renseignées dans son adresse.

| Colonne | Signification métier |
|---|---|
| location_id | Identifiant du lieu |
| person_id | Référence vers le patient concerné |
| latitude | Latitude géographique |
| longitude | Longitude géographique |
