<template>
  <div v-if="show" class="modal" @click.self="close">
    <div class="modal-content" @click.stop>
      <span class="close-modal" @click="close">&times;</span>
      <h2 class="modal-title">Registrar</h2>

      <form @submit.prevent="handleSubmit">
        <p class="info-message">
          Tu nombre de usuario se generará automáticamente y se te mostrará al
          completar el registro.
        </p>

        <!-- Sección de Foto de Perfil -->
        <div class="form-group">
          <label for="avatar">Foto de Perfil <span class="optional">(Opcional)</span></label>
          <div class="avatar-upload-section">
            <div class="avatar-preview">
              <img 
                v-if="avatarPreview" 
                :src="avatarPreview" 
                class="avatar-preview-img"
              />
              <div v-else class="avatar-placeholder">
                👤
              </div>
            </div>
            <input 
              type="file" 
              id="avatar"
              ref="avatarInput"
              accept="image/*" 
              @change="handleAvatarSelect"
              style="display: none"
            />
            <div class="avatar-buttons">
              <button 
                type="button" 
                @click="$refs.avatarInput.click()" 
                class="btn-avatar-upload"
              >
                📷 Elegir Foto
              </button>
              <button 
                v-if="avatarPreview" 
                type="button" 
                @click="removeAvatar" 
                class="btn-avatar-remove"
              >
                ❌ Quitar
              </button>
            </div>
          </div>
          <p class="avatar-hint">Tamaño máximo: 1MB • Formatos: JPG, PNG</p>
        </div>

        <div class="form-group">
          <label for="nombre">Nombre</label>
          <input 
            type="text" 
            id="nombre" 
            v-model="form.nombre"
            :class="{ error: errors.nombre }"
            required 
          />
          <span v-if="errors.nombre" class="error-message">{{ errors.nombre }}</span>
        </div>

        <div class="form-group">
          <label for="apellido">Apellido</label>
          <input 
            type="text" 
            id="apellido" 
            v-model="form.apellido"
            :class="{ error: errors.apellido }"
            required 
          />
          <span v-if="errors.apellido" class="error-message">{{ errors.apellido }}</span>
        </div>

        <div class="form-group">
          <label for="email">Correo Electrónico <span class="optional">(Requerido)</span></label>
          <input 
            type="email" 
            id="email" 
            v-model="form.email"
            :class="{ error: errors.email }"
            required
          />
          <span v-if="errors.email" class="error-message">{{ errors.email }}</span>
        </div>

        <div class="form-group">
          <label for="telefono">Teléfono <span class="optional">(Opcional)</span></label>
          <input 
            type="tel" 
            id="telefono" 
            v-model="form.telefono"
            :class="{ error: errors.telefono }"
            placeholder="1234-5678"
            maxlength="9"
          />
          <span v-if="errors.telefono" class="error-message">{{ errors.telefono }}</span>
        </div>

        <div class="form-group">
          <label for="region">Región</label>
          <select 
            id="region" 
            v-model="form.region"
            :class="{ error: errors.region }"
            required
          >
            <option value="" disabled selected>Selecciona tu región</option>
            <option value="norte">Norte</option>
            <option value="centro">Centro</option>
            <option value="sur">Sur</option>
          </select>
          <span v-if="errors.region" class="error-message">{{ errors.region }}</span>
        </div>

        <!-- Contraseña con botón ver/ocultar -->
        <div class="form-group">
          <label for="password">Contraseña (mínimo 10 caracteres)</label>
          <div class="password-input-container">
            <input 
              :type="showPassword ? 'text' : 'password'" 
              id="password" 
              v-model="form.password"
              :class="{ error: errors.password }"
              required 
              autocomplete="new-password"
            />
            <button 
              type="button" 
              class="password-toggle"
              @click="showPassword = !showPassword"
            >
              {{ showPassword ? '🙈' : '👁️' }}
            </button>
          </div>
          <span v-if="errors.password" class="error-message">{{ errors.password }}</span>
        </div>

        <!-- Confirmar contraseña con botón ver/ocultar -->
        <div class="form-group">
          <label for="confirmPassword">Confirmar Contraseña</label>
          <div class="password-input-container">
            <input 
              :type="showConfirmPassword ? 'text' : 'password'" 
              id="confirmPassword" 
              v-model="form.confirmPassword"
              :class="{ error: errors.confirmPassword }"
              required 
              autocomplete="new-password"
            />
            <button 
              type="button" 
              class="password-toggle"
              @click="showConfirmPassword = !showConfirmPassword"
            >
              {{ showConfirmPassword ? '🙈' : '👁️' }}
            </button>
          </div>
          <span v-if="errors.confirmPassword" class="error-message">{{ errors.confirmPassword }}</span>
        </div>

        <div v-if="message" :class="messageType">
          {{ message }}
        </div>

        <button type="submit" class="btn-login" :disabled="isSubmitting">
          {{ isSubmitting ? 'Registrando...' : 'Registrar' }}
        </button>
      </form>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, defineProps, defineEmits, watch } from 'vue'
