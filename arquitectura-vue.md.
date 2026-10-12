# Arquitectura frontend en Vue: separación de responsabilidades

Plantilla de referencia para construir módulos nuevos (Vue 3 + Pinia + PrimeVue + axios + SweetAlert2). El ejemplo es un CRUD de **productos**, pero la estructura sirve para cualquier módulo (clientes, proveedores, compras, etc.).

---

## 1. La idea en una imagen

```
                 ┌─────────────────────────────────────┐
                 │  View (padre / container)           │
                 │  ProductosView.vue                  │
                 │  - Decide QUÉ pasa y CUÁNDO         │
                 │  - Llama al store                   │
                 │  - Muestra toast / confirmación     │
                 └──────┬───────────────▲──────────────┘
          props (datos) │               │ emits (eventos)
                 ┌──────▼───────────────┴──────────────┐
                 │  Components (hijos / presentational)│
                 │  ProductoTable.vue                  │
                 │  ProductoDialog.vue                 │
                 │  - Solo dibujan y avisan            │
                 │  - No llaman a la API               │
                 └─────────────────────────────────────┘

   View ──► Store (Pinia) ──► Service (axios) ──► API Laravel
          estado + acciones     solo HTTP
```

**Regla de oro:** los datos bajan por props, los eventos suben por emits.

---

## 2. Estructura de carpetas

```
src/
├── api/
│   └── http.js                  # instancia de axios + interceptores
├── services/
│   ├── productosService.js      # solo llamadas HTTP de productos
│   └── confirmService.js        # envoltura de SweetAlert (confirmaciones)
├── stores/
│   └── productos.js             # estado compartido + acciones
├── composables/
│   └── useNotify.js             # envoltura de Toast de PrimeVue
├── views/
│   └── ProductosView.vue        # padre: orquesta todo
├── components/
│   └── Productos/
│       ├── ProductoTable.vue    # hijo: tabla
│       └── ProductoDialog.vue   # hijo: formulario crear/editar
└── router/
    └── index.js                 # rutas con lazy loading
```

---

## 3. Capa 1: cliente HTTP (`api/http.js`)

Una sola instancia de axios. Aquí van los errores **generales** para no repetir `try/catch` en cada vista.

```js
// src/api/http.js
import axios from 'axios'
import router from '@/router'

const http = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  headers: { Accept: 'application/json' },
})

// Agrega el token a cada petición
http.interceptors.request.use((config) => {
  const token = localStorage.getItem('token')
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})

// Normaliza errores
http.interceptors.response.use(
  (response) => response,
  (error) => {
    // Sesión expirada: se maneja una sola vez aquí
    if (error.response?.status === 401) {
      localStorage.removeItem('token')
      router.push('/login')
    }

    // Mensaje listo para mostrar en la View
    error.userMessage =
      error.response?.data?.message ??
      (error.request ? 'Sin conexión con el servidor' : 'Error inesperado')

    // Errores de validación de Laravel (422)
    error.validationErrors = error.response?.data?.errors ?? null

    return Promise.reject(error)
  }
)

export default http
```

> El interceptor **no muestra toasts**: `useToast()` solo funciona dentro de componentes. Solo deja el error listo (`userMessage`) y la View decide qué mostrar.

---

## 4. Capa 2: service (`services/productosService.js`)

Solo habla con la API. No guarda nada, no conoce Pinia ni la interfaz.

```js
// src/services/productosService.js
import http from '@/api/http'

export const productosService = {
  async listar(params = {}) {
    const { data } = await http.get('/productos', { params })
    return data // { data: [...], meta: { total, current_page, ... } }
  },

  async crear(payload) {
    const { data } = await http.post('/productos', payload)
    return data
  },

  async actualizar(id, payload) {
    const { data } = await http.put(`/productos/${id}`, payload)
    return data
  },

  async eliminar(id) {
    await http.delete(`/productos/${id}`)
  },
}
```

---

## 5. Capa 3: store (`stores/productos.js`)

Guarda el **estado compartido** y expone **acciones**. Lanza los errores hacia arriba (no los traga) para que la View decida qué hacer.

