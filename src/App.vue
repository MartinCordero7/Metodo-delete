<script setup>
import { ref, computed, onMounted } from 'vue'
import jsPDF from 'jspdf'
import 'jspdf-autotable'
import DocumentTable from './components/DocumentTable.vue'

const API_BASE = import.meta.env.VITE_API_BASE || 'http://localhost:3000'

const documents = ref([])
const searchQuery = ref('')
const loading = ref(false)
const error = ref(null)
const currentPage = ref(1)
const itemsPerPage = 15
const dataSource = ref('productos') // 'productos' o 'carrito'

onMounted(() => {
  // Asegurar que siempre empieza en productos
  dataSource.value = 'productos'
  loadDocumentsFromAPI()
})

// Cargar documentos desde MongoDB
const loadDocumentsFromAPI = async () => {
  loading.value = true
  error.value = null
  try {
    const endpoint = dataSource.value === 'productos' 
      ? `${API_BASE}/api/productos`
      : `${API_BASE}/api/carrito`
    
    const response = await fetch(endpoint)
    
    if (!response.ok) {
      throw new Error(`Error del servidor: ${response.status}`)
    }
    
    const data = await response.json()
    
    // Mapear los datos de MongoDB al formato de nuestra tabla
    if (Array.isArray(data)) {
      if (dataSource.value === 'productos') {
        documents.value = data.map((doc) => ({
          _id: doc._id,
          tipo: 'producto',
          nombre: doc.nombre || 'Sin nombre',
          descripcion: doc.descripcion || 'Sin descripción',
          categoria: doc.categoria || 'N/A',
          precio: doc.precio || 0,
          stock: doc.stock || 0,
          marca: doc.marca || 'N/A',
          sku: doc.sku || 'N/A',
          fechaCreacion: doc.fechaCreacion ? new Date(doc.fechaCreacion).toLocaleDateString('es-ES') : 'N/A',
          activo: doc.estado === 'Activo' ? true : doc.estado === 'Inactivo' ? false : (doc.activo !== undefined ? doc.activo : true)
        }))
      } else {
        // Mapeo para carrito
        documents.value = data.map((doc) => ({
          _id: doc._id,
          tipo: 'carrito',
          usuarioId: doc.usuarioId,
          productos: doc.productos || [],
          cantidadProductos: Array.isArray(doc.productos) ? doc.productos.length : 0,
          total: doc.total || 0,
          estado: doc.estado || 'N/A',
          fechaCreacion: doc.fechaCreacion ? new Date(doc.fechaCreacion).toLocaleDateString('es-ES') : 'N/A',
          fechaActualizacion: doc.fechaActualizacion ? new Date(doc.fechaActualizacion).toLocaleDateString('es-ES') : 'N/A',
          metodoPago: doc.metodoPago || 'Sin especificar',
          direccionEnvio: doc.direccionEnvio || 'Sin dirección'
        }))
      }
    } else {
      documents.value = []
    }
  } catch (err) {
    error.value = 'Error al cargar los datos de MongoDB: ' + err.message
    console.error('Error:', err)
  } finally {
    loading.value = false
  }
}

// Filtrar documentos basado en la búsqueda
const filteredDocuments = computed(() => {
  if (!searchQuery.value) {
    return documents.value
  }
  
  const query = searchQuery.value.toLowerCase()
  return documents.value.filter(doc => {
    if (doc.tipo === 'producto') {
      return (
        doc.nombre?.toLowerCase().includes(query) ||
        doc.marca?.toLowerCase().includes(query) ||
        doc.categoria?.toLowerCase().includes(query) ||
        doc.descripcion?.toLowerCase().includes(query) ||
        doc.sku?.toLowerCase().includes(query)
      )
    } else {
      return (
        doc.estado?.toLowerCase().includes(query) ||
        doc.direccionEnvio?.toLowerCase().includes(query) ||
        doc.metodoPago?.toLowerCase().includes(query)
      )
    }
  })
})

// Obtener documentos paginados
const paginatedDocuments = computed(() => {
  const start = (currentPage.value - 1) * itemsPerPage
  const end = start + itemsPerPage
  return filteredDocuments.value.slice(start, end)
})

// Calcular total de páginas
const totalPages = computed(() => {
  return Math.ceil(filteredDocuments.value.length / itemsPerPage)
})

// Reiniciar a página 1 cuando se busca
const handleSearch = () => {
  currentPage.value = 1
}

