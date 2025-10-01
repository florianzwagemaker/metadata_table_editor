This document outlines the basic minimum viable product (MVP) requirements for the Metadata-Table-Editor project. The goal of the MVP is to provide a functional and user-friendly interface for loading, editing, and uploading metadata tables in TSV format. The following sections in detail the key features and functionalities that must be included in the MVP.

The project should be developed using Vue.js and should be hosted on GitHub Pages as a static web application.
The application should utilize PrimeVue for UI components, specifically the DataTable component for rendering and editing the metadata table. Additionally, Vite should be used as the build tool for the project.
Basic responsiveness and usability should be warranted but for the purpose of this MVP, mobile device compatibility is not a requirement.

## Minimum Viable Product details

## File Handling: 

* Load and parse a metadata table in TSV format through a URL. This URL can both be provided through a textbox input field or through a URL parameter called `input-metadata`.
* Display the loaded metadata table in a tabular format on the web interface.
* Allow users to download the edited metadata table in TSV format.

## Table rendering and editing:

* The application should use PrimeVue's DataTable component to render the metadata table.
* The table should support inline editing of cell values.
* The table *should not* support adding or deleting rows or columns.
* The table should support sorting and filtering of columns.
* The table should support pagination if the number of rows exceeds a certain threshold (e.g., 50 rows).
* The table should support resizing of columns.
* The table should support reordering of columns through drag-and-drop functionality.
* There should be an option to select which columns are visible in the table, e.g. through a multi-select dropdown with checkboxes.
* The application should provide a way to reset the table to its initial state (e.g., by reloading the original metadata table).

* The table information should be stored in the browser's local storage to preserve changes across page reloads.
* There should be a button to clear the local storage and reset the table to its initial state.

## Upload of edited metadata:

For the upload of the edited metadata table, a few more details are needed that should be provided through URL parameters:
* a `return-url` parameter that specifies the base URL to which the edited metadata table should be uploaded.
* an `organism` parameter that specifies the organism name to be included in the upload request.
  * the final upload url should be constructed as follows: 
  ```
  const revisionUrl = `${returnUrl.replace(/\/$/, '')}/${organism}/revise?groupId=${groupId}&dataUseTermsType=OPEN`
  ```
  Here, the `groupId` is an inferred value from another API call (see below).
* an `auth-endpoint` parameter that specifies the URL of an authentication endpoint to obtain an access token for the upload request. 
* an `auth-realm` parameter that specifies the authentication realm to be used when obtaining the access token.
* an `auth-client-id` parameter that specifies the client ID to be used when obtaining the access token.
  * To obtain the access token, the application should make a request to the authentication endpoint with the provided realm and client ID. The authentication service is keycloak and the protocol to obtain the token is OpenID Connect.
* the `groupId` parameter can be inferred by making a GET request to the return-url with the /user/groups endpoint, using the obtained access token for authentication. The response will contain a list of groups, and the application should select the first group from this list to use as the `groupId`.
* The application should provide a button to upload the edited metadata table to the constructed upload URL.
* the upload should be done through a POST request with the following specifications:
  * The request should include the access token in the Authorization header as a Bearer token.
  * The request should include the edited metadata table as a TSV file in a multipart/form-data payload. The file should be named `metadata.tsv`.
  * The request should include a description field in the multipart/form-data payload as "metadataFile".

* The application should handle the response from the upload request and provide appropriate feedback to the user (e.g., success or error message).
* The application should handle any errors that may occur during the upload process (e.g., network errors, authentication errors, etc.) and provide appropriate feedback to the user.


