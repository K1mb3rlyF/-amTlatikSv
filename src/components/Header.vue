<script setup>
import { RouterLink, useRouter } from 'vue-router'
import { ref, onMounted, computed } from 'vue';
import { auth, db } from '@/firebase';
import { onAuthStateChanged, signOut } from 'firebase/auth';
import { doc, getDoc } from 'firebase/firestore';
import UserProfile from './UserProfile.vue';

const router = useRouter();
const isLoggedIn = ref(false);
const user = ref(null);
const userData = ref(null);
const showMenu = ref(false);
const showProfileModal = ref(false);
const showLogoutConfirm = ref(false);
let hoverTimeout;

// Verificar autenticación con Firebase
onMounted(() => {
  onAuthStateChanged(auth, async (firebaseUser) => {
    if (firebaseUser) {
      isLoggedIn.value = true;
      user.value = firebaseUser;
      
      // Cargar datos adicionales de Firestore
      try {
        const userDoc = await getDoc(doc(db, 'users', firebaseUser.uid));
        if (userDoc.exists()) {
          userData.value = userDoc.data();
        }
      } catch (error) {
        console.error('Error cargando datos del usuario:', error);
      }
    } else {
      isLoggedIn.value = false;
      user.value = null;
      userData.value = null;
    }
  });
});

// Computed para mostrar nombre y avatar
const displayName = computed(() => {
  if (userData.value) {
    return userData.value.nombre || user.value?.displayName || 'Usuario';
  }
  return user.value?.displayName || 'Usuario';
});

// SOLUCIÓN: Usar solo Firestore para el avatar (evita error de longitud)
const userAvatar = computed(() => {
  return userData.value?.avatar || '';
});

// VERIFICACIÓN MÁS ROBUSTA DEL USUARIO
const currentUser = computed(() => {
  if (!user.value) return null;
  if (!user.value.uid) return null;
  if (typeof user.value.uid !== 'string') return null;
  if (user.value.uid.length < 5) return null;
  
  return user.value;
});

// ESCUCHAR ACTUALIZACIONES DEL PERFIL - NUEVO MÉTODO
const handleProfileUpdated = async (updatedData) => {
  console.log('🔄 Perfil actualizado recibido en Header:', updatedData);
  
  // Si hay un avatar actualizado
  if (updatedData.avatar !== undefined) {
    console.log('🖼️ Actualizando avatar en header...');
    
    // Forzar actualización del avatar en el header
    if (userData.value) {
      userData.value.avatar = updatedData.avatar;
    }
    
    // Recargar datos del usuario desde Firestore para sincronizar completamente
    if (auth.currentUser) {
      try {
        const userDoc = await getDoc(doc(db, 'users', auth.currentUser.uid));
        if (userDoc.exists()) {
          userData.value = { ...userData.value, ...userDoc.data() };
          console.log('✅ Datos del usuario recargados desde Firestore');
        }
      } catch (error) {
        console.error('Error recargando datos del usuario:', error);
      }
    }
    
    // Forzar re-render del componente
    userData.value = { ...userData.value };
  }
};

// Mostrar modal de confirmación
function confirmLogout() {
  showLogoutConfirm.value = true;
  showMenu.value = false;
}

// Confirmar cierre de sesión
function handleLogout() {
  signOut(auth).then(() => {
    isLoggedIn.value = false;
    user.value = null;
    userData.value = null;
    showLogoutConfirm.value = false;
    // Recargar la página para ir al menú principal
    window.location.href = '/';
  }).catch((error) => {
    console.error('Error cerrando sesión:', error);
    alert('Error al cerrar sesión. Por favor intenta nuevamente.');
  });
}

// Cancelar cierre de sesión
function cancelLogout() {
  showLogoutConfirm.value = false;
}

function goToLogin() {
  if (!isLoggedIn.value) {
    router.push('/auth/login');
  } else {
    // Si ya está logueado, mostrar menú
    showMenu.value = !showMenu.value;
  }
}

function openProfile() {
  if (currentUser.value) {
    showProfileModal.value = true;
    showMenu.value = false;
  } else {
    // Si no hay usuario autenticado, redirigir a login
    router.push('/auth/login');
  }
}

function handleMouseEnter() {
  clearTimeout(hoverTimeout);
  if (isLoggedIn.value) {
    showMenu.value = true;
  }
}

function handleMouseLeave() {
  hoverTimeout = setTimeout(() => {
    showMenu.value = false;
  }, 300);
}
</script>

