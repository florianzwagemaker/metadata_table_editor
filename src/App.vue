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
const isAuthenticated = ref(false);
const authenticationInProgress = ref(false);

let keycloak = null;

const visibleColumns = computed(() => {
  return allColumns.value.filter(col => selectedColumns.value.includes(col.field));
});

// Check if we have all required parameters for authentication
const hasAuthParams = computed(() => {
  const urlParams = new URLSearchParams(window.location.search);
  return urlParams.get('return-url') && 
         urlParams.get('organism') && 
         urlParams.get('auth-endpoint') && 
         urlParams.get('auth-realm') && 
         urlParams.get('auth-client-id');
});

async function initializeAuthentication() {
  if (!hasAuthParams.value || authenticationInProgress.value) return;
  
  authenticationInProgress.value = true;
  
  const urlParams = new URLSearchParams(window.location.search);
  const authEndpoint = urlParams.get('auth-endpoint');
  const authRealm = urlParams.get('auth-realm');
  const authClientId = urlParams.get('auth-client-id');

  try {
    if (!keycloak) {
      keycloak = new Keycloak({
        url: authEndpoint,
        realm: authRealm,
        clientId: authClientId
      });
    }

    console.log('Initializing Keycloak authentication...');
    const authenticated = await keycloak.init({ 
      onLoad: 'check-sso',
      checkLoginIframe: false, // Disable iframe checking to avoid 403 errors
      enableLogging: true // Add logging for debugging
    });

    console.log('Authentication result:', authenticated);
    if (authenticated) {
      isAuthenticated.value = true;
      toast.add({ 
        severity: 'success', 
        summary: 'Authentication Successful', 
        detail: 'You are now authenticated', 
        life: 3000 
      });
    } else {
      isAuthenticated.value = false;
      toast.add({ 
        severity: 'info', 
        summary: 'Authentication Available', 
        detail: 'Click Upload to authenticate when ready', 
        life: 3000 
      });
    }
  } catch (error) {
    console.error('Authentication initialization error:', error);
    isAuthenticated.value = false;
    toast.add({ 
      severity: 'warn', 
      summary: 'Authentication Available', 
      detail: 'Authentication setup complete. Click Upload to sign in.', 
      life: 3000 
    });
  } finally {
    authenticationInProgress.value = false;
  }
}

async function validateAndRefreshToken() {
  if (!keycloak || !keycloak.authenticated) return false;

  try {
    // Check if token needs refresh (refresh if expires within 30 seconds)
    const tokenValid = await keycloak.updateToken(30);
    if (tokenValid) {
      isAuthenticated.value = true;
      return true;
    } else {
      // Token is still valid
      isAuthenticated.value = true;
      return true;
    }
  } catch (error) {
    console.error('Token refresh failed:', error);
    isAuthenticated.value = false;
    return false;
  }
}

