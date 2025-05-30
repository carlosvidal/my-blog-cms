<template>
  <ViewLayout>
    <template #title>Dashboard</template>
    <template #subtitle>Bienvenido a tu panel de control</template>
    
    <div class="space-y-8">
      <!-- Advertencia de blog no válido -->
      <section v-if="blogError" class="bg-red-50 border border-red-200 rounded-lg p-4">
        <div class="flex items-start">
          <AlertTriangle class="w-5 h-5 text-red-500 mt-0.5 mr-3" />
          <div>
            <h3 class="font-medium text-red-800">Error con el blog seleccionado</h3>
            <p class="text-red-700 mt-1">El blog seleccionado no existe o no se puede acceder. Por favor, selecciona otro blog.</p>
            <button 
              @click="router.push('/blogs')" 
              class="mt-3 px-4 py-2 bg-red-100 hover:bg-red-200 text-red-800 rounded-md transition-colors"
            >
              Seleccionar blog
            </button>
          </div>
        </div>
      </section>
      
      <!-- Stats Overview -->
      <section>
        <div class="flex items-center justify-between mb-4">
          <h2 class="text-lg font-medium text-panel-heading">Resumen</h2>
          <button @click="fetchStats" class="p-1 rounded hover:bg-panel text-panel-muted transition-colors" title="Refrescar estadísticas">
            <RefreshCw class="w-4 h-4" />
          </button>
        </div>
        
        <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
          <StatCard title="Posts" :value="stats.posts" icon="article" color="blue" />
          <StatCard title="Páginas" :value="stats.pages" icon="description" color="green">
            <template #footer>Última actualización: {{ new Date().toLocaleDateString() }}</template>
          </StatCard>
          <StatCard title="Categorías" :value="stats.categories" icon="category" color="purple" />
        </div>
      </section>
      
      <!-- Recent Activity -->
      <section class="border border-panel rounded-lg bg-panel">
        <div class="px-6 py-4 border-b border-panel">
          <h2 class="text-lg font-medium text-panel-heading">Actividad reciente</h2>
        </div>
        
        <div v-if="isLoading" class="p-8 text-center text-panel-muted">
          <div class="animate-pulse flex justify-center">
            <div class="h-4 w-24 bg-panel rounded"></div>
          </div>
          <p class="mt-2 text-sm">Cargando actividad...</p>
        </div>
        
        <div v-else-if="!recentActivity.length" class="p-12 text-center">
          <div class="text-panel-muted mb-3">
            <History class="w-10 h-10" />
          </div>
          <h3 class="text-lg font-medium text-panel-text mb-1">Sin actividad reciente</h3>
          <p class="text-panel-muted text-sm">Comienza a crear contenido para ver tu actividad aquí</p>
        </div>
        
        <div v-else class="divide-y border-panel">
          <div v-for="(activity, index) in recentActivity" :key="index" class="px-6 py-4 flex items-start hover:bg-panel transition-colors">
            <div class="flex-shrink-0 h-8 w-8 rounded-full bg-panel flex items-center justify-center text-gray-600 mr-3">
              <component :is="getIconComponent(activity.icon)" class="w-4 h-4" />
            </div>
            <div>
              <p class="text-sm text-panel-heading">{{ activity.message }}</p>
              <p class="text-xs text-panel-muted mt-1">{{ activity.time }}</p>
            </div>
          </div>
        </div>
      </section>
      
      <!-- Quick Actions -->
      <section>
        <h2 class="text-lg font-medium text-panel-heading mb-4">Acciones rápidas</h2>
        <div class="grid grid-cols-1 md:grid-cols-4 gap-4">
          <div v-for="action in quickActions" :key="action.name" 
               class="p-4 border border-panel rounded-lg bg-panel hover:shadow-sm transition-all cursor-pointer"
               @click="action.action">
            <div class="flex items-center">
              <div :class="`w-8 h-8 rounded flex items-center justify-center mr-3 ${action.bgColor}`">
                <component :is="getIconComponent(action.icon)" class="w-4 h-4 text-white" />
              </div>
              <span class="text-sm font-medium text-panel-text">{{ action.name }}</span>
            </div>
          </div>
        </div>
      </section>
    </div>
  </ViewLayout>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import ViewLayout from './ViewLayout.vue'
import StatCard from '@/components/StatCard.vue'
import api from '@/utils/api'
import { RefreshCw, AlertTriangle, FileText, File, Tag, Settings, History, Edit, PlusCircle, Trash2 } from 'lucide-vue-next'

const router = useRouter()
const isLoading = ref(false)
const blogError = ref(false)

const stats = ref({
  posts: 0,
  pages: 0,
  categories: 0
})

