---
name: vue-optimization
description: Analyzes Vue 3 component structure and suggests optimizations for performance, reactivity, and code reuse. Use when reviewing Vue components or planning component refactoring.
---

# Vue Component Optimization & Analysis Guide

This skill provides a systematic approach to analyzing Vue 3 (Composition API) components for performance bottlenecks, reactivity issues, and code reuse opportunities.

## Quick Analysis Checklist

When reviewing any Vue component, run through these checks:

- [ ] **Reactivity**: Are refs/computed/watchers properly declared?
- [ ] **Props Validation**: Do props have type definitions and defaults?
- [ ] **Computed vs Methods**: Using computed for derivations (cached) vs methods (re-run)?
- [ ] **Watchers**: Are there unnecessary watchers? Can they be replaced with computed?
- [ ] **Component Reusability**: Could this be split into smaller, reusable components?
- [ ] **Key Binding**: Are v-for loops using stable, unique keys (not index)?
- [ ] **Scoped Styles**: Is CSS scoped to prevent leaks?
- [ ] **Lifecycle**: Are resources cleaned up in onBeforeUnmount()?
- [ ] **Slots**: Would named slots improve flexibility?
- [ ] **Emit Events**: Are events properly typed and documented?

## 1. Reactivity Analysis

### Problem: Unreactive Data

**Anti-pattern:**
```javascript
// ❌ This won't be reactive
let counter = 0;

const increment = () => {
  counter++; // Changes won't trigger updates
};
```

**Solution:**
```javascript
// ✅ Use ref for reactive primitives
const counter = ref(0);

const increment = () => {
  counter.value++;
};

// ✅ Use reactive for objects
const state = reactive({
  count: 0,
  items: []
});

// Later: state.count = 5; // Reactive
```

### Problem: Computed vs Watchers

**Anti-pattern:**
```javascript
// ❌ Watcher doing what computed should do
const total = ref(0);

watch([price, quantity], () => {
  total.value = price.value * quantity.value;
});
```

**Solution:**
```javascript
// ✅ Use computed for derivations (cached, dependencies tracked)
const total = computed(() => price.value * quantity.value);

// ✅ Use watch only for side effects (API calls, logging, etc.)
watch(
  () => props.filters,
  async (newFilters) => {
    const data = await fetchData(newFilters);
    items.value = data;
  },
  { deep: true }
);
```

**Key difference:**
- **Computed**: Caches result, re-calculates only when dependencies change
- **Watcher**: Runs every time tracked values change (use for side effects)

### Problem: Deep Watch Performance

**Anti-pattern:**
```javascript
// ❌ Deep watchers are expensive on large objects
watch(
  () => largeObject,
  () => console.log('changed'),
  { deep: true }
);
```

**Solution:**
```javascript
// ✅ Watch specific properties instead
watch(
  () => largeObject.specificField,
  () => console.log('field changed')
);

// ✅ Or use computed + shallow watch
const relevantData = computed(() => ({
  name: largeObject.name,
  status: largeObject.status
}));

watch(relevantData, () => console.log('relevant data changed'));
```

## 2. Performance Optimization

### Problem: Unnecessary Re-renders (Key Binding)

**Anti-pattern:**
```vue
<!-- ❌ Using index as key - causes re-renders and bugs with reordering -->
<div v-for="(item, index) in items" :key="index">
  {{ item.name }}
</div>
```

**Solution:**
```vue
<!-- ✅ Use stable, unique identifier -->
<div v-for="item in items" :key="item.id">
  {{ item.name }}
</div>

<!-- ✅ For months/time-based data, use the time value -->
<div v-for="month in months" :key="month">
  {{ month }}
</div>

<!-- ✅ For custom objects with no ID, use computed key -->
<div v-for="item in items" :key="`${item.sku}-${item.warehouse}`">
  {{ item.sku }}
</div>
```

### Problem: Expensive Computations in Render

**Anti-pattern:**
```javascript
// ❌ Heavy filter runs on every render
const filteredItems = items.value.filter(item => {
  return expensiveCalculation(item);
});
```

