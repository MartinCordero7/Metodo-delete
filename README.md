# Vue Table App

Aplicación Vue.js para visualizar, filtrar y descargar documentos en formato PDF.

## 🚀 Características

- ✅ Lectura de archivos JSON
- 🔍 Filtrado en tiempo real por título, autor, categoría o descripción
- 📥 Descarga del listado en formato PDF
- 📂 Carga de archivos JSON personalizados
- 📱 Diseño responsive
- 🎨 Interfaz moderna con modo claro/oscuro

## 📦 Instalación

```bash
npm install
```

## 🏃 Ejecutar en modo desarrollo

```bash
npm run dev
```

## 🏗️ Construir para producción

```bash
npm run build
```

## 📝 Uso

1. La aplicación carga automáticamente los datos desde `src/data/documents.json`
2. Usa el campo de búsqueda para filtrar documentos
3. Haz clic en "Descargar PDF" para exportar el listado filtrado
4. Usa "Cargar JSON" para importar tu propio archivo de datos

## 📄 Formato del JSON

```json
[
  {
    "id": 1,
    "titulo": "Título del documento",
    "autor": "Nombre del autor",
    "categoria": "Categoría",
    "fecha": "2024-01-15",
    "descripcion": "Descripción del documento"
  }
]
```

## 🛠️ Tecnologías

- Vue.js 3
- Vite
- jsPDF (generación de PDFs)
- jspdf-autotable (tablas en PDF)
