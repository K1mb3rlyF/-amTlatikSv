<template>
  <!-- Modal para usuario autenticado -->
  <div class="profile-modal" v-if="show && isUserAuthenticated" @click.self="close">
    <div class="profile-content" @click.stop>
      <span class="close-modal" @click="close">&times;</span>
      <h2 class="modal-title">Mi Perfil</h2>

      <!-- Sección de Foto de Perfil -->
      <div class="avatar-section">
        <div class="avatar-container">
          <img 
            v-if="userProfile.avatar && userProfile.avatar.startsWith('data:image')" 
            :src="userProfile.avatar" 
            class="user-avatar"
            alt="Foto de perfil"
          />
          <img
            v-else-if="userProfile.avatar"
            :src="userProfile.avatar"
            class="user-avatar"
            alt="Foto de perfil"
            @error="handleImageError"
          />
          <div v-else class="user-avatar placeholder">
            {{ getUserInitials }}
          </div>
        </div>
        
        <input 
          type="file" 
          ref="avatarInput"
          accept="image/*" 
          @change="handleAvatarUpload" 
          style="display: none"
        />
        <div class="avatar-buttons">
          <button @click="$refs.avatarInput.click()" class="btn-change-avatar">
            📷 Cambiar Foto
          </button>
          <button 
            v-if="userProfile.avatar" 
            @click="removeAvatar" 
            class="btn-remove-avatar"
          >
            ❌ Eliminar
          </button>
        </div>
        <p class="avatar-hint">Tamaño máximo: 1MB • Formatos: JPG, PNG</p>
        
        <!-- Mensaje de éxito -->
        <div v-if="showSuccessMessage" class="success-message">
          ✅ {{ successMessage }}
        </div>

        <!-- Debug info (puedes quitar esto después) -->
        <div v-if="debugInfo" class="debug-info">
          <p><strong>Debug:</strong> Avatar URL length: {{ userProfile.avatar ? userProfile.avatar.length : 0 }}</p>
          <p><strong>Tipo:</strong> {{ userProfile.avatar ? (userProfile.avatar.startsWith('data:image') ? 'Base64' : 'URL') : 'No hay avatar' }}</p>
        </div>
      </div>

      <!-- Información del Usuario -->
      <div class="user-info">
        <div class="info-item">
          <label>Nombre:</label>
          <span class="info-value">{{ userProfile.displayName || 'No establecido' }}</span>
        </div>
        <div class="info-item">
          <label>Email:</label>
          <span class="info-value">{{ userProfile.email }}</span>
        </div>
        <div class="info-item">
          <label>Rol:</label>
          <span class="role-badge" :class="userProfile.role">
            {{ userProfile.role === 'admin' ? '👑 Administrador' : '👤 Usuario' }}
          </span>
        </div>
        <div class="info-item" v-if="userProfile.createdAt">
          <label>Miembro desde:</label>
          <span class="info-value">{{ formatDate(userProfile.createdAt) }}</span>
        </div>
      </div>

      <!-- Progreso de Subida -->
      <div v-if="uploadProgress > 0" class="upload-progress">
        <p>Subiendo imagen: {{ uploadProgress }}%</p>
        <div class="progress-bar">
          <div class="progress-fill" :style="{ width: uploadProgress + '%' }"></div>
        </div>
      </div>

      <div class="profile-actions">
        <button @click="close" class="btn-close">Cerrar</button>
      </div>
    </div>
  </div>

  <!-- Modal para usuario NO autenticado -->
  <div class="profile-modal" v-else-if="show" @click.self="close">
    <div class="profile-content" @click.stop>
      <span class="close-modal" @click="close">&times;</span>
      <h2 class="modal-title">Iniciar Sesión</h2>
      
      <div class="auth-required-message">
        <div class="auth-icon">🔒</div>
        <h3>Acceso Requerido</h3>
        <p>Para acceder a tu perfil y dejar comentarios, necesitas iniciar sesión.</p>
        
        <div class="auth-actions">
          <button @click="redirectToLogin" class="btn-login">
            🚀 Iniciar Sesión
          </button>
          <button @click="close" class="btn-close">
            Cancelar
          </button>
        </div>
        
        <p class="auth-hint">
          ¿No tienes cuenta? <a href="#" @click.prevent="redirectToRegister">Regístrate aquí</a>
        </p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, computed, watch } from 'vue'
import { useRouter } from 'vue-router'
import { 
  db, 
  auth 
} from '@/firebase'
import { doc, updateDoc, getDoc } from 'firebase/firestore'

const props = defineProps({
  show: Boolean,
  user: Object
})

const emit = defineEmits(['close', 'profile-updated'])
const router = useRouter()