import { useRouter } from 'vue-router'
import { 
  createUserWithEmailAndPassword,
  updateProfile,
  signOut
} from 'firebase/auth';
import { 
  auth,
  db,
  doc, 
  setDoc, 
  serverTimestamp 
} from '@/firebase';

const router = useRouter();
const props = defineProps({
  show: Boolean
})

const emit = defineEmits(['close', 'register-success'])

const form = reactive({
  nombre: '',
  apellido: '',
  email: '',
  telefono: '',
  region: '',
  password: '',
  confirmPassword: ''
})

const errors = reactive({
  nombre: '',
  apellido: '',
  email: '',
  telefono: '',
  region: '',
  password: '',
  confirmPassword: ''
})

// Variables para avatar
const avatarPreview = ref('')
const avatarFile = ref(null)
const message = ref('')
const messageType = ref('success-message')
const isSubmitting = ref(false)

// Variables para mostrar/ocultar contraseña
const showPassword = ref(false)
const showConfirmPassword = ref(false)

const close = () => {
  emit('close')
}

// Función para formatear teléfono
const formatPhoneNumber = (value) => {
  // Remover todo excepto números
  const numbers = value.replace(/\D/g, '')
  
  // Limitar a 8 dígitos
  const limited = numbers.slice(0, 8)
  
  // Aplicar formato ####-#### después del 4to dígito
  if (limited.length > 4) {
    return `${limited.slice(0, 4)}-${limited.slice(4)}`
  }
  
  return limited
}

// Watcher para formatear automáticamente el teléfono
watch(() => form.telefono, (newValue) => {
  form.telefono = formatPhoneNumber(newValue)
})

// Manejar selección de avatar
const handleAvatarSelect = (event) => {
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

  avatarFile.value = file
  
  // Crear preview
  const reader = new FileReader()
  reader.onload = (e) => {
    avatarPreview.value = e.target.result
  }
  reader.readAsDataURL(file)
}

// Remover avatar
const removeAvatar = () => {
  avatarPreview.value = ''
  avatarFile.value = null
  if (document.getElementById('avatar')) {
    document.getElementById('avatar').value = ''
  }
}

// Convertir imagen a Base64
const convertToBase64 = (file) => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.readAsDataURL(file)
    reader.onload = () => resolve(reader.result)
    reader.onerror = error => reject(error)
  })
}

