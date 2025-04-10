<script setup>
import { ref, onMounted, computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import api from '@/utils/api'
import TipTapEditor from '@/components/TipTapEditor.vue'
import BaseButton from '@/components/BaseButton.vue'
import { transitions, rounded, shadows } from '@/styles/designSystem'
import { Save } from 'lucide-vue-next'

const route = useRoute()
const router = useRouter()
const isLoading = ref(false)
const isSaving = ref(false)
const post = ref({
  title: '',
  slug: '',
  content: '',
  status: 'draft',
  excerpt: '',
  featuredImage: null,
  author: '' // Add author field
})

// Remove the HTML template code from here
const error = ref('')
const categories = ref([])

const isEditMode = computed(() => {
  return !!route.params.id
})

const pageTitle = computed(() => {
  return isEditMode.value ? 'Editar Post' : 'Crear Post'
})

const fetchPost = async (uuid) => {
  isLoading.value = true
  try {
    console.log(`Obteniendo post con UUID: ${uuid}`)
    const response = await api.get(`/api/posts/uuid/${uuid}`)
    post.value = response.data
    console.log('Post obtenido:', response.data)
  } catch (err) {
    console.error('Error al obtener post:', err)
    error.value = err.response?.data?.message || err.response?.data?.error || err.message || 'Error al cargar el post'
  } finally {
    isLoading.value = false
  }
}

const handleSubmit = async () => {
  const authToken = localStorage.getItem('authToken');
  if (!authToken) {
    router.push('/login');
    return;
  }

  try {
    // Extract user ID from token
    const userId = authToken.split('-').pop();
    if (!userId) {
      throw new Error('Invalid token format');
    }

    isSaving.value = true;
    error.value = '';

    // Validate required fields
    if (!post.value.title?.trim()) {
      throw new Error('Title is required');
    }
    if (!post.value.content || typeof post.value.content !== 'string' || !post.value.content.trim()) {
      throw new Error('Content is required');
    }
    if (!post.value.categoryId) {
      throw new Error('Category is required');
    }

    // Remove this duplicate check - we already validated userId above
    // if (!authUser?.id) {
    //   throw new Error('User not authenticated');
    // }

    // Prepare payload according to API schema - usando campos directos
    const categoryId = Number(post.value.categoryId || 0);
    
    // Asegurarse de que el ID de la categoría es un número válido
    if (isNaN(categoryId)) {
      console.error('ID de categoría inválido:', post.value.categoryId);
      throw new Error('El ID de categoría no es válido');
    }
    
    // Crear un payload simplificado con campos directos
    const payload = {
      title: post.value.title.trim(),
      content: post.value.content.trim(),
      excerpt: post.value.excerpt?.trim() || '',
      status: post.value.status || 'draft',
      authorId: userId, // Enviar authorId directamente
      categoryId: categoryId, // Enviar categoryId directamente
      slug: post.value.slug?.trim() || ''
    };

    // First save/update the post without image
    console.log('Guardando datos del post...');
    console.log('Payload completo:', JSON.stringify(payload, null, 2));
    console.log('authorId:', payload.authorId);
    console.log('categoryId:', payload.categoryId);
    const postResponse = isEditMode.value
      ? await api.put(`/api/posts/${route.params.id}`, payload)
      : await api.post('/api/cms-posts', payload); // Usar la nueva ruta específica para el CMS

    // Verify post was saved successfully
    if (!postResponse.data?.id) {
      throw new Error('No se recibió un ID válido del post guardado');
    }
    console.log('Post guardado exitosamente con ID:', postResponse.data.id);

    // Only include slug if provided
    if (post.value.slug?.trim()) {
      payload.slug = post.value.slug.trim();
    }

    // Si hay una imagen, intentar subirla después de guardar el post
    if (post.value.featuredImage) {
      console.log('Iniciando subida de imagen...');
      try {
        const imageFormData = new FormData();
        imageFormData.append('file', post.value.featuredImage, post.value.featuredImage.name);
        imageFormData.append('userId', userId);
        imageFormData.append('purpose', 'post_featured_image');

        console.log('Preparando FormData para la imagen:', {
          fileName: post.value.featuredImage.name,
          size: post.value.featuredImage.size
        });

        const imageResponse = await api.post('/api/media/upload', imageFormData, {
          headers: {
            'Authorization': `Bearer ${authToken}`,
            'Content-Type': 'multipart/form-data'
          },
          timeout: 60000
        });

        if (!imageResponse.data?.path) {
          console.error('Respuesta inválida de subida:', imageResponse.data);
          console.warn('La imagen no se pudo subir, pero el post fue guardado');
        } else {
          console.log('Imagen subida exitosamente:', imageResponse.data.path);
          const mediaUrl = `http://localhost:3000/${imageResponse.data.path}`;
          console.log('Actualizando post con URL de la imagen:', mediaUrl);
          await api.put(`/api/posts/${postResponse.data.id}`, {
            featuredImage: mediaUrl
          });
          console.log('Post actualizado exitosamente con la imagen');
        }
      } catch (uploadError) {
        console.error('Error en la subida:', uploadError);
        console.warn('Error al subir la imagen, pero el post fue guardado:', uploadError.message);
      }
    }

    router.push('/posts');

  } catch (err) {
    console.error('Post error:', err);
    console.error('Post error details:', {
      message: err.message,
      response: err.response?.data,
      stack: err.stack
    });
    error.value = err.response?.data?.message || err.message || 'Error saving post';
  } finally {
    isSaving.value = false;
  }
}

onMounted(async () => {
  const authToken = localStorage.getItem('authToken');
  if (!authToken) {
    router.push('/login');
    return;
  }

  try {
    // Extract user ID from token (assuming it's encoded in the token)
    // This is a temporary solution - you should ideally decode the JWT properly
    const userId = authToken.split('-').pop();
    if (!userId) {
      throw new Error('Invalid token format');
    }
    post.value.author = userId;
  } catch (error) {
    console.error('Auth error:', error);
    router.push('/login');
  }

  // Load categories
  const response = await api.get('/api/categories')
  categories.value = response.data

  // If in edit mode, fetch the post
  if (isEditMode.value) {
    await fetchPost(route.params.id)
  }
})

const handleImageUpload = (event) => {
  const file = event.target.files[0];
  if (!file) return;

  // Validate file type
  if (!file.type.match('image/jpeg')) {
    error.value = 'Solo se permiten imágenes JPG';
    return;
  }

  // Validate image dimensions
  const img = new Image();
  img.onload = function () {
    if (this.width < 800 || this.height < 600) {
      error.value = `La imagen debe ser de al menos 800×600px (actual: ${this.width}×${this.height}px)`;
      return;
    }
    post.value.featuredImage = file;
  };
  img.onerror = () => {
    error.value = 'Error al cargar la imagen';
  };
  img.src = URL.createObjectURL(file);
};
</script>

<template>
  <div class="max-w-4xl mx-auto py-4 px-2 sm:py-8 sm:px-6 lg:px-8">
    <!-- Loading state -->
    <div v-if="isLoading" class="text-center py-12">
      <div class="animate-spin rounded-full h-12 w-12 border-t-2 border-b-2 border-gray-800 mx-auto"></div>
      <p class="mt-4 text-gray-600">Cargando...</p>
    </div>

    <template v-else>
      <!-- Error message -->
      <div v-if="error" class="mb-6 p-4 bg-red-50 rounded-lg border border-red-200">
        <p class="text-red-600 font-medium">{{ error }}</p>
      </div>

      <div :class="['bg-white overflow-hidden sm:border sm:border-gray-200 sm:rounded sm:shadow-md']">
        <div class="p-3 sm:p-6 border-b border-gray-200 flex justify-between items-center">
          <h1 class="text-2xl font-bold text-gray-900">{{ pageTitle }}</h1>
        </div>

        <form @submit.prevent="handleSubmit" class="p-3 sm:p-6 space-y-4 sm:space-y-6">
          <!-- Title Input -->
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Título</label>
            <input v-model="post.title" required placeholder="Escribe el título del post"
              class="w-full px-4 py-2 rounded border border-gray-300 focus:ring-2 focus:ring-gray-300 focus:border-gray-300 transition-all duration-200">
          </div>

          <!-- Category Select -->
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Categoría</label>
            <select v-model="post.categoryId" required
              class="w-full px-4 py-2 rounded border border-gray-300 focus:ring-2 focus:ring-gray-300 focus:border-gray-300 transition-all duration-200">
              <option value="" disabled>Selecciona una categoría</option>
              <option v-for="category in categories" :key="category.id" :value="category.id">
                {{ category.name }}
              </option>
            </select>
          </div>

          <!-- Slug Input -->
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Slug</label>
            <div
              class="flex rounded border border-gray-300 overflow-hidden focus-within:ring-2 focus-within:ring-gray-300 focus-within:border-gray-300">
              <span class="bg-gray-50 px-3 py-2 text-gray-500 text-sm border-r border-gray-300">/blog/</span>
              <input v-model="post.slug" placeholder="mi-post-url" class="flex-1 px-4 py-2 focus:outline-none">
            </div>
          </div>

          <!-- Status Selector -->
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Estado</label>
            <select v-model="post.status"
              class="w-full px-4 py-2 rounded border border-gray-300 focus:ring-2 focus:ring-gray-300 focus:border-gray-300 transition-all duration-200">
              <option value="draft">Borrador</option>
              <option value="published">Publicado</option>
            </select>
          </div>

          <!-- Content Editor -->
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Contenido</label>
            <TipTapEditor v-model="post.content"
              class="min-h-[200px] border border-gray-300 rounded focus:ring-2 focus:ring-gray-300 overflow-hidden transition-all duration-200" />
          </div>

          <!-- Excerpt Input -->
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Resumen (máx. 160 caracteres)</label>
            <textarea v-model="post.excerpt" maxlength="160" placeholder="Breve descripción del post"
              class="w-full px-4 py-2 rounded border border-gray-300 focus:ring-2 focus:ring-gray-300 focus:border-gray-300 transition-all duration-200"
              rows="3"></textarea>
            <p class="text-xs text-gray-500 mt-1">{{ post.excerpt?.length || 0 }}/160 caracteres</p>
          </div>

          <!-- Featured Image Upload -->
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Imagen destacada (JPG, min. 800×600px)</label>
            <div class="mt-1 flex items-center">
              <input type="file" id="featuredImage" accept="image/jpeg" @change="handleImageUpload" class="sr-only">
              <label for="featuredImage"
                class="cursor-pointer rounded-md bg-white py-2 px-3 text-sm font-medium text-gray-700 shadow-sm border border-gray-300 hover:bg-gray-50 focus:outline-none focus:ring-2 focus:ring-gray-300">
                Seleccionar imagen
              </label>
              <span class="ml-4 text-sm text-gray-500" v-if="post.featuredImage">
                {{ post.featuredImage.name }}
              </span>
            </div>
          </div>

          <!-- Form Actions -->
          <div class="flex flex-wrap justify-end gap-3 pt-6 border-t border-gray-100">
            <BaseButton type="button" variant="secondary" @click="router.push('/posts')" :disabled="isSaving">
              Cancelar
            </BaseButton>
            <BaseButton type="submit" variant="primary" :disabled="isSaving">
              <span class="flex items-center whitespace-nowrap">
                <svg v-if="isSaving" class="animate-spin mr-2 h-4 w-4 text-white" xmlns="http://www.w3.org/2000/svg"
                  fill="none" viewBox="0 0 24 24">
                  <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                  <path class="opacity-75" fill="currentColor"
                    d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z">
                  </path>
                </svg>
                <Save v-else class="w-4 h-4 mr-2" />
                {{ isSaving ? 'Guardando...' : 'Guardar Post' }}
              </span>
            </BaseButton>
          </div>
        </form>
      </div>
    </template>
  </div>
</template>