// VERIFICACIÓN ROBUSTA DE AUTENTICACIÓN
const isUserAuthenticated = computed(() => {
  return props.user && 
         props.user.uid && 
         typeof props.user.uid === 'string' &&
         props.user.uid.length > 5
})

// Datos del perfil
const userProfile = reactive({
  displayName: '',
  email: '',
  avatar: '',
  role: 'user',
  createdAt: null
})

const uploadProgress = ref(0)
const showSuccessMessage = ref(false)
const successMessage = ref('')
const debugInfo = ref(true) // Cambiar a false cuando funcione

// Referencia al input de archivo
const avatarInput = ref(null)

// Redirigir a login
const redirectToLogin = () => {
  close()
  router.push('/auth/login')
}

// Redirigir a registro
const redirectToRegister = () => {
  close()
  router.push('/auth/register')
}

// Iniciales del usuario para avatar por defecto
const getUserInitials = computed(() => {
  if (!userProfile.displayName) return '👤'
  return userProfile.displayName
    .split(' ')
    .map(n => n[0])
    .join('')
    .toUpperCase()
    .slice(0, 2)
})

// Cargar datos del perfil (solo si está autenticado)
const loadUserProfile = async () => {
  if (!isUserAuthenticated.value) {
    console.log('No se carga perfil: usuario no autenticado')
    return
  }
  
  try {
    // Cargar datos de Firestore
    const userDoc = await getDoc(doc(db, 'users', props.user.uid))
    if (userDoc.exists()) {
      const userData = userDoc.data()
      
      Object.assign(userProfile, {
        displayName: props.user.displayName || `${userData.nombre} ${userData.apellido}`,
        email: props.user.email,
        avatar: userData.avatar || '',
        role: userData.role || 'user',
        createdAt: userData.createdAt
      })

      console.log('🔄 Perfil cargado:', {
        avatar: userProfile.avatar ? userProfile.avatar.substring(0, 50) + '...' : 'No hay avatar',
        tipo: userProfile.avatar ? (userProfile.avatar.startsWith('data:image') ? 'Base64' : 'URL') : 'Vacío'
      })
    }
  } catch (error) {
    console.error('Error cargando perfil:', error)
  }
}

// Convertir imagen a Base64
const convertToBase64 = (file) => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.readAsDataURL(file)
    reader.onload = () => {
      console.log('📸 Base64 generado, longitud:', reader.result.length)
      resolve(reader.result)
    }
    reader.onerror = error => reject(error)
  })
}

// Mostrar mensaje de éxito
const showSuccess = (message) => {
  successMessage.value = message
  showSuccessMessage.value = true
  setTimeout(() => {
    showSuccessMessage.value = false
    successMessage.value = ''
  }, 3000)
}

// EMITIR EVENTO GLOBAL para sincronizar todos los componentes
const emitGlobalProfileUpdate = (updateData) => {
  // Emitir evento local para el componente padre (Header)
  emit('profile-updated', updateData)
  
  // EMITIR EVENTO GLOBAL para ComunidadView y otros componentes
  window.dispatchEvent(new CustomEvent('profile-updated', { 
    detail: updateData
  }))
  
  console.log('🌐 Evento global emitido:', updateData)
}

// Manejar error de imagen
const handleImageError = (event) => {
  console.error('❌ Error cargando imagen:', event)
  userProfile.avatar = '' // Limpiar avatar si hay error
}

// SOLUCIÓN SIMPLIFICADA: Manejar subida de avatar
const handleAvatarUpload = async (event) => {
  if (!isUserAuthenticated.value) {
    alert('Debes iniciar sesión para subir una foto')
    return
  }

  const file = event.target.files[0]
  if (!file) return

  // Validaciones
  if (!file.type.startsWith('image/')) {
    alert('Por favor selecciona una imagen válida (JPEG, PNG, etc.)')
    return
  }

  if (file.size > 1 * 1024 * 1024) {
    alert('La imagen debe ser menor a 1MB')
    return
  }

  try {
    uploadProgress.value = 10

    // Convertir a Base64
    const base64Image = await convertToBase64(file)
    uploadProgress.value = 60

    console.log('🖼️ Base64 completo:', base64Image.substring(0, 100) + '...')

    // SOLUCIÓN: Usar Base64 directamente SIN modificar
    const avatarData = base64Image
    
    // Actualizar inmediatamente en la interfaz
    userProfile.avatar = avatarData
    uploadProgress.value = 70

    console.log('✅ Avatar actualizado localmente')

    // Guardar en Firestore
    await updateDoc(doc(db, 'users', props.user.uid), {
      avatar: avatarData,
      lastUpdated: new Date()
    })
    uploadProgress.value = 90

    console.log('💾 Avatar guardado en Firestore')

    // EMITIR EVENTOS DE ACTUALIZACIÓN
    const updateData = { 
      avatar: avatarData,
      userId: props.user.uid,
      timestamp: new Date().getTime(),
      type: 'avatar_updated'
    }
    
    emitGlobalProfileUpdate(updateData)

    // Mostrar mensaje de éxito
    showSuccess('¡Foto de perfil actualizada correctamente!')
    uploadProgress.value = 100

    // Reset progress después de 1 segundo
    setTimeout(() => {
      uploadProgress.value = 0
      // Limpiar el input file para permitir subir la misma imagen otra vez
      if (avatarInput.value) {
        avatarInput.value.value = ''
      }
    }, 1000)

  } catch (error) {
    console.error('❌ Error subiendo avatar:', error)
    alert('Error al subir la imagen: ' + error.message)
    uploadProgress.value = 0
    // Limpiar el input file en caso de error
    if (avatarInput.value) {
      avatarInput.value.value = ''
    }
  }
}

