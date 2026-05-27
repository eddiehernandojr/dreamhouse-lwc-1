# Dreamhouse LWC Build Simulation Guide

This guide assumes you are rebuilding the `dreamhouse-lwc` app in a fresh Salesforce Developer Org with VS Code and Salesforce CLI. It is written as a learning sequence, not as the fastest deployment path.

The app is a real estate sample app. Its core idea is simple:

- `Broker__c` stores real estate brokers.
- `Property__c` stores homes for sale and links each property to a broker.
- Apex returns filtered property data, imports sample data, creates files, and geocodes addresses.
- Lightning Web Components display filters, tiles, maps, summaries, record page widgets, and sample data tools.
- Lightning pages and tabs compose those pieces into the Dreamhouse app.

## Repository Map

Before building, orient yourself to the project:

| Area | Files | Purpose |
| --- | --- | --- |
| Salesforce project config | `sfdx-project.json`, `config/project-scratch-def.json` | Defines source API version, package directory, and scratch org shape. |
| Data model | `force-app/main/default/objects` | Custom objects, fields, compact layouts, list views. |
| Apex | `force-app/main/default/classes` | Controllers, services, helper data transfer objects, tests. |
| LWC | `force-app/main/default/lwc` | User interface components. |
| Messaging | `force-app/main/default/messageChannels` | Lightning Message Service channels for component communication. |
| App and pages | `applications`, `tabs`, `flexipages`, `layouts`, `aura` | Dreamhouse Lightning app, tabs, record pages, app pages, page template. |
| Security | `permissionsets/dreamhouse.permissionset-meta.xml` | Object, field, Apex, tab, and app access. |
| External access | `cspTrustedSites`, `remoteSiteSettings` | Allows images, map tiles, and Apex geocoding callouts. |
| Static/content resources | `staticresources`, `contentassets` | Leaflet library, sample data JSON, app logo. |
| Flow | `flows/Create_property.flow-meta.xml` | Guided property creation flow with geocoding and file upload. |
| Sample data | `data/*.json`, `data/sample-data-plan.json` | CLI-loadable sample records. |
| Local tests/tooling | `package.json`, `jest.config.js`, `force-app/test/jest-mocks` | LWC Jest tests and mocks. |

## Recommended Learning Strategy

Build in layers:

1. Create the org and project connection.
2. Build the custom objects and fields.
3. Add layouts, tabs, and permissions.
4. Add Apex classes that depend on the data model.
5. Add message channels and LWC utilities.
6. Add UI components from small to composed.
7. Add Lightning pages, app configuration, flow, and external access.
8. Load sample data.
9. Run final functional and automated tests.

This order lets you test each Salesforce concept before adding the next one.

## Phase 1: Org Setup

### Step 1. Confirm the Salesforce DX project

What to build:

- A local Salesforce DX project connected to your Developer Org.

Files involved:

- `sfdx-project.json`
- `config/project-scratch-def.json`
- `.forceignore`

Why now:

- Every later command depends on knowing the package directory and API version.

Concept taught:

- Salesforce DX source format and project configuration.

Useful commands:

```bash
sf org login web --alias dreamhouse-dev --set-default
sf org display --target-org dreamhouse-dev
sf project deploy preview --target-org dreamhouse-dev
```

Notes:

- `sfdx-project.json` uses `force-app` as the default package directory.
- The source API version is `64.0`.
- `config/project-scratch-def.json` is useful if you later choose a scratch org, but for a normal Developer Org you mainly need the login command.

Verify:

- In VS Code, confirm the default org is set.
- Run:

```bash
sf org list
```

## Phase 2: Data Model Setup

Build the data model before Apex or UI. Apex references these objects and fields, and LWCs import them with `@salesforce/schema`.

### Step 2. Build the `Broker__c` object

What to build:

- Custom object: `Broker__c`
- Label: `Broker`
- Plural label: `Brokers`
- Name field: `Broker Name`, Text
- Sharing model: Read/Write
- Search and reports enabled

Files involved:

- `force-app/main/default/objects/Broker__c/Broker__c.object-meta.xml`

Why now:

- `Property__c.Broker__c` is a lookup to `Broker__c`, so Broker must exist first.

Concept taught:

- Custom objects, name fields, object features, and org-wide sharing model metadata.

Deploy or build manually:

```bash
sf project deploy start --source-dir force-app/main/default/objects/Broker__c --target-org dreamhouse-dev
```

Verify:

- In Setup, open Object Manager and find `Broker`.
- Create one test broker manually.

### Step 3. Add Broker fields

What to build:

| Field | Type | Purpose |
| --- | --- | --- |
| `Broker_Id__c` | Number, External ID | Optional stable identifier for imports. |
| `Title__c` | Text | Broker title, such as Senior Broker. |
| `Phone__c` | Phone | Office phone. |
| `Mobile_Phone__c` | Phone | Mobile phone. |
| `Email__c` | Email | Broker email. |
| `Picture__c` | URL | Image URL. |
| `Picture_IMG__c` | Formula/Text | Displays `Picture__c` as an image. |

Files involved:

- `force-app/main/default/objects/Broker__c/fields/*.field-meta.xml`

Why now:

- Broker record pages and `brokerCard` depend on these fields.

Concept taught:

- Field types, external IDs, URL fields, formula image rendering, and field-level security later.

Verify:

- Add values to your test broker.
- Confirm the formula image field renders if the URL points to an allowed image source.

### Step 4. Build the `Property__c` object

What to build:

- Custom object: `Property__c`
- Label: `Property`
- Plural label: `Properties`
- Name field: `Property Name`, Text
- Activities, feed tracking, history, reports, search, sharing, streaming API enabled

Files involved:

- `force-app/main/default/objects/Property__c/Property__c.object-meta.xml`

Why now:

- This is the main app object. Almost every Apex class, LWC, page, and flow depends on it.

Concept taught:

- Custom object capabilities and how enabling features affects record pages and related metadata.

Verify:

- In Object Manager, open `Property`.
- Create a property with just a name to confirm the object exists.

### Step 5. Add simple Property fields

What to build:

Start with non-dependent fields:

| Field | Type | Purpose |
| --- | --- | --- |
| `Address__c` | Text | Street address. |
| `City__c` | Text | City search and display. |
| `State__c` | Text | State display. |
| `Zip__c` | Text | Postal code. |
| `Description__c` | Long Text Area | Listing description. |
| `Tags__c` | Text | Search/filter keywords. |
| `Beds__c` | Number | Minimum bedroom filtering. |
| `Baths__c` | Number | Minimum bathroom filtering. |
| `Price__c` | Currency | Asking price and filtering. |
| `Assessed_Value__c` | Currency | Additional valuation data. |
| `Price_Sold__c` | Currency | Closed sale price. |
| `Picture__c` | URL | Main image URL. |
| `Thumbnail__c` | URL | Small image URL. |
| `Record_Link__c` | Text | Link display helper. |

Files involved:

- `force-app/main/default/objects/Property__c/fields/Address__c.field-meta.xml`
- `City__c`, `State__c`, `Zip__c`, `Description__c`, `Tags__c`
- `Beds__c`, `Baths__c`, `Price__c`, `Assessed_Value__c`, `Price_Sold__c`
- `Picture__c`, `Thumbnail__c`, `Record_Link__c`

Why now:

- These fields do not require other custom fields. They let you create useful records before adding formulas, lookup, location, and picklists.

Concept taught:

- Basic field types used by UI API, Apex SOQL, and LWC schema imports.

Verify:

- Create or edit a test property.
- Fill in price, bed, bath, city, and image URLs.

### Step 6. Add dependent Property fields

What to build:

| Field | Type | Depends on | Purpose |
| --- | --- | --- | --- |
| `Broker__c` | Lookup to `Broker__c` | `Broker__c` object | Links each property to a broker. |
| `Status__c` | Picklist | None | Tracks `Contracted`, `Pre Market`, `Available`, `Under Agreement`, `Closed`. |
| `Location__c` | Location | None | Stores latitude and longitude. |
| `Date_Listed__c` | Date | None | Used by days-on-market logic. |
| `Date_Pre_Market__c` | Date | None | Lifecycle date. |
| `Date_Contracted__c` | Date | None | Lifecycle date. |
| `Date_Agreement__c` | Date | None | Lifecycle date. |
| `Date_Closed__c` | Date | None | Lifecycle date. |
| `Days_On_Market__c` | Formula Number | `Date_Listed__c` | Calculates `TODAY() - Date_Listed__c`. |
| `Picture_IMG__c` | Formula/Text | `Picture__c` | Renders main picture. |
| `Thumbnail_IMG__c` | Formula/Text | `Thumbnail__c` | Renders thumbnail. |

Files involved:

- `force-app/main/default/objects/Property__c/fields/Broker__c.field-meta.xml`
- `Status__c.field-meta.xml`
- `Location__c.field-meta.xml`
- Date fields
- Formula fields

Why now:

- Lookup, formula, location, and picklist fields teach more advanced data modeling after the basic object works.

Concept taught:

- Relationships, formula fields, geolocation fields, and picklists.

Verify:

- Link a property to a broker.
- Set `Date_Listed__c` and confirm `Days_On_Market__c` calculates.
- Enter latitude and longitude, then verify map components can later read `Location__Latitude__s` and `Location__Longitude__s`.

### Step 7. Add list views and compact layouts

What to build:

- Broker compact layout
- Property compact layout
- All list views for both custom objects

Files involved:

- `force-app/main/default/objects/Broker__c/compactLayouts/Broker_Compact.compactLayout-meta.xml`
- `force-app/main/default/objects/Property__c/compactLayouts/Property_Compact_Layout.compactLayout-meta.xml`
- `force-app/main/default/objects/Broker__c/listViews/All.listView-meta.xml`
- `force-app/main/default/objects/Property__c/listViews/All.listView-meta.xml`

Why now:

- Record pages and tabs feel usable once the object has list and compact display metadata.

Concept taught:

- Record summaries, list views, and how metadata shapes standard Salesforce UI.

Verify:

- Open the Broker and Property tabs after tabs are added later.
- Confirm record highlights show useful fields.

## Phase 3: Security and Permissions

### Step 8. Build the permission set shell

What to build:

- Permission set: `dreamhouse`

Files involved:

