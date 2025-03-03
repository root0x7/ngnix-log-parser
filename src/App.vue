<template>
  <div class="container mx-auto px-4 py-6">
    <h1 class="text-2xl font-bold mb-4">NGINX Log Analyzer</h1>
    
    <div class="mb-6">
      <label for="file-input" class="block mb-2 font-medium">Select NGINX log file:</label>
      <input
        id="file-input"
        type="file"
        @change="handleFileUpload"
        class="p-2 border rounded-md w-full"
        accept=".log,.txt"
      />
      <p v-if="fileName" class="mt-2 text-sm text-gray-600">
        Selected file: {{ fileName }}
      </p>
    </div>
    
    <div v-if="isLoading" class="text-center py-4">
      <p>Processing log file...</p>
    </div>
    
    <div v-if="parsedLogs.length > 0" class="overflow-x-auto">
      <div class="flex justify-between items-center mb-4">
        <h2 class="text-lg font-semibold">{{ parsedLogs.length }} Log Entries</h2>
        <div class="flex space-x-2">
          <input 
            v-model="searchQuery" 
            placeholder="Search logs..." 
            class="px-3 py-1 border rounded-md"
            @input="filterLogs"
          />
        </div>
      </div>
      
      <table class="w-full border-collapse border">
        <thead>
          <tr class="bg-gray-100">
            <th class="p-2 border text-left">IP Address</th>
            <th class="p-2 border text-left">Timestamp</th>
            <th class="p-2 border text-left">Method</th>
            <th class="p-2 border text-left">URL</th>
            <th class="p-2 border text-left">Status</th>
            <th class="p-2 border text-left">Size</th>
            <th class="p-2 border text-left">Referrer</th>
            <th class="p-2 border text-left">User Agent</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(log, index) in filteredLogs" :key="index" :class="getStatusClass(log.status)">
            <td class="p-2 border">{{ log.ipAddress }}</td>
            <td class="p-2 border">{{ log.timestamp }}</td>
            <td class="p-2 border">{{ log.method }}</td>
            <td class="p-2 border truncate max-w-xs">{{ log.url }}</td>
            <td class="p-2 border">{{ log.status }}</td>
            <td class="p-2 border">{{ formatBytes(log.size) }}</td>
            <td class="p-2 border truncate max-w-xs">{{ log.referrer }}</td>
            <td class="p-2 border truncate max-w-xs">{{ log.userAgent }}</td>
          </tr>
        </tbody>
      </table>
    </div>
    
    <div v-else-if="fileProcessed && !isLoading" class="text-red-500 mt-4">
      No valid NGINX logs detected in the file. Please check the format and try again.
    </div>
  </div>
</template>