<template>
  <header class="glass-header">
    <div class="logo-container">
      <img src="/logo.png" alt="Ñam Tlatik SV" class="logo animate__animated animate__fadeIn" />
      <h1 class="animate__animated animate__fadeIn">Ñam Tlatik SV</h1>
    </div>

    <div
      class="login-container"
      @mouseenter="handleMouseEnter"
      @mouseleave="handleMouseLeave"
    >
      <button class="auth-btn animate__animated animate__fadeIn" @click="goToLogin">
        <div class="user-avatar-small" v-if="isLoggedIn && userAvatar">
          <img :src="userAvatar" alt="Avatar" :key="userAvatar" />
        </div>
        <div class="user-avatar-small placeholder" v-else-if="isLoggedIn">
          {{ displayName.charAt(0) }}
        </div>
        <i class="fas fa-user" v-else></i>
        {{ isLoggedIn ? displayName : 'Iniciar Sesión' }}
      </button>

      <div v-show="showMenu && isLoggedIn" class="login-menu">
        <div class="user-info-menu">
          <div class="user-avatar-menu" v-if="userAvatar">
            <img :src="userAvatar" alt="Avatar" :key="userAvatar" />
          </div>
          <div class="user-avatar-menu placeholder" v-else>
            {{ displayName.charAt(0) }}
          </div>
          <div class="user-details">
            <p style="margin: 0 0 5px 0;">
              <strong>{{ displayName }}</strong>
            </p>
            <p style="margin: 0; font-size: 0.8rem; color: #f0f0f0;">
              {{ userData?.email || user?.email }}
            </p>
          </div>
        </div>
        
        <div class="menu-actions">
          <button @click="openProfile" class="menu-btn profile-btn">
            👤 Mi Perfil
          </button>
          <button @click="confirmLogout" class="menu-btn logout-btn">
            🚪 Cerrar sesión
          </button>
        </div>
      </div>
    </div>

    <!-- Modal de Perfil - SOLO se muestra si hay usuario autenticado -->
    <UserProfile 
      v-if="currentUser"
      :show="showProfileModal" 
      :user="currentUser"
      @close="showProfileModal = false"
      @profile-updated="handleProfileUpdated"
    />

    <!-- Modal de Confirmación de Cierre de Sesión -->
    <div v-if="showLogoutConfirm" class="logout-confirm-modal" @click.self="cancelLogout">
      <div class="logout-confirm-content">
        <div class="logout-icon">🚪</div>
        <h3>¿Estás seguro de cerrar sesión?</h3>
        <p>Serás redirigido a la página principal</p>
        <div class="logout-actions">
          <button @click="cancelLogout" class="btn-cancel">Cancelar</button>
          <button @click="handleLogout" class="btn-confirm">Sí, Cerrar Sesión</button>
        </div>
      </div>
    </div>
  </header>

  <nav class="main-nav animate__animated animate__fadeInDown">
    <RouterLink to="/" title="Geolocalización">Geolocalización</RouterLink>
    <RouterLink to="/calendarizacion" title="Calendarización">Calendarización</RouterLink>
    <RouterLink to="/recetas" title="Recetas">Recetas</RouterLink>
    <RouterLink to="/historia" title="Historia">Historia</RouterLink>
    <RouterLink to="/comunidad" title="Comunidad">Comunidad</RouterLink>
  </nav>
</template>

<style scoped>
header.glass-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: rgba(255, 255, 255, 0.15);
  backdrop-filter: blur(8px);
  padding: 15px 30px;
  border-radius: 0 0 15px 15px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  position: relative;
  z-index: 100;
  transition: box-shadow 0.3s ease, background 0.3s ease;
}

.logo-container {
  display: flex;
  align-items: center;
  gap: 15px;
  cursor: pointer;
}

.logo-container img.logo {
  height: 50px;
  transition: transform 0.3s ease, filter 0.3s ease;
}

.logo-container h1 {
  color: #222;
  font-weight: 700;
  font-size: 1.8rem;
  margin: 0;
  text-shadow: 1px 1px 2px rgba(255, 255, 255, 0.3);
  transition: transform 0.3s ease, filter 0.3s ease;
}