- `force-app/main/default/permissionsets/dreamhouse.permissionset-meta.xml`

Why now:

- You need object and field access before testing LWCs and Apex as a normal user.

Concept taught:

- Permission sets as app-specific access bundles.

Deploy:

```bash
sf project deploy start --source-dir force-app/main/default/permissionsets --target-org dreamhouse-dev
```

Assign:

```bash
sf org assign permset --name dreamhouse --target-org dreamhouse-dev
```

Verify:

```bash
sf data query --query "SELECT PermissionSet.Name, Assignee.Username FROM PermissionSetAssignment WHERE PermissionSet.Name = 'dreamhouse'" --target-org dreamhouse-dev
```

### Step 9. Understand what the permission set grants

What it grants:

- Object CRUD for `Broker__c`: create, read, edit, delete.
- Object CRUD for `Property__c`: create, read, edit, delete.
- Field read/edit for most Broker and Property fields.
- Read-only access to formula fields such as `Picture_IMG__c`, `Thumbnail_IMG__c`, `Days_On_Market__c`, and `Record_Link__c`.
- Apex class access to `PagedResult`, `PropertyController`, and `SampleDataController`.
- Tab visibility for `Broker__c`, `Property__c`, `Property_Explorer`, `Property_Finder`, and `Settings`.
- App visibility for `Dreamhouse`.

Important learning note:

- `FileUtilities` and `GeocodingService` are not listed in this permission set. Admin users can still test them, but a stricter non-admin learning org may need explicit Apex class access if those features are used outside system contexts.

Concept taught:

- Object permissions, field-level security, Apex class access, app access, and tab visibility.

Verify:

- Temporarily switch to a non-admin test user with the permission set and confirm they can see the custom objects and fields.

## Phase 4: External Access and Static Assets

### Step 10. Add CSP Trusted Sites

What to build:

- CSP Trusted Site for OpenStreetMap tiles: `https://tile.openstreetmap.org`
- CSP Trusted Site for sample images: `https://s3-us-west-2.amazonaws.com`

Files involved:

- `force-app/main/default/cspTrustedSites/openStreetMap.cspTrustedSite-meta.xml`
- `force-app/main/default/cspTrustedSites/s3_us_west_2_amazonaws_com.cspTrustedSite-meta.xml`

Why now:

- Map tiles and sample listing images are external resources. Lightning blocks untrusted content by default.

Concept taught:

- Content Security Policy in Lightning Experience.

Verify:

- Later, property images and OpenStreetMap tiles should load without browser console CSP errors.

### Step 11. Add Remote Site Setting

What to build:

- Remote site for `https://nominatim.openstreetmap.org`

Files involved:

- `force-app/main/default/remoteSiteSettings/nominatim_openstreetmap.remoteSite-meta.xml`

Why now:

- Apex callouts in `GeocodingService` need callout permission before the flow can geocode addresses.

Concept taught:

- Apex callout security.

Verify:

- Later, run the create property flow and confirm it can geocode a valid address.

### Step 12. Add static resources and content asset

What to build:

- Static resource `leafletjs`, containing Leaflet JavaScript, CSS, and marker images.
- Static resources:
  - `sample_data_brokers`
  - `sample_data_properties`
  - `sample_data_contacts`
- Content asset `dreamhouselogosquare`.

Files involved:

- `force-app/main/default/staticresources/leafletjs`
- `force-app/main/default/staticresources/leafletjs.resource-meta.xml`
- `force-app/main/default/staticresources/sample_data_*.json`
- `force-app/main/default/staticresources/sample_data_*.resource-meta.xml`
- `force-app/main/default/contentassets/dreamhouselogosquare.asset`
- `force-app/main/default/contentassets/dreamhouselogosquare.asset-meta.xml`

Why now:

- Map LWCs import `leafletjs`.
- `SampleDataController` reads sample data from static resources.
- The app branding references the content asset logo.

Concept taught:

- Static resources, content assets, and resource imports in LWC.

Verify:

```bash
sf data query --query "SELECT Name FROM StaticResource WHERE Name LIKE 'sample_data_%' OR Name = 'leafletjs'" --target-org dreamhouse-dev
```

## Phase 5: Apex Backend

Build Apex after the schema exists.

### Step 13. Add `PagedResult`

What to build:

- Simple Apex data transfer object with Aura-enabled properties:
  - `pageSize`
  - `pageNumber`
  - `totalItemCount`
  - `records`

Files involved:

- `force-app/main/default/classes/PagedResult.cls`
- `PagedResult.cls-meta.xml`

Why now:

- `PropertyController.getPagedPropertyList` returns this type.

Concept taught:

- Apex wrapper classes for LWC data.

Verify:

```bash
sf apex run --target-org dreamhouse-dev
```

Then run:

```apex
PagedResult result = new PagedResult();
result.pageSize = 9;
System.debug(result);
```

### Step 14. Add `PropertyController`

What to build:

- `getPagedPropertyList(...)`
  - Filters by search text, max price, minimum bedrooms, and minimum bathrooms.
  - Returns a page of `Property__c` records ordered by price.
- `getPictures(propertyId)`
  - Reads image files linked to a property using `ContentDocumentLink` and `ContentVersion`.