```js
// src/stores/productos.js
import { defineStore } from 'pinia'
import { ref } from 'vue'
import { productosService } from '@/services/productosService'

export const useProductosStore = defineStore('productos', () => {
  // ---- estado ----
  const items = ref([])
  const total = ref(0)
  const pagina = ref(1)
  const porPagina = ref(10)
  const cargando = ref(false)

  // ---- acciones ----
  async function cargar() {
    cargando.value = true
    try {
      const res = await productosService.listar({
        page: pagina.value,
        per_page: porPagina.value,
      })
      items.value = res.data
      total.value = res.meta.total
    } finally {
      cargando.value = false
    }
  }

  async function crear(datos) {
    await productosService.crear(datos)
    await cargar()
  }

  async function actualizar(datos) {
    await productosService.actualizar(datos.id, datos)
    await cargar()
  }

  async function eliminar(id) {
    await productosService.eliminar(id)
    await cargar()
  }

  function cambiarPagina(nueva, filas) {
    pagina.value = nueva
    porPagina.value = filas
    return cargar()
  }

  return {
    items, total, pagina, porPagina, cargando,
    cargar, crear, actualizar, eliminar, cambiarPagina,
  }
})
```

**Qué NO va aquí:** toasts, confirmaciones, si un dialog está abierto, textos de inputs.

---

## 6. Servicios de interfaz (alertas y confirmaciones)

### 6.1 Toast: `composables/useNotify.js`

En `App.vue` debe existir **un solo** `<Toast />` y `app.use(ToastService)` en `main.js`.

```js
// src/composables/useNotify.js
import { useToast } from 'primevue/usetoast'

export function useNotify() {
  const toast = useToast()

  return {
    ok:    (msg) => toast.add({ severity: 'success', summary: msg, life: 3000 }),
    info:  (msg) => toast.add({ severity: 'info',    summary: msg, life: 3000 }),
    aviso: (msg) => toast.add({ severity: 'warn',    summary: msg, life: 4000 }),
    error: (msg) => toast.add({ severity: 'error',   summary: msg, life: 5000 }),
  }
}
```

```vue
<!-- App.vue -->
<template>
  <Toast />
  <router-view />
</template>
```

### 6.2 Confirmación: `services/confirmService.js`

SweetAlert envuelto en una función que devuelve `true` o `false`. Si algún día cambias de librería, solo tocas este archivo.

```js
// src/services/confirmService.js
import Swal from 'sweetalert2'

export async function confirmar(titulo, texto = '') {
  const r = await Swal.fire({
    title: titulo,
    text: texto,
    icon: 'warning',
    showCancelButton: true,
    confirmButtonText: 'Sí, continuar',
    cancelButtonText: 'Cancelar',
  })
  return r.isConfirmed
}
```

---

## 7. Componentes hijos (presentational)

### 7.1 Tabla: `components/Productos/ProductoTable.vue`

Recibe datos por props y avisa con eventos. No sabe de dónde vienen los datos.

```vue
<script setup>
defineProps({
  productos: { type: Array, required: true },
  total: { type: Number, default: 0 },
  filas: { type: Number, default: 10 },
  cargando: { type: Boolean, default: false },
})

const emit = defineEmits(['editar', 'eliminar', 'pagina'])

function onPage(event) {
  // event.page empieza en 0
  emit('pagina', { pagina: event.page + 1, filas: event.rows })
}
</script>

<template>
  <DataTable
    :value="productos"
    :loading="cargando"
    :rows="filas"
    :totalRecords="total"
    lazy
    paginator
    @page="onPage"
  >
    <Column field="codigo" header="Código" />
    <Column field="nombre" header="Nombre" />
    <Column field="precio" header="Precio" />
    <Column header="Acciones">
      <template #body="{ data }">
        <Button icon="pi pi-pencil" text @click="emit('editar', data)" />
        <Button icon="pi pi-trash" text severity="danger"
                @click="emit('eliminar', data)" />
      </template>
    </Column>
  </DataTable>
</template>
```

### 7.2 Dialog: `components/Productos/ProductoDialog.vue`

Mantiene una **copia local** del formulario para no modificar el dato original hasta que el usuario guarde. No llama a la API: emite `guardar` con los datos.