**Solution:**
```javascript
// ✅ Use computed to cache expensive operations
const filteredItems = computed(() =>
  items.value.filter(item => expensiveCalculation(item))
);
```

### Problem: Component Re-mounts on Condition

**Anti-pattern:**
```vue
<!-- ❌ Condition changes component identity, forcing remount -->
<DetailsModal v-if="showDetails" :item="selectedItem" />
```

**Solution:**
```vue
<!-- ✅ Use v-show for frequent toggling (keeps DOM, just hides) -->
<DetailsModal v-show="showDetails" :item="selectedItem" />

<!-- ✅ Or keep mounted but disabled -->
<DetailsModal :item="selectedItem" :disabled="!showDetails" />
```

### Problem: Excessive Watcher Creation

**Anti-pattern:**
```javascript
// ❌ New watchers created for each filter change
items.forEach(item => {
  watch(() => item.status, () => {
    recalculate();
  });
});
```

**Solution:**
```javascript
// ✅ Single watcher on array changes
watch(
  () => items.value.map(i => i.status),
  () => recalculate(),
  { deep: false }
);

// ✅ Or use computed dependency
const statuses = computed(() => items.value.map(i => i.status));
watch(statuses, () => recalculate());
```

## 3. Code Reuse & Composition

### Problem: Duplicated Component Logic

**Anti-pattern:**
```vue
<!-- InventoryDetailModal.vue -->
<script setup>
import { ref, computed } from 'vue';

const isOpen = ref(false);
const selectedItem = ref(null);

const formattedPrice = computed(() => 
  formatCurrency(selectedItem.value?.price)
);

const openModal = (item) => {
  selectedItem.value = item;
  isOpen.value = true;
};

const closeModal = () => {
  isOpen.value = false;
  selectedItem.value = null;
};
</script>

<!-- ProductDetailModal.vue - IDENTICAL LOGIC -->
<script setup>
import { ref, computed } from 'vue';

const isOpen = ref(false);
const selectedItem = ref(null);

const formattedPrice = computed(() => 
  formatCurrency(selectedItem.value?.price)
);

const openModal = (item) => {
  selectedItem.value = item;
  isOpen.value = true;
};

const closeModal = () => {
  isOpen.value = false;
  selectedItem.value = null;
};
</script>
```

**Solution - Extract as Composable:**
```javascript
// composables/useModal.js
import { ref, computed } from 'vue';

export function useModal(formatFn = (v) => v) {
  const isOpen = ref(false);
  const selectedItem = ref(null);

  const formatted = computed(() => 
    selectedItem.value ? formatFn(selectedItem.value) : null
  );

  const openModal = (item) => {
    selectedItem.value = item;
    isOpen.value = true;
  };

  const closeModal = () => {
    isOpen.value = false;
    selectedItem.value = null;
  };

  return {
    isOpen,
    selectedItem,
    formatted,
    openModal,
    closeModal
  };
}
```

Then in components:
```vue
<!-- InventoryDetailModal.vue -->
<script setup>
import { useModal } from '@/composables/useModal';
import { formatCurrency } from '@/utils/currency';

const modal = useModal((item) => formatCurrency(item.price));
</script>

<template>
  <div v-show="modal.isOpen">
    <div>{{ modal.formatted }}</div>
  </div>
</template>
```

### Problem: Component Size (Too Many Responsibilities)

**Anti-pattern:**
```vue
<!-- Dashboard.vue - 800 lines doing everything -->
<script setup>
// - Fetches data
// - Filters data
// - Displays charts
// - Handles modals
// - Manages user settings
// - Handles exports
// - etc.
</script>
```

**Solution - Break into Smaller Components:**
```
Dashboard/
├── Dashboard.vue (main orchestrator)
├── filters/
│   └── FilterBar.vue
├── sections/
│   ├── SalesChart.vue
│   ├── InventoryWidget.vue
│   ├── OrdersTable.vue
│   └── MetricsCards.vue
├── modals/
│   └── DetailModal.vue
└── composables/
    └── useDashboardData.js
```

