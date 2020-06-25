# ADT Explorer

This is a preview release of the ADT Explorer for the ADTv2 public preview service.

Please see the Microsoft [Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct)

## Requirements

Node.js 10+

## Getting Started

### Running locally

1. Ensure you provision your service environment and setup an Azure AD client as per the instructions in the [main repo](https://github.com/azure/azure-digital-twins), specifically:
    * [Set up an Azure Digital Twins instance](https://github.com/Azure/azure-digital-twins/blob/private-preview/Documentation/how-to-set-up-instance.md)
    * [Authenticate against Azure Digital Twins](https://github.com/Azure/azure-digital-twins/blob/private-preview/Documentation/how-to-authenticate.md): part 1 only (i.e. creating the app registration)
      > When adding callback URLs, make sure to add `http://localhost:3000`. While it is possible to customize the port used by ADT Explorer, this is the default.
1. From a command prompt in the `client/src` folder, run `npm install`.
1. From the same command prompt, run `npm run start`.
    > By default, the app runs on port 3000. To customize the port, change the run command. For example, to use port 8080:
    >  * Linux/Mac (Bash): `PORT=8080 npm run start`
    >  * Windows (cmd): `set PORT=8080 && npm run start`
    > Note: Your ADT app registration must have a reply URL using the same port you are using - e.g. localhost:7000 if that is the port you are using.
1. Your browser should open and the app should appear.

### Running in the cloud

1. Deploy the ARM template called `template.json` located under the `deployment` folder into your Azure subscription.
1. Package the client app using `npm run build`. You may need to set `NODE_OPTIONS=--max_old_space_size=4096` if you receive memory-related errors.
1. From the new `build` file, upload each file to the `web` container in the new storage account created by the ARM template.
1. Package the functions app using `dotnet public -c Release -o ./publish`.
1. Zip the contents of the `./publish` folder. E.g. from within the publish folder, run `zip -r AdtExplorerFunctions.zip *`.
1. Publish the functions app using the CLI: `az functionapp deployment source config-zip -g <resource_group> -n <app_name> --src <zip_file_path>`.
1. [Optional] For each Azure Digital Twins environment used with the tool *where live telemetry through SignalR is required*, deploy the `template-eventgrid.json` template in your Azure subscription.
1. Update your Azure AD client to add a new callback URL for the application (i.e. `https://adtexplorer-<your suffix>.azurewebsites.net/`).

### Advanced

When running locally, the Event Grid and SignalR services required for telemetry streaming are not available. However, if you have completed the cloud deployment, you can leverage these services locally to enable the full set of capabilities.

This requires setting the `REACT_APP_BASE_ADT_URL` environment variable to point to your Azure Functions host (e.g. `https://adtexplorer-<your suffix>.azurewebsites.net`). This can be set in the shell environment before starting `npm` or by creating a `.env` file in the `client` folder with `REACT_APP_BASE_ADT_URL=https://...`.

Also, the local URL needs to be added to the allowed origins for the Azure Function and SignalR service. In the ARM template, the default `http://localhost:3000` path is added during deployment; however, if the site is run on a different port locally then both services will need to be updated through the Azure Portal.

## Key features

### First run

Initial authentication is triggered by:
1. Clicking on the sign in button in the top right, or
1. Clicking on an operation that requires calling the service.

Before continuing, you'll need to provide:
1. The Azure AD client ID configured earlier.
1. The Azure AD tenant ID where the app is defined.
1. The target Azure DT URL.

To change these properties at any time, click on the sign in button in the top right.

### Querying

Queries can be issued from the *Query Explorer* panel.

To save a query, click on the Save icon next to the *Run Query* button. This query will then be saved locally and be available in the *Saved Queries* drop down to the left of the query text box. To delete a saved query, click on the *X* icon next to the name of the query when the *Saved Queries* drop down is open.

For large graphs, it's suggested to query only a limited subset and then load the remainder as required. More specifically, you can double click on twins in the graph view to retrieve additional related nodes.

To the right side of the *Query Explorer* toolbar, there are a number of controls to change the layout of the graph. Four different layout algorithms are available alongside options to center, fit to screen, and re-run layout.

### Models

Models can be viewed and managed from the *Model View* panel. The panel will automatically show all available models in your environment on first connection; however, to trigger it explicitly, click on the *Download models* button.

To upload a model, click on the *Upload a model* button and select one more JSON-formatted model files when prompted.

For each model, you can:
1. Delete: remove the definition from your ADT environment.
1. View: see the raw JSON definition of the model.
1. Create a new twin: create a new instance of the model as a twin in the ADT environment. No properties are set as part of this process (aside from name).

### Import/Export

From the *Graph View*, import/export functionality is available.

Export serializes the most recent query results to a JSON-based format, including models, twins, and relationships.

Import deserializes from either a custom Excel-based format (see the `examples` folder) or the JSON-based format generated on export. Before import is executed, a preview of the graph is presented for validation.

### Editing twins

Selecting a node in the *Graph View* shows its properties in the *Property Explorer*. This includes default values for properties that have not yet been set.

To edit writeable properties, update thier values inline and click the Save button at the top of the view. The resulting patch operation on the API is then shown in a modal.

Selecting two nodes allows for the creation of relationships. Multi-selection is enabled by holding down CTRL/CMD keys. Ensure twins are selected in the order of the relationship direction (i.e the first twin selected will be the source). Once two twins are selected, click on the `Create Relationship` button and select the type of relationship.

Multi-select is also enabled for twin deletion.

### Advanced Settings

Clicking the settings cog in the top right corner allows the configuration of the following advanced features:
1. Eager Loading: in the case the twins returned by a query have relationships to twins *not* returned by the query, this feature will load these missing twins before rendering the graph.
1. Caching: this keeps a local cache of relationships and models in memory to improve query performance. These caches are cleared on any write operations on the relevant components (or on browser refresh).
1. Console & Output windows: these are hidden by default. The console window enables the use of simple shell functions for workign with the graph. The output window shows a diagnostic trace of operations.
1. Number of layers to expand: when double clicking on a node, this number indicates how many layers of relationships to fetch.
1. Expansion direction: when double clicking on a node, this indicates which kinds of relationships to follow when expanding.

## Extensibility points

### Import

Import plugins are found in `src/services/plugins` directory within the client code base. Each plugin should be defined as a class with a single function:

```ts
tryLoad(file: File): Promise<ImportModel | boolean>
```

If the plugin can import the file, it should return an `ImportModel`. If it cannot import the file, it should return `false` so the import service can share the file with other plugins.

The `ImportModel` should be structured as follows:

```ts
class DataModel {
  digitalTwinsFileInfo: DataFileInfoModel;
  digitalTwinsGraph: DataGraphModel;
  digitalTwinsModels: DigitalTwinModel[];
}

class DataFileInfoModel {
  fileVersion: string; // should be "1.0.0"
}

class DataGraphModel {
  digitalTwins: DigitalTwin[]; // objects align with structure returned by API
  relationships: DigitalTwinRelationship[]; // objects align with structure returned by API
}
```

New plugins need to be registered in `ImportPlugins` collection at the top of the `src/services/ImportService.js` file.

Currently, import plugins for Excel and JSON are provided. To support custom formats of either, the new plugins would need to be placed first in the `ImportPlugins` collection or they would need to be extended to detect the custom format (and either parse in place or return `false` to allow another plugin to parse).

The `ExcelImportPlugin` is designed to support additional Excel-based formats. Currently, all files are parsed through the `StandardExcelImportFormat` class; however, it would be relatively straightforward to inspect cell content to detect specific structures and call an alternative import class instead.

### Export

Graphs can be exported as JSON files (which can then be re-imported). The structured of the files follows the `DataModel` class described in the previous section.

Export is managed by the `ExportService` class in `src/services/ExportService.js` file.

To alter the export format structure, the existing logic within the `ExportService` to extract the contents of the graph could be reused and then re-formatted as desired.

### Views

All panels are defined in the `src/App.js` file. These configuration objects are defined by the requirements of the Golden Layout Component.

For temporary panels within the application (e.g. import preview), two approaches can be considered:
1. For panels like output & console, the new panel can be added to the `optionalComponentsConfig` collection. This allows the panel's state (i.e. open or closed) to be managed through the app state, regardless of whether it is closed via the 'X' on the tab or when it is closed via configuration (like available in the preferences dialog).
1. For panels like import preview, these can be manaully added on demand to the layout. This can be cleanly done via the pub/sub mechanism (see below and the `componentDidMount` method in `App.js`).

### View commands

Where a view has commands, it's suggested that a dedicated command bar component is created (based on components like that found in `src/components/GraphViewerComponent/GraphViewerCommandBarComponent.js`). These leverage the Office Fabric UI `CommandBar` component and either expose callbacks for functionality via props or manage operations directly.

### Pub/Sub

The Golden Layout Component includes a pub/sub message bus for communication between components. This is key part of the ADT Explorer and is used to pass messages between components.

All events - via publish and subscribe methods - are defined in the `src/services/EventService.js` file. Additional events can be defined by adding to this file.

The pub/sub message bus is not immediately available on application load; however, the event service will buffer any pub or sub requests during this period and then apply them once available.

## Services

### Local

When running locally, all requests to the ADT service are proxied through the same local web server used for hosting the client app. This is configured in the `client/src/setupProxy.js` file.

### Cloud

When running in the cloud, Azure Functions hosts three services to support the front end application:
1. Proxy: this proxies requests through the ADT service (much in the same way as the proxy used when running locally).
1. SignalR: this allows clients to retrieve credentials to access the SignalR service for live telemetry updates. It also validates that the endpoint and route required to stream information from the ADT service to the ADT Explorer app is in place. If the managed service identity for the Function is configured correctly (i.e. has write permissions on the resource group and can administer the ADT service), then it can create these itself.
1. EventGrid: this receives messages from the Event Grid to broadcasts them to any listening clients using SignalR. The messages are sent from ADT to the function via ADT endpoint and route.