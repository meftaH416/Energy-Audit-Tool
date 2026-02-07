# Energy Audit Tool - Technical Documentation

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Data Structure](#data-structure)
3. [Core Functions](#core-functions)
4. [UI Components](#ui-components)
5. [Storage System](#storage-system)
6. [PDF Generation](#pdf-generation)
7. [Event Handling](#event-handling)
8. [Future Development Ideas](#future-development-ideas)

---

## Architecture Overview

### Technology Stack
- **Pure HTML/CSS/JavaScript** - No framework dependencies
- **External Libraries**:
  - `jsPDF` (2.5.1) - PDF generation
  - `html2canvas` (1.4.1) - Currently imported but not used (optional for future features)

### Design Pattern
- **Single Page Application (SPA)** - All functionality in one HTML file
- **Event-driven architecture** - User interactions trigger auto-save
- **Client-side storage** - localStorage for persistence
- **Dynamic DOM manipulation** - JavaScript creates/destroys UI elements

### File Structure
```
energy-audit-tool-improved.html
├── <head>
│   ├── Metadata & SEO
│   ├── External library imports (jsPDF, html2canvas)
│   └── <style> - All CSS inline
└── <body>
    ├── UI Structure (HTML)
    └── <script> - All JavaScript inline
```

---

## Data Structure

### Storage Schema

```javascript
{
  basic: {
    name: String,              // Company name
    address: String,           // Company address
    date: String,              // Audit date (YYYY-MM-DD)
    products: String,          // Products produced
    volume: String,            // Production volume
    shift: String,             // Operating hours
    employee: String,          // Number of employees
    'raw-materials': String,   // Raw materials description
    'plant-area': String,      // Plant area description
    'process-flow': String     // Process description
  },
  plantLayoutPhoto: String|null,    // Base64 image data
  processPhoto: String|null,        // Base64 image data
  equipmentTypes: [
    {
      name: String,            // Equipment type name
      instances: [
        {
          desc: String,        // Description/model/location
          hours: String,       // Hours of operation
          inputs: [String],    // Energy inputs array
          photos: [String]     // Base64 image array
        }
      ]
    }
  ]
}
```

### localStorage Key
- **Key**: `'energy-audit-draft'`
- **Format**: JSON string
- **Max Size**: ~5-10MB (browser dependent, mostly limited by images)

---

## Core Functions

### 1. State Management

#### `getFormState()`
**Purpose**: Extracts all form data into a structured object  
**Returns**: Object matching the storage schema  
**Called by**: `saveDraft()`

```javascript
function getFormState() {
  // 1. Collect basic fields from form inputs
  // 2. Extract plant/process photos from preview divs
  // 3. Loop through equipment type blocks
  // 4. For each type, loop through instances
  // 5. Collect instance data (desc, hours, inputs, photos)
  // 6. Return structured object
}
```

**Key Logic**:
- Uses `querySelectorAll` to find all equipment blocks dynamically
- Filters empty values from inputs array
- Only includes equipment types that have name OR instances

---

#### `saveDraft()`
**Purpose**: Saves current form state to localStorage  
**Timing**: Debounced by 800ms  
**Error Handling**: Catches `QuotaExceededError` for storage limits

```javascript
function saveDraft() {
  try {
    const state = getFormState();
    
    // Only save if there's actual content
    const hasContent = /* check if any field has data */;
    
    if (hasContent) {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
      // Update status message
    }
  } catch (err) {
    // Handle storage quota errors
  }
}
```

**Status Messages**:
- ✓ Success: Green text with timestamp
- ⚠ Error: Red text
- No data: Gray text

---

#### `restoreDraft()`
**Purpose**: Loads saved data from localStorage on page load  
**Called**: Once on initialization

```javascript
function restoreDraft() {
  // 1. Get saved data from localStorage
  // 2. Parse JSON (with error handling)
  // 3. Restore basic form fields
  // 4. Restore plant/process photos
  // 5. Rebuild equipment types dynamically
  // 6. For each type, rebuild instances
  // 7. Restore instance photos and inputs
}
```

**Important**: Uses `skipDefaultInstance` and `skipDefaults` flags to prevent adding empty rows during restoration.

---

#### `clearDraft()`
**Purpose**: Wipes all saved data and reloads page  
**Safety**: Requires confirmation dialog

```javascript
function clearDraft() {
  if (confirm("Clear ALL saved draft data?")) {
    isClearing = true;  // Prevent auto-save during clear
    localStorage.removeItem(STORAGE_KEY);
    window.location.replace(window.location.href);
  }
}
```

---

### 2. Utility Functions

#### `debounce(fn, delay)`
**Purpose**: Rate-limits function calls to prevent excessive saves  
**Default delay**: 800ms

```javascript
function debounce(fn, delay = 800) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}
```

**Usage**: `const debouncedSave = debounce(saveDraft);`

---

#### `attachAutoSave()`
**Purpose**: Attaches event listeners to all inputs for auto-save  
**Called**: After adding new UI elements

```javascript
function attachAutoSave() {
  document.querySelectorAll('input, textarea, [type="file"]').forEach(el => {
    el.removeEventListener('input', debouncedSave);  // Prevent duplicates
    el.removeEventListener('change', debouncedSave);
    el.addEventListener('input', debouncedSave);
    el.addEventListener('change', debouncedSave);
  });
}
```

**Events Listened**:
- `input` - For text fields
- `change` - For file inputs and dates

---

## UI Components

### 1. Equipment Type Block

**Structure**:
```
equipment-type-block (div)
├── equipment-type-header (div)
│   ├── Title: "Equipment Type #N"
│   └── Delete Type button
├── Type name input field
├── "Add Instance" button
└── instance-container (div)
    └── [Multiple equipment instances]
```

**Function**: `addEquipmentType(skipDefaultInstance = false)`

```javascript
function addEquipmentType(skipDefaultInstance = false) {
  typeCounter++;  // Global counter for unique IDs
  const typeId = `type-${typeCounter}`;
  
  // Create DOM structure
  const typeBlock = document.createElement('div');
  typeBlock.className = 'equipment-type-block';
  typeBlock.id = typeId;
  typeBlock.innerHTML = `...`;
  
  container.appendChild(typeBlock);
  
  if (!skipDefaultInstance) {
    addInstance(typeId, typeCounter);  // Add first instance
  }
  
  attachAutoSave();
}
```

**Key Features**:
- Each type gets unique ID: `type-1`, `type-2`, etc.
- Can be deleted individually
- Contains multiple instances

---

### 2. Equipment Instance

**Structure**:
```
equipment-instance (div)
├── Delete button
├── instance-header (div) - "Instance #N"
├── Description textarea
├── Hours of operation input
├── inputs-section (div)
│   ├── inputs-list (div)
│   │   └── [Multiple input rows]
│   └── "Add Input" button
└── photo-upload (div)
    ├── "Add Photo" button
    └── instance-photo-preview (div)
        └── [Photo input wrappers with previews]
```

**Function**: `addInstance(typeId, typeNum, skipDefaults = false)`

```javascript
function addInstance(typeId, typeNum, skipDefaults = false) {
  const instancesDiv = document.getElementById(`instances-${typeId}`);
  const currentInstances = instancesDiv.querySelectorAll('.equipment-instance').length;
  const instanceNumber = currentInstances + 1;
  const instId = `inst-${typeId}-${instanceNumber}`;
  
  // Create instance DOM
  const instance = document.createElement('div');
  instance.className = 'equipment-instance';
  instance.id = instId;
  instance.innerHTML = `...`;
  
  instancesDiv.appendChild(instance);
  
  if (!skipDefaults) {
    addPhotoInput(instId);
    addInput(instId);
  }
  
  attachAutoSave();
}
```

**Naming Convention**: `inst-type-1-1`, `inst-type-1-2`, etc.

---

### 3. Input Row (Energy Sources)

**Function**: `addInput(instId)`

```javascript
function addInput(instId) {
  const list = document.getElementById(`inputs-${instId}`);
  const inputRowId = `input-${instId}-${Date.now()}`;  // Unique timestamp ID
  
  const row = document.createElement('div');
  row.className = 'input-row';
  row.id = inputRowId;
  row.innerHTML = `
    <input type="text" placeholder="e.g. Electricity (kWh)...">
    <button onclick="deleteInput('${inputRowId}')">×</button>
  `;
  
  list.appendChild(row);
  attachAutoSave();
}
```

**Delete**: `deleteInput(rowId)` - Removes row and triggers save

---

### 4. Photo Input

**Function**: `addPhotoInput(instId)`

```javascript
function addPhotoInput(instId) {
  const previewContainer = document.getElementById(`photo-preview-${instId}`);
  
  // Create file input
  const input = document.createElement('input');
  input.type = 'file';
  input.accept = 'image/*';
  input.capture = 'environment';  // Mobile camera hint
  
  // Create remove button
  const removeBtn = document.createElement('button');
  removeBtn.onclick = () => { wrapper.remove(); debouncedSave(); };
  
  // Create wrapper
  const wrapper = document.createElement('div');
  wrapper.appendChild(input);
  wrapper.appendChild(removeBtn);
  
  // Create preview div
  const photoPreviewDiv = document.createElement('div');
  wrapper.appendChild(photoPreviewDiv);
  
  // Handle file selection
  input.addEventListener('change', e => {
    const file = e.target.files[0];
    if (!file) return;
    
    const reader = new FileReader();
    reader.onload = ev => {
      const img = document.createElement('img');
      img.src = ev.target.result;  // Base64 data URL
      photoPreviewDiv.innerHTML = '';
      photoPreviewDiv.appendChild(img);
      debouncedSave();
    };
    reader.readAsDataURL(file);  // Convert to base64
  });
  
  previewContainer.parentElement.insertBefore(wrapper, previewContainer);
  attachAutoSave();
}
```

**Image Storage**:
- Uses `FileReader.readAsDataURL()` to convert to base64
- Stores base64 string in localStorage
- No server upload required

---

## Storage System

### localStorage Implementation

**Key**: `'energy-audit-draft'`

**Advantages**:
- ✅ Persistent across sessions
- ✅ No server required
- ✅ ~5-10MB capacity (browser dependent)
- ✅ Synchronous API (simple to use)

**Limitations**:
- ❌ Not synced across devices
- ❌ Cleared if user clears browser data
- ❌ Domain-specific (can't share across sites)
- ❌ Size limits can be hit with many high-res photos

### Auto-Save Strategy

```
User types → Input event → Debounce (800ms) → saveDraft() → localStorage
```

**Why debounce?**
- Prevents saving on every keystroke
- Reduces localStorage write operations
- Better performance

### Data Recovery

**On page load**:
1. `restoreDraft()` runs immediately
2. Checks for `STORAGE_KEY` in localStorage
3. If found, parses JSON and rebuilds UI
4. If corrupted, clears and starts fresh

**On page unload**:
1. `beforeunload` event triggers
2. Calls `saveDraft()` one final time
3. Unless `isClearing` flag is set

---

## PDF Generation

### Overview
Uses **jsPDF** library to create multi-page PDFs with embedded images.

### Main Function: `downloadPDF()`

```javascript
document.getElementById('download-pdf').onclick = async () => {
  // 1. Check if jsPDF is loaded
  if (!window.jspdf?.jsPDF) {
    alert("PDF library not loaded");
    return;
  }
  
  // 2. Show progress indicator
  progressDiv.style.display = 'block';
  
  try {
    const { jsPDF } = window.jspdf;
    const pdf = new jsPDF('p', 'mm', 'a4');  // Portrait, mm units, A4 size
    
    // 3. Initialize variables
    const pageWidth = 210;   // A4 width in mm
    const pageHeight = 297;  // A4 height in mm
    const margin = 15;
    const maxWidth = pageWidth - 2 * margin;
    let y = margin;  // Current Y position
    
    // 4. Add content with helper functions
    // ... (see detailed breakdown below)
    
    // 5. Save PDF
    const filename = `${companyName}_Energy_Audit_${date}.pdf`;
    pdf.save(filename);
    
  } catch (err) {
    alert("PDF generation failed: " + err.message);
  } finally {
    progressDiv.style.display = 'none';
  }
};
```

---

### Helper Functions

#### `checkPageBreak(neededSpace)`
**Purpose**: Adds new page if content won't fit

```javascript
const checkPageBreak = (neededSpace = 15) => {
  if (y + neededSpace > pageHeight - margin) {
    pdf.addPage();
    y = margin;  // Reset Y position
  }
};
```

---

#### `addText(text, fontSize, isBold)`
**Purpose**: Adds text with word wrapping

```javascript
const addText = (text, fontSize = 11, isBold = false) => {
  checkPageBreak();  // Ensure space
  pdf.setFontSize(fontSize);
  pdf.setFont(undefined, isBold ? 'bold' : 'normal');
  
  const lines = pdf.splitTextToSize(text, maxWidth);  // Word wrap
  pdf.text(lines, margin, y);
  
  y += lines.length * (fontSize * 0.4) + 3;  // Update Y position
};
```

**Line height calculation**: `fontSize * 0.4` approximates line spacing

---

#### `addImage(imgSrc, label)`
**Purpose**: Embeds base64 image in PDF

```javascript
const addImage = async (imgSrc, label) => {
  if (!imgSrc) return;
  
  checkPageBreak(80);  // Images need more space
  addText(label, 10, true);  // Image caption
  
  try {
    // Load image to get dimensions
    const img = new Image();
    img.src = imgSrc;
    await new Promise((resolve, reject) => {
      img.onload = resolve;
      img.onerror = reject;
    });
    
    // Calculate scaled dimensions
    const imgWidth = maxWidth;
    const imgHeight = (img.height / img.width) * imgWidth;
    const finalHeight = Math.min(imgHeight, 120);  // Cap at 120mm
    
    checkPageBreak(finalHeight + 5);
    
    // Add image to PDF
    pdf.addImage(imgSrc, 'JPEG', margin, y, imgWidth, finalHeight);
    y += finalHeight + 8;
    
  } catch (err) {
    console.error("Error adding image:", err);
    addText("[Image could not be loaded]", 9);
  }
};
```

**Key Points**:
- Async function (uses `await` for image loading)
- Maintains aspect ratio
- Caps height to prevent overflow
- Graceful error handling

---

### PDF Content Structure

```
1. Title & Date
2. Basic Information (all fields)
3. Plant Layout Photo
4. Process Flow Photo
5. Equipment Details
   For each equipment type:
     - Type name
     For each instance:
       - Description
       - Operating hours
       - Energy inputs (bulleted list)
       - Instance photos
```

---

### Filename Generation

```javascript
const companyName = document.getElementById("name")?.value.trim() || "Energy_Audit";
const date = document.getElementById("date")?.value || new Date().toISOString().split('T')[0];
const filename = `${companyName.replace(/\s+/g, '_')}_Energy_Audit_${date}.pdf`;
```

**Format**: `CompanyName_Energy_Audit_2024-01-15.pdf`

---

## Event Handling

### Global Event Listeners

#### 1. Add Equipment Type Button
```javascript
document.getElementById('add-equipment-type')
  .addEventListener('click', () => addEquipmentType());
```

#### 2. Clear Draft Button
```javascript
document.getElementById('clear-draft')
  .addEventListener('click', clearDraft);
```

#### 3. Download PDF Button
```javascript
document.getElementById('download-pdf')
  .onclick = async () => { /* PDF generation */ };
```

#### 4. Plant Layout Photo
```javascript
document.getElementById('plant-layout')
  .addEventListener('change', e => {
    const file = e.target.files[0];
    // FileReader to convert to base64
    // Display preview
    // Trigger auto-save
  });
```

#### 5. Process Flow Photo
```javascript
document.getElementById('process-photo')
  .addEventListener('change', e => {
    // Same as plant layout
  });
```

#### 6. Before Unload (Auto-save on close)
```javascript
let isClearing = false;
window.addEventListener('beforeunload', (e) => {
  if (!isClearing) {
    saveDraft();
  }
});
```

**Flag `isClearing`**: Prevents saving during intentional clear

---

### Dynamic Event Listeners

Attached via `onclick` attributes in innerHTML:
- `deleteType('typeId')`
- `deleteInstance('instId')`
- `addInstance('typeId', typeNum)`
- `addInput('instId')`
- `deleteInput('rowId')`
- `addPhotoInput('instId')`

**Note**: These are global functions accessible from onclick handlers

---

## Future Development Ideas

### 1. Data Export/Import

**Export Feature**:
```javascript
function exportData() {
  const state = getFormState();
  const json = JSON.stringify(state, null, 2);
  const blob = new Blob([json], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  
  const a = document.createElement('a');
  a.href = url;
  a.download = 'audit-data.json';
  a.click();
}
```

**Import Feature**:
```javascript
function importData(file) {
  const reader = new FileReader();
  reader.onload = e => {
    try {
      const state = JSON.parse(e.target.result);
      localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
      location.reload();
    } catch (err) {
      alert('Invalid file format');
    }
  };
  reader.readAsText(file);
}
```

---

### 2. Cloud Sync (Optional)

**Approach 1: Backend API**
```javascript
async function syncToCloud() {
  const state = getFormState();
  const response = await fetch('https://api.example.com/audits', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(state)
  });
  const { id } = await response.json();
  localStorage.setItem('audit-id', id);
}
```

**Approach 2: Firebase/Supabase**
```javascript
import { initializeApp } from 'firebase/app';
import { getFirestore, doc, setDoc } from 'firebase/firestore';

async function saveToFirebase() {
  const state = getFormState();
  const userId = getCurrentUserId();
  await setDoc(doc(db, 'audits', userId), state);
}
```

---

### 3. Image Compression

**Problem**: Large images fill localStorage quickly

**Solution**: Compress before saving
```javascript
function compressImage(base64, maxWidth = 1200, quality = 0.7) {
  return new Promise((resolve) => {
    const img = new Image();
    img.onload = () => {
      const canvas = document.createElement('canvas');
      const ctx = canvas.getContext('2d');
      
      let width = img.width;
      let height = img.height;
      
      if (width > maxWidth) {
        height = (height / width) * maxWidth;
        width = maxWidth;
      }
      
      canvas.width = width;
      canvas.height = height;
      ctx.drawImage(img, 0, 0, width, height);
      
      resolve(canvas.toDataURL('image/jpeg', quality));
    };
    img.src = base64;
  });
}

// Usage in photo upload:
reader.onload = async ev => {
  const compressed = await compressImage(ev.target.result);
  img.src = compressed;
  // ...
};
```

---

### 4. Offline-First with Service Worker

**Service Worker** (`sw.js`):
```javascript
const CACHE_NAME = 'energy-audit-v1';
const urlsToCache = [
  '/',
  '/energy-audit-tool-improved.html',
  'https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js'
];

self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(urlsToCache))
  );
});

self.addEventListener('fetch', event => {
  event.respondWith(
    caches.match(event.request)
      .then(response => response || fetch(event.request))
  );
});
```

**Register in HTML**:
```javascript
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js');
}
```

---

### 5. Multi-Language Support

**Approach**: Externalize strings to translation object

```javascript
const translations = {
  en: {
    title: "Energy Audit Tool",
    companyName: "Company Name",
    addEquipment: "Add Equipment Type",
    // ...
  },
  es: {
    title: "Herramienta de Auditoría Energética",
    companyName: "Nombre de la Empresa",
    addEquipment: "Agregar Tipo de Equipo",
    // ...
  }
};

let currentLang = 'en';

function t(key) {
  return translations[currentLang][key] || key;
}

// Usage:
document.getElementById('title').textContent = t('title');
```

---

### 6. Template System

**Pre-configured equipment templates**:

```javascript
const templates = {
  manufacturing: {
    equipmentTypes: [
      { name: 'Boiler', instances: [/* pre-filled */] },
      { name: 'Air Compressor', instances: [/* pre-filled */] },
      // ...
    ]
  },
  warehouse: {
    equipmentTypes: [
      { name: 'HVAC', instances: [/* pre-filled */] },
      { name: 'Lighting', instances: [/* pre-filled */] },
      // ...
    ]
  }
};

function loadTemplate(templateName) {
  const template = templates[templateName];
  // Apply template to form
  localStorage.setItem(STORAGE_KEY, JSON.stringify(template));
  location.reload();
}
```

---

### 7. Enhanced PDF Features

**Charts/Graphs**:
```javascript
// Using Chart.js to generate energy consumption graph
const canvas = document.createElement('canvas');
const ctx = canvas.getContext('2d');
const chart = new Chart(ctx, {
  type: 'bar',
  data: { /* energy data */ }
});

// Convert chart to image
const chartImage = canvas.toDataURL();
pdf.addImage(chartImage, 'PNG', margin, y, maxWidth, 80);
```

**Calculations**:
```javascript
function calculateTotalEnergy() {
  const equipmentTypes = document.querySelectorAll('.equipment-type-block');
  let totalKwh = 0;
  
  equipmentTypes.forEach(type => {
    // Parse inputs and sum energy
  });
  
  return totalKwh;
}

// Add to PDF:
addText(`Total Estimated Energy: ${calculateTotalEnergy()} kWh/year`, 12, true);
```

---

### 8. Data Validation

```javascript
function validateForm() {
  const errors = [];
  
  // Check required fields
  if (!document.getElementById('name').value.trim()) {
    errors.push('Company name is required');
  }
  
  if (!document.getElementById('date').value) {
    errors.push('Audit date is required');
  }
  
  // Check equipment
  const hasEquipment = document.querySelectorAll('.equipment-type-block').length > 0;
  if (!hasEquipment) {
    errors.push('At least one equipment type is required');
  }
  
  return errors;
}

// Use before PDF generation:
document.getElementById('download-pdf').onclick = async () => {
  const errors = validateForm();
  if (errors.length > 0) {
    alert('Please fix:\n' + errors.join('\n'));
    return;
  }
  // Generate PDF...
};
```

---

### 9. Undo/Redo Functionality

```javascript
const history = {
  states: [],
  currentIndex: -1,
  
  push(state) {
    this.states = this.states.slice(0, this.currentIndex + 1);
    this.states.push(JSON.parse(JSON.stringify(state)));
    this.currentIndex++;
  },
  
  undo() {
    if (this.currentIndex > 0) {
      this.currentIndex--;
      return this.states[this.currentIndex];
    }
  },
  
  redo() {
    if (this.currentIndex < this.states.length - 1) {
      this.currentIndex++;
      return this.states[this.currentIndex];
    }
  }
};

// Track changes:
function saveDraft() {
  const state = getFormState();
  history.push(state);
  localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
}

// Keyboard shortcuts:
document.addEventListener('keydown', e => {
  if (e.ctrlKey && e.key === 'z') {
    const state = history.undo();
    if (state) restoreState(state);
  }
  if (e.ctrlKey && e.key === 'y') {
    const state = history.redo();
    if (state) restoreState(state);
  }
});
```

---

### 10. Accessibility Improvements

```javascript
// Add ARIA labels
function addEquipmentType() {
  // ...
  typeBlock.setAttribute('role', 'region');
  typeBlock.setAttribute('aria-label', `Equipment Type ${typeCounter}`);
  // ...
}

// Keyboard navigation
document.addEventListener('keydown', e => {
  if (e.key === 'Escape') {
    // Close modals or cancel operations
  }
});

// Focus management
function addInstance(typeId, typeNum) {
  // ...
  const firstInput = instance.querySelector('textarea');
  firstInput.focus();  // Auto-focus first field
}
```

---

## Performance Considerations

### Current Bottlenecks

1. **Large images in localStorage**
   - Solution: Implement compression (see section 3)
   - Alternative: Use IndexedDB for larger storage

2. **PDF generation with many images**
   - Solution: Add loading progress bar per image
   - Alternative: Generate PDF on server-side

3. **DOM manipulation on restore**
   - Current: Acceptable for typical use
   - Future: Use document fragments for batch inserts

---

### Optimization Tips

**1. Lazy load images in PDF**:
```javascript
const imagesToLoad = [];
// Collect all images first
// Load them in batches of 5
for (let i = 0; i < imagesToLoad.length; i += 5) {
  await Promise.all(imagesToLoad.slice(i, i + 5).map(loadImage));
}
```

**2. Use IndexedDB for images**:
```javascript
const openDB = () => {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open('EnergyAuditDB', 1);
    request.onsuccess = () => resolve(request.result);
    request.onerror = () => reject(request.error);
    request.onupgradeneeded = (e) => {
      const db = e.target.result;
      db.createObjectStore('images', { keyPath: 'id' });
    };
  });
};
```

**3. Virtual scrolling for large equipment lists**:
- Only render visible equipment types
- Render off-screen items on scroll

---

## Testing Checklist

### Unit Testing (Manual)
- [ ] Save draft with text inputs
- [ ] Save draft with images
- [ ] Restore draft after refresh
- [ ] Clear draft completely
- [ ] Add/delete equipment types
- [ ] Add/delete instances
- [ ] Add/delete inputs
- [ ] Add/delete photos
- [ ] Generate PDF with all content
- [ ] Generate PDF with images
- [ ] Handle storage quota error

### Cross-Browser Testing
- [ ] Chrome (Desktop)
- [ ] Chrome (Mobile)
- [ ] Safari (Desktop)
- [ ] Safari (iOS)
- [ ] Firefox
- [ ] Edge

### Edge Cases
- [ ] Very long company name (>100 chars)
- [ ] 50+ equipment types
- [ ] 100+ instances
- [ ] High-resolution images (>5MB each)
- [ ] No internet (after initial load)
- [ ] Private/Incognito mode
- [ ] Browser with localStorage disabled

---

## Debugging Tips

### 1. Check localStorage contents
```javascript
console.log(localStorage.getItem('energy-audit-draft'));
```

### 2. Monitor save operations
```javascript
function saveDraft() {
  console.log('Saving draft...');
  const state = getFormState();
  console.log('State size:', JSON.stringify(state).length);
  // ...
}
```

### 3. Test image compression
```javascript
reader.onload = async ev => {
  const original = ev.target.result;
  console.log('Original size:', original.length);
  
  const compressed = await compressImage(original);
  console.log('Compressed size:', compressed.length);
  console.log('Reduction:', ((1 - compressed.length/original.length) * 100).toFixed(2) + '%');
  // ...
};
```

### 4. PDF generation debug mode
```javascript
document.getElementById('download-pdf').onclick = async () => {
  console.log('Starting PDF generation...');
  
  try {
    // ... existing code
    console.log('Adding basic info...');
    // ...
    console.log('Adding equipment details...');
    // ...
    console.log('PDF saved successfully');
  } catch (err) {
    console.error('PDF Error Details:', err);
    console.error('Stack trace:', err.stack);
  }
};
```

---

## Code Style Guide

### Naming Conventions
- **Functions**: camelCase (`addEquipmentType`, `saveDraft`)
- **Constants**: UPPER_SNAKE_CASE (`STORAGE_KEY`)
- **DOM IDs**: kebab-case (`equipment-types-container`)
- **CSS classes**: kebab-case (`equipment-type-block`)

### Code Organization
1. Constants at top
2. Utility functions
3. Core functions (state management)
4. UI builder functions
5. Event handlers
6. Initialization code

### Comments
```javascript
// Single-line for brief explanations

/**
 * Multi-line for complex functions
 * @param {string} typeId - Equipment type ID
 * @returns {void}
 */
```

---

## Deployment

### GitHub Pages
1. Create repository
2. Upload `energy-audit-tool-improved.html`
3. Enable GitHub Pages in settings
4. Access at: `https://username.github.io/repo-name/energy-audit-tool-improved.html`

### Custom Domain
1. Add `CNAME` file to repo
2. Configure DNS records
3. Enable HTTPS in GitHub settings

### Versioning
```html
<!-- Add version in HTML comment -->
<!-- Energy Audit Tool v1.0.0 | 2024-01-15 -->
```

---

## License & Attribution

**Libraries Used**:
- jsPDF (MIT License)
- html2canvas (MIT License)

**Credits**:
- Created by: Meftah Uddin
- LinkedIn: [Profile Link]
- Website: [Portfolio Link]

---

## Changelog

### v1.0.0 (Current)
- Initial release
- Auto-save functionality
- PDF generation with images
- Offline support via localStorage
- Mobile-friendly camera integration

### Future Versions
- v1.1.0: Image compression
- v1.2.0: Export/Import JSON
- v2.0.0: Cloud sync (optional)

---

## Support & Contribution

### Reporting Issues
1. Check existing issues on GitHub
2. Provide browser/device info
3. Include steps to reproduce
4. Attach screenshots if applicable

### Contributing
1. Fork repository
2. Create feature branch
3. Make changes
4. Test thoroughly
5. Submit pull request

---

**End of Documentation**

*Last updated: [Date]*  
*Version: 1.0.0*