Files involved:

- `force-app/main/default/classes/PropertyController.cls`
- `PropertyController.cls-meta.xml`
- `force-app/main/default/classes/TestPropertyController.cls`

Why now:

- Core list, map, tile, and carousel LWCs depend on this controller.

Concept taught:

- `@AuraEnabled(cacheable=true)`, SOQL, pagination, `WITH USER_MODE`, file queries, and LWC Apex wiring.

Dependencies:

- `Property__c` and its fields.
- `PagedResult`.
- User must have object and field permissions.

Verify with Apex:

```bash
sf data query --query "SELECT Id, Name, City__c, Price__c FROM Property__c LIMIT 5" --target-org dreamhouse-dev
sf apex run test --class-names TestPropertyController --target-org dreamhouse-dev --result-format human
```

If you have no data yet, the query returns no rows. That is fine at this stage.

### Step 15. Add `SampleDataController`

What to build:

- `importSampleData()`
  - Deletes existing `Case`, `Property__c`, `Broker__c`, and `Contact` records.
  - Reads JSON from static resources.
  - Inserts brokers, properties, and contacts.
  - Randomizes `Date_Listed__c` for properties.

Files involved:

- `force-app/main/default/classes/SampleDataController.cls`
- `SampleDataController.cls-meta.xml`
- `force-app/main/default/classes/TestSampleDataController.cls`
- `force-app/main/default/staticresources/sample_data_*.json`

Why now:

- The Settings page and `sampleDataImporter` LWC call this method.

Concept taught:

- JSON deserialization into sObjects, static resource access from Apex, and DML.

Important warning:

- This method deletes all records in `Case`, `Property__c`, `Broker__c`, and `Contact` in the org where it runs. Use it only in a learning or disposable org.

Verify:

```bash
sf apex run test --class-names TestSampleDataController --target-org dreamhouse-dev --result-format human
```

### Step 16. Add `FileUtilities`

What to build:

- `createFile(base64data, filename, recordId)`
  - Creates a `ContentVersion`.
  - Reads its `ContentDocumentId`.
  - Links the file to a record with `ContentDocumentLink`.

Files involved:

- `force-app/main/default/classes/FileUtilities.cls`
- `FileUtilities.cls-meta.xml`
- `FileUtilitiesTest.cls`

Why now:

- `propertyCarousel` uses this to upload and attach images from the property record page.

Concept taught:

- Salesforce Files, `ContentVersion`, `ContentDocumentLink`, binary data, and `AuraHandledException`.

Verify:

```bash
sf apex run test --class-names FileUtilitiesTest --target-org dreamhouse-dev --result-format human
```

### Step 17. Add `GeocodingService`

What to build:

- Invocable Apex method `geocodeAddresses`.
- Inner input class `GeocodingAddress`.
- Inner output class `Coordinates`.
- HTTP GET callout to OpenStreetMap Nominatim.

Files involved:

- `force-app/main/default/classes/GeocodingService.cls`
- `GeocodingService.cls-meta.xml`
- `GeocodingServiceTest.cls`
- `force-app/main/default/remoteSiteSettings/nominatim_openstreetmap.remoteSite-meta.xml`

Why now:

- The `Create_property` flow calls this service before creating a property.

Concept taught:

- Invocable Apex, Flow integration, callouts, HTTP mocks, and remote site settings.

Verify:

```bash
sf apex run test --class-names GeocodingServiceTest --target-org dreamhouse-dev --result-format human
```

### Step 18. Run all Apex tests

Command:

```bash
sf apex run test --test-level RunLocalTests --target-org dreamhouse-dev --wait 10 --result-format human
```

Verify:

- All local Apex tests pass.
- If a test fails because the org has unexpected data, reset the learning org or inspect test setup assumptions.

## Phase 6: Lightning Message Channels

### Step 19. Add message channels

What to build:

- `FiltersChange__c`
- `PropertySelected__c`

Files involved:

- `force-app/main/default/messageChannels/FiltersChange.messageChannel-meta.xml`
- `force-app/main/default/messageChannels/PropertySelected.messageChannel-meta.xml`

Why now:

- Several LWCs communicate without a direct parent-child relationship.

Concept taught:

- Lightning Message Service.

Dependencies:

- None on Apex, but the components that use these channels depend on them existing.

Verify:

- After LWCs are added, changing filters should update the list/map.
- Selecting a property should update summary, map, and days-on-market components.

## Phase 7: Lightning Web Components

Build LWCs from smallest utility pieces to composed app views.

### Step 20. Add utility and display components

What to build:

| Component | Files | Purpose | Concept |
| --- | --- | --- | --- |
| `ldsUtils` | `lwc/ldsUtils` | Converts LDS/Apex errors into display strings. | Shared JS module. |
| `errorPanel` | `lwc/errorPanel` | Reusable error display with alternate templates. | Conditional template rendering. |
| `paginator` | `lwc/paginator` | Emits previous/next pagination events. | Public API and custom events. |
| `propertyTile` | `lwc/propertyTile` | Displays one property card/tile. | `@api`, navigation, responsive form factor. |

Why now:

- Larger components reuse these pieces.

Dependencies:

- `propertyTile` needs `Property__c` record data shape.
- `errorPanel` imports `ldsUtils`.

Verify:

```bash
npm install
npm run test:unit -- -- --findRelatedTests force-app/main/default/lwc/paginator/__tests__/paginator.test.js
```

### Step 21. Add record-oriented components

What to build:

| Component | Purpose | Key dependencies |
| --- | --- | --- |
| `brokerCard` | Shows the broker for a property and navigates to broker record. | `Property__c.Broker__c`, Broker fields, UI API. |
| `daysOnMarket` | Shows date listed and calculated days on market. | `PropertySelected__c`, `Date_Listed__c`, `Days_On_Market__c`. |
| `propertySummary` | Shows selected property summary and navigates to record. | `PropertySelected__c`, property schema fields. |
| `propertyMap` | Shows one property location in a `lightning-map`. | `PropertySelected__c`, `Location__c`, address fields. |
| `propertyLocation` | Uses mobile location service on record page. | `Location__c`, mobile capabilities. |
| `propertyCarousel` | Shows and uploads property pictures. | `PropertyController.getPictures`, `FileUtilities.createFile`, Salesforce Files. |

Files involved:

- `force-app/main/default/lwc/brokerCard`
- `daysOnMarket`
- `propertySummary`
- `propertyMap`
- `propertyLocation`
- `propertyCarousel`

Why now:

- These components are useful on record pages and respond to selected property context.

Concept taught:

- UI Record API, `@wire(getRecord)`, NavigationMixin, mobile capabilities, file upload patterns, and component public properties.

Verify:

```bash
npm run test:unit -- -- --findRelatedTests force-app/main/default/lwc/propertySummary/__tests__/propertySummary.test.js
npm run test:unit -- -- --findRelatedTests force-app/main/default/lwc/propertyMap/__tests__/propertyMap.test.js
```

### Step 22. Add search, filter, list, and map composition components

What to build:

| Component | Purpose | Key dependencies |
| --- | --- | --- |
| `propertyFilter` | Publishes search, price, bed, and bath filters. | `FiltersChange__c`. |
| `propertyTileList` | Displays paged property tiles and publishes selected property. | `PropertyController.getPagedPropertyList`, `paginator`, `propertyTile`, both message channels. |
| `propertyListMap` | Displays a list and Leaflet map together. | `leafletjs`, `PropertyController.getPagedPropertyList`, both message channels. |

Files involved:

- `force-app/main/default/lwc/propertyFilter`
- `force-app/main/default/lwc/propertyTileList`
- `force-app/main/default/lwc/propertyListMap`

Why now:

- These are the first app-page-level experiences. They require Apex, data, static resources, and message channels.

Concept taught:

- Reactive Apex parameters, LMS publish/subscribe, third-party JavaScript static resources, map markers, and composed components.

Verify:

```bash
npm run test:unit -- -- --findRelatedTests force-app/main/default/lwc/propertyTileList/__tests__/propertyTileList.test.js
npm run test:unit -- -- --findRelatedTests force-app/main/default/lwc/propertyListMap/__tests__/propertyListMap.test.js
```

### Step 23. Add Settings and mobile helper components

What to build:

| Component | Purpose | Concept |
| --- | --- | --- |
| `sampleDataImporter` | Button that calls `SampleDataController.importSampleData`. | Imperative Apex and toast events. |
| `barcodeScanner` | Scans a barcode and navigates to a record. | Salesforce mobile barcode scanner. |
| `listContactsFromDevice` | Reads contacts from device on mobile. | Salesforce mobile contacts service. |
| `navigateToRecord` | Flow screen component that navigates to a record. | Flow screen LWC and NavigationMixin. |

Files involved:

- `force-app/main/default/lwc/sampleDataImporter`
- `barcodeScanner`
- `listContactsFromDevice`
- `navigateToRecord`

Why now:

- These are optional or specialized features. They make more sense after core property browsing works.

Concept taught:

- Imperative Apex, toasts, mobile-only APIs, and embedding LWCs in Flow screens.

Verify:

```bash
npm run test:unit -- -- --findRelatedTests force-app/main/default/lwc/sampleDataImporter/__tests__/sampleDataImporter.test.js
```

### Step 24. Run all LWC tests

Command:

```bash
npm run test:unit
```

Optional lint:

```bash
npm run lint
```

## Phase 8: Flow

### Step 25. Build `Create_property`

What to build:

- Screen Flow named `Create Property`.
- Screens:
  - New Property
  - Address
  - Property Details
  - Upload Picture
  - Navigate to Record
- Apex action:
  - `GeocodingService`
- Record operations:
  - Create `Property__c`
  - Read `ContentDocumentLink`
  - Read `ContentVersion`
  - Update `Picture__c` and `Thumbnail__c`
- Flow screen LWC:
  - `navigateToRecord`

Files involved:

- `force-app/main/default/flows/Create_property.flow-meta.xml`
- `force-app/main/default/classes/GeocodingService.cls`
- `force-app/main/default/lwc/navigateToRecord`

Why now:

- The flow depends on Apex, `Property__c`, file objects, and the screen LWC.

Concept taught:

- Screen Flow, Apex actions, file upload handling, record create/update, and LWC in Flow.

