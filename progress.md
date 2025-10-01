# MVP Development Progress

This document tracks the progress of the Minimum Viable Product (MVP) development for the Metadata-Table-Editor project.

## Project Setup
- [x] Initialize Vue.js project with Vite
- [x] Install PrimeVue, PrimeFlex, and PrimeIcons
- [x] Install PapaParse for TSV parsing
- [x] Configure PrimeVue in the application

## File Handling
- [x] Load metadata TSV from a URL provided in a textbox
- [x] Load metadata TSV from `input-metadata` URL parameter
- [x] Parse the TSV data
- [x] Display data in a table
- [x] Allow users to download the edited table as TSV

## Table Rendering and Editing (PrimeVue DataTable)
- [x] Render table using PrimeVue's DataTable
- [x] Support inline editing of cells
- [x] Support column sorting
- [x] Support column filtering
- [x] Support pagination
- [x] Support column resizing
- [x] Support column reordering (drag-and-drop)
- [x] Implement column visibility selection
- [x] Implement "Reset" button to reload original data
- [x] Persist table state in local storage
- [x] Implement "Clear Local Storage" button

## Upload of Edited Metadata
- [x] Create "Upload" button
- [x] Implement authentication with Keycloak to get access token
  - [x] Read `auth-endpoint`, `auth-realm`, `auth-client-id` from URL parameters
- [x] Infer `groupId` from `/user/groups` endpoint
- [x] Construct the final upload URL
- [x] Implement the upload POST request with multipart/form-data
- [x] Provide user feedback for upload success or failure
- [x] Handle errors during the upload process