Each component has **single responsibility**:
- `FilterBar.vue`: Manages filters only
- `SalesChart.vue`: Displays sales data only
- `useD ashboardData.js`: Fetches and transforms data

### Problem: Component Props Explosion

**Anti-pattern:**
```vue
<script setup>
defineProps({
  orderId: String,
  orderNumber: String,
  status: String,
  warehouse: String,
  category: String,
  items: Array,
  total: Number,
  createdDate: String,
  updatedDate: String,
  customer: Object,
  // ... 20 more props
});
</script>
```

**Solution - Use Object Props:**
```vue
<script setup>
defineProps({
  order: {
    type: Object,
    required: true,
    // Implicitly validates structure
  }
});
</script>

<template>
  <div>
    <h3>{{ order.orderNumber }}</h3>
    <p>Status: {{ order.status }}</p>
    <p>Total: {{ order.total }}</p>
  </div>
</template>
```

## 4. Data Flow & Filtering

### Problem: Filtering at Multiple Levels

**Anti-pattern:**
```javascript
// filters.js
const allOrders = ref([]);
const filteredByStatus = computed(() => 
  allOrders.value.filter(o => o.status === selectedStatus.value)
);

// Component: filters again!
const finalOrders = computed(() =>
  filteredByStatus.value.filter(o => o.warehouse === selectedWarehouse.value)
);
```

**Solution - Single Filtering Function:**
```javascript
// api.js
export const getOrders = async (filters) => {
  const params = new URLSearchParams(filters);
  const response = await fetch(`/api/orders?${params}`);
  return response.json();
};

// composables/useOrders.js
export function useOrders() {
  const orders = ref([]);
  const filters = reactive({
    status: null,
    warehouse: null,
    category: null
  });

  const loadOrders = async () => {
    orders.value = await getOrders(filters);
  };

  watch(filters, loadOrders, { deep: true });

  return { orders, filters };
}
```

## 5. Common Patterns to Look For

### Pattern: Modal/Dialog Management

**Reusable pattern:**
```javascript
export function useDialog() {
  const isOpen = ref(false);
  
  const open = () => { isOpen.value = true; };
  const close = () => { isOpen.value = false; };
  const toggle = () => { isOpen.value = !isOpen.value; };
  
  return { isOpen, open, close, toggle };
}
```

### Pattern: Data Loading

**Reusable pattern:**
```javascript
export function useAsync(asyncFn) {
  const data = ref(null);
  const error = ref(null);
  const loading = ref(false);

  const execute = async (...args) => {
    loading.value = true;
    error.value = null;
    try {
      data.value = await asyncFn(...args);
    } catch (e) {
      error.value = e;
    } finally {
      loading.value = false;
    }
  };

  onMounted(execute);
  return { data, error, loading, execute };
}
```

### Pattern: Form State

**Reusable pattern:**
```javascript
export function useForm(initialValues, onSubmit) {
  const formData = reactive(initialValues);
  const errors = reactive({});
  const touched = reactive({});

  const handleChange = (field, value) => {
    formData[field] = value;
  };

  const handleBlur = (field) => {
    touched[field] = true;
    // Validate field
  };

  const handleSubmit = async () => {
    await onSubmit(formData);
  };

  const reset = () => {
    Object.assign(formData, initialValues);
    Object.assign(errors, {});
    Object.assign(touched, {});
  };

  return {
    formData,
    errors,
    touched,
    handleChange,
    handleBlur,
    handleSubmit,
    reset
  };
}
```

## 6. Props Validation Best Practices

**Anti-pattern:**
```vue
<script setup>
defineProps(['item', 'onUpdate']);
</script>
```

**Solution:**
```vue
<script setup>
import { PropType } from 'vue';

defineProps({
  item: {
    type: Object as PropType<{
      id: string;
      name: string;
      quantity: number;
      price: number;
    }>,
    required: true
  },
  onUpdate: {
    type: Function as PropType<(item: any) => void>,
    required: true
  },
  disabled: {
    type: Boolean,
    default: false
  }
});
</script>
```