Verify:

- Open the Flow in Flow Builder.
- Debug it with a valid address.
- Confirm a new Property record is created.
- Confirm uploaded file URLs are written to `Picture__c` and `Thumbnail__c` when a file is uploaded.

## Phase 9: App, Tabs, Layouts, and Pages

### Step 26. Add tabs

What to build:

| Tab | Type | Purpose |
| --- | --- | --- |
| `Property__c` | Custom object tab | Standard property records. |
| `Broker__c` | Custom object tab | Standard broker records. |
| `Property_Explorer` | Lightning page tab | Three-column exploration UI. |
| `Property_Finder` | Lightning page tab | List/map search UI. |
| `Settings` | Lightning page tab | Sample data import UI. |

Files involved:

- `force-app/main/default/tabs/*.tab-meta.xml`

Why now:

- Tabs expose pages and objects in the app navigation.

Concept taught:

- Object tabs and Lightning page tabs.

Verify:

- In Setup, search for Tabs and confirm each tab exists.

### Step 27. Add page layouts

What to build:

- Broker page layout.
- Property page layout.

Files involved:

- `force-app/main/default/layouts/Broker__c-Broker Layout.layout-meta.xml`
- `force-app/main/default/layouts/Property__c-Property Layout.layout-meta.xml`

Why now:

- Layouts control standard record detail fields and are referenced by record pages.

Concept taught:

- Page layout metadata versus Lightning record page metadata.

Verify:

- Open a Broker record and a Property record in standard layout/edit mode.

### Step 28. Add the Aura page template

What to build:

- `pageTemplate_2_7_3` Aura component.

Files involved:

- `force-app/main/default/aura/pageTemplate_2_7_3/pageTemplate_2_7_3.cmp`
- `.design`
- `.svg`
- `.cmp-meta.xml`

Why now:

- App pages use this template shape for custom column layouts.

Concept taught:

- Aura page templates still matter for Lightning App Builder page structure.

Verify:

- Open Lightning App Builder and confirm the template is available or pages using it render.

### Step 29. Add Lightning app pages

What to build:

#### `Property Explorer`

Files:

- `force-app/main/default/flexipages/Property_Explorer.flexipage-meta.xml`

Composition:

- Left: `propertyFilter`, `flowruntime:interview` for `Create_property`
- Center: `propertyTileList`
- Right: `propertySummary`, `propertyMap`

Purpose:

- Teaches component communication through LMS.

#### `Property Finder`

Files:

- `force-app/main/default/flexipages/Property_Finder.flexipage-meta.xml`

Composition:

- Left: `barcodeScanner`, `propertyFilter`
- Main/right experience: `propertyListMap`, `propertySummary`, `daysOnMarket`

Purpose:

- Teaches combining list, map, filters, selection, and mobile scanning.

#### `Settings`

Files:

- `force-app/main/default/flexipages/Settings.flexipage-meta.xml`

Composition:

- `sampleDataImporter`

Purpose:

- Teaches app utility pages and admin/helper functions.

Why now:

- These pages require tabs, LWCs, Apex, message channels, static resources, and permissions.

Concept taught:

- Lightning App Builder pages and component composition.

Verify:

- Open each page from the app navigation after the app is created.
- Check browser console for missing component or permission errors.

### Step 30. Add record pages

What to build:

#### `Property_Record_Page`

Files:

- `force-app/main/default/flexipages/Property_Record_Page.flexipage-meta.xml`

Custom components:

- `brokerCard`
- `daysOnMarket`
- `propertyMap`
- `propertyLocation`
- `propertyCarousel`

Purpose:

- Enhances the standard Property record page with broker, map, date, location, and image features.

#### `Broker_Record_Page`

Files:

- `force-app/main/default/flexipages/Broker_Record_Page.flexipage-meta.xml`

Purpose:

- Shows broker fields and related properties.

Why now:

- Record pages need objects, layouts, fields, and components in place.

Concept taught:

- Lightning record pages, custom component placement, standard related lists, and action overrides.

Verify:

- Open a Property record.
- Confirm broker card, map, carousel, and days-on-market areas render.
- Open a Broker record and confirm related Properties appear.

### Step 31. Add the Dreamhouse app

What to build:

- Lightning app `Dreamhouse`
- Branding logo `dreamhouselogosquare`
- Navigation items:
  - Home
  - Property Explorer
  - Property Finder
  - Contacts
  - Properties
  - Brokers
  - Files
  - Settings
- Record page overrides:
  - `Property__c` view action uses `Property_Record_Page`
  - `Broker__c` view action uses `Broker_Record_Page`

Files involved:

- `force-app/main/default/applications/Dreamhouse.app-meta.xml`
- `force-app/main/default/contentassets/dreamhouselogosquare.asset-meta.xml`

Why now:

- The app ties the whole experience together.

Concept taught:

- Lightning apps, navigation, branding, and record page activation through metadata.

Verify:

- Open App Launcher and launch Dreamhouse.
- Confirm all nav items appear.

## Phase 10: Prompts

### Step 32. Add prompt metadata

What to build:

- `Property.prompt-meta.xml`
- `PropertyExplorer.prompt-meta.xml`
- `PropertyFinder.prompt-meta.xml`