// Verificar si el blog activo existe
const verifyActiveBlog = async () => {
  const activeBlogUuid = localStorage.getItem('activeBlog')
  console.log('Verificando blog activo con UUID:', activeBlogUuid)
  
  // Si no hay un blog seleccionado, redirigir a la página de selección de blogs
  if (!activeBlogUuid) {
    console.log('No hay un blog seleccionado, redirigiendo a la página de blogs')
    router.push('/blogs')
    return false
  }
  
  try {
    // URL directa sin usar api.baseURL
    console.log('Consultando blog con UUID:', activeBlogUuid)
    
    // Intentar obtener información del blog para verificar que existe
    const response = await api.get(`/api/blogs/uuid/${activeBlogUuid}`)
    console.log('Blog activo verificado:', response.data)
    return true
  } catch (error) {
    console.error('Error al verificar el blog activo:', error)
    console.log('Detalles del error:', {
      status: error.response?.status,
      statusText: error.response?.statusText,
      data: error.response?.data,
      message: error.message
    })
    
    // Limpiar localStorage y redirigir a la página de blogs
    localStorage.removeItem('activeBlog')
    console.log('Se ha limpiado la selección de blog debido a un error')
    alert(`El blog seleccionado no existe o no se puede acceder. Error: ${error.response?.status || error.message}. Serás redirigido a la página de selección de blogs.`)
    router.push('/blogs')
    return false
  }
}

const recentActivity = ref([
  {
    icon: 'edit',
    message: 'Editaste el post "Introducción a Vue 3"',
    time: 'Hace 2 horas'
  },
  {
    icon: 'plus-circle',
    message: 'Creaste una nueva categoría "Tutoriales"',
    time: 'Ayer'
  },
  {
    icon: 'trash',
    message: 'Eliminaste el post "Borrador"',
    time: 'Hace 2 días'
  }
])

const quickActions = [
  {
    name: 'Crear post',
    icon: 'file-text',
    bgColor: 'bg-panel',
    action: () => router.push('/cms/posts/new')
  },
  {
    name: 'Crear página',
    icon: 'file',
    bgColor: 'bg-panel',
    action: () => router.push('/cms/pages/new')
  },
  {
    name: 'Gestionar categorías',
    icon: 'tag',
    bgColor: 'bg-panel',
    action: () => router.push('/cms/categories')
  },
  {
    name: 'Configuración',
    icon: 'settings',
    bgColor: 'bg-panel',
    action: () => router.push('/cms/settings')
  }
]

const fetchStats = async () => {
  isLoading.value = true
  try {
    // Verificar si hay un token de autenticación antes de hacer las llamadas a la API
    const token = localStorage.getItem('authToken')
    if (!token) {
      console.warn('No authentication token found')
      return
    }
    
    // Función para cargar los contadores
    const loadCounts = async () => {
      try {
        // Obtener el blog activo
        const activeBlogUuid = localStorage.getItem('activeBlog');
        if (!activeBlogUuid) {
          console.warn('No hay un blog activo seleccionado');
          return;
        }
        
        // Obtener el ID del blog
        const blogResponse = await api.get(`/api/blogs/uuid/${activeBlogUuid}`);
        const blogId = blogResponse.data?.id;
        
        if (!blogId) {
          console.error('No se pudo obtener el ID del blog activo');
          return;
        }
        
        console.log('Obteniendo estadísticas para blogId:', blogId);
        
        // Hacer las peticiones de estadísticas incluyendo el blogId
        const [postsRes, pagesRes, categoriesRes] = await Promise.allSettled([
          api.get(`/api/stats/posts/count?blogId=${blogId}`),
          api.get(`/api/stats/pages/count?blogId=${blogId}`),
          api.get(`/api/stats/categories/count?blogId=${blogId}`)
        ]);
        
        console.log('Respuestas de estadísticas:', { postsRes, pagesRes, categoriesRes });

        // Procesar cada respuesta individualmente
        const processResponse = (response, type) => {
          if (response.status === 'fulfilled') {
            return response.value.data?.count || 0;
          } else {
            console.error(`Error cargando estadísticas de ${type}:`, {
              message: response.reason?.message,
              response: response.reason?.response?.data,
              status: response.reason?.response?.status
            });
            return 0; // Valor por defecto en caso de error
          }
        };

        stats.value.posts = processResponse(postsRes, 'posts');
        stats.value.pages = processResponse(pagesRes, 'pages');
        stats.value.categories = processResponse(categoriesRes, 'categories');
      } catch (error) {
        console.error('Error en loadCounts:', {
          message: error.message,
          response: error.response?.data,
          status: error.response?.status
        });
      }
    }
    await loadCounts();
  } catch (error) {
    console.error('Fetch error:', error)
  } finally {
    isLoading.value = false
  }
}

// Función para obtener el componente de icono correspondiente
const getIconComponent = (iconName) => {
  const iconMap = {
    'edit': Edit,
    'plus-circle': PlusCircle,
    'trash': Trash2,
    'file-text': FileText,
    'file': File,
    'tag': Tag,
    'settings': Settings
  }
  return iconMap[iconName] || Edit
}

onMounted(async () => {
  // Verificar si el blog activo existe antes de cargar las estadísticas
  const blogValid = await verifyActiveBlog()
  
  // Solo cargar estadísticas si el blog es válido
  if (blogValid) {
    fetchStats()
  }
})
</script>