## 7. Lifecycle & Cleanup

**Anti-pattern:**
```javascript
// Leaks resources if component unmounts
watch(() => props.itemId, async () => {
  const data = await fetchData(props.itemId);
  item.value = data;
});
```

**Solution:**
```javascript
let unsubscribe;

watch(
  () => props.itemId,
  async () => {
    const data = await fetchData(props.itemId);
    item.value = data;
  }
);

onBeforeUnmount(() => {
  if (unsubscribe) unsubscribe();
  // Cancel any pending async operations
});
```

## Analysis Workflow

When analyzing a component for optimization:

### Step 1: Data Flow Analysis
- [ ] Identify all reactive sources (ref, reactive, props)
- [ ] Check computed properties vs watchers
- [ ] Verify data flows from parent → child, events flow child → parent
- [ ] Look for unnecessary data transformations

### Step 2: Performance Analysis
- [ ] Scan v-for loops for proper key binding
- [ ] Check for computed properties that should be memoized
- [ ] Identify expensive operations (heavy filters, calculations)
- [ ] Look for unnecessary re-renders (v-if vs v-show)

### Step 3: Reusability Analysis
- [ ] Identify duplicated logic across components
- [ ] Look for opportunities to extract composables
- [ ] Check if component does too much (single responsibility)
- [ ] Evaluate prop drilling (too many levels of props)

### Step 4: Code Quality Analysis
- [ ] Verify props have type definitions
- [ ] Check event emissions are properly typed
- [ ] Look for missing lifecycle cleanup
- [ ] Verify error handling exists

## Example: Complete Optimization

**Before:**
```vue
<script setup>
import { ref, watch } from 'vue';

const props = defineProps(['filters']);
const orders = ref([]);

watch(() => props.filters, async (newFilters) => {
  const response = await fetch(
    `/api/orders?warehouse=${newFilters.warehouse}&status=${newFilters.status}`
  );
  const data = await response.json();
  orders.value = data;
}, { deep: true });

const byStatus = ref({});

watch(orders, (newOrders) => {
  byStatus.value = {};
  newOrders.forEach(order => {
    if (!byStatus.value[order.status]) {
      byStatus.value[order.status] = [];
    }
    byStatus.value[order.status].push(order);
  });
});
</script>

<template>
  <div>
    <div v-for="(list, status) in byStatus" :key="status">
      <h3>{{ status }}</h3>
      <div v-for="(order, i) in list" :key="i">
        {{ order.id }}: {{ order.total }}
      </div>
    </div>
  </div>
</template>
```

**Issues identified:**
1. Manual data fetching (should use composable)
2. v-for key uses index (should use order.id)
3. Watcher doing derived data transform (should use computed)
4. Deep watch on entire filters object

**After:**
```vue
<script setup>
import { computed } from 'vue';
import { useOrders } from '@/composables/useOrders';

const props = defineProps({
  filters: {
    type: Object,
    required: true
  }
});

const { orders } = useOrders(props.filters);

const ordersByStatus = computed(() => {
  const grouped = {};
  orders.value.forEach(order => {
    if (!grouped[order.status]) {
      grouped[order.status] = [];
    }
    grouped[order.status].push(order);
  });
  return grouped;
});
</script>

<template>
  <div>
    <div v-for="(list, status) in ordersByStatus" :key="status">
      <h3>{{ status }}</h3>
      <div v-for="order in list" :key="order.id">
        {{ order.id }}: {{ order.total }}
      </div>
    </div>
  </div>
</template>
```

## Optimization Checklist Summary

✅ **Always do:**
- Use computed for derived/cached data
- Use watch for side effects only
- Stable keys in v-for (id, not index)
- Extract repeated logic to composables
- Validate all props with types
- Clean up in onBeforeUnmount

❌ **Never do:**
- Use watchers for simple data derivations
- Use index as v-for key
- Duplicate component logic across files
- Create watchers inside loops
- Forget to cleanup listeners/subscriptions
- Pass too many props (use objects instead)
