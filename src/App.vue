<script setup>
import { ref, onMounted, computed, watch } from 'vue';
import InputText from 'primevue/inputtext';
import Button from 'primevue/button';
import DataTable from 'primevue/datatable';
import Column from 'primevue/column';
import Papa from 'papaparse';
import Toast from 'primevue/toast';
import { useToast } from 'primevue/usetoast';
import MultiSelect from 'primevue/multiselect';
import Keycloak from 'keycloak-js';

const metadataUrl = ref('');
const tableData = ref([]);
const allColumns = ref([]);
const selectedColumns = ref([]);
const toast = useToast();

let keycloak = null;

const visibleColumns = computed(() => {
  return allColumns.value.filter(col => selectedColumns.value.includes(col.field));
});

function loadMetadata() {
  if (!metadataUrl.value) return;

  Papa.parse(metadataUrl.value, {
    download: true,
    header: true,
    skipEmptyLines: true,
    complete: (results) => {
      tableData.value = results.data;
      if (results.data.length > 0) {
        const fields = Object.keys(results.data[0]);
        allColumns.value = fields.map(field => ({ field, header: field }));
        selectedColumns.value = fields;
      }
      toast.add({ severity: 'success', summary: 'Success', detail: 'Metadata loaded successfully', life: 3000 });
    },
    error: (error) => {
      console.error('Error parsing TSV:', error);
      toast.add({ severity: 'error', summary: 'Error', detail: 'Failed to load or parse metadata', life: 3000 });
    }
  });
}

function downloadTSV() {
  const tsv = Papa.unparse(tableData.value);
  const blob = new Blob([tsv], { type: 'text/tab-separated-values;charset=utf-8;' });
  const link = document.createElement('a');
  link.href = URL.createObjectURL(blob);
  link.setAttribute('download', 'metadata.tsv');
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}

function onCellEditComplete(event) {
    let { data, newValue, field } = event;
    data[field] = newValue;
    toast.add({ severity: 'success', summary: 'Cell Updated', detail: `Value changed to "${newValue}"`, life: 3000 });
}

function onColReorder(event) {
    allColumns.value = event.columns;
}

function resetTable() {
    loadMetadata();
}

function clearLocalStorage() {
    localStorage.removeItem('metadata-table-editor-state');
    toast.add({ severity: 'warn', summary: 'Local Storage Cleared', detail: 'Please reload the page to see the changes', life: 3000 });
}

async function uploadMetadata() {
    const urlParams = new URLSearchParams(window.location.search);
    const returnUrl = urlParams.get('return-url');
    const organism = urlParams.get('organism');
    const authEndpoint = urlParams.get('auth-endpoint');
    const authRealm = urlParams.get('auth-realm');
    const authClientId = urlParams.get('auth-client-id');
    const fastaUrl = urlParams.get('input-fasta');

    if (!returnUrl || !organism || !authEndpoint || !authRealm || !authClientId || !fastaUrl) {
        toast.add({ severity: 'error', summary: 'Error', detail: 'Missing URL parameters for upload', life: 3000 });
        return;
    }

    if (!keycloak) {
        keycloak = new Keycloak({
            url: authEndpoint,
            realm: authRealm,
            clientId: authClientId
        });
    }

    try {
        const authenticated = await keycloak.init({ onLoad: 'login-required' });
        if (!authenticated) {
            toast.add({ severity: 'error', summary: 'Authentication Failed', detail: 'User not authenticated', life: 3000 });
            return;
        }

        const token = keycloak.token;

        const [groupsResponse, fastaResponse] = await Promise.all([
            fetch(`${returnUrl.replace(/\/$/, '')}/user/groups`, {
                headers: { 'Authorization': `Bearer ${token}` }
            }),
            fetch(fastaUrl)
        ]);

        if (!groupsResponse.ok) throw new Error('Failed to fetch user groups');
        if (!fastaResponse.ok) throw new Error('Failed to fetch FASTA file');

        const groups = await groupsResponse.json();
        const fastaBlob = await fastaResponse.blob();
        const groupId = groups[0].groupId;

        const revisionUrl = `${returnUrl.replace(/\/$/, '')}/${organism}/revise?groupId=${groupId}&dataUseTermsType=OPEN`;

        const dataForUpload = tableData.value.map(row => {
            const newRow = { ...row };
            const accession = newRow['accessionVersion'];
            const sample = newRow['SampleID'];

            if (accession && sample) {
                newRow.id = `${accession}|${sample}`;
            } else if (!newRow.hasOwnProperty('id')) {
                newRow.id = '';
            }
            return newRow;
        });

        const originalHeaders = allColumns.value.map(c => c.field);
        const headersForUpload = ['id', ...originalHeaders.filter(h => h.toLowerCase() !== 'id' && h !== 'submissionId')];
        const tsv = Papa.unparse(dataForUpload, { columns: headersForUpload, delimiter: '\t' });
        const metadataBlob = new Blob([tsv], { type: 'text/tab-separated-values;charset=utf-8;' });

        const formData = new FormData();
        formData.append('metadataFile', metadataBlob, 'metadata.tsv');
        formData.append('sequenceFile', fastaBlob, 'sequence.fasta');
        formData.append('description', 'metadataFile');
        const uploadResponse = await fetch(revisionUrl, {
            method: 'POST',
            headers: { 'Authorization': `Bearer ${token}` },
            body: formData
        });

        if (uploadResponse.ok) {
            toast.add({ severity: 'success', summary: 'Success', detail: 'Metadata uploaded successfully', life: 3000 });
        } else {
            throw new Error('Upload failed');
        }

    } catch (error) {
        console.error('Upload error:', error);
        toast.add({ severity: 'error', summary: 'Error', detail: 'An error occurred during upload', life: 3000 });
    }
}