<script>
export default {
  name: 'App',
  data() {
    return {
      parsedLogs: [],
      fileName: '',
      isLoading: false,
      fileProcessed: false,
      searchQuery: '',
      filteredLogs: []
    }
  },
  methods: {
    handleFileUpload(event) {
      const file = event.target.files[0];
      if (!file) return;
      
      this.fileName = file.name;
      this.isLoading = true;
      this.fileProcessed = false;
      this.parsedLogs = [];
      
      const reader = new FileReader();
      
      reader.onload = (e) => {
        const content = e.target.result;
        this.parseNginxLogs(content);
        this.isLoading = false;
        this.fileProcessed = true;
      };
      
      reader.onerror = () => {
        console.error('Error reading file');
        this.isLoading = false;
        this.fileProcessed = true;
      };
      
      reader.readAsText(file);
    },
    
    parseNginxLogs(content) {
      const logs = content.trim().split('\n');
      this.parsedLogs = [];
      
      const logPattern = /^(\S+) - - \[(.*?)\] "(\S+) (.*?) (\S+)" (\d+) (\d+) "([^"]*)" "([^"]*)"$/;
      
      logs.forEach(logLine => {
        const match = logLine.match(logPattern);
        if (match) {
          this.parsedLogs.push({
            ipAddress: match[1],
            timestamp: match[2],
            method: match[3],
            url: match[4],
            protocol: match[5],
            status: match[6],
            size: match[7],
            referrer: match[8] !== '-' ? match[8] : '',
            userAgent: match[9]
          });
        }
      });
      
      this.filteredLogs = [...this.parsedLogs];
    },
    
    filterLogs() {
      if (!this.searchQuery.trim()) {
        this.filteredLogs = [...this.parsedLogs];
        return;
      }
      
      const query = this.searchQuery.toLowerCase();
      this.filteredLogs = this.parsedLogs.filter(log => {
        return (
          log.ipAddress.toLowerCase().includes(query) ||
          log.url.toLowerCase().includes(query) ||
          log.status.includes(query) ||
          log.method.toLowerCase().includes(query) ||
          log.userAgent.toLowerCase().includes(query)
        );
      });
    },
    
    formatBytes(bytes) {
      if (bytes === '0' || bytes === '-') return '0 B';
      const sizes = ['B', 'KB', 'MB', 'GB', 'TB'];
      const i = Math.floor(Math.log(parseInt(bytes)) / Math.log(1024));
      return (parseInt(bytes) / Math.pow(1024, i)).toFixed(2) + ' ' + sizes[i];
    },
    
    getStatusClass(status) {
      const statusCode = parseInt(status);
      if (statusCode >= 200 && statusCode < 300) return 'bg-green-50';
      if (statusCode >= 300 && statusCode < 400) return 'bg-blue-50';
      if (statusCode >= 400 && statusCode < 500) return 'bg-yellow-50';
      if (statusCode >= 500) return 'bg-red-50';
      return '';
    }
  }
}
</script>

<style>
.container {
  max-width: 1200px;
  margin-left: auto;
  margin-right: auto;
  padding-left: 1rem;
  padding-right: 1rem;
  padding-top: 1.5rem;
  padding-bottom: 1.5rem;
}

h1 {
  font-size: 1.5rem;
  font-weight: 700;
  margin-bottom: 1rem;
}

.mb-6 {
  margin-bottom: 1.5rem;
}

.mb-4 {
  margin-bottom: 1rem;
}

.mt-2 {
  margin-top: 0.5rem;
}

.block {
  display: block;
}

.font-medium {
  font-weight: 500;
}

.p-2 {
  padding: 0.5rem;
}

.border {
  border-width: 1px;
  border-style: solid;
  border-color: #e5e7eb;
}

.rounded-md {
  border-radius: 0.375rem;
}

.w-full {
  width: 100%;
}

.text-center {
  text-align: center;
}

.py-4 {
  padding-top: 1rem;
  padding-bottom: 1rem;
}

.text-sm {
  font-size: 0.875rem;
}

.text-gray-600 {
  color: #4b5563;
}

.text-red-500 {
  color: #ef4444;
}

.overflow-x-auto {
  overflow-x: auto;
}

.flex {
  display: flex;
}

.justify-between {
  justify-content: space-between;
}

.items-center {
  align-items: center;
}

.text-lg {
  font-size: 1.125rem;
}

.font-semibold {
  font-weight: 600;
}

.space-x-2 > * + * {
  margin-left: 0.5rem;
}

.px-3 {
  padding-left: 0.75rem;
  padding-right: 0.75rem;
}

.py-1 {
  padding-top: 0.25rem;
  padding-bottom: 0.25rem;
}

table {
  width: 100%;
  border-collapse: collapse;
}

th {
  text-align: left;
  padding: 0.5rem;
  border: 1px solid #e5e7eb;
}

td {
  padding: 0.5rem;
  border: 1px solid #e5e7eb;
}

.bg-gray-100 {
  background-color: #f3f4f6;
}

.bg-green-50 {
  background-color: #ecfdf5;
}

.bg-blue-50 {
  background-color: #eff6ff;
}

.bg-yellow-50 {
  background-color: #fefce8;
}

.bg-red-50 {
  background-color: #fef2f2;
}

.truncate {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.max-w-xs {
  max-width: 20rem;
}
</style>