const validateForm = () => {
  let isValid = true
  const newErrors = {}

  if (!form.nombre.trim()) {
    newErrors.nombre = 'El nombre es obligatorio'
    isValid = false
  } else if (!/^[a-zA-ZáéíóúÁÉÍÓÚñÑ\s]+$/.test(form.nombre)) {
    newErrors.nombre = 'Solo se permiten letras en el nombre'
    isValid = false
  }

  if (!form.apellido.trim()) {
    newErrors.apellido = 'El apellido es obligatorio'
    isValid = false
  } else if (!/^[a-zA-ZáéíóúÁÉÍÓÚñÑ\s]+$/.test(form.apellido)) {
    newErrors.apellido = 'Solo se permiten letras en el apellido'
    isValid = false
  }

  if (!form.email.trim()) {
    newErrors.email = 'El email es obligatorio para el registro'
    isValid = false
  } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(form.email)) {
    newErrors.email = 'Por favor ingrese un email válido'
    isValid = false
  }

  // Validación de teléfono (opcional pero con formato si se ingresa)
  if (form.telefono && !/^\d{4}-\d{4}$/.test(form.telefono)) {
    newErrors.telefono = 'El teléfono debe tener formato 1234-5678'
    isValid = false
  }

  if (!form.region) {
    newErrors.region = 'Por favor seleccione una región'
    isValid = false
  }

  if (!form.password) {
    newErrors.password = 'La contraseña es obligatoria'
    isValid = false
  } else if (form.password.length < 10) {
    newErrors.password = 'La contraseña debe tener al menos 10 caracteres'
    isValid = false
  }

  if (form.password !== form.confirmPassword) {
    newErrors.confirmPassword = 'Las contraseñas no coinciden'
    isValid = false
  }

  // Actualizar errors
  Object.keys(errors).forEach(key => {
    errors[key] = newErrors[key] || ''
  })

  return isValid
}

const handleSubmit = async () => {
  if (!validateForm()) {
    return
  }

  isSubmitting.value = true
  message.value = ''
  
  try {
    // Crear usuario en Firebase Authentication
    const userCredential = await createUserWithEmailAndPassword(
      auth, 
      form.email, 
      form.password
    );
    
    const user = userCredential.user;
    
    // Actualizar perfil con nombre completo
    await updateProfile(user, {
      displayName: `${form.nombre} ${form.apellido}`
    });

    // Convertir avatar a Base64 si existe
    let avatarBase64 = ''
    if (avatarFile.value) {
      avatarBase64 = await convertToBase64(avatarFile.value)
    }

    // Guardar información adicional en Firestore
    await setDoc(doc(db, 'users', user.uid), {
      nombre: form.nombre,
      apellido: form.apellido,
      email: form.email,
      telefono: form.telefono || '', // Guardar vacío si no se ingresó
      region: form.region,
      createdAt: serverTimestamp(),
      isMember: true,
      username: `user_${user.uid.slice(0, 8)}`,
      role: 'user',
      // Guardar avatar si existe
      avatar: avatarBase64
    });

    // 🔥 CORRECCIÓN: Cerrar sesión inmediatamente después del registro
    await signOut(auth);

    message.value = '✅ ¡Registro exitoso! Ahora inicia sesión con tus credenciales.';
    messageType.value = 'success-message';

    // Limpiar el formulario
    Object.keys(form).forEach(key => {
      form[key] = '';
    });
    
    // Limpiar avatar
    removeAvatar()

    // 🔥 CORRECCIÓN: Cerrar el modal después de 2 segundos (sin redirigir)
    setTimeout(() => {
      close();
    }, 2000);

  } catch (error) {
    console.error('Error en registro:', error);
    
    // Manejo de errores de Firebase
    if (error.code === 'auth/email-already-in-use') {
      message.value = '❌ Este email ya está registrado';
    } else if (error.code === 'auth/weak-password') {
      message.value = '❌ La contraseña es muy débil';
    } else if (error.code === 'auth/invalid-email') {
      message.value = '❌ El formato del email es inválido';
    } else {
      message.value = `❌ Error: ${error.message}`;
    }
    
    messageType.value = 'error-message';
  } finally {
    isSubmitting.value = false;
  }
}
</script>

<style scoped>
.modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  z-index: 100;
  display: flex;
  justify-content: center;
  align-items: center;
}