Files involved:

- `force-app/main/default/prompts/*.prompt-meta.xml`

Why now:

- These are app experience metadata, not core data model or UI dependencies. Add them after the main app works.

Concept taught:

- Prompt metadata used for in-app guidance or walkthrough-style experiences.

Verify:

- Deploy succeeds.
- If your org does not support the prompt metadata type or related features, keep it as a later optional step.

## Phase 11: Sample Data Loading

You have two sample data paths.

### Step 33A. Load sample data with Salesforce CLI

What to build:

- 8 brokers
- 12 properties
- 5 contacts

Files involved:

- `data/sample-data-plan.json`
- `data/brokers-data.json`
- `data/properties-data.json`
- `data/contacts-data.json`

Why now:

- The full UI is much easier to test with realistic records.

Concept taught:

- Data tree import, reference IDs, and relationship resolution.

Command:

```bash
sf data import tree --plan data/sample-data-plan.json --target-org dreamhouse-dev
```

Verify:

```bash
sf data query --query "SELECT COUNT() FROM Broker__c" --target-org dreamhouse-dev
sf data query --query "SELECT COUNT() FROM Property__c" --target-org dreamhouse-dev
sf data query --query "SELECT COUNT() FROM Contact" --target-org dreamhouse-dev
```

Expected counts from the local data folder:

- Brokers: 8
- Properties: 12
- Contacts: 5

### Step 33B. Load sample data from the Settings tab

What to build:

- Use the `sampleDataImporter` LWC to call `SampleDataController.importSampleData`.

Files involved:

- `force-app/main/default/lwc/sampleDataImporter`
- `force-app/main/default/classes/SampleDataController.cls`
- `force-app/main/default/staticresources/sample_data_*.json`
- `force-app/main/default/flexipages/Settings.flexipage-meta.xml`

Why use this path:

- It teaches imperative Apex from LWC and Apex JSON deserialization from static resources.

Warning:

- This deletes existing `Case`, `Property__c`, `Broker__c`, and `Contact` records first.

Verify:

- Open Dreamhouse > Settings.
- Click the import button.
- Confirm success toast.
- Query record counts.

## Phase 12: Final Testing

### Step 34. Deploy the full app

Once you understand each piece, deploy the complete metadata set:

```bash
sf project deploy start --source-dir force-app --target-org dreamhouse-dev --wait 20
```

### Step 35. Assign permissions

```bash
sf org assign permset --name dreamhouse --target-org dreamhouse-dev
```

### Step 36. Run Apex tests

```bash
sf apex run test --test-level RunLocalTests --target-org dreamhouse-dev --wait 10 --result-format human
```

### Step 37. Run LWC tests

```bash
npm install
npm run test:unit
```

### Step 38. Manual UI test checklist

Verify these in the Dreamhouse app:

- Dreamhouse app appears in App Launcher.
- Property Explorer tab loads.
- Property Finder tab loads.
- Settings tab loads.
- Brokers and Properties tabs are visible.
- Sample data appears in lists.
- Property filters update results.
- Selecting a property updates the summary.
- Selecting a property updates the map.
- Property record page shows:
  - Broker card
  - Days on market
  - Map
  - Location component
  - Carousel
- Broker record page shows broker fields and related properties.
- Property images load from S3.
- Map tiles load from OpenStreetMap.
- Create Property flow can create a property.

## Main Dependency Map

Use this map when something fails.

| Feature | Depends on |
| --- | --- |
| `Property__c.Broker__c` | `Broker__c` object. |
| Broker page and `brokerCard` | Broker fields and `Property__c.Broker__c`. |
| `Days_On_Market__c` | `Date_Listed__c`. |
| `PropertyController.getPagedPropertyList` | `Property__c` fields, `PagedResult`, permissions. |
| `propertyTileList` | `PropertyController`, `propertyTile`, `paginator`, `FiltersChange__c`, `PropertySelected__c`. |
| `propertyListMap` | `PropertyController`, `leafletjs`, CSP for OpenStreetMap, message channels. |
| `propertySummary`, `propertyMap`, `daysOnMarket` | `PropertySelected__c` and property fields. |
| `propertyCarousel` | `PropertyController.getPictures`, `FileUtilities.createFile`, Salesforce Files. |
| `sampleDataImporter` | `SampleDataController`, static resource sample data, Apex class access. |
| `Create_property` flow | `Property__c`, `GeocodingService`, remote site setting, `navigateToRecord`. |
| Dreamhouse app | Tabs, flexipages, content asset logo, permission set visibility. |
| Record page overrides | `Property_Record_Page`, `Broker_Record_Page`, custom objects. |

## Simplest-to-Most-Advanced Build Order Summary

1. Project and org connection.
2. `Broker__c` object and fields.
3. `Property__c` object and simple fields.
4. Lookup, formula, location, picklist, and date fields.
5. Compact layouts, list views, and page layouts.
6. Permission set.
7. Static resources, content asset, CSP, and remote site settings.
8. `PagedResult`.
9. `PropertyController`.
10. `SampleDataController`.
11. `FileUtilities`.
12. `GeocodingService`.
13. Message channels.
14. Utility LWCs.
15. Record display LWCs.
16. Filter, tile list, and list/map LWCs.
17. Settings and mobile helper LWCs.
18. Flow.
19. Tabs.
20. App pages and record pages.
21. Dreamhouse app.
22. Prompt metadata.
23. Sample data.
24. Final tests.

