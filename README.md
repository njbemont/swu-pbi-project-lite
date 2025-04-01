# swu-pbi-project

## Goals
The main goal of this project is to provide a dashboard for players and judges to search for, analyze, and find rulings on Star Wars: Unlimited cards.

## Getting Started

### Pre-requisites
1. Github account
2. Power BI Desktop installed - https://www.microsoft.com/en-us/power-platform/products/power-bi/desktop

### Initial Setup
   
1. Download and Extract ZIP File
2. Open PBIP file (SWU Dashboard.pbip)
3. Click "Transform Data" in the top tool bar
4. In the top folder "Parameter", change the "File Path" Parameter to the path your GitHub folder is located (ex. if your full folder path to the project is "C:\Users\NoahBemont\OneDrive - StarPoint Technologies Inc\Documents\GitHub\swu-pbi-project", change this parameter to "C:\Users\NoahBemont\OneDrive - StarPoint Technologies Inc\Documents\"). This project always assumes that your project files will be in a GitHub\swu-pbi-project folder structure. 
5. Click "Close & Apply"
6. Wait for it to load. If it asks about privacy settings, check the box to ignore privacy settings.
7. If you get an error on Pricing, go to File > Options and Settings > Options > Current File > Privacy > Check "Ignore Privacy Levels"

### Collection Import
1. Open the collection_spreadsheet file in the Collection folder. Using the userInput tab, enter the count of each card you have in column J. Save your version of this file with the same file name in the Collection folder.
2. Right-click on the "Collection" table in the table list (far right side of the application under "Data") and click refresh.

### Decklists Import
1. Download your Decklist JSON files from SWUDB
2. Put each JSON file in the "Decklists" folder
3. Right-click on the "Decks" table in the table list (far right side of the application under "Data") and click refresh. 

## Dashboards
### !!Visualizations are a work in progress and are subject to change!!
### Card Gallery
![image](https://github.com/user-attachments/assets/ea32161f-c798-4ac3-8a62-16cf6a3df87a)
### Slice & Dice
![image](https://github.com/user-attachments/assets/d9d1be25-c155-4952-85af-f4e15f62b788)
### Collection
![image](https://github.com/user-attachments/assets/6c040fa8-b7f9-49de-8c8d-217a6dd1312d)
### Collection Pricing
![image](https://github.com/user-attachments/assets/8a6e36c2-b8ce-4936-9330-5b73c5bb3689)
### Watchlist
![image](https://github.com/user-attachments/assets/0e20b16e-d58b-414b-9cd5-8581052e22f7)
### Trade Tool
![image](https://github.com/user-attachments/assets/20b0a6aa-306e-4f3b-87a3-3954fa3ca131)
### Decks
![image](https://github.com/user-attachments/assets/e6346a65-99f3-49ba-a5c2-e9af66219b3d)

## Documentation
### API
This project uses the official Star Wars: Unlimited card database API. This API does not have public documentation, but it is publicly accessible. The effort to extract information from this API was a lot of trial and error. I'll document any relevant filters or API call information here.  
Base URL: https://admin.starwarsunlimited.com/api/card-list  
Filters in use:
1. locale
2. pagination[page]
3. pagination[pageSize]

### API Extraction within PBIX:
#### Functions:
##### Dynamic
This function loops through the official Star Wars: Unlimited card database to extract all card information for every card on the official website.  
Definition:
```
(page as number) =>
let
     Source = Json.Document(Web.Contents("https://admin.starwarsunlimited.com/api/card-list?locale=en&pagination[page]=" & Number.ToText(page) & "&pagination[pageSize]=" & Number.ToText(#"Rows") & "")),
     data = Source[data]
in
     data
```
#### Parameters
These are mainly used as default values for the loop, but can also be used for troubleshooting purposes
##### Page
Sets the page value for the API call.  
Default: 1
##### Row
Sets the number of row responses per page for the API call.  
Default: 25

#### Tables
##### Base
This table is the complete API call. All other tables are transformations off of this table.  
Use a reference to this table to create any new tables from the API data.  
This code uses the Dynamic function to grab all records from the API, converts the list of records to a table, and then expands all the records into columns.  
Definition:  
```
let
    Source = List.Generate(
        () => [Result = try Dynamic(1) otherwise null, pagenumber=1],
        each List.IsEmpty([Result]) = false,
        each [Result = try Dynamic(pagenumber) otherwise null, pagenumber = [pagenumber] + 1],
        each [Result]),
    #"Converted to Table" = Table.FromList(Source, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
    #"Expanded Column1" = Table.ExpandListColumn(#"Converted to Table", "Column1"),
    #"Expanded Column2" = Table.ExpandRecordColumn(#"Expanded Column1", "Column1", {"attributes"}, {"attributes"}),
    #"Expanded attributes" = Table.ExpandRecordColumn(#"Expanded Column2", "attributes", {"title", "subtitle", "cardNumber", "cardCount", "artist", "artFrontHorizontal", "artBackHorizontal", "hasFoil", "cost", "hp", "power", "text", "textStyled", "deployBox", "deployBoxStyled", "epicAction", "epicActionStyled", "linkHtml", "cardId", "createdAt", "updatedAt", "publishedAt", "locale", "hyperspace", "unique", "showcase", "cardUid", "rules", "rulesStyled", "serialCode", "upgradeHp", "upgradePower", "artFront", "artBack", "artThumbnail", "localizations", "aspects", "aspectDuplicates", "type", "type2", "traits", "arenas", "keywords", "rarity", "expansion", "variantTypes", "variantOf", "reprintOf", "validationId"}, {"title", "subtitle", "cardNumber", "cardCount", "artist", "artFrontHorizontal", "artBackHorizontal", "hasFoil", "cost", "hp", "power", "text", "textStyled", "deployBox", "deployBoxStyled", "epicAction", "epicActionStyled", "linkHtml", "cardId", "createdAt", "updatedAt", "publishedAt", "locale", "hyperspace", "unique", "showcase", "cardUid", "rules", "rulesStyled", "serialCode", "upgradeHp", "upgradePower", "artFront", "artBack", "artThumbnail", "localizations", "aspects", "aspectDuplicates", "type", "type2", "traits", "arenas", "keywords", "rarity", "expansion", "variantTypes", "variantOf", "reprintOf", "validationId"})
in
    #"Expanded attributes"
```

##### Cards Fact
This table is essentially a bridge table so that all dimensional tables can work together. It's just IDs for relational purposes.  
Definition:
```
let
    Source = Base,
    #"Removed Other Columns" = Table.SelectColumns(Source,{"cardUid", "aspects", "traits", "arenas", "keywords", "variantTypes"}),
    #"Expanded aspects" = Table.ExpandRecordColumn(#"Removed Other Columns", "aspects", {"data"}, {"aspects.data"}),
    #"Expanded aspects.data" = Table.ExpandListColumn(#"Expanded aspects", "aspects.data"),
    #"Expanded aspects.data1" = Table.ExpandRecordColumn(#"Expanded aspects.data", "aspects.data", {"id"}, {"aspects.data.id"}),
    #"Expanded traits" = Table.ExpandRecordColumn(#"Expanded aspects.data1", "traits", {"data"}, {"traits.data"}),
    #"Expanded traits.data" = Table.ExpandListColumn(#"Expanded traits", "traits.data"),
    #"Expanded traits.data1" = Table.ExpandRecordColumn(#"Expanded traits.data", "traits.data", {"id"}, {"traits.data.id"}),
    #"Expanded arenas" = Table.ExpandRecordColumn(#"Expanded traits.data1", "arenas", {"data"}, {"arenas.data"}),
    #"Expanded arenas.data" = Table.ExpandListColumn(#"Expanded arenas", "arenas.data"),
    #"Expanded arenas.data1" = Table.ExpandRecordColumn(#"Expanded arenas.data", "arenas.data", {"id"}, {"arenas.data.id"}),
    #"Expanded keywords" = Table.ExpandRecordColumn(#"Expanded arenas.data1", "keywords", {"data"}, {"keywords.data"}),
    #"Expanded keywords.data" = Table.ExpandListColumn(#"Expanded keywords", "keywords.data"),
    #"Expanded keywords.data1" = Table.ExpandRecordColumn(#"Expanded keywords.data", "keywords.data", {"id"}, {"keywords.data.id"}),
    #"Expanded variantTypes" = Table.ExpandRecordColumn(#"Expanded keywords.data1", "variantTypes", {"data"}, {"variantTypes.data"}),
    #"Expanded variantTypes.data" = Table.ExpandListColumn(#"Expanded variantTypes", "variantTypes.data"),
    #"Expanded variantTypes.data1" = Table.ExpandRecordColumn(#"Expanded variantTypes.data", "variantTypes.data", {"id"}, {"variantTypes.data.id"})
in
    #"Expanded variantTypes.data1"
```

##### Cards
This table contains the majority of card information. I plan to break this down into a few more dimensional items, so this table may change a bit.  
Definition:
```
let
    Source = Base,
    #"Expanded artFront" = Table.ExpandRecordColumn(Source, "artFront", {"data"}, {"artFront.data"}),
    #"Expanded artFront.data" = Table.ExpandRecordColumn(#"Expanded artFront", "artFront.data", {"attributes"}, {"artFront.data.attributes"}),
    #"Expanded artFront.data.attributes" = Table.ExpandRecordColumn(#"Expanded artFront.data", "artFront.data.attributes", {"url"}, {"artFront.data.attributes.url"}),
    #"Expanded artBack" = Table.ExpandRecordColumn(#"Expanded artFront.data.attributes", "artBack", {"data"}, {"artBack.data"}),
    #"Expanded artBack.data" = Table.ExpandRecordColumn(#"Expanded artBack", "artBack.data", {"attributes"}, {"artBack.data.attributes"}),
    #"Expanded artBack.data.attributes" = Table.ExpandRecordColumn(#"Expanded artBack.data", "artBack.data.attributes", {"url"}, {"artBack.data.attributes.url"}),
    #"Expanded artThumbnail" = Table.ExpandRecordColumn(#"Expanded artBack.data.attributes", "artThumbnail", {"data"}, {"artThumbnail.data"}),
    #"Expanded artThumbnail.data" = Table.ExpandRecordColumn(#"Expanded artThumbnail", "artThumbnail.data", {"attributes"}, {"artThumbnail.data.attributes"}),
    #"Expanded artThumbnail.data.attributes" = Table.ExpandRecordColumn(#"Expanded artThumbnail.data", "artThumbnail.data.attributes", {"url"}, {"artThumbnail.data.attributes.url"}),
    #"Expanded type" = Table.ExpandRecordColumn(#"Expanded artThumbnail.data.attributes", "type", {"data"}, {"type.data"}),
    #"Expanded type.data" = Table.ExpandRecordColumn(#"Expanded type", "type.data", {"attributes"}, {"type.data.attributes"}),
    #"Expanded type.data.attributes" = Table.ExpandRecordColumn(#"Expanded type.data", "type.data.attributes", {"name"}, {"type.data.attributes.name"}),
    #"Expanded type2" = Table.ExpandRecordColumn(#"Expanded type.data.attributes", "type2", {"data"}, {"type2.data"}),
    #"Expanded type2.data" = Table.ExpandRecordColumn(#"Expanded type2", "type2.data", {"attributes"}, {"type2.data.attributes"}),
    #"Expanded type2.data.attributes" = Table.ExpandRecordColumn(#"Expanded type2.data", "type2.data.attributes", {"name"}, {"type2.data.attributes.name"}),
    #"Expanded rarity" = Table.ExpandRecordColumn(#"Expanded type2.data.attributes", "rarity", {"data"}, {"rarity.data"}),
    #"Expanded rarity.data" = Table.ExpandRecordColumn(#"Expanded rarity", "rarity.data", {"attributes"}, {"rarity.data.attributes"}),
    #"Expanded rarity.data.attributes" = Table.ExpandRecordColumn(#"Expanded rarity.data", "rarity.data.attributes", {"name"}, {"rarity.data.attributes.name"}),
    #"Expanded expansion" = Table.ExpandRecordColumn(#"Expanded rarity.data.attributes", "expansion", {"data"}, {"expansion.data"}),
    #"Expanded expansion.data" = Table.ExpandRecordColumn(#"Expanded expansion", "expansion.data", {"attributes"}, {"expansion.data.attributes"}),
    #"Expanded expansion.data.attributes" = Table.ExpandRecordColumn(#"Expanded expansion.data", "expansion.data.attributes", {"name", "code"}, {"expansion.data.attributes.name", "expansion.data.attributes.code"}),
    #"Removed Columns" = Table.RemoveColumns(#"Expanded expansion.data.attributes",{"textStyled", "deployBoxStyled", "epicActionStyled", "linkHtml", "localizations", "aspects", "traits", "arenas", "keywords", "variantTypes", "variantOf", "validationId", "cardId", "aspectDuplicates", "rulesStyled", "rules"}),
    #"Expanded reprintOf" = Table.ExpandRecordColumn(#"Removed Columns", "reprintOf", {"data"}, {"reprintOf.data"}),
    #"Expanded reprintOf.data" = Table.ExpandRecordColumn(#"Expanded reprintOf", "reprintOf.data", {"id"}, {"reprintOf.data.id"})
in
    #"Expanded reprintOf.data"
```

#### Arenas
This table contains Arena information.  
Definition:
```
let
    Source = Base,
    #"Removed Other Columns" = Table.SelectColumns(Source,{"arenas"}),
    #"Expanded arenas" = Table.ExpandRecordColumn(#"Removed Other Columns", "arenas", {"data"}, {"data"}),
    #"Expanded data" = Table.ExpandListColumn(#"Expanded arenas", "data"),
    #"Expanded data1" = Table.ExpandRecordColumn(#"Expanded data", "data", {"id", "attributes"}, {"id", "attributes"}),
    #"Filtered Rows" = Table.SelectRows(#"Expanded data1", each [id] <> null and [id] <> ""),
    #"Expanded attributes" = Table.ExpandRecordColumn(#"Filtered Rows", "attributes", {"name", "description", "createdAt", "updatedAt", "publishedAt", "locale"}, {"name", "description", "createdAt", "updatedAt", "publishedAt", "locale"}),
    #"Removed Duplicates" = Table.Distinct(#"Expanded attributes", {"id"})
in
    #"Removed Duplicates"
```

#### Aspects
This table contains Aspect information.  
Definition:
```
let
    Source = Base,
    #"Removed Other Columns" = Table.SelectColumns(Source,{"aspects"}),
    #"Expanded aspects" = Table.ExpandRecordColumn(#"Removed Other Columns", "aspects", {"data"}, {"data"}),
    #"Expanded data" = Table.ExpandListColumn(#"Expanded aspects", "data"),
    #"Expanded data1" = Table.ExpandRecordColumn(#"Expanded data", "data", {"id", "attributes"}, {"id", "attributes"}),
    #"Filtered Rows" = Table.SelectRows(#"Expanded data1", each [id] <> null and [id] <> ""),
    #"Expanded attributes" = Table.ExpandRecordColumn(#"Filtered Rows", "attributes", {"name", "description", "color", "createdAt", "updatedAt", "publishedAt", "locale", "englishName", "sortValue"}, {"name", "description", "color", "createdAt", "updatedAt", "publishedAt", "locale", "englishName", "sortValue"}),
    #"Removed Duplicates" = Table.Distinct(#"Expanded attributes", {"id"})
in
    #"Removed Duplicates"
```

#### Keywords
This table contains Keyword information.  
Definition:
```
let
    Source = Base,
    #"Removed Other Columns" = Table.SelectColumns(Source,{"keywords"}),
    #"Expanded keywords" = Table.ExpandRecordColumn(#"Removed Other Columns", "keywords", {"data"}, {"data"}),
    #"Expanded data" = Table.ExpandListColumn(#"Expanded keywords", "data"),
    #"Expanded data1" = Table.ExpandRecordColumn(#"Expanded data", "data", {"id", "attributes"}, {"id", "attributes"}),
    #"Filtered Rows" = Table.SelectRows(#"Expanded data1", each [id] <> null and [id] <> ""),
    #"Expanded attributes" = Table.ExpandRecordColumn(#"Filtered Rows", "attributes", {"name", "description", "createdAt", "updatedAt", "publishedAt", "locale"}, {"name", "description", "createdAt", "updatedAt", "publishedAt", "locale"}),
    #"Removed Duplicates" = Table.Distinct(#"Expanded attributes", {"id"})
in
    #"Removed Duplicates"
```

#### Rules
This table contains Rules information.  It extracts the data from the Additional Rulings tab for each card on the official card database.  
Definition:
```
let
    Source = Base,
    #"Removed Other Columns" = Table.SelectColumns(Source,{"cardUid", "rulesStyled"}),
    #"Filtered Rows" = Table.SelectRows(#"Removed Other Columns", each [rulesStyled] <> null and [rulesStyled] <> ""),
    #"Replaced Value" = Table.ReplaceValue(#"Filtered Rows","</td></tr></tbody></table></figure>","",Replacer.ReplaceText,{"rulesStyled"}),
    #"Replaced Value1" = Table.ReplaceValue(#"Replaced Value","<figure class=""table""><table><tbody><tr><td>","",Replacer.ReplaceText,{"rulesStyled"}),
    #"Trimmed Text" = Table.TransformColumns(#"Replaced Value1",{{"rulesStyled", Text.Trim, type text}}),
    #"Split Column by Delimiter" = Table.SplitColumn(#"Trimmed Text", "rulesStyled", Splitter.SplitTextByDelimiter("</td></tr><tr><td>", QuoteStyle.Csv), {"rulesStyled.1", "rulesStyled.2", "rulesStyled.3", "rulesStyled.4", "rulesStyled.5", "rulesStyled.6"}),
    #"Unpivoted Other Columns" = Table.UnpivotOtherColumns(#"Split Column by Delimiter", {"cardUid"}, "Attribute", "Value"),
    #"Split Column by Delimiter1" = Table.SplitColumn(#"Unpivoted Other Columns", "Value", Splitter.SplitTextByDelimiter("</td><td>", QuoteStyle.Csv), {"Value.1", "Value.2", "Value.3"}),
    #"Changed Type" = Table.TransformColumnTypes(#"Split Column by Delimiter1",{{"Value.1", type date}, {"Value.2", type text}, {"Value.3", type text}}),
    #"Removed Columns" = Table.RemoveColumns(#"Changed Type",{"Value.3"}),
    #"Renamed Columns" = Table.RenameColumns(#"Removed Columns",{{"Value.1", "date"}, {"Value.2", "ruling"}})
in
    #"Renamed Columns"
```

#### Traits
This table contains Trait information.   
Definition:
```
let
    Source = Base,
    #"Removed Other Columns" = Table.SelectColumns(Source,{"traits"}),
    #"Expanded traits" = Table.ExpandRecordColumn(#"Removed Other Columns", "traits", {"data"}, {"data"}),
    #"Expanded data" = Table.ExpandListColumn(#"Expanded traits", "data"),
    #"Expanded data1" = Table.ExpandRecordColumn(#"Expanded data", "data", {"id", "attributes"}, {"id", "attributes"}),
    #"Filtered Rows" = Table.SelectRows(#"Expanded data1", each [id] <> null and [id] <> ""),
    #"Expanded attributes" = Table.ExpandRecordColumn(#"Filtered Rows", "attributes", {"name", "description", "createdAt", "updatedAt", "publishedAt", "locale"}, {"name", "description", "createdAt", "updatedAt", "publishedAt", "locale"}),
    #"Removed Duplicates" = Table.Distinct(#"Expanded attributes", {"id"})
in
    #"Removed Duplicates"
```

#### Variants
This table contains Variant information.   
Definition:
```
let
    Source = Base,
    #"Removed Other Columns" = Table.SelectColumns(Source,{"variantTypes"}),
    #"Expanded variantTypes" = Table.ExpandRecordColumn(#"Removed Other Columns", "variantTypes", {"data"}, {"data"}),
    #"Expanded data" = Table.ExpandListColumn(#"Expanded variantTypes", "data"),
    #"Expanded data1" = Table.ExpandRecordColumn(#"Expanded data", "data", {"id", "attributes"}, {"id", "attributes"}),
    #"Filtered Rows" = Table.SelectRows(#"Expanded data1", each [id] <> null and [id] <> ""),
    #"Expanded attributes" = Table.ExpandRecordColumn(#"Filtered Rows", "attributes", {"name"}, {"name"}),
    #"Removed Duplicates" = Table.Distinct(#"Expanded attributes", {"id"})
in
    #"Removed Duplicates"
```

### Table Relationships
| ID | Name | Relationship | Model | IsActive | CrossFilteringBehavior | RelyOnReferentialIntegrity | FromTable | FromColumn | FromCardinality | ToTable | ToColumn | ToCardinality | State | SecurityFilteringBehavior |
|----|------|-------------|-------|----------|------------------------|----------------------------|-----------|------------|----------------|--------|----------|---------------|-------|--------------------------|
| 43 | f9a00a22-fb23-0afd-db9e-d1c475e5bbd9 | 'Cards Fact'[arenas.data.id] *[<->]1 'Arenas'[id] | 9d34e8bc-5268-4a8c-ab36-79656a9bdff2 | true | BothDirections | false | Cards Fact | arenas.data.id | Many | Arenas | id | One | Ready | Single |
| 44 | 9d5b7c52-4021-9574-2e26-de4fd47ac5e4 | 'Cards Fact'[aspects.data.id] *[<->]1 'Aspects'[id] | 9d34e8bc-5268-4a8c-ab36-79656a9bdff2 | true | BothDirections | false | Cards Fact | aspects.data.id | Many | Aspects | id | One | Ready | Single |
| 45 | 14d759a8-e43e-2c77-f861-371551d865db | 'Cards Fact'[cardUid] *[<->]1 'Cards'[cardUid] | 9d34e8bc-5268-4a8c-ab36-79656a9bdff2 | true | BothDirections | false | Cards Fact | cardUid | Many | Cards | cardUid | One | Ready | Single |
| 46 | 5dab5f92-2f18-5d29-2b40-11d8ffa6b6b9 | 'Cards Fact'[keywords.data.id] *[<->]1 'Keywords'[id] | 9d34e8bc-5268-4a8c-ab36-79656a9bdff2 | true | BothDirections | false | Cards Fact | keywords.data.id | Many | Keywords | id | One | Ready | Single |
| 47 | 023ec232-5024-54d7-0e19-46d3639872ae | 'Cards Fact'[traits.data.id] *[<->]1 'Traits'[id] | 9d34e8bc-5268-4a8c-ab36-79656a9bdff2 | true | BothDirections | false | Cards Fact | traits.data.id | Many | Traits | id | One | Ready | Single |
| 48 | 627873dd-7ac0-bae2-fa47-8542021f189b | 'Cards Fact'[variantTypes.data.id] *[<->]1 'Variants'[id] | 9d34e8bc-5268-4a8c-ab36-79656a9bdff2 | true | BothDirections | false | Cards Fact | variantTypes.data.id | Many | Variants | id | One | Ready | Single |
| 50 | 17e1da67-c7a0-79bd-7745-672eecdff521 | 'Cards Fact'[cardUid] [<->] 'Rules'[cardUid] | 9d34e8bc-5268-4a8c-ab36-79656a9bdff2 | true | BothDirections | false | Cards Fact | cardUid | Many | Rules | cardUid | Many | Ready | Single |
| 49 | 4f38265f-390e-486b-87af-b82a37f9fd3a | 'Rules'[date] *[<-]1 'LocalDateTable_e3d70a23-d04e-4c99-bf90-293b16bed457'[Date] | 9d34e8bc-5268-4a8c-ab36-79656a9bdff2 | true | OneDirection | false | Rules | date | Many | LocalDateTable_e3d70a23-d04e-4c99-bf90-293b16bed457 | Date | One | Ready | Single |
