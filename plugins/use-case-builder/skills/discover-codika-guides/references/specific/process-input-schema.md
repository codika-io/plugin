# Process Input Schema Guide

> Define configurable parameters for process installations using deployment input schemas.

## Table of Contents

1. [Introduction](#1-introduction)
2. [Field Types Reference](#2-field-types-reference)
3. [Sections](#3-sections)
4. [ObjectArray Deep Dive](#4-objectarray-deep-dive)
5. [INSTPARM Placeholder Patterns](#5-instparm-placeholder-patterns)
6. [Complete Example](#6-complete-example)
7. [Reference Files](#7-reference-files)

---

## 1. Introduction

### What It Enables

When users install a process (creating a `ProcessInstance`), they can customize their installation via **deployment parameters**. This makes each process instance unique per user/organization.

The deployment input schema defines:
- What form fields users see during installation
- Validation rules for each field
- Default values for auto-installation

### Data Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│ config.ts                                                               │
│ └── getDeploymentInputSchema()     ← Schema definition                  │
│ └── getDefaultDeploymentParameters() ← Default values                   │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Installation Form (UI)                                                  │
│ └── User fills form fields                                              │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ ProcessDeploymentInstance                                               │
│ └── deploymentParameters: { COMPANY_NAME: "Acme", MAX_ITEMS: 50, ... }  │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ INSTPARM Replacement                                                    │
│ └── {{INSTPARM_COMPANY_NAME_MRAPTSNI}} → "Acme"                         │
│ └── {{INSTPARM_MAX_ITEMS_MRAPTSNI}} → 50                                │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ n8n Workflow Execution                                                  │
│ └── Workflow runs with actual user-configured values                    │
└─────────────────────────────────────────────────────────────────────────┘
```

### Where to Define

In your use case's `config.ts`, define two functions:

```typescript
export function getDeploymentInputSchema(): DeploymentInputSchema {
  return [
    // Field and section definitions here
  ];
}

export function getDefaultDeploymentParameters(): DeploymentParameterValues {
  return {
    // Default values for auto-installation
  };
}
```

### Default Values

Default values are specified on each field via the `defaultValue` property. The UI uses these to pre-populate the installation form.

```typescript
// In getDeploymentInputSchema()
{
  key: 'MAX_ITEMS',
  type: 'number',
  label: 'Maximum Items',
  defaultValue: 10,  // ← Pre-populates form input
}
```

### `getDefaultDeploymentParameters()` - Exporting Defaults for Backend

The `getDefaultDeploymentParameters()` function exports all default values as a complete map. The backend uses this during auto-installation when deploying.

```typescript
export function getDefaultDeploymentParameters(): DeploymentParameterValues {
  return {
    COMPANY_NAME: 'Demo Corp',
    MAX_ITEMS: 10,
    ENABLE_FEATURE: true,
    OUTPUT_FORMAT: 'json',
    PRIORITY: 'normal',
    ALLOWED_DOMAINS: ['example.com'],
    TEAM_MEMBERS: [
      {
        name: 'John Doe',
        age: 32,
        active: true,
        role: 'developer',
        skills: ['JavaScript'],
      },
    ],
  };
}
```

**When used:** Deployment creates a process instance for the project owner using these defaults, enabling immediate testing without manual form submission.

### Keeping Defaults in Sync

The `defaultValue` on each field and the values in `getDefaultDeploymentParameters()` represent the **same defaults** - just in two formats for different consumers:

- **UI** reads `defaultValue` per-field to render the form
- **Backend** reads `getDefaultDeploymentParameters()` for auto-installation

**Important:** There is no automatic validation that these match. Keep them synchronized manually. The keys in `getDefaultDeploymentParameters()` must match the `key` values in the schema.

### Key Naming Convention

Field keys use **CONSTANT_CASE**:
- `COMPANY_NAME`
- `MAX_ITEMS`
- `ENABLE_FEATURE`
- `TEAM_MEMBERS`

---

## 2. Field Types Reference

**Eleven field types** are available for deployment parameters. All field properties are defined at the top level of each field object.

### 2.1 String Field

Single-line text input.

**TypeScript Interface:**

```typescript
interface DeploymentStringField {
  key: string;              // CONSTANT_CASE
  type: 'string';
  label: string;
  description?: string;
  tooltip?: string;
  placeholder?: string;
  required?: boolean;
  defaultValue?: string;
  minLength?: number;
  maxLength?: number;
  regex?: string;           // Regex pattern for validation
  regexError?: string;      // Custom error message when regex fails
}
```

**Example:**

```typescript
{
  key: 'COMPANY_NAME',
  type: 'string',
  label: 'Company Name',
  description: 'The name of your company or organization',
  placeholder: 'Enter company name...',
  required: true,
  defaultValue: 'Demo Corp',
  tooltip: 'This will appear in generated outputs',
}
```

**INSTPARM Usage (no extra quotes needed - value includes quotes):**

```javascript
const companyName = {{INSTPARM_COMPANY_NAME_MRAPTSNI}};
```

---

### 2.2 Text Field

Multi-line textarea input. Use for longer text content like descriptions, notes, prompts, or JSON input.

**TypeScript Interface:**

```typescript
interface DeploymentTextField {
  key: string;              // CONSTANT_CASE
  type: 'text';
  label: string;
  description?: string;
  tooltip?: string;
  placeholder?: string;
  required?: boolean;
  defaultValue?: string;
  minLength?: number;
  maxLength?: number;
  rows?: number;            // Number of textarea rows (default: 4)
  regex?: string;           // Regex pattern for validation
  regexError?: string;      // Custom error message when regex fails
}
```

**Example:**

```typescript
{
  key: 'SYSTEM_PROMPT',
  type: 'text',
  label: 'System Prompt',
  description: 'The system prompt for the AI assistant',
  placeholder: 'Enter the system prompt...',
  required: true,
  defaultValue: 'You are a helpful assistant.',
  rows: 8,
  maxLength: 5000,
}
```

**INSTPARM Usage (no extra quotes needed - value includes quotes):**

```javascript
const systemPrompt = {{INSTPARM_SYSTEM_PROMPT_MRAPTSNI}};
```

> **Note:** Text fields properly handle multiline content with newlines. The value is serialized using JSON.stringify, ensuring special characters are correctly escaped.

---

### 2.3 Number Field

Numeric input with optional min/max/step constraints.

**TypeScript Interface:**

```typescript
interface DeploymentNumberField {
  key: string;
  type: 'number';
  label: string;
  description?: string;
  tooltip?: string;
  placeholder?: string;
  required?: boolean;
  defaultValue?: number;
  min?: number;
  max?: number;
  step?: number;
}
```

**Example:**

```typescript
{
  key: 'MAX_ITEMS',
  type: 'number',
  label: 'Maximum Items',
  description: 'Maximum number of items to process per batch',
  required: false,
  defaultValue: 10,
  min: 1,
  max: 100,
  step: 1,
}
```

**INSTPARM Usage (unquoted):**

```javascript
const maxItems = {{INSTPARM_MAX_ITEMS_MRAPTSNI}};
```

---

### 2.4 Boolean Field

Toggle/checkbox for true/false values.

**TypeScript Interface:**

```typescript
interface DeploymentBooleanField {
  key: string;
  type: 'boolean';
  label: string;
  description?: string;
  tooltip?: string;
  required?: boolean;
  defaultValue?: boolean;
}
```

**Example:**

```typescript
{
  key: 'ENABLE_FEATURE',
  type: 'boolean',
  label: 'Enable Advanced Feature',
  description: 'Toggle to enable the advanced processing feature',
  required: false,
  defaultValue: true,
}
```

**INSTPARM Usage (unquoted):**

```javascript
const enableFeature = {{INSTPARM_ENABLE_FEATURE_MRAPTSNI}};
// Results in: const enableFeature = true;
```

---

### 2.5 Date Field

Date picker for selecting calendar dates.

**TypeScript Interface:**

```typescript
interface DeploymentDateField {
  key: string;              // CONSTANT_CASE
  type: 'date';
  label: string;
  description?: string;
  tooltip?: string;
  required?: boolean;
  defaultValue?: string;    // ISO date string: YYYY-MM-DD
  minDate?: string;         // Minimum selectable date (ISO string)
  maxDate?: string;         // Maximum selectable date (ISO string)
}
```

**Example:**

```typescript
{
  key: 'START_DATE',
  type: 'date',
  label: 'Start Date',
  description: 'When the process should begin',
  required: true,
  defaultValue: '2025-01-01',
  minDate: '2024-01-01',
  maxDate: '2030-12-31',
}
```

**INSTPARM Usage (no extra quotes - value includes quotes):**

```javascript
const startDate = {{INSTPARM_START_DATE_MRAPTSNI}};
// Results in: const startDate = "2025-01-01";
```

---

### 2.6 Select Field

Dropdown for single selection from predefined options.

**TypeScript Interface:**

```typescript
interface DeploymentSelectField {
  key: string;
  type: 'select';
  label: string;
  description?: string;
  tooltip?: string;
  placeholder?: string;
  required?: boolean;
  options: Array<{ value: string; label: string }>;
  defaultValue?: string;
}
```

**Example:**

```typescript
{
  key: 'OUTPUT_FORMAT',
  type: 'select',
  label: 'Output Format',
  description: 'Choose the format for output data',
  required: true,
  defaultValue: 'json',
  options: [
    { label: 'JSON', value: 'json' },
    { label: 'XML', value: 'xml' },
    { label: 'CSV', value: 'csv' },
    { label: 'Plain Text', value: 'text' },
  ],
}
```

**INSTPARM Usage (no extra quotes - value includes quotes):**

```javascript
const outputFormat = {{INSTPARM_OUTPUT_FORMAT_MRAPTSNI}};
// Results in: const outputFormat = 'json';
```

---

### 2.7 Multiselect Field

Multiple selection from a list of options (checkboxes/tags).

**TypeScript Interface:**

```typescript
interface DeploymentMultiSelectField {
  key: string;              // CONSTANT_CASE
  type: 'multiselect';
  label: string;
  description?: string;
  tooltip?: string;
  required?: boolean;
  options: Array<{ value: string; label: string }>;  // REQUIRED
  defaultValue?: string[];  // Array of selected option values
  minSelections?: number;   // Minimum required selections
  maxSelections?: number;   // Maximum allowed selections
}
```

**Example:**

```typescript
{
  key: 'ENABLED_FEATURES',
  type: 'multiselect',
  label: 'Enabled Features',
  description: 'Select the features to enable for this installation',
  required: true,
  defaultValue: ['analytics', 'notifications'],
  minSelections: 1,
  maxSelections: 5,
  options: [
    { label: 'Analytics', value: 'analytics' },
    { label: 'Notifications', value: 'notifications' },
    { label: 'Reports', value: 'reports' },
    { label: 'API Access', value: 'api' },
    { label: 'Admin Panel', value: 'admin' },
  ],
}
```

**INSTPARM Usage (unquoted - returns array):**

```javascript
const enabledFeatures = {{INSTPARM_ENABLED_FEATURES_MRAPTSNI}};
// Results in: const enabledFeatures = ['analytics','notifications'];

// Use array methods:
enabledFeatures.includes('analytics');  // true
enabledFeatures.length;                  // 2
```

---

### 2.8 Radio Field

Radio button group for single selection with visible options.

**TypeScript Interface:**

```typescript
interface DeploymentRadioField {
  key: string;
  type: 'radio';
  label: string;
  description?: string;
  tooltip?: string;
  required?: boolean;
  options: Array<{ value: string; label: string }>;
  defaultValue?: string;
}
```

**Example:**

```typescript
{
  key: 'PRIORITY',
  type: 'radio',
  label: 'Processing Priority',
  description: 'Select the priority level for processing',
  required: true,
  defaultValue: 'normal',
  options: [
    { label: 'Low', value: 'low' },
    { label: 'Normal', value: 'normal' },
    { label: 'High', value: 'high' },
  ],
}
```

**INSTPARM Usage (no extra quotes - value includes quotes):**

```javascript
const priority = {{INSTPARM_PRIORITY_MRAPTSNI}};
// Results in: const priority = "normal";
```

---

### 2.9 Array Field

Repeatable list of primitive values with add/remove functionality.

**TypeScript Interface:**

```typescript
interface DeploymentArrayField {
  key: string;
  type: 'array';
  label: string;
  description?: string;
  tooltip?: string;
  placeholder?: string;
  required?: boolean;
  defaultValue?: (string | number | boolean)[];
  itemField?: DeploymentArrayItemField;  // Defines item type and properties
  minItems?: number;
  maxItems?: number;
}
```

**The `itemField` property** defines the complete field configuration for each array item. If not specified, defaults to `{ type: 'string' }`.

**Supported Item Types (7 types):**

| Type | Description | Example Properties |
|------|-------------|-------------------|
| `string` | Single-line text | `placeholder`, `minLength`, `maxLength`, `regex`, `regexError` |
| `text` | Multi-line textarea | `placeholder`, `rows`, `minLength`, `maxLength` |
| `number` | Numeric input | `min`, `max`, `step`, `numberType` |
| `boolean` | Toggle switch | `enabledLabel`, `disabledLabel` |
| `date` | Date picker | `minDate`, `maxDate` |
| `select` | Dropdown | `options` (required), `placeholder` |
| `radio` | Radio buttons | `options` (required) |

> **Note:** `multiselect` is NOT supported as an array item type because it would create nested arrays, which Firestore cannot store.

**Example - String Array (simple):**

```typescript
{
  key: 'ALLOWED_DOMAINS',
  type: 'array',
  label: 'Allowed Domains',
  description: 'List of domains that are allowed for processing',
  required: false,
  defaultValue: ['example.com', 'demo.org'],
  minItems: 1,
  maxItems: 10,
  itemField: {
    type: 'string',
    placeholder: 'e.g., example.com',
    regex: '^[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$',
    regexError: 'Invalid domain format',
  },
}
```

**Example - Text Array (multi-line notes):**

```typescript
{
  key: 'MEETING_NOTES',
  type: 'array',
  label: 'Meeting Notes',
  description: 'Add meeting notes or call transcripts',
  defaultValue: [],
  minItems: 0,
  maxItems: 5,
  itemField: {
    type: 'text',
    placeholder: 'Enter notes...',
    rows: 4,
    maxLength: 2000,
  },
}
```

**Example - Number Array:**

```typescript
{
  key: 'SCORE_VALUES',
  type: 'array',
  label: 'Test Scores',
  defaultValue: [85, 90, 78],
  itemField: {
    type: 'number',
    min: 0,
    max: 100,
    step: 1,
  },
}
```

**Example - Select Array:**

```typescript
{
  key: 'PRIORITY_LEVELS',
  type: 'array',
  label: 'Priority Levels',
  defaultValue: ['medium'],
  itemField: {
    type: 'select',
    options: [
      { value: 'low', label: 'Low' },
      { value: 'medium', label: 'Medium' },
      { value: 'high', label: 'High' },
    ],
  },
}
```

**INSTPARM Usage (unquoted - returns array):**

```javascript
const allowedDomains = {{INSTPARM_ALLOWED_DOMAINS_MRAPTSNI}};
// Results in: const allowedDomains = ['example.com','demo.org'];

const scores = {{INSTPARM_SCORE_VALUES_MRAPTSNI}};
// Results in: const scores = [85,90,78];

// Use array methods:
allowedDomains.length;           // 2
allowedDomains[0];               // 'example.com'
scores.reduce((a, b) => a + b);  // 253
```

---

### 2.10 Object Field

Single structured object with nested fields. Serialized as JSON.

**TypeScript Interface:**

```typescript
interface DeploymentObjectField {
  key: string;              // CONSTANT_CASE
  type: 'object';
  label: string;
  description?: string;
  tooltip?: string;
  required?: boolean;
  schema: DeploymentInputItem[];  // REQUIRED: Nested field definitions
  defaultValue?: Record<string, unknown>;  // Default object value
}
```

**Example:**

```typescript
{
  key: 'API_CONFIG',
  type: 'object',
  label: 'API Configuration',
  description: 'Configure the external API connection',
  required: true,
  schema: [
    {
      key: 'endpoint',
      type: 'string',
      label: 'API Endpoint',
      placeholder: 'https://api.example.com',
      required: true,
    },
    {
      key: 'timeout',
      type: 'number',
      label: 'Timeout (seconds)',
      defaultValue: 30,
      min: 5,
      max: 120,
    },
    {
      key: 'retry_enabled',
      type: 'boolean',
      label: 'Enable Retries',
      defaultValue: true,
    },
  ],
  defaultValue: {
    endpoint: '',
    timeout: 30,
    retry_enabled: true,
  },
}
```

**INSTPARM Usage (unquoted - returns object):**

```javascript
const apiConfig = {{INSTPARM_API_CONFIG_MRAPTSNI}};
// Results in: const apiConfig = {'endpoint':'https://api.example.com','timeout':30,'retry_enabled':true};

// Access properties:
apiConfig.endpoint;       // 'https://api.example.com'
apiConfig.timeout;        // 30
apiConfig.retry_enabled;  // true
```

---

### 2.11 ObjectArray Field

Array of structured objects with a defined schema for each item.

**TypeScript Interface:**

```typescript
interface DeploymentObjectArrayField {
  key: string;
  type: 'objectArray';
  label: string;
  description?: string;
  tooltip?: string;
  required?: boolean;
  /** Schema for each object - supports all field types including nested objectArrays */
  itemSchema: DeploymentInputItem[];
  defaultValue?: Record<string, unknown>[];
  minItems?: number;
  maxItems?: number;
}
```

**Example:**

```typescript
{
  key: 'TEAM_MEMBERS',
  type: 'objectArray',
  label: 'Team Members',
  description: 'Configure team member details',
  required: false,
  minItems: 1,
  maxItems: 5,
  itemSchema: [
    {
      key: 'name',
      type: 'string',
      label: 'Name',
      placeholder: 'Full name',
      required: true,
    },
    {
      key: 'age',
      type: 'number',
      label: 'Age',
      min: 18,
      max: 100,
      defaultValue: 30,
    },
    {
      key: 'active',
      type: 'boolean',
      label: 'Active',
      defaultValue: true,
    },
    {
      key: 'role',
      type: 'select',
      label: 'Role',
      required: true,
      defaultValue: 'developer',
      options: [
        { label: 'Developer', value: 'developer' },
        { label: 'Designer', value: 'designer' },
        { label: 'Manager', value: 'manager' },
      ],
    },
    {
      key: 'skills',
      type: 'array',
      label: 'Skills',
      description: 'List of skills',
      minItems: 1,
      maxItems: 5,
    },
  ],
  defaultValue: [
    {
      name: 'John Doe',
      age: 32,
      active: true,
      role: 'developer',
      skills: ['JavaScript', 'TypeScript', 'React'],
    },
  ],
}
```

**INSTPARM Usage (unquoted - returns full JavaScript array/object):**

```javascript
const teamMembers = {{INSTPARM_TEAM_MEMBERS_MRAPTSNI}};
// Results in: const teamMembers = [{'name':'John Doe','age':32,'active':true,'role':'developer','skills':['JavaScript','TypeScript','React']}];

// Full array/object access:
teamMembers.length;              // 1
teamMembers[0].name;            // 'John Doe'
teamMembers[0].age;             // 32
teamMembers[0].active;          // true
teamMembers[0].role;            // 'developer'
teamMembers[0].skills;          // ['JavaScript', 'TypeScript', 'React']
teamMembers[0].skills[0];       // 'JavaScript'

// Array methods work:
teamMembers.map(m => m.name);                    // ['John Doe']
teamMembers.filter(m => m.active);               // [{ ... }]
teamMembers.reduce((sum, m) => sum + m.age, 0);  // 32
```

---

## 3. Sections

Sections group related fields visually with optional collapsible behavior.

### Structure

```typescript
interface DeploymentSection {
  type: 'section';
  title: string;
  description?: string;
  collapsible?: boolean;    // Enable collapse/expand
  defaultOpen?: boolean;    // Initial state when collapsible
  inputSchema: DeploymentInputItem[];  // Nested fields or sections
}
```

### Example

```typescript
{
  type: 'section',
  title: 'Basic Configuration',
  description: 'Core settings for your installation',
  collapsible: true,
  defaultOpen: true,
  inputSchema: [
    {
      key: 'COMPANY_NAME',
      type: 'string',
      label: 'Company Name',
      required: true,
      defaultValue: 'Demo Corp',
    },
    {
      key: 'MAX_ITEMS',
      type: 'number',
      label: 'Maximum Items',
      defaultValue: 10,
      min: 1,
      max: 100,
    },
    {
      key: 'ENABLE_FEATURE',
      type: 'boolean',
      label: 'Enable Advanced Feature',
      defaultValue: true,
    },
  ],
}
```

### Nesting Sections

Sections can be nested inside objectArrays:

```typescript
{
  key: 'DEPARTMENTS',
  type: 'objectArray',
  label: 'Departments',
  itemSchema: [
    {
      key: 'name',
      type: 'string',
      label: 'Department Name',
      required: true,
    },
    {
      type: 'section',                    // Section inside objectArray
      title: 'Staff',
      description: 'Employees in this department',
      collapsible: true,
      defaultOpen: true,
      inputSchema: [
        {
          key: 'employees',
          type: 'objectArray',            // Nested objectArray
          label: 'Employees',
          itemSchema: [
            { key: 'name', type: 'string', label: 'Name', required: true },
            { key: 'title', type: 'string', label: 'Job Title' },
          ],
        },
      ],
    },
  ],
}
```

---

## 4. ObjectArray Deep Dive

ObjectArrays enable complex, structured configuration data.

### Supported Nested Types

The `itemSchema` supports **all field types**:

| Type | Supported in itemSchema |
|------|------------------------|
| `string` | Yes |
| `text` | Yes |
| `number` | Yes |
| `boolean` | Yes |
| `date` | Yes |
| `select` | Yes |
| `multiselect` | Yes |
| `radio` | Yes |
| `array` | Yes |
| `object` | Yes (recursive) |
| `objectArray` | Yes (recursive) |
| `section` | Yes |

### Recursive Composition

ObjectArrays can contain sections, which can contain objectArrays:

```
DEPARTMENTS (objectArray)
├── name (string)
├── budget (number)
└── Staff (section)
    └── employees (objectArray)
        ├── name (string)
        └── title (string)
```

### Complete Nested Example

```typescript
{
  key: 'DEPARTMENTS',
  type: 'objectArray',
  label: 'Departments',
  description: 'Configure departments with nested employee lists',
  minItems: 0,
  maxItems: 3,
  itemSchema: [
    {
      key: 'name',
      type: 'string',
      label: 'Department Name',
      placeholder: 'e.g., Engineering',
      required: true,
    },
    {
      key: 'budget',
      type: 'number',
      label: 'Budget',
      min: 0,
      defaultValue: 100000,
    },
    {
      type: 'section',
      title: 'Staff',
      description: 'Employees in this department',
      collapsible: true,
      defaultOpen: true,
      inputSchema: [
        {
          key: 'employees',
          type: 'objectArray',
          label: 'Employees',
          minItems: 0,
          maxItems: 5,
          itemSchema: [
            {
              key: 'name',
              type: 'string',
              label: 'Name',
              required: true,
            },
            {
              key: 'title',
              type: 'string',
              label: 'Job Title',
              placeholder: 'e.g., Senior Engineer',
            },
          ],
        },
      ],
    },
  ],
}
```

### Accessing Nested Data in n8n

```javascript
const departments = {{INSTPARM_DEPARTMENTS_MRAPTSNI}};

// Access nested objectArray:
const firstDept = departments[0];
const deptName = firstDept.name;                    // 'Engineering'
const deptBudget = firstDept.budget;               // 100000
const employees = firstDept.employees;              // Array of employee objects
const firstEmployee = employees[0];                 // { name: 'Alice', title: 'Senior Engineer' }
const employeeCount = employees.length;             // Number of employees

// Iterate over nested data:
departments.forEach(dept => {
  console.log(`${dept.name}: ${dept.employees.length} employees`);
});
```

---

## 5. INSTPARM Placeholder Patterns

### Pattern Format

```
{{INSTPARM_<KEY>_MRAPTSNI}}
```

- `INSTPARM_` - Prefix
- `<KEY>` - The field key in CONSTANT_CASE
- `_MRAPTSNI` - Suffix (reversed "INSTPARM_")

### Serialization Rules

The system serializes values for safe embedding in n8n workflow JSON:

| Field Type | Value Type | Serialized Format | n8n Usage |
|------------|------------|-------------------|-----------|
| `string` | `string` | `"value"` (JSON.stringify) | `{{INSTPARM_X_MRAPTSNI}}` (unquoted) |
| `text` | `string` | `"value"` (JSON.stringify) | `{{INSTPARM_X_MRAPTSNI}}` (unquoted) |
| `number` | `number` | String representation | `{{INSTPARM_X_MRAPTSNI}}` (unquoted) |
| `boolean` | `boolean` | `true` or `false` | `{{INSTPARM_X_MRAPTSNI}}` (unquoted) |
| `date` | `string` | `"2024-01-15"` (JSON.stringify) | `{{INSTPARM_X_MRAPTSNI}}` (unquoted) |
| `select` | `string` | `"value"` (JSON.stringify) | `{{INSTPARM_X_MRAPTSNI}}` (unquoted) |
| `multiselect` | `string[]` | `['a','b','c']` | `{{INSTPARM_X_MRAPTSNI}}` (unquoted) |
| `radio` | `string` | `"value"` (JSON.stringify) | `{{INSTPARM_X_MRAPTSNI}}` (unquoted) |
| `array` | `string[]` | `['a','b','c']` | `{{INSTPARM_X_MRAPTSNI}}` (unquoted) |
| `object` | `object` | `{'k':'v'}` | `{{INSTPARM_X_MRAPTSNI}}` (unquoted) |
| `objectArray` | `object[]` | `[{'k':'v'}]` | `{{INSTPARM_X_MRAPTSNI}}` (unquoted) |

**String types use JSON.stringify** which includes surrounding double quotes and properly escapes special characters (including newlines in multiline text fields).

**Arrays/objects use single quotes** to avoid breaking JSON string contexts in n8n Code nodes.

> **Context-Aware Serialization:** INSTPARM placeholders work in two contexts: **jsCode context** (inside JSON strings, as shown in examples below) and **JSON value context** (directly in JSON, e.g., `"value": {{INSTPARM_X_MRAPTSNI}}`). The system automatically detects the context and applies appropriate escaping. Use the same syntax everywhere.
>
> See [placeholder-patterns.md](./placeholder-patterns.md#context-aware-serialization) for detailed examples of both contexts.

### Usage Rules

**All field types are unquoted** - do NOT add extra quotes around INSTPARM placeholders:

```javascript
// String field (value includes quotes via JSON.stringify)
const companyName = {{INSTPARM_COMPANY_NAME_MRAPTSNI}};

// Text field (multiline content properly escaped)
const systemPrompt = {{INSTPARM_SYSTEM_PROMPT_MRAPTSNI}};

// Date field (returns quoted ISO date string)
const startDate = {{INSTPARM_START_DATE_MRAPTSNI}};

// Select field (returns quoted string value)
const outputFormat = {{INSTPARM_OUTPUT_FORMAT_MRAPTSNI}};

// Radio field (returns quoted string value)
const priority = {{INSTPARM_PRIORITY_MRAPTSNI}};

// Number field
const maxItems = {{INSTPARM_MAX_ITEMS_MRAPTSNI}};

// Boolean field
const enableFeature = {{INSTPARM_ENABLE_FEATURE_MRAPTSNI}};

// Multiselect field (returns array of strings)
const enabledFeatures = {{INSTPARM_ENABLED_FEATURES_MRAPTSNI}};

// Array field (strings)
const allowedDomains = {{INSTPARM_ALLOWED_DOMAINS_MRAPTSNI}};

// Object field (single object)
const apiConfig = {{INSTPARM_API_CONFIG_MRAPTSNI}};

// ObjectArray field (array of objects)
const teamMembers = {{INSTPARM_TEAM_MEMBERS_MRAPTSNI}};
```

### Complete n8n Code Node Example

```javascript
// ============================================================================
// INSTPARM Usage Example
// ============================================================================
// NOTE: String types (string, text, date, select, radio) include quotes
// automatically via JSON.stringify - do NOT add extra quotes!

// --- Basic Fields ---
const companyName = {{INSTPARM_COMPANY_NAME_MRAPTSNI}};
const maxItems = {{INSTPARM_MAX_ITEMS_MRAPTSNI}};
const enableFeature = {{INSTPARM_ENABLE_FEATURE_MRAPTSNI}};

// --- Selection Fields ---
const outputFormat = {{INSTPARM_OUTPUT_FORMAT_MRAPTSNI}};
const priority = {{INSTPARM_PRIORITY_MRAPTSNI}};

// --- Array Fields ---
const allowedDomains = {{INSTPARM_ALLOWED_DOMAINS_MRAPTSNI}};

// --- ObjectArray Fields ---
const teamMembers = {{INSTPARM_TEAM_MEMBERS_MRAPTSNI}};

// ============================================================================
// Working with ObjectArray Data
// ============================================================================

// Get count
const memberCount = teamMembers.length;

// Access first item properties
const firstMember = teamMembers[0] || {};
const firstName = firstMember.name;
const firstRole = firstMember.role;
const firstSkills = firstMember.skills || [];

// Use array methods
const allNames = teamMembers.map(m => m.name);
const activeMembers = teamMembers.filter(m => m.active === true);
const totalAge = teamMembers.reduce((sum, m) => sum + (m.age || 0), 0);
const seniorMembers = teamMembers.filter(m => m.level === 'senior');

// Access nested arrays
const allSkills = teamMembers.flatMap(m => m.skills || []);

return [
  {
    json: {
      company_name: companyName,
      max_items: maxItems,
      enable_feature: enableFeature,
      output_format: outputFormat,
      priority: priority,
      allowed_domains: allowedDomains.join(', '),
      member_count: memberCount,
      first_member_name: firstName,
      all_member_names: allNames.join(', '),
      active_member_count: activeMembers.length,
    }
  }
];
```

---

## 6. Complete Example

### Demo Use Case

A complete working example is available at:

```
/use-cases-demo/process-input-demo/
├── config.ts                              ← Schema + defaults
└── workflows/
    └── process-input-workflow.json        ← Workflow with INSTPARM usage
```

### Schema Definition (config.ts)

```typescript
export function getDeploymentInputSchema(): DeploymentInputSchema {
  return [
    // Section 1: Basic Fields
    {
      type: 'section',
      title: 'Basic Configuration',
      description: 'Simple field types: string, number, boolean',
      collapsible: true,
      defaultOpen: true,
      inputSchema: [
        {
          key: 'COMPANY_NAME',
          type: 'string',
          label: 'Company Name',
          required: true,
          defaultValue: 'Demo Corp',
        },
        {
          key: 'MAX_ITEMS',
          type: 'number',
          label: 'Maximum Items',
          defaultValue: 10,
          min: 1,
          max: 100,
        },
        {
          key: 'ENABLE_FEATURE',
          type: 'boolean',
          label: 'Enable Advanced Feature',
          defaultValue: true,
        },
      ],
    },

    // Section 2: Selection Fields
    {
      type: 'section',
      title: 'Selection Options',
      description: 'Select and radio field types',
      collapsible: true,
      defaultOpen: true,
      inputSchema: [
        {
          key: 'OUTPUT_FORMAT',
          type: 'select',
          label: 'Output Format',
          required: true,
          defaultValue: 'json',
          options: [
            { label: 'JSON', value: 'json' },
            { label: 'XML', value: 'xml' },
            { label: 'CSV', value: 'csv' },
          ],
        },
        {
          key: 'PRIORITY',
          type: 'radio',
          label: 'Processing Priority',
          defaultValue: 'normal',
          options: [
            { label: 'Low', value: 'low' },
            { label: 'Normal', value: 'normal' },
            { label: 'High', value: 'high' },
          ],
        },
      ],
    },

    // Section 3: Array Fields
    {
      type: 'section',
      title: 'Array Configuration',
      description: 'Array of strings and array of objects',
      collapsible: true,
      defaultOpen: true,
      inputSchema: [
        {
          key: 'ALLOWED_DOMAINS',
          type: 'array',
          label: 'Allowed Domains',
          defaultValue: ['example.com', 'demo.org'],
          minItems: 1,
          maxItems: 10,
        },
        {
          key: 'TEAM_MEMBERS',
          type: 'objectArray',
          label: 'Team Members',
          minItems: 1,
          maxItems: 5,
          itemSchema: [
            { key: 'name', type: 'string', label: 'Name', required: true },
            { key: 'age', type: 'number', label: 'Age', min: 18, max: 100 },
            { key: 'active', type: 'boolean', label: 'Active', defaultValue: true },
            {
              key: 'role',
              type: 'select',
              label: 'Role',
              options: [
                { label: 'Developer', value: 'developer' },
                { label: 'Designer', value: 'designer' },
                { label: 'Manager', value: 'manager' },
              ],
            },
            {
              key: 'skills',
              type: 'array',
              label: 'Skills',
              minItems: 1,
              maxItems: 5,
            },
          ],
          defaultValue: [
            {
              name: 'John Doe',
              age: 32,
              active: true,
              role: 'developer',
              skills: ['JavaScript', 'TypeScript'],
            },
          ],
        },
      ],
    },
  ];
}
```

### Default Parameters (config.ts)

```typescript
export function getDefaultDeploymentParameters(): DeploymentParameterValues {
  return {
    COMPANY_NAME: 'Demo Corp',
    MAX_ITEMS: 10,
    ENABLE_FEATURE: true,
    OUTPUT_FORMAT: 'json',
    PRIORITY: 'normal',
    ALLOWED_DOMAINS: ['example.com', 'demo.org'],
    TEAM_MEMBERS: [
      {
        name: 'John Doe',
        age: 32,
        active: true,
        role: 'developer',
        skills: ['JavaScript', 'TypeScript'],
      },
    ],
  };
}
```

---

## 7. Reference Files

| File | Description |
|------|-------------|
| `use-cases-demo/process-input-demo/config.ts` | Complete schema example with all 7 field types |
| `use-cases-demo/process-input-demo/workflows/process-input-workflow.json` | Workflow demonstrating INSTPARM usage |
| `codika` | TypeScript type definitions (npm package) |

### Type Definitions Location

All deployment input schema types are exported from the `codika` package:

```typescript
import { type DeploymentInputSchema, type DeploymentInputField } from 'codika';
```

Key types:
- `DeploymentInputSchema` - Root schema array type
- `DeploymentInputItem` - Union of field or section
- `DeploymentInputField` - Union of all field types
- `DeploymentSection` - Section grouping type
- `DeploymentParameterValues` - Values map type

---

## Related Documentation

- [placeholder-patterns.md](./placeholder-patterns.md) - Complete placeholder reference including INSTPARM
- [config-patterns.md](./config-patterns.md) - config.ts structure and patterns

---

*Last Updated: January 2025*