// Función para descargar PDF
const downloadPDF = () => {
  const doc = new jsPDF()
  
  // Título del documento
  doc.setFontSize(18)
  doc.text(`Listado de ${dataSource.value === 'productos' ? 'Productos' : 'Carrito'} - MongoDB`, 14, 22)
  
  // Información adicional
  doc.setFontSize(11)
  doc.text(`Total de documentos: ${filteredDocuments.value.length}`, 14, 30)
  doc.text(`Generado el: ${new Date().toLocaleDateString('es-ES')}`, 14, 36)
  
  // Preparar datos para la tabla
  const tableData = filteredDocuments.value.map(doc => [
    doc.nombre,
    doc.marca,
    doc.categoria,
    `$${doc.precio}`,
    doc.stock,
    doc.descripcion?.substring(0, 60) + '...'
  ])
  
  // Crear tabla
  doc.autoTable({
    startY: 42,
    head: [['Nombre', 'Marca', 'Categoría', 'Precio', 'Stock', 'Descripción']],
    body: tableData,
    theme: 'grid',
    styles: {
      fontSize: 8,
      cellPadding: 3
    },
    headStyles: {
      fillColor: [100, 108, 255],
      textColor: 255,
      fontStyle: 'bold'
    },
    columnStyles: {
      0: { cellWidth: 35 },
      1: { cellWidth: 25 },
      2: { cellWidth: 25 },
      3: { cellWidth: 20 },
      4: { cellWidth: 15 },
      5: { cellWidth: 50 }
    }
  })
  
  // Guardar el PDF
  doc.save(`documentos_plos_${new Date().getTime()}.pdf`)
}

// Función para cargar archivo JSON personalizado
const loadCustomJSON = (event) => {
  const file = event.target.files[0]
  if (file) {
    const reader = new FileReader()
    reader.onload = (e) => {
      try {
        const data = JSON.parse(e.target.result)
        documents.value = Array.isArray(data) ? data : [data]
      } catch (error) {
        alert('Error al leer el archivo JSON: ' + error.message)
      }
    }
    reader.readAsText(file)
  }
}

// Eliminar producto con confirmación si está vinculado a carrito
const handleDelete = async (doc) => {
  if (!doc?._id) return

  const baseUrl = `http://localhost:3000/api/productos/${doc._id}`

  const confirmBasic = confirm(`¿Eliminar "${doc.nombre}"?`)
  if (!confirmBasic) return

  loading.value = true
  error.value = null
  try {
    let response = await fetch(baseUrl, { method: 'DELETE' })

    if (response.status === 409) {
      const data = await response.json()
      const force = confirm(`${data.message || 'El producto está vinculado a compras.'}\n¿Deseas eliminarlo de todas formas?`)
      if (!force) return
      response = await fetch(`${baseUrl}?force=true`, { method: 'DELETE' })
    }

    if (!response.ok) {
      const txt = await response.text()
      throw new Error(txt || 'No se pudo eliminar el producto')
    }

    await loadDocumentsFromAPI()
  } catch (err) {
    error.value = 'Error al eliminar: ' + err.message
  } finally {
    loading.value = false
  }
}
</script>

<template>
  <div>
    <h1>� Tienda MongoDB - {{ dataSource === 'productos' ? 'Productos' : 'Carrito' }}</h1>
    
    <div v-if="error" class="error-message">
      ⚠️ {{ error }}
      <button @click="loadDocumentsFromAPI" class="btn" style="margin-left: 1rem;">
        🔄 Reintentar
      </button>
    </div>
    
    <div v-if="loading" class="loading">
      ⏳ Cargando datos desde MongoDB...
    </div>
    
    <template v-if="!loading">
      <div class="controls">
        <select v-model="dataSource" @change="loadDocumentsFromAPI" class="select-box">
          <option value="productos">📦 Productos</option>
          <option value="carrito">🛒 Carrito</option>
        </select>
        
        <input
          v-model="searchQuery"
          @input="handleSearch"
          type="text"
          class="search-box"
          placeholder="🔍 Buscar..."
        />
        
        <button @click="downloadPDF" class="btn" :disabled="documents.length === 0">
          📥 Descargar PDF
        </button>
        
        <button @click="loadDocumentsFromAPI" class="btn btn-secondary">
          🔄 Recargar
        </button>
      </div>
      
      <DocumentTable 
        :documents="paginatedDocuments" 
        :data-source="dataSource"
        @delete="handleDelete"
      />
      
      <!-- Paginación -->
      <div class="pagination">
        <button 
          @click="currentPage--" 
          :disabled="currentPage === 1"
          class="btn btn-pagination"
        >
          ← Anterior
        </button>
        
        <div class="pagination-info">
          Página {{ currentPage }} de {{ totalPages }}
        </div>
        
        <button 
          @click="currentPage++" 
          :disabled="currentPage === totalPages"
          class="btn btn-pagination"
        >
          Siguiente →
        </button>
      </div>
      
      <div class="stats">
        Mostrando {{ paginatedDocuments.length }} de {{ filteredDocuments.length }} documentos (Total: {{ documents.length }})
      </div>
    </template>
  </div>
</template>