```vue
<script setup>
import { ref, watch } from 'vue'

const props = defineProps({
  producto: { type: Object, default: null }, // null = crear, objeto = editar
  guardando: { type: Boolean, default: false },
  errores: { type: Object, default: null },  // errores 422 de Laravel
})

const visible = defineModel('visible', { type: Boolean, default: false })
const emit = defineEmits(['guardar'])

const vacio = () => ({ id: null, codigo: '', nombre: '', precio: 0 })
const form = ref(vacio())

// Cada vez que se abre, carga la copia
watch(visible, (abierto) => {
  if (abierto) form.value = props.producto ? { ...props.producto } : vacio()
})
</script>

<template>
  <Dialog
    v-model:visible="visible"
    :header="producto ? 'Editar producto' : 'Nuevo producto'"
    modal
    :style="{ width: '28rem' }"
  >
    <div class="flex flex-col gap-4">
      <div>
        <label>Código</label>
        <InputText v-model="form.codigo" fluid />
        <small v-if="errores?.codigo" class="text-red-500">
          {{ errores.codigo[0] }}
        </small>
      </div>

      <div>
        <label>Nombre</label>
        <InputText v-model="form.nombre" fluid />
        <small v-if="errores?.nombre" class="text-red-500">
          {{ errores.nombre[0] }}
        </small>
      </div>

      <div>
        <label>Precio</label>
        <InputNumber v-model="form.precio" mode="currency" currency="USD" fluid />
      </div>
    </div>

    <template #footer>
      <Button label="Cancelar" text @click="visible = false" />
      <Button label="Guardar" :loading="guardando" @click="emit('guardar', form)" />
    </template>
  </Dialog>
</template>
```

---

## 8. El padre (View): `views/ProductosView.vue`

Es el único que **coordina**: conoce el store, decide cuándo abrir el dialog, llama a las acciones y muestra el resultado al usuario.

```vue
<script setup>
import { ref, onMounted } from 'vue'
import { useProductosStore } from '@/stores/productos'
import { useNotify } from '@/composables/useNotify'
import { confirmar } from '@/services/confirmService'
import ProductoTable from '@/components/Productos/ProductoTable.vue'
import ProductoDialog from '@/components/Productos/ProductoDialog.vue'

const store = useProductosStore()
const { ok, error } = useNotify()

// Estado LOCAL de la vista (no va al store)
const mostrarDialog = ref(false)
const seleccionado = ref(null)
const guardando = ref(false)
const erroresForm = ref(null)

onMounted(() => store.cargar())

function nuevo() {
  seleccionado.value = null
  erroresForm.value = null
  mostrarDialog.value = true
}

function editar(producto) {
  seleccionado.value = producto
  erroresForm.value = null
  mostrarDialog.value = true
}

async function guardar(datos) {
  guardando.value = true
  erroresForm.value = null
  try {
    if (datos.id) {
      await store.actualizar(datos)
      ok('Producto actualizado')
    } else {
      await store.crear(datos)
      ok('Producto creado')
    }
    mostrarDialog.value = false
  } catch (e) {
    if (e.validationErrors) {
      erroresForm.value = e.validationErrors // se muestran junto a cada campo
    } else {
      error(e.userMessage)
    }
  } finally {
    guardando.value = false
  }
}

async function eliminar(producto) {
  if (!(await confirmar(`¿Eliminar "${producto.nombre}"?`))) return
  try {
    await store.eliminar(producto.id)
    ok('Producto eliminado')
  } catch (e) {
    error(e.userMessage)
  }
}
</script>

<template>
  <div class="flex justify-between mb-4">
    <h2>Productos</h2>
    <Button label="Nuevo" icon="pi pi-plus" @click="nuevo" />
  </div>

  <ProductoTable
    :productos="store.items"
    :total="store.total"
    :filas="store.porPagina"
    :cargando="store.cargando"
    @editar="editar"
    @eliminar="eliminar"
    @pagina="({ pagina, filas }) => store.cambiarPagina(pagina, filas)"
  />

  <ProductoDialog
    v-model:visible="mostrarDialog"
    :producto="seleccionado"
    :guardando="guardando"
    :errores="erroresForm"
    @guardar="guardar"
  />
</template>
```