watch([tableData, allColumns, selectedColumns], (newState) => {
  const state = {
    tableData: newState[0],
    allColumns: newState[1],
    selectedColumns: newState[2],
    metadataUrl: metadataUrl.value
  };
  localStorage.setItem('metadata-table-editor-state', JSON.stringify(state));
}, { deep: true });

onMounted(() => {
  const savedState = localStorage.getItem('metadata-table-editor-state');
  if (savedState) {
    const state = JSON.parse(savedState);
    tableData.value = state.tableData;
    allColumns.value = state.allColumns;
    selectedColumns.value = state.selectedColumns;
    metadataUrl.value = state.metadataUrl;
  } else {
    const urlParams = new URLSearchParams(window.location.search);
    const inputMetadataUrl = urlParams.get('input-metadata');
    if (inputMetadataUrl) {
      metadataUrl.value = inputMetadataUrl;
      loadMetadata();
    }
  }
});
</script>

<template>
  <div id="app">
    <Toast />
    <header>
      <h1>Metadata Table Editor</h1>
    </header>
    <main>
      <div class="card">
        <div class="p-fluid">
          <div class="p-field">
            <label for="metadata-url">Metadata URL</label>
            <InputText id="metadata-url" v-model="metadataUrl" type="text" />
          </div>
          <Button label="Load" @click="loadMetadata" />
        </div>
      </div>
      <div class="card">
        <div class="flex justify-content-between mb-4">
          <MultiSelect v-model="selectedColumns" :options="allColumns.map(c => c.field)" placeholder="Select Columns" style="width: 20rem" />
          <div>
            <Button label="Upload" icon="pi pi-upload" class="p-button-success mr-2" @click="uploadMetadata" />
            <Button label="Clear Local Storage" icon="pi pi-trash" class="p-button-danger mr-2" @click="clearLocalStorage" />
            <Button label="Reset" icon="pi pi-refresh" class="p-button-secondary mr-2" @click="resetTable" />
            <Button label="Download TSV" icon="pi pi-download" @click="downloadTSV" />
          </div>
        </div>
        <DataTable :value="tableData" responsiveLayout="scroll" editMode="cell" @cell-edit-complete="onCellEditComplete"
                   paginator :rows="10" :rowsPerPageOptions="[10, 20, 50]" sortMode="multiple" removableSort filterDisplay="row"
                   resizableColumns columnResizeMode="fit" reorderableColumns @column-reorder="onColReorder">
          <Column v-for="col of visibleColumns" :key="col.field" :field="col.field" :header="col.header" sortable filter>
            <template #editor="{ data, field }">
                <InputText v-model="data[field]" autofocus />
            </template>
          </Column>
        </DataTable>
      </div>
    </main>
  </div>
</template>

<style scoped>
#app {
  padding: 2rem;
}

header {
  margin-bottom: 2rem;
}

.card {
  background: #ffffff;
  padding: 2rem;
  border-radius: 10px;
  margin-bottom: 2rem;
  box-shadow: 0 2px 1px -1px rgba(0,0,0,.2), 0 1px 1px 0 rgba(0,0,0,.14), 0 1px 3px 0 rgba(0,0,0,.12);
}

.p-fluid .p-field {
  margin-bottom: 1rem;
}

.flex {
    display: flex;
}

.justify-content-between {
    justify-content: space-between;
}

.mb-4 {
    margin-bottom: 1rem;
}

.mr-2 {
    margin-right: 0.5rem;
}
</style>