// Eliminar avatar (solo si está autenticado)
const removeAvatar = async () => {
  if (!isUserAuthenticated.value) return
  
  if (!confirm('¿Estás seguro de que quieres eliminar tu foto de perfil?')) {
    return
  }

  try {
    // Eliminar de Firestore
    await updateDoc(doc(db, 'users', props.user.uid), {
      avatar: '',
      lastUpdated: new Date()
    })

    // Actualizar localmente
    userProfile.avatar = ''

    // EMITIR EVENTOS DE ACTUALIZACIÓN
    const updateData = { 
      avatar: '',
      userId: props.user.uid,
      timestamp: new Date().getTime(),
      type: 'avatar_removed'
    }
    
    emitGlobalProfileUpdate(updateData)

    // Mostrar mensaje de éxito
    showSuccess('Foto de perfil eliminada correctamente')

    // Limpiar el input file
    if (avatarInput.value) {
      avatarInput.value.value = ''
    }

  } catch (error) {
    console.error('Error eliminando avatar:', error)
    alert('Error al eliminar la foto de perfil: ' + error.message)
  }
}

// Formatear fecha
const formatDate = (timestamp) => {
  if (!timestamp) return 'N/A'
  const date = new Date(timestamp.seconds * 1000)
  return date.toLocaleDateString('es-ES', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}

const close = () => {
  emit('close')
}

// Cargar perfil cuando se muestre el modal y el usuario esté autenticado
watch(() => props.show, (newVal) => {
  if (newVal) {
    if (isUserAuthenticated.value) {
      loadUserProfile()
    }
  }
})
</script>

<style scoped>
.profile-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  z-index: 1000;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 20px;
  box-sizing: border-box;
}

.profile-content {
  background: white;
  border-radius: 16px;
  padding: 2rem;
  width: 90%;
  max-width: 450px;
  max-height: 80vh;
  overflow-y: auto;
  position: relative;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
  margin: auto;
}

.close-modal {
  position: absolute;
  top: 1rem;
  right: 1rem;
  cursor: pointer;
  font-size: 1.5rem;
  color: #666;
  background: none;
  border: none;
  width: 30px;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  transition: background 0.3s ease;
  z-index: 1001;
}

.close-modal:hover {
  background: #f5f5f5;
}

.modal-title {
  color: #ff6f61;
  margin-bottom: 1.5rem;
  text-align: center;
  font-size: 1.5rem;
}