.modal-content {
  background-color: white;
  padding: 30px;
  border-radius: 15px;
  width: 90%;
  max-width: 500px;
  max-height: 90vh;
  overflow-y: auto;
  position: relative;
}

.modal-title {
  color: var(--primary-color);
  margin-bottom: 20px;
  text-align: center;
}

.close-modal {
  position: absolute;
  top: 15px;
  right: 15px;
  cursor: pointer;
  font-size: 1.5rem;
  color: var(--gray);
}

.info-message {
  color: var(--gray);
  font-size: 0.8rem;
  margin-top: -10px;
  margin-bottom: 15px;
}

.form-group {
  margin-bottom: 1.2rem;
  position: relative;
}

.form-group label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: 500;
  color: var(--text-color);
}

.form-group input,
.form-group select {
  width: 100%;
  padding: 12px 15px;
  border: 2px solid var(--input-border);
  border-radius: 8px;
  background-color: white;
  font-size: 1rem;
}

.form-group input.error,
.form-group select.error {
  border-color: var(--error-color);
}

.error-message {
  color: var(--error-color);
  font-size: 0.8rem;
  margin-top: 5px;
  display: block;
}

.success-message {
  color: var(--success-color);
  font-size: 0.9rem;
  margin-top: 10px;
  text-align: center;
}

.btn-login {
  background-color: var(--primary-color);
  color: white;
  border: none;
  border-radius: 8px;
  padding: 14px;
  font-size: 1rem;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.3s;
  margin-top: 10px;
  width: 100%;
}

.btn-login:hover {
  background-color: #7a3434;
}

.btn-login:disabled {
  background-color: var(--gray);
  cursor: not-allowed;
}

.optional {
  color: var(--gray);
  font-size: 0.8rem;
}

select {
  appearance: none;
  background-image: url("data:image/svg+xml;charset=UTF-8,%3csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3e%3cpolyline points='6 9 12 15 18 9'%3e%3c/polyline%3e%3c/svg%3e");
  background-repeat: no-repeat;
  background-position: right 10px center;
  background-size: 1em;
}

/* Estilos para avatar */
.avatar-upload-section {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 15px;
  margin-bottom: 10px;
}

.avatar-preview {
  width: 100px;
  height: 100px;
  border-radius: 50%;
  border: 3px solid var(--primary-color);
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #f5f5f5;
}

.avatar-preview-img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.avatar-placeholder {
  font-size: 2.5rem;
  color: var(--gray);
}

.avatar-buttons {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
  justify-content: center;
}

.btn-avatar-upload {
  background: var(--primary-color);
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9rem;
  transition: background-color 0.3s;
}

.btn-avatar-upload:hover {
  background: #7a3434;
}

.btn-avatar-remove {
  background: #dc2626;
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9rem;
  transition: background-color 0.3s;
}

.btn-avatar-remove:hover {
  background: #b91c1c;
}

.avatar-hint {
  color: var(--gray);
  font-size: 0.75rem;
  text-align: center;
  margin-top: 5px;
}

/* Estilos para el input de contraseña con botón */
.password-input-container {
  position: relative;
  display: flex;
  align-items: center;
}

.password-input-container input {
  padding-right: 50px;
  width: 100%;
}

.password-toggle {
  position: absolute;
  right: 10px;
  background: none;
  border: none;
  cursor: pointer;
  font-size: 1.2rem;
  padding: 5px;
  border-radius: 4px;
  transition: background-color 0.3s ease;
}

.password-toggle:hover {
  background: rgba(0, 0, 0, 0.1);
}

/* Estilos para el input de teléfono */
.form-group input[type="tel"] {
  letter-spacing: 1px;
  font-family: monospace;
  font-size: 1.1rem;
}

@media (max-width: 768px) {
  .modal-content {
    padding: 20px;
  }
  
  .avatar-buttons {
    flex-direction: column;
    width: 100%;
  }
  
  .btn-avatar-upload,
  .btn-avatar-remove {
    width: 100%;
  }
}
</style>