async function ensureAuthenticated() {
  if (!hasAuthParams.value) {
    toast.add({ 
      severity: 'error', 
      summary: 'Error', 
      detail: 'Missing authentication parameters', 
      life: 3000 
    });
    return false;
  }

  // If already authenticated, validate the token
  if (isAuthenticated.value && keycloak && keycloak.authenticated) {
    return await validateAndRefreshToken();
  }

  // If not authenticated, try to authenticate
  try {
    if (!keycloak) {
      // Initialize keycloak if not already done
      await initializeAuthentication();
    }
    
    // If still not authenticated after initialization, trigger login
    if (!keycloak.authenticated) {
      // This will redirect to the login page
      await keycloak.login({
        redirectUri: window.location.href
      });
      // This line won't be reached as login() redirects
      return false;
    }
    
    isAuthenticated.value = true;
    return true;
  } catch (error) {
    console.error('Authentication failed:', error);
    toast.add({ 
      severity: 'error', 
      summary: 'Authentication Failed', 
      detail: 'Could not authenticate user', 
      life: 3000 
    });
    return false;
  }
}

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
    const fastaUrl = urlParams.get('input-fasta');

    if (!returnUrl || !organism || !fastaUrl) {
        toast.add({ severity: 'error', summary: 'Error', detail: 'Missing URL parameters for upload', life: 3000 });
        return;
    }

    // Ensure we're authenticated before proceeding
    const authenticated = await ensureAuthenticated();
    if (!authenticated) {
        return;
    }

    try {
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

onMounted(async () => {
  // First, load saved state or URL parameters for metadata
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

  // Initialize authentication if parameters are available
  if (hasAuthParams.value) {
    await initializeAuthentication();
  }
});
</script>

<template>
  <div class="min-h-screen surface-50">
    <Toast />
    
    <!-- Header -->
    <div class="surface-card shadow-2 px-4 py-2 md:px-6 lg:px-8">
      <div class="flex flex-row gap-4" style="display: flex;">
        <!-- Left side: Title and Auth info -->
        <div class="flex flex-column gap-3" style="flex: 1 1 33.3333%; max-width: 33.3333%;">
          <div class="text-900 font-medium text-2xl mb-1">Metadata Table Editor</div>
          <div class="text-700 text-sm">Load, edit, and upload your metadata tables with ease</div>
          <div v-if="hasAuthParams" class="flex align-items-center gap-2 mb-2">
            <i :class="isAuthenticated ? 'pi pi-check-circle text-green-500' : 'pi pi-exclamation-circle text-orange-500'"></i>
            <span :class="isAuthenticated ? 'text-green-600' : 'text-orange-600'" class="text-sm">
              {{ isAuthenticated ? 'Authenticated' : 'Authentication Available' }}
            </span>
          </div>
        </div>
        
        <!-- Right side: Metadata loading -->
        <div class="flex flex-column gap-3" style="flex: 2 1 66.6667%; max-width: 66.6667%;">
          <div class="flex flex-column gap-2">
            <label for="metadata-url" class="font-semibold text-sm">Metadata URL</label>
            <InputText 
              id="metadata-url" 
              v-model="metadataUrl" 
              placeholder="Enter TSV metadata file URL..."
              class="w-full"
              size="small"
            />
          </div>
          <Button 
            label="Load Metadata" 
            icon="pi pi-download" 
            @click="loadMetadata" 
            :disabled="!metadataUrl"
            size="small"
            class="w-full"
          />
        </div>
      </div>
    </div>

    <!-- Main Content -->
    <div class="px-4 py-3 md:px-6 lg:px-8">

      <!-- Data Table -->
      <div v-if="tableData.length > 0" class="surface-card shadow-1 border-round">
        <div class="p-4 border-bottom-1 surface-border">
          <div class="flex align-items-center justify-content-between">
            <div class="flex align-items-center gap-2">
              <i class="pi pi-table text-primary text-xl"></i>
              <span class="text-900 font-medium text-xl">Metadata Table</span>
            </div>
            <div class="flex align-items-center gap-2">
              <span class="text-600 text-sm">{{ tableData.length }} rows</span>
            </div>
          </div>
        </div>
        
        <div class="p-4">
          <!-- Controls -->
          <div class="flex flex-column lg:flex-row lg:justify-content-between lg:align-items-center gap-4 mb-4">
            <div class="flex flex-column gap-2">
              <label class="font-semibold text-sm">Visible Columns</label>
              <MultiSelect 
                v-model="selectedColumns" 
                :options="allColumns.map(c => c.field)" 
                placeholder="Select columns to display" 
                class="w-full md:w-20rem"
                :maxSelectedLabels="3"
                size="small"
              />
            </div>
            
            <div class="flex flex-wrap gap-2">
              <Button 
                v-if="hasAuthParams"
                label="Upload" 
                icon="pi pi-upload" 
                severity="success"
                @click="uploadMetadata"
                :disabled="authenticationInProgress"
                :loading="authenticationInProgress"
                size="small"
              />
              <Button 
                label="Reset" 
                icon="pi pi-refresh" 
                severity="secondary"
                @click="resetTable" 
                size="small"
              />
              <Button 
                label="Download TSV" 
                icon="pi pi-download" 
                @click="downloadTSV" 
                size="small"
              />
              <Button 
                label="Clear Storage" 
                icon="pi pi-trash" 
                severity="danger"
                outlined
                @click="clearLocalStorage" 
                size="small"
              />
            </div>
          </div>

          <!-- Data Table -->
          <DataTable 
            :value="tableData" 
            editMode="cell" 
            @cell-edit-complete="onCellEditComplete"
            paginator 
            :rows="15" 
            :rowsPerPageOptions="[10, 15, 25, 50]" 
            sortMode="multiple" 
            removableSort 
            filterDisplay="row"
            resizableColumns 
            columnResizeMode="fit" 
            reorderableColumns 
            @column-reorder="onColReorder"
            class="p-datatable-sm"
            responsiveLayout="scroll"
            :pt="{
              table: { style: 'min-width: 50rem' },
              header: { class: 'text-900 font-medium' }
            }"
          >
            <Column 
              v-for="col of visibleColumns" 
              :key="col.field" 
              :field="col.field" 
              :header="col.header" 
              sortable 
              filter
            >
              <template #editor="{ data, field }">
                <InputText v-model="data[field]" class="w-full" />
              </template>
            </Column>
          </DataTable>
        </div>
      </div>

      <!-- Empty State -->
      <div v-else class="surface-card shadow-1 border-round">
        <div class="text-center py-8 px-4">
          <i class="pi pi-inbox text-6xl text-400 mb-4"></i>
          <div class="text-900 font-medium text-xl mb-2">No Data Loaded</div>
          <div class="text-600">Load a metadata file using the URL input in the header above or via URL parameters</div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Remove custom styles - let PrimeVue handle styling */
</style>