/* Sección Avatar */
.avatar-section {
  text-align: center;
  margin-bottom: 2rem;
  padding-bottom: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.avatar-container {
  margin-bottom: 1rem;
}

.user-avatar {
  width: 120px;
  height: 120px;
  border-radius: 50%;
  object-fit: cover;
  border: 4px solid #ff6f61;
  margin: 0 auto;
  display: block;
  background: #f5f5f5;
}

.user-avatar.placeholder {
  background: linear-gradient(135deg, #ff6f61, #ff8e76);
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 2rem;
  font-weight: bold;
  margin: 0 auto;
}

.avatar-buttons {
  display: flex;
  gap: 0.5rem;
  justify-content: center;
  margin-bottom: 0.5rem;
  flex-wrap: wrap;
}

.btn-change-avatar {
  background: #ff6f61;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9rem;
  transition: background 0.3s ease;
}

.btn-change-avatar:hover {
  background: #e85b4e;
}

.btn-remove-avatar {
  background: #dc2626;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9rem;
  transition: background 0.3s ease;
}

.btn-remove-avatar:hover {
  background: #b91c1c;
}

.avatar-hint {
  color: #666;
  font-size: 0.75rem;
  margin: 0;
}

/* Mensaje de éxito */
.success-message {
  background: #d1fae5;
  color: #065f46;
  padding: 0.75rem;
  border-radius: 8px;
  margin-top: 1rem;
  font-weight: 600;
  border: 1px solid #a7f3d0;
  animation: slideDown 0.3s ease;
}

/* Debug info */
.debug-info {
  background: #fef3c7;
  color: #92400e;
  padding: 0.5rem;
  border-radius: 6px;
  margin-top: 0.5rem;
  font-size: 0.75rem;
  border: 1px solid #f59e0b;
}

.debug-info p {
  margin: 0.25rem 0;
}

@keyframes slideDown {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* INFORMACIÓN DEL USUARIO - CORREGIDO */
.user-info {
  margin-bottom: 2rem;
}

.info-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  padding: 0.75rem;
  background: #f8fafc;
  border-radius: 8px;
  min-height: 44px; /* Altura mínima consistente */
}

.info-item label {
  font-weight: 600;
  color: #374151;
  min-width: 120px;
  flex-shrink: 0; /* Evita que se encoja */
}

.info-value {
  color: #6b7280;
  text-align: right;
  flex: 1;
  word-break: break-word; /* Permite que el texto se ajuste */
  padding-left: 0.5rem;
}

/* BADGE DE ROL - CORREGIDO */
.role-badge {
  padding: 0.25rem 0.75rem;
  border-radius: 12px;
  font-size: 0.875rem;
  font-weight: 600;
  display: inline-block;
  text-align: center;
  white-space: nowrap;
  flex-shrink: 0; /* Evita que se estire */
  margin-left: auto; /* Empuja el badge hacia la derecha */
}

.role-badge.user {
  background: #e2e8f0;
  color: #475569;
}

.role-badge.admin {
  background: linear-gradient(135deg, #ff6f61, #ff8e76);
  color: white;
}

/* Progreso de Subida */
.upload-progress {
  text-align: center;
  margin: 1rem 0;
  padding: 1rem;
  background: #f1f5f9;
  border-radius: 8px;
}

.upload-progress p {
  margin: 0 0 0.5rem 0;
  color: #475569;
}

.progress-bar {
  width: 100%;
  height: 8px;
  background: #e2e8f0;
  border-radius: 4px;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #ff6f61, #ff8e76);
  transition: width 0.3s ease;
}

/* Acciones */
.profile-actions {
  text-align: center;
}

.btn-close {
  background: #6b7280;
  color: white;
  border: none;
  padding: 0.75rem 2rem;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1rem;
  transition: background 0.3s ease;
}

.btn-close:hover {
  background: #4b5563;
}

/* Nuevos estilos para el mensaje de autenticación */
.auth-required-message {
  text-align: center;
  padding: 2rem 1rem;
}

.auth-icon {
  font-size: 3rem;
  margin-bottom: 1rem;
}

.auth-required-message h3 {
  color: #ff6f61;
  margin-bottom: 1rem;
  font-size: 1.5rem;
}

.auth-required-message p {
  color: #666;
  margin-bottom: 2rem;
  line-height: 1.5;
}

.auth-actions {
  display: flex;
  gap: 1rem;
  justify-content: center;
  margin-bottom: 1.5rem;
  flex-wrap: wrap;
}

.btn-login {
  background: #ff6f61;
  color: white;
  border: none;
  padding: 0.75rem 1.5rem;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1rem;
  transition: background 0.3s ease;
}

.btn-login:hover {
  background: #e85b4e;
}

.auth-hint {
  font-size: 0.9rem;
  color: #666;
}

.auth-hint a {
  color: #ff6f61;
  text-decoration: none;
}

.auth-hint a:hover {
  text-decoration: underline;
}

/* RESPONSIVE - CORREGIDO */
@media (max-width: 768px) {
  .profile-modal {
    padding: 10px;
    align-items: flex-start;
    padding-top: 50px;
  }
  
  .profile-content {
    padding: 1.5rem;
    margin: 0;
    max-height: 85vh;
  }
  
  .avatar-buttons {
    flex-direction: column;
  }
  
  .info-item {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.5rem;
    min-height: auto; /* Quita altura mínima en móvil */
  }
  
  .info-item label {
    min-width: auto;
  }
  
  .info-value {
    text-align: left;
    padding-left: 0;
  }
  
  .role-badge {
    margin-left: 0; /* Quita el margen automático en móvil */
    align-self: flex-start; /* Alinea a la izquierda en móvil */
  }

  /* Responsive para el mensaje de auth */
  .auth-actions {
    flex-direction: column;
  }
  
  .auth-actions button {
    width: 100%;
  }
}

@media (max-height: 600px) {
  .profile-modal {
    align-items: flex-start;
    padding-top: 20px;
  }
  
  .profile-content {
    max-height: 90vh;
  }
}
</style>