---

## 9. Ruta con lazy loading (`router/index.js`)

```js
{
  path: '/productos',
  component: () => import('@/views/ProductosView.vue'),
}
```

Cada vista se descarga solo cuando el usuario entra a ella.

---

## 10. Tabla de decisiones: ¿dónde va cada cosa?

| Qué es | Dónde vive | Por qué |
|---|---|---|
| Llamada HTTP | `services/` | Solo conoce la API |
| Datos compartidos (lista, usuario, carrito, caja abierta) | `stores/` | Varias vistas los usan |
| Si un dialog está abierto | View (`ref`) | Estado local, no lo necesita nadie más |
| Copia del formulario en edición | Dialog (`ref` local) | No se modifica el original hasta guardar |
| Toast de "Guardado" / "Error" | View llama a `useNotify()` | La View sabe cuándo terminó la acción |
| Confirmación "¿Eliminar?" | View llama a `confirmar()` | La View sabe qué eligió el usuario |
| Error 401 / sin conexión | Interceptor de axios | Se maneja una sola vez |
| Errores de validación 422 | Store los lanza, View los pasa al Dialog por prop | El Dialog solo los muestra |
| Lógica reutilizable con estado (paginación, búsqueda con debounce) | `composables/` | Se comparte entre vistas |
| Formateo (moneda, fechas) | `utils/` o composable | Función pura |

---

## 11. Errores comunes a evitar

1. **El hijo llama a la API o al store directamente.** Pierde reutilización y es difícil de probar. Que emita un evento.
2. **El hijo modifica una prop.** Las props son de solo lectura. Usa una copia local o `v-model`.
3. **El store muestra toasts o abre dialogs.** El store no debe conocer la interfaz.
4. **Todo en el store.** Un `ref(false)` de un modal no necesita Pinia.
5. **`try/catch` repetido para errores genéricos.** Usa el interceptor.
6. **Varios `<Toast />` en distintos componentes.** Con uno en `App.vue` basta.
7. **Vistas gigantes.** Si una View pasa de ~300 líneas, extrae la lógica a un composable (`useProductos.js`) o divide en más componentes.

---

## 12. Checklist para un módulo nuevo

- [ ] Crear `services/xService.js` con `listar`, `crear`, `actualizar`, `eliminar`.
- [ ] Crear `stores/x.js` con estado, `cargar` y las acciones.
- [ ] Crear `components/X/XTable.vue` (props + emits).
- [ ] Crear `components/X/XDialog.vue` (`v-model:visible`, copia local, emite `guardar`).
- [ ] Crear `views/XView.vue` que orquesta, con `useNotify` y `confirmar`.
- [ ] Registrar la ruta con `() => import(...)`.
- [ ] Probar los tres casos: éxito, error de validación (422) y error de red.

---

## 13. Optimización de rendimiento (referencia rápida)

Separar responsabilidades mejora el **orden y mantenimiento**; el rendimiento es otro tema:

| Técnica | Cuándo aplicarla |
|---|---|
| Lazy loading de rutas | Siempre (cambio de una línea por ruta) |
| `npx vite-bundle-visualizer` | Una vez, para ver qué pesa |
| Paginación del lado del servidor | Tablas con muchos registros |
| `computed` en lugar de métodos | Valores derivados en plantillas |
| `debounce` en buscadores | Búsquedas que llaman a la API |
| `shallowRef` / `v-memo` / `v-once` | Solo si mides un problema real con miles de elementos |

**Regla general:** medir primero (Chrome DevTools → Performance, Lighthouse) y optimizar después.

---

## 14. Equivalencias en React y Angular

La arquitectura es la misma; cambian las herramientas.

| Capa | Vue | React | Angular |
|---|---|---|---|
| Service | `services/*.js` + axios | igual | `HttpClient` en `@Injectable` |
| Store | Pinia | Zustand / Redux Toolkit | Signals / NgRx |
| Lógica reutilizable | Composables | Custom hooks | Services |
| Padre (View) | `views/` | `pages/` | Smart component |
| Hijos | props + `emit` | props + callbacks | `@Input` + `@Output` |
| Evitar prop drilling | `provide/inject` | Context | Inyección de dependencias |