.logo-container:hover img.logo,
.logo-container:hover h1 {
  transform: scale(1.1);
  filter: drop-shadow(0 0 5px #ff6f61);
}

.auth-btn {
  background: #ff6f61;
  color: #fff;
  border: none;
  border-radius: 25px;
  padding: 10px 18px;
  font-weight: 600;
  cursor: pointer;
  box-shadow: 0 3px 6px rgba(255, 111, 97, 0.5);
  transition: background 0.3s ease;
  animation: pulseBtn 3s infinite;
  position: relative;
  display: flex;
  align-items: center;
  gap: 8px;
}

.auth-btn:hover {
  background: #e85b4e;
}

@keyframes pulseBtn {
  0%,
  100% {
    box-shadow: 0 3px 6px rgba(255, 111, 97, 0.5);
    background-color: #ff6f61;
  }
  50% {
    box-shadow: 0 6px 12px rgba(255, 111, 97, 0.8);
    background-color: #ff826f;
  }
}

.user-avatar-small {
  width: 30px;
  height: 30px;
  border-radius: 50%;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(255, 255, 255, 0.2);
}

.user-avatar-small img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.user-avatar-small.placeholder {
  background: rgba(255, 255, 255, 0.3);
  color: white;
  font-weight: bold;
  font-size: 0.9rem;
}

.login-container {
  position: relative;
}

.login-menu {
  position: absolute;
  top: 110%;
  right: 0;
  background: rgba(255, 111, 97, 0.95);
  padding: 15px;
  border-radius: 10px;
  color: white;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
  z-index: 150;
  transition: opacity 0.3s ease;
  min-width: 200px;
}

.user-info-menu {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 10px;
  padding-bottom: 10px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.3);
}

.user-avatar-menu {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(255, 255, 255, 0.2);
  border: 2px solid white;
}

.user-avatar-menu img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.user-avatar-menu.placeholder {
  background: rgba(255, 255, 255, 0.3);
  color: white;
  font-weight: bold;
  font-size: 1rem;
}

.user-details {
  flex: 1;
}

.menu-actions {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.menu-btn {
  background: rgba(255, 255, 255, 0.2);
  color: white;
  border: none;
  border-radius: 5px;
  padding: 8px 12px;
  cursor: pointer;
  text-align: left;
  transition: background 0.3s ease;
  font-size: 0.9rem;
}

.menu-btn:hover {
  background: rgba(255, 255, 255, 0.3);
}

.profile-btn {
  background: rgba(255, 255, 255, 0.15);
}

.logout-btn {
  background: rgba(220, 38, 38, 0.7);
}

.logout-btn:hover {
  background: rgba(220, 38, 38, 0.9);
}

.logout-confirm-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  z-index: 2000;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 20px;
  box-sizing: border-box;
}

.logout-confirm-content {
  background: white;
  border-radius: 16px;
  padding: 2rem;
  width: 90%;
  max-width: 400px;
  text-align: center;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
  animation: scaleIn 0.3s ease;
}

@keyframes scaleIn {
  from {
    transform: scale(0.8);
    opacity: 0;
  }
  to {
    transform: scale(1);
    opacity: 1;
  }
}

.logout-icon {
  font-size: 3rem;
  margin-bottom: 1rem;
}

.logout-confirm-content h3 {
  color: #ff6f61;
  margin-bottom: 0.5rem;
  font-size: 1.3rem;
}

.logout-confirm-content p {
  color: #666;
  margin-bottom: 2rem;
}

.logout-actions {
  display: flex;
  gap: 1rem;
  justify-content: center;
}

.btn-cancel {
  background: #6b7280;
  color: white;
  border: none;
  padding: 0.75rem 1.5rem;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1rem;
  transition: background 0.3s ease;
  flex: 1;
}

.btn-cancel:hover {
  background: #4b5563;
}

.btn-confirm {
  background: #dc2626;
  color: white;
  border: none;
  padding: 0.75rem 1.5rem;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1rem;
  transition: background 0.3s ease;
  flex: 1;
}

.btn-confirm:hover {
  background: #b91c1c;
}

nav.main-nav {
  display: flex;
  justify-content: center;
  background: rgba(255, 255, 255, 0.15);
  backdrop-filter: blur(8px);
  border-radius: 10px;
  margin: 15px auto 30px auto;
  max-width: 800px;
  padding: 10px 0;
  transition: background 0.3s ease;
}

nav.main-nav a {
  color: #222;
  text-decoration: none;
  margin: 0 25px;
  font-weight: 600;
  font-size: 1.1rem;
  position: relative;
  padding: 5px 0;
  transition: color 0.3s ease;
  text-shadow: 1px 1px 1px rgba(255, 255, 255, 0.2);
}

nav.main-nav a::after {
  content: "";
  position: absolute;
  left: 0;
  bottom: 0;
  width: 0;
  height: 3px;
  background: #ff6f61;
  transition: width 0.3s ease;
  border-radius: 3px;
}

nav.main-nav a:hover,
nav.main-nav a:focus {
  color: #ff6f61;
}

nav.main-nav a:hover::after,
nav.main-nav a:focus::after {
  width: 100%;
}

@media (max-width: 768px) {
  .login-menu {
    min-width: 180px;
  }
  
  .auth-btn {
    padding: 8px 15px;
    font-size: 0.9rem;
  }
  
  .user-avatar-small {
    width: 25px;
    height: 25px;
    font-size: 0.8rem;
  }
  
  .logout-actions {
    flex-direction: column;
  }
}
</style>