## Practical Rebuild Commands

For a staged rebuild, deploy in this approximate order:

```bash
sf project deploy start --source-dir force-app/main/default/objects/Broker__c --target-org dreamhouse-dev
sf project deploy start --source-dir force-app/main/default/objects/Property__c --target-org dreamhouse-dev
sf project deploy start --source-dir force-app/main/default/layouts --target-org dreamhouse-dev
sf project deploy start --source-dir force-app/main/default/staticresources --target-org dreamhouse-dev
sf project deploy start --source-dir force-app/main/default/contentassets --target-org dreamhouse-dev
sf project deploy start --source-dir force-app/main/default/cspTrustedSites --target-org dreamhouse-dev
sf project deploy start --source-dir force-app/main/default/remoteSiteSettings --target-org dreamhouse-dev
sf project deploy start --source-dir force-app/main/default/classes --target-org dreamhouse-dev
sf project deploy start --source-dir force-app/main/default/messageChannels --target-org dreamhouse-dev
sf project deploy start --source-dir force-app/main/default/lwc --target-org dreamhouse-dev
sf project deploy start --source-dir force-app/main/default/flows --target-org dreamhouse-dev
sf project deploy start --source-dir force-app/main/default/aura --target-org dreamhouse-dev
sf project deploy start --source-dir force-app/main/default/flexipages --target-org dreamhouse-dev
sf project deploy start --source-dir force-app/main/default/tabs --target-org dreamhouse-dev
sf project deploy start --source-dir force-app/main/default/applications --target-org dreamhouse-dev
sf project deploy start --source-dir force-app/main/default/permissionsets --target-org dreamhouse-dev
sf org assign permset --name dreamhouse --target-org dreamhouse-dev
sf data import tree --plan data/sample-data-plan.json --target-org dreamhouse-dev
```

For the final complete deployment:

```bash
sf project deploy start --source-dir force-app --target-org dreamhouse-dev --wait 20
sf org assign permset --name dreamhouse --target-org dreamhouse-dev
sf data import tree --plan data/sample-data-plan.json --target-org dreamhouse-dev
npm install
npm run test:unit
sf apex run test --test-level RunLocalTests --target-org dreamhouse-dev --wait 10 --result-format human
```

## What Each Major File Teaches

| File or folder | Teaches |
| --- | --- |
| `objects/Broker__c` | Simple custom object modeling. |
| `objects/Property__c` | Rich custom object modeling with lookup, geolocation, formulas, picklists, and dates. |
| `permissionsets/dreamhouse.permissionset-meta.xml` | How Salesforce access is layered over metadata. |
| `classes/PagedResult.cls` | Apex DTO for UI data. |
| `classes/PropertyController.cls` | LWC Apex controllers, SOQL filtering, pagination, files. |
| `classes/SampleDataController.cls` | Static resource JSON import and DML. |
| `classes/FileUtilities.cls` | Salesforce Files API from Apex. |
| `classes/GeocodingService.cls` | Invocable Apex and callouts from Flow. |
| `messageChannels/*.xml` | Cross-component communication. |
| `lwc/propertyFilter` | Publishing filter state. |
| `lwc/propertyTileList` | Wired Apex, pagination, child components, selection messages. |
| `lwc/propertyListMap` | Third-party static resources and map/list composition. |
| `lwc/propertySummary` | UI API and record navigation. |
| `lwc/propertyMap` | Selected-record map rendering. |
| `lwc/propertyCarousel` | Record files and image upload. |
| `lwc/sampleDataImporter` | Imperative Apex and toast UX. |
| `flows/Create_property.flow-meta.xml` | Screen Flow, Apex action, file upload, record navigation. |
| `flexipages/*.xml` | How Lightning pages assemble components. |
| `applications/Dreamhouse.app-meta.xml` | App navigation, branding, and record page overrides. |
| `tabs/*.xml` | How users reach objects and app pages. |
| `staticresources/leafletjs` | Loading external JS/CSS in LWC. |
| `data/*.json` | Data import references and relationships. |

## Suggested Learning Exercises

1. Build only `Broker__c`, then create one Broker manually.
2. Build only basic `Property__c` fields, then create one Property manually.
3. Add the Broker lookup, then relate the Property to the Broker.
4. Add `Date_Listed__c` and `Days_On_Market__c`, then watch the formula calculate.
5. Deploy only `PagedResult` and `PropertyController`, then call the controller from Apex tests.
6. Deploy `propertyTile` and `propertyTileList`, then inspect how data flows from Apex to the UI.
7. Deploy message channels and observe how selecting one tile updates other components.
8. Add `propertyListMap` and inspect how `leafletjs` is loaded from a static resource.
9. Add the Flow last and trace how it calls Apex, creates a record, reads files, updates image URLs, and navigates.

The key lesson: Dreamhouse is not one big feature. It is a layered Salesforce app where metadata, security, Apex, LWC, Flow, and app configuration each contribute one understandable piece.
