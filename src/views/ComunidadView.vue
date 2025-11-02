<template>
  <div class="comunidad-view">
    <!-- Navegación -->
    <nav class="nav-container">
      <button 
        class="nav-button" 
        :class="{ active: activeSection === 'comunidad' }"
        @click="activeSection = 'comunidad'"
      >
        <span class="nav-icon">💬</span>
        Comunidad
      </button>
    </nav>

    <div class="main-content">
      <!-- Sidebar de filtros -->
      <aside class="sidebar">
        <div class="sidebar-header">
          <h3 class="filter-title">Filtros</h3>
          <button class="clear-filters" @click="clearFilters" v-if="hasActiveFilters">
            Limpiar
          </button>
        </div>
        
        <div class="filter-group">
          <label class="filter-label">Valoración</label>
          <div class="stars-filter">
            <button 
              v-for="n in 5" 
              :key="n"
              class="star-filter-btn"
              :class="{ active: filterStars === (6 - n) }"
              @click="toggleStarFilter(6 - n)"
            >
              <span class="star-filter">{{ '★'.repeat(6 - n) }}</span>
              <span class="star-count">({{ getStarCount(6 - n) }})</span>
            </button>
          </div>
        </div>

        <div class="filter-group">
          <label class="filter-label">Ordenar por</label>
          <select v-model="sortBy" class="filter-select">
            <option value="recent">Más recientes</option>
            <option value="popular">Más populares</option>
            <option value="highest">Mejor valorados</option>
            <option value="lowest">Peor valorados</option>
          </select>
        </div>

        <div class="filter-group">
          <label class="filter-label">Buscar</label>
          <div class="search-container">
            <span class="search-icon">🔍</span>
            <input 
              v-model="searchTerm"
              type="text" 
              placeholder="Buscar comentarios..." 
              class="search-input"
            />
          </div>
        </div>

        <div class="stats-card">
          <div class="stat-item">
            <span class="stat-value">{{ comentarios.length }}</span>
            <span class="stat-label">Total valoraciones</span>
          </div>
          <div class="stat-item">
            <span class="stat-value">{{ averageRating.toFixed(1) }}</span>
            <span class="stat-label">Promedio</span>
          </div>
        </div>
      </aside>

      <!-- Área de contenido principal -->
      <main class="content-area">
        <!-- Header con información de usuario -->
        <header class="comunidad-header">
          <div class="header-content">
            <div class="header-text">
              <h1>Comunidad Ñam Tlatik</h1>
              <p>Comparte tu experiencia y descubre lo que otros opinan</p>
            </div>
            <div class="user-section">
              <div class="user-info-header">
                <div class="user-avatar-header" :class="{ anonimo: !currentUser.avatar }">
                  <img 
                    v-if="currentUser.avatar" 
                    :src="currentUser.avatar" 
                    class="user-avatar-img"
                    :key="currentUser.avatar"
                  />
                  <span v-else>👤</span>
                </div>
                <div class="user-details-header">
                  <span class="user-name-header">{{ currentUser.name }}</span>
                  <span class="user-status">{{ currentUser.status }}</span>
                </div>
              </div>
            </div>
          </div>
        </header>

        <!-- Formulario para nuevo comentario - SOLO PARA USUARIOS AUTENTICADOS -->
        <div class="new-comment-card" v-if="isUserAuthenticated">
          <div class="comment-card-header">
            <h3>Deja tu Valoración</h3>
            <div class="user-badge">
              <div class="user-avatar-small" :class="{ anonimo: !currentUser.avatar }">
                <img 
                  v-if="currentUser.avatar" 
                  :src="currentUser.avatar" 
                  class="user-avatar-img"
                  :key="currentUser.avatar"
                />
                <span v-else>👤</span>
              </div>
              <span>Publicando como <strong>{{ currentUser.name }}</strong></span>
            </div>
          </div>
          
          <form @submit.prevent="submitComment" class="comment-form">
            <div class="form-group">
              <label class="form-label">Calificación del Platillo</label>
              <div class="star-rating-container">
                <div class="star-rating">
                  <span 
                    v-for="star in 5" 
                    :key="star" 
                    class="star"
                    :class="{ 
                      filled: star <= (hoverRating || newComment.rating),
                      selected: star <= newComment.rating,
                      animated: star <= newComment.rating && starAnimation
                    }"
                    @click="rateStar(star)"
                    @mouseover="hoverRating = star"
                    @mouseleave="hoverRating = 0"
                  >
                    ★
                  </span>
                </div>
                <div class="rating-feedback">
                  <span v-if="newComment.rating === 0 && formSubmitted" class="error-message">
                    ⚠️ Por favor selecciona una calificación
                  </span>
                  <span v-else-if="newComment.rating > 0" class="rating-text">
                    {{ getRatingText(newComment.rating) }}
                  </span>
                  <span v-else class="rating-hint">
                    Selecciona de 1 a 5 estrellas
                  </span>
                </div>
              </div>
            </div>

            <div class="form-group">
              <label for="commentText" class="form-label">Tu Experiencia</label>
              <textarea 
                id="commentText"
                v-model="newComment.text" 
                placeholder="Comparte los detalles de tu experiencia con este platillo..."
                rows="4"
                class="comment-textarea"
                :class="{ error: !newComment.text && formSubmitted }"
              ></textarea>
              <div class="textarea-footer">
                <span class="error-message" v-if="!newComment.text && formSubmitted">
                  ⚠️ El comentario no puede estar vacío
                </span>
                <span class="char-count" :class="{ warning: newComment.text.length > 250 }">
                  {{ newComment.text.length }}/300
                </span>
              </div>
            </div>

            <div class="form-actions">
              <button 
                type="submit" 
                class="submit-btn primary"
                :disabled="isSubmitting"
              >
                <span v-if="isSubmitting" class="loading-spinner"></span>
                <span v-else class="btn-icon">📝</span>
                {{ isSubmitting ? 'Publicando...' : 'Publicar Valoración' }}
              </button>
            </div>
          </form>
        </div>

        <!-- Mensaje para usuarios NO autenticados - MANTIENE EL MISMO DISEÑO -->
        <div class="new-comment-card" v-else>
          <div class="comment-card-header">
            <h3>Deja tu Valoración</h3>
            <div class="user-badge">
              <div class="user-avatar-small anonimo">🔒</div>
              <span>Inicia sesión para comentar</span>
            </div>
          </div>
          
          <div class="auth-required-section">
            <div class="auth-icon">👤</div>
            <div class="auth-content">
              <h4>Acceso Requerido</h4>
              <p>Para compartir tu experiencia y dejar comentarios, necesitas iniciar sesión.</p>
              <div class="auth-buttons">
                <button @click="redirectToLogin" class="auth-btn login">
                  Iniciar Sesión
                </button>
                <button @click="redirectToLogin" class="auth-btn register">
                  Registrarse
                </button>
              </div>
            </div>
          </div>
        </div>

        <!-- Sección de comentarios -->
        <div class="comments-section">
          <div class="section-header">
            <h2>Valoraciones de la Comunidad</h2>
            <div class="section-controls">
              <div class="results-info">
                <span class="results-count">{{ filteredComments.length }} valoraciones</span>
                <span class="results-filter" v-if="hasActiveFilters">(filtradas)</span>
              </div>
              <div class="view-controls">
                <button class="view-btn" :class="{ active: viewMode === 'grid' }" @click="viewMode = 'grid'">
                  ▦ Grid
                </button>
                <button class="view-btn" :class="{ active: viewMode === 'list' }" @click="viewMode = 'list'">
                  ☰ Lista
                </button>
              </div>
            </div>
          </div>

          <div v-if="filteredComments.length === 0" class="no-comments">
            <div class="no-comments-icon">💬</div>
            <h3>No hay valoraciones</h3>
            <p>No se encontraron comentarios que coincidan con tus filtros.</p>
            <button class="clear-filters-btn" @click="clearFilters" v-if="hasActiveFilters">
              Limpiar filtros
            </button>
          </div>

          <div class="comments-container" :class="viewMode">
            <article 
              v-for="(comment, index) in paginatedComments" 
              :key="comment.id"
              class="comment-card"
              :class="{ 
                highlighted: highlightedIndex === index,
                'user-comment': comment.userId === currentUser.id 
              }"
            >
              <div class="comment-header">
                <div class="user-info">
                  <div class="user-avatar" :class="{ anonimo: !comment.avatar }">
                    <img 
                      v-if="comment.avatar" 
                      :src="comment.avatar" 
                      class="user-avatar-img"
                      :key="comment.avatar"
                    />
                    <span v-else>👤</span>
                  </div>
                  <div class="user-details">
                    <div class="user-name-row">
                      <h3 class="user-name">{{ comment.usuario }}</h3>
                      <span v-if="comment.userId === currentUser.id && isUserAuthenticated" class="your-comment-badge">
                        Tu valoración
                      </span>
                    </div>
                    <div class="comment-meta">
                      <div class="comment-star-rating">
                        <span 
                          v-for="star in 5" 
                          :key="star" 
                          class="comment-star" 
                          :class="{ filled: star <= comment.estrellas }"
                        >
                          ★
                        </span>
                      </div>
                      <span class="comment-date">{{ formatDate(comment.fecha) }}</span>
                    </div>
                  </div>
                </div>
                <div class="comment-actions-header">
                  <div class="rating-badge">
                    <span class="rating-number">{{ comment.estrellas }}</span>
                    <span class="rating-icon">★</span>
                  </div>
                  <!-- Menú de tres puntos para comentarios del usuario -->
                  <div class="comment-menu" v-if="comment.userId === currentUser.id && isUserAuthenticated">
                    <button class="menu-toggle" @click="toggleMenu(index)">
                      <span class="menu-dots">⋯</span>
                    </button>
                    <div class="menu-dropdown" v-if="activeMenuIndex === index">
                      <button class="menu-item delete" @click="deleteComment(comment.id)">
                        <span class="menu-icon">🗑️</span>
                        Eliminar comentario
                      </button>
                    </div>
                  </div>
                </div>
              </div>
              
              <div class="comment-content">
                <p class="comment-text">{{ comment.texto }}</p>
              </div>
              
              <div class="comment-response" v-if="comment.respuesta">
                <div class="response-header">
                  <div class="response-badge">
                    <div class="response-avatar anonimo">👤</div>
                    <span>Ñam Tlatik SV</span>
                  </div>
                  <span class="response-date">{{ formatDate(comment.fecha) }}</span>
                </div>
                <p class="response-text">{{ comment.respuesta }}</p>
              </div>

              <!-- Respuestas de usuarios -->
              <div class="user-responses" v-if="comment.respuestas && comment.respuestas.length > 0">
                <div 
                  v-for="respuesta in comment.respuestas" 
                  :key="respuesta.id"
                  class="user-response"
                  :class="{ 'own-response': respuesta.userId === currentUser.id }"
                >
                  <div class="response-header">
                    <div class="response-badge user-response-badge">
                      <div class="response-avatar" :class="{ anonimo: !respuesta.avatar }">
                        <img 
                          v-if="respuesta.avatar" 
                          :src="respuesta.avatar" 
                          class="user-avatar-img"
                          :key="respuesta.avatar"
                        />
                        <span v-else>👤</span>
                      </div>
                      <span>{{ respuesta.usuario }}</span>
                      <span v-if="respuesta.userId === currentUser.id && isUserAuthenticated" class="response-own-badge">Tú</span>
                    </div>
                    <span class="response-date">{{ formatDate(respuesta.fecha) }}</span>
                  </div>
                  <p class="response-text">{{ respuesta.texto }}</p>
                  
                  <div class="response-menu" v-if="respuesta.userId === currentUser.id && isUserAuthenticated">
                    <button class="menu-toggle small" @click="toggleResponseMenu(respuesta.id)">
                      <span class="menu-dots">⋯</span>
                    </button>
                    <div class="menu-dropdown" v-if="activeResponseMenu === respuesta.id">
                      <button class="menu-item delete" @click="deleteResponse(comment.id, respuesta.id)">
                        <span class="menu-icon">🗑️</span>
                        Eliminar respuesta
                      </button>
                    </div>
                  </div>
                </div>
              </div>

              <!-- Formulario para responder - SOLO PARA USUARIOS AUTENTICADOS -->
              <div class="reply-section" v-if="showReplyForm === comment.id && isUserAuthenticated">
                <div class="reply-form">
                  <div class="user-badge small">
                    <div class="user-avatar-small" :class="{ anonimo: !currentUser.avatar }">
                      <img 
                        v-if="currentUser.avatar" 
                        :src="currentUser.avatar" 
                        class="user-avatar-img"
                        :key="currentUser.avatar"
                      />
                      <span v-else>👤</span>
                    </div>
                    <span>Respondiendo como <strong>{{ currentUser.name }}</strong></span>
                  </div>
                  <textarea 
                    v-model="replyTexts[comment.id]"
                    placeholder="Escribe tu respuesta..."
                    rows="2"
                    class="reply-textarea"
                  ></textarea>
                  <div class="reply-actions">
                    <button class="reply-cancel" @click="cancelReply(comment.id)">Cancelar</button>
                    <button 
                      class="reply-submit" 
                      @click="submitReply(comment.id)"
                      :disabled="!replyTexts[comment.id]?.trim()"
                    >
                      Responder
                    </button>
                  </div>
                </div>
              </div>
              
              <div class="comment-actions">
                <button 
                  class="action-btn like-btn" 
                  @click="likeComment(comment.id, index)"
                  :disabled="!isUserAuthenticated"
                  :title="!isUserAuthenticated ? 'Inicia sesión para dar like' : 'Dar like'"
                >
                  <span class="action-icon" :class="{ 
                    liked: comment.userLikes && comment.userLikes.includes(currentUser.id),
                    disabled: !isUserAuthenticated
                  }">👍</span>
                  <span class="action-count">{{ comment.likes || 0 }}</span>
                </button>
                
                <button 
                  class="action-btn reply-btn" 
                  @click="toggleReplyForm(comment.id)"
                  :disabled="!isUserAuthenticated"
                  :title="!isUserAuthenticated ? 'Inicia sesión para responder' : 'Responder'"
                >
                  <span class="action-icon">↩️</span>
                  Responder
                </button>
                
                <button class="action-btn share-btn" @click="shareComment(comment)">
                  <span class="action-icon">↗️</span>
                  Compartir
                </button>
              </div>
            </article>
          </div>

          <!-- Paginación -->
          <div class="pagination-container" v-if="totalPages > 1">
            <div class="pagination-info">
              Página {{ currentPage }} de {{ totalPages }}
            </div>
            <div class="pagination">
              <button 
                class="pagination-btn prev"
                :disabled="currentPage === 1"
                @click="currentPage--"
              >
                ‹ Anterior
              </button>
              
              <button 
                v-for="page in visiblePages" 
                :key="page"
                @click="currentPage = page"
                :class="{ active: currentPage === page }"
                class="pagination-btn"
              >
                {{ page }}
              </button>
              
              <button 
                class="pagination-btn next"
                :disabled="currentPage === totalPages"
                @click="currentPage++"
              >
                Siguiente ›
              </button>
            </div>
          </div>
        </div>
      </main>
    </div>

    <!-- Notificación de éxito -->
    <div v-if="showSuccessNotification" class="success-notification">
      <div class="notification-content">
        <div class="notification-icon">✅</div>
        <div class="notification-text">
          <strong>¡Valoración publicada!</strong>
          <span>Tu comentario ya es visible en la comunidad</span>
        </div>
        <button class="notification-close" @click="showSuccessNotification = false">
          <span>×</span>
        </button>
      </div>
    </div>

    <!-- Notificación de eliminación -->
    <div v-if="showDeleteNotification" class="success-notification delete-notification">
      <div class="notification-content">
        <div class="notification-icon">🗑️</div>
        <div class="notification-text">
          <strong>¡Comentario eliminado!</strong>
          <span>Tu valoración ha sido eliminada correctamente</span>
        </div>
        <button class="notification-close" @click="showDeleteNotification = false">
          <span>×</span>
        </button>
      </div>
    </div>

    <!-- Notificación de compartir -->
    <div v-if="showShareNotification" class="success-notification share-notification">
      <div class="notification-content">
        <div class="notification-icon">↗️</div>
        <div class="notification-text">
          <strong>¡Enlace copiado!</strong>
          <span>Puedes compartir este comentario</span>
        </div>
        <button class="notification-close" @click="showShareNotification = false">
          <span>×</span>
        </button>
      </div>
    </div>
  </div>
</template>

<script>
// IMPORTACIONES DE FIREBASE
import { db, auth } from '../firebase.js';
import { 
  collection, 
  addDoc, 
  updateDoc, 
  deleteDoc, 
  doc, 
  onSnapshot,
  serverTimestamp,
  query,
  orderBy,
  arrayUnion,
  arrayRemove,
  getDoc
} from 'firebase/firestore';
import { 
  onAuthStateChanged
} from 'firebase/auth';
import { useRouter } from 'vue-router';

export default {
  name: 'ComunidadView',
  setup() {
    const router = useRouter();
    return { router };
  },
  data() {
    return {
      activeSection: 'comunidad',
      filterStars: '',
      searchTerm: '',
      sortBy: 'recent',
      currentPage: 1,
      commentsPerPage: 6,
      highlightedIndex: null,
      isSubmitting: false,
      formSubmitted: false,
      showSuccessNotification: false,
      showDeleteNotification: false,
      showShareNotification: false,
      activeMenuIndex: null,
      viewMode: 'grid',
      starAnimation: false,
      hoverRating: 0,
      
      // Variables para respuestas
      showReplyForm: null,
      replyTexts: {},
      activeResponseMenu: null,
      
      // Usuario actual
      currentUser: {
        id: null,
        name: "Conectando...",
        avatar: "",
        status: "Turista"
      },
      
      newComment: {
        rating: 0,
        text: ''
      },
      
      // Comentarios desde Firebase
      comentarios: [],
    }
  },
  computed: {
    // VERIFICACIÓN DE USUARIO AUTENTICADO
    isUserAuthenticated() {
      return this.currentUser.id && 
             !this.currentUser.id.includes('anonymous') && 
             this.currentUser.id.length > 10;
    },
    
    filteredComments() {
      let result = [...this.comentarios];
      
      // Filtrar por estrellas
      if (this.filterStars) {
        result = result.filter(c => c.estrellas == this.filterStars);
      }
      
      // Filtrar por término de búsqueda
      if (this.searchTerm) {
        const term = this.searchTerm.toLowerCase();
        result = result.filter(c => 
          c.usuario.toLowerCase().includes(term) || 
          c.texto.toLowerCase().includes(term) ||
          (c.respuesta && c.respuesta.toLowerCase().includes(term))
        );
      }
      
      // Ordenar
      switch(this.sortBy) {
        case 'recent':
          result.sort((a, b) => new Date(b.fecha) - new Date(a.fecha));
          break;
        case 'popular':
          result.sort((a, b) => b.likes - a.likes);
          break;
        case 'highest':
          result.sort((a, b) => b.estrellas - a.estrellas);
          break;
        case 'lowest':
          result.sort((a, b) => a.estrellas - b.estrellas);
          break;
      }
      
      return result;
    },
    totalPages() {
      return Math.ceil(this.filteredComments.length / this.commentsPerPage);
    },
    paginatedComments() {
      const start = (this.currentPage - 1) * this.commentsPerPage;
      return this.filteredComments.slice(start, start + this.commentsPerPage);
    },
    visiblePages() {
      const pages = [];
      const maxVisible = 5;
      let start = Math.max(1, this.currentPage - Math.floor(maxVisible / 2));
      let end = Math.min(this.totalPages, start + maxVisible - 1);
      
      if (end - start + 1 < maxVisible) {
        start = Math.max(1, end - maxVisible + 1);
      }
      
      for (let i = start; i <= end; i++) {
        pages.push(i);
      }
      
      return pages;
    },
    averageRating() {
      if (this.comentarios.length === 0) return 0;
      const sum = this.comentarios.reduce((acc, comment) => acc + comment.estrellas, 0);
      return sum / this.comentarios.length;
    },
    hasActiveFilters() {
      return this.filterStars !== '' || this.searchTerm !== '';
    }
  },
  methods: {
    // REDIRECCIONES PARA LOGIN/REGISTRO - AMBOS AL LOGIN
    redirectToLogin() {
      this.router.push('/auth/login');
    },

    // NUEVO MÉTODO: ESCUCHAR ACTUALIZACIONES DEL PERFIL
    handleProfileUpdated(updatedData) {
      console.log('🔄 ComunidadView: Perfil actualizado recibido', updatedData);
      
      if (updatedData.avatar !== undefined && updatedData.userId === this.currentUser.id) {
        console.log('🖼️ Actualizando avatar en ComunidadView...');
        
        // Actualizar avatar del usuario actual
        this.currentUser.avatar = updatedData.avatar;
        
        // Actualizar avatar en comentarios existentes del usuario
        this.comentarios.forEach(comment => {
          if (comment.userId === this.currentUser.id) {
            comment.avatar = updatedData.avatar;
          }
        });
        
        // Actualizar avatar en respuestas existentes del usuario
        this.comentarios.forEach(comment => {
          if (comment.respuestas && comment.respuestas.length > 0) {
            comment.respuestas.forEach(respuesta => {
              if (respuesta.userId === this.currentUser.id) {
                respuesta.avatar = updatedData.avatar;
              }
            });
          }
        });
        
        // Forzar re-render
        this.comentarios = [...this.comentarios];
      }
    },

    // INICIALIZAR FIREBASE
    async initializeFirebase() {
      try {
        // SOLO autenticación con usuarios reales, NO anónima
        onAuthStateChanged(auth, async (user) => {
          if (user) {
            this.currentUser.id = user.uid;
            this.currentUser.name = user.displayName || `Usuario_${user.uid.slice(-4)}`;
            this.currentUser.status = "Miembro";
            
            // CARGAR AVATAR DEL USUARIO ACTUAL
            await this.loadUserAvatar(user.uid);
          } else {
            // Usuario NO autenticado
            this.currentUser.id = null;
            this.currentUser.name = "Invitado";
            this.currentUser.status = "Turista";
            this.currentUser.avatar = "";
          }
        });

        // Cargar comentarios de Firebase
        this.loadCommentsFromFirebase();
        
        // ESCUCHAR EVENTOS GLOBALES DE ACTUALIZACIÓN DE PERFIL
        this.setupGlobalEventListener();
        
      } catch (error) {
        console.error('Error inicializando Firebase:', error);
      }
    },

    // NUEVO MÉTODO: CONFIGURAR EVENT LISTENER GLOBAL
    setupGlobalEventListener() {
      // Escuchar eventos personalizados para actualizaciones de perfil
      window.addEventListener('profile-updated', (event) => {
        if (event.detail && event.detail.userId === this.currentUser.id) {
          this.handleProfileUpdated(event.detail);
        }
      });
    },

    // CARGAR AVATAR DEL USUARIO
    async loadUserAvatar(userId) {
      try {
        const userDoc = await getDoc(doc(db, 'users', userId));
        if (userDoc.exists()) {
          const userData = userDoc.data();
          this.currentUser.avatar = userData.avatar || "";
        }
      } catch (error) {
        console.error('Error cargando avatar:', error);
      }
    },

    // CARGAR COMENTARIOS DE FIREBASE
    loadCommentsFromFirebase() {
      const commentsRef = collection(db, 'comentarios');
      const q = query(commentsRef, orderBy('fecha', 'desc'));
      
      onSnapshot(q, (snapshot) => {
        this.comentarios = snapshot.docs.map(doc => {
          const data = doc.data();
          return {
            id: doc.id,
            userId: data.userId,
            usuario: data.usuario,
            estrellas: data.estrellas,
            texto: data.texto,
            respuesta: data.respuesta || null,
            respuestas: data.respuestas || [],
            likes: data.likes || 0,
            userLikes: data.userLikes || [],
            fecha: data.fecha?.toDate() || new Date(),
            avatar: data.avatar || ""
          };
        });
      });
    },

    // MÉTODOS ORIGINALES
    rateStar(star) {
      if (!this.isUserAuthenticated) {
        alert('⚠️ Debes iniciar sesión para calificar');
        return;
      }
      this.newComment.rating = star;
      this.starAnimation = true;
      
      setTimeout(() => {
        this.starAnimation = false;
      }, 600);
    },
    
    shareComment(comment) {
      const commentUrl = `${window.location.origin}${window.location.pathname}#comment-${comment.id}`;
      
      if (navigator.share) {
        navigator.share({
          title: `Valoración de ${comment.usuario} en Ñam Tlatik`,
          text: `${comment.texto.substring(0, 100)}...`,
          url: commentUrl
        })
        .then(() => console.log('Compartido exitosamente'))
        .catch((error) => {
          this.copyToClipboard(commentUrl);
        });
      } else {
        this.copyToClipboard(commentUrl);
      }
    },
    
    copyToClipboard(text) {
      navigator.clipboard.writeText(text).then(() => {
        this.showShareNotification = true;
        setTimeout(() => {
          this.showShareNotification = false;
        }, 3000);
      });
    },
    
    toggleMenu(index) {
      this.activeMenuIndex = this.activeMenuIndex === index ? null : index;
    },
    
    // ELIMINAR COMENTARIO DE FIREBASE
    async deleteComment(commentId) {
      if (confirm('¿Estás seguro de que quieres eliminar este comentario?')) {
        try {
          await deleteDoc(doc(db, 'comentarios', commentId));
          
          this.activeMenuIndex = null;
          this.showDeleteNotification = true;
          setTimeout(() => {
            this.showDeleteNotification = false;
          }, 3000);
        } catch (error) {
          console.error('Error eliminando comentario:', error);
          alert('❌ Error al eliminar el comentario');
        }
      }
    },
    
    toggleStarFilter(rating) {
      this.filterStars = this.filterStars === rating ? '' : rating;
    },
    
    clearFilters() {
      this.filterStars = '';
      this.searchTerm = '';
      this.sortBy = 'recent';
      this.currentPage = 1;
    },
    
    getStarCount(rating) {
      return this.comentarios.filter(c => c.estrellas === rating).length;
    },
    
    validateComment() {
      if (!this.isUserAuthenticated) {
        alert('⚠️ Debes iniciar sesión para comentar');
        return false;
      }
      
      if (this.newComment.rating === 0) {
        alert("⚠️ Por favor selecciona una valoración con estrellas (mínimo 1 estrella)");
        return false;
      }
      
      if (!this.newComment.text.trim()) {
        alert("⚠️ El comentario no puede estar vacío");
        return false;
      }
      
      return true;
    },
    
    getRatingText(rating) {
      const texts = {
        1: "Malo - No lo recomiendo",
        2: "Regular - Hay que mejorar", 
        3: "Bueno - Cumple expectativas",
        4: "Muy Bueno - Lo recomiendo",
        5: "Excelente - Increíble"
      };
      return texts[rating] || "";
    },
    
    // LIKE CON CONTROL DE USUARIO
    async likeComment(commentId, index) {
      if (!this.isUserAuthenticated) {
        alert('⚠️ Debes iniciar sesión para dar like');
        return;
      }
      
      try {
        const commentRef = doc(db, 'comentarios', commentId);
        const comment = this.comentarios.find(c => c.id === commentId);
        
        const userLikes = comment.userLikes || [];
        const hasLiked = userLikes.includes(this.currentUser.id);
        
        if (hasLiked) {
          // Quitar like
          await updateDoc(commentRef, {
            likes: Math.max(0, (comment.likes || 0) - 1),
            userLikes: arrayRemove(this.currentUser.id)
          });
        } else {
          // Dar like
          await updateDoc(commentRef, {
            likes: (comment.likes || 0) + 1,
            userLikes: arrayUnion(this.currentUser.id)
          });
          
          // Animación
          this.highlightedIndex = index;
          setTimeout(() => {
            this.highlightedIndex = null;
          }, 1000);
        }
        
      } catch (error) {
        console.error('Error dando like:', error);
        alert('❌ Error al dar like');
      }
    },
    
    // GUARDAR COMENTARIO EN FIREBASE (CON AVATAR)
    async submitComment() {
      this.formSubmitted = true;
      
      if (!this.validateComment()) return;
      
      this.isSubmitting = true;
      
      try {
        // Guardar en Firebase CON AVATAR
        const commentData = {
          userId: this.currentUser.id,
          usuario: this.currentUser.name,
          estrellas: this.newComment.rating,
          texto: this.newComment.text.trim(),
          respuesta: null,
          respuestas: [],
          likes: 0,
          userLikes: [],
          avatar: this.currentUser.avatar || "", // INCLUIR AVATAR
          fecha: serverTimestamp()
        };
        
        await addDoc(collection(db, 'comentarios'), commentData);
        
        // Resetear formulario
        this.newComment = {
          rating: 0,
          text: ''
        };
        
        this.formSubmitted = false;
        
        // Mostrar notificación de éxito
        this.showSuccessNotification = true;
        setTimeout(() => {
          this.showSuccessNotification = false;
        }, 3000);
        
      } catch (error) {
        console.error('Error publicando comentario:', error);
        alert('❌ Error al publicar la valoración. Intenta nuevamente.');
      } finally {
        this.isSubmitting = false;
      }
    },
    
    // MÉTODOS PARA RESPONDER COMENTARIOS
    toggleReplyForm(commentId) {
      if (!this.isUserAuthenticated) {
        alert('⚠️ Debes iniciar sesión para responder comentarios');
        return;
      }
      
      if (this.showReplyForm === commentId) {
        this.showReplyForm = null;
        this.replyTexts[commentId] = '';
      } else {
        this.showReplyForm = commentId;
        this.replyTexts[commentId] = '';
      }
    },
    
    cancelReply(commentId) {
      this.showReplyForm = null;
      this.replyTexts[commentId] = '';
    },
    
    // GUARDAR RESPUESTA EN FIREBASE (CON AVATAR)
    async submitReply(commentId) {
      if (!this.isUserAuthenticated) {
        alert('⚠️ Debes iniciar sesión para responder');
        return;
      }
      
      const replyText = this.replyTexts[commentId]?.trim();
      if (!replyText) return;
      
      try {
        const commentRef = doc(db, 'comentarios', commentId);
        
        const newResponse = {
          id: Date.now().toString(),
          userId: this.currentUser.id,
          usuario: this.currentUser.name,
          texto: replyText,
          avatar: this.currentUser.avatar || "", // INCLUIR AVATAR
          fecha: new Date().toISOString()
        };
        
        // Agregar respuesta a Firebase
        await updateDoc(commentRef, {
          respuestas: arrayUnion(newResponse)
        });
        
        // Limpiar el formulario
        this.showReplyForm = null;
        this.replyTexts[commentId] = '';
        
        // Mostrar notificación de éxito
        this.showSuccessNotification = true;
        setTimeout(() => {
          this.showSuccessNotification = false;
        }, 3000);
        
      } catch (error) {
        console.error('Error enviando respuesta:', error);
        alert('❌ Error al enviar la respuesta');
      }
    },
    
    toggleResponseMenu(responseId) {
      this.activeResponseMenu = this.activeResponseMenu === responseId ? null : responseId;
    },
    
    // ELIMINAR RESPUESTA DE FIREBASE
    async deleteResponse(commentId, responseId) {
      if (confirm('¿Estás seguro de que quieres eliminar esta respuesta?')) {
        try {
          const commentRef = doc(db, 'comentarios', commentId);
          const comment = this.comentarios.find(c => c.id === commentId);
          
          const response = comment.respuestas.find(r => r.id === responseId);
          if (!response || response.userId !== this.currentUser.id) {
            alert('❌ No tienes permisos para eliminar esta respuesta');
            return;
          }
          
          // Eliminar respuesta de Firebase
          await updateDoc(commentRef, {
            respuestas: arrayRemove(response)
          });
          
          this.activeResponseMenu = null;
          
          // Mostrar notificación de eliminación
          this.showDeleteNotification = true;
          setTimeout(() => {
            this.showDeleteNotification = false;
          }, 3000);
          
        } catch (error) {
          console.error('Error eliminando respuesta:', error);
          alert('❌ Error al eliminar la respuesta');
        }
      }
    },
    
    formatDate(dateString) {
      const date = new Date(dateString);
      const now = new Date();
      const diffTime = Math.abs(now - date);
      const diffDays = Math.floor(diffTime / (1000 * 60 * 60 * 24));
      
      if (diffDays === 0) {
        return 'Hoy a las ' + date.toLocaleTimeString('es-ES', {
          hour: '2-digit',
          minute: '2-digit'
        });
      } else if (diffDays === 1) {
        return 'Ayer a las ' + date.toLocaleTimeString('es-ES', {
          hour: '2-digit',
          minute: '2-digit'
        });
      } else if (diffDays < 7) {
        return `Hace ${diffDays} días`;
      } else {
        return date.toLocaleDateString('es-ES', {
          year: 'numeric',
          month: 'long',
          day: 'numeric'
        });
      }
    }
  },
  mounted() {
    // INICIALIZAR FIREBASE
    this.initializeFirebase();
    
    // Cerrar menús al hacer click fuera
    document.addEventListener('click', (e) => {
      if (!e.target.closest('.comment-menu')) {
        this.activeMenuIndex = null;
      }
      if (!e.target.closest('.response-menu')) {
        this.activeResponseMenu = null;
      }
    });
  },
  watch: {
    filterStars() {
      this.currentPage = 1;
    },
    searchTerm() {
      this.currentPage = 1;
    },
    sortBy() {
      this.currentPage = 1;
    }
  }
}
</script>

<style scoped>
/* TODOS LOS ESTILOS ORIGINALES SE MANTIENEN EXACTAMENTE IGUAL */
/* ... (todo el CSS original permanece igual) ... */

.comunidad-view {
  font-family: 'Inter', 'Segoe UI', system-ui, sans-serif;
  color: #1a1a1a;
  max-width: 1400px;
  margin: 0 auto;
  padding: 0;
  background: #f8fafc;
  min-height: 100vh;
}

.nav-container {
  background: white;
  border-bottom: 1px solid #e2e8f0;
  padding: 0 2rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.nav-button {
  background: none;
  border: none;
  padding: 1.25rem 0;
  font-size: 1.1rem;
  font-weight: 600;
  color: #64748b;
  cursor: pointer;
  transition: all 0.3s ease;
  position: relative;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.nav-button:hover {
  color: #8B4513;
}

.nav-button.active {
  color: #8B4513;
}

.nav-button.active::after {
  content: '';
  position: absolute;
  bottom: -1px;
  left: 0;
  width: 100%;
  height: 3px;
  background-color: #8B4513;
  border-radius: 3px 3px 0 0;
}

.nav-icon {
  font-size: 1.2rem;
}

.main-content {
  display: flex;
  gap: 2rem;
  padding: 2rem;
  max-width: 1400px;
  margin: 0 auto;
}

.sidebar {
  flex: 0 0 320px;
  background: white;
  border-radius: 16px;
  padding: 1.5rem;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
  height: fit-content;
  position: sticky;
  top: 2rem;
}

.sidebar-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
}

.filter-title {
  color: #1e293b;
  font-weight: 700;
  font-size: 1.25rem;
  margin: 0;
}

.clear-filters {
  background: none;
  border: none;
  color: #64748b;
  font-size: 0.875rem;
  cursor: pointer;
  padding: 0.25rem 0.5rem;
  border-radius: 6px;
  transition: all 0.2s ease;
}

.clear-filters:hover {
  color: #8B4513;
  background: #F5F5DC;
}

.filter-group {
  margin-bottom: 1.5rem;
}

.filter-label {
  display: block;
  color: #374151;
  font-weight: 600;
  font-size: 0.875rem;
  margin-bottom: 0.5rem;
}

.stars-filter {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.star-filter-btn {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: none;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 0.75rem 1rem;
  cursor: pointer;
  transition: all 0.2s ease;
  width: 100%;
}

.star-filter-btn:hover {
  border-color: #8B4513;
  background: #F5F5DC;
}

.star-filter-btn.active {
  border-color: #8B4513;
  background: #F5F5DC;
}

.star-filter {
  color: #8B4513;
  font-size: 1.1rem;
}

.star-count {
  color: #64748b;
  font-size: 0.875rem;
}

.filter-select {
  width: 100%;
  padding: 0.75rem 1rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.875rem;
  background: white;
  cursor: pointer;
  transition: all 0.2s ease;
}

.filter-select:focus {
  outline: none;
  border-color: #8B4513;
  box-shadow: 0 0 0 3px rgba(139, 69, 19, 0.1);
}

.search-container {
  position: relative;
  display: flex;
  align-items: center;
}

.search-icon {
  position: absolute;
  left: 1rem;
  color: #64748b;
  font-size: 1rem;
}

.search-input {
  width: 100%;
  padding: 0.75rem 1rem 0.75rem 2.5rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.875rem;
  transition: all 0.2s ease;
}

.search-input:focus {
  outline: none;
  border-color: #8B4513;
  box-shadow: 0 0 0 3px rgba(139, 69, 19, 0.1);
}

.stats-card {
  background: linear-gradient(135deg, #F5F5DC, #DEB887);
  border: 1px solid #8B4513;
  border-radius: 12px;
  padding: 1.5rem;
  margin-top: 1rem;
}

.stat-item {
  text-align: center;
  padding: 0.5rem 0;
}

.stat-item:not(:last-child) {
  border-bottom: 1px solid #D2B48C;
}

.stat-value {
  display: block;
  font-size: 1.5rem;
  font-weight: 700;
  color: #8B4513;
}

.stat-label {
  font-size: 0.875rem;
  color: #654321;
}

.content-area {
  flex: 1;
  min-width: 0;
}

.comunidad-header {
  background: white;
  border-radius: 16px;
  padding: 2rem;
  margin-bottom: 2rem;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
}

.header-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 2rem;
}

.header-text h1 {
  font-size: 2rem;
  font-weight: 800;
  color: #1e293b;
  margin: 0 0 0.5rem 0;
  background: linear-gradient(135deg, #8B4513, #654321);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.header-text p {
  color: #64748b;
  margin: 0;
  font-size: 1.1rem;
}

.user-section {
  flex-shrink: 0;
}

.user-info-header {
  display: flex;
  align-items: center;
  gap: 1rem;
  background: #F5F5DC;
  padding: 1rem 1.5rem;
  border-radius: 12px;
  border: 1px solid #D2B48C;
}

.user-avatar-header {
  width: 3rem;
  height: 3rem;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.5rem;
  color: white;
  border: 2px solid #8B4513;
  overflow: hidden;
}

.user-avatar-header.anonimo {
  background: #8B4513;
}

.user-avatar-img {
  width: 100%;
  height: 100%;
  border-radius: 50%;
  object-fit: cover;
}

.user-details-header {
  display: flex;
  flex-direction: column;
}

.user-name-header {
  font-weight: 600;
  color: #1e293b;
}

.user-status {
  font-size: 0.875rem;
  color: #8B4513;
  font-weight: 500;
}

.new-comment-card {
  background: white;
  border-radius: 16px;
  padding: 2rem;
  margin-bottom: 2rem;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
  border: 1px solid #e2e8f0;
}

.comment-card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
  padding-bottom: 1rem;
  border-bottom: 1px solid #f1f5f9;
}

.comment-card-header h3 {
  color: #1e293b;
  font-size: 1.5rem;
  font-weight: 700;
  margin: 0;
}

.user-badge {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  background: #f8fafc;
  padding: 0.75rem 1rem;
  border-radius: 8px;
  border: 1px solid #e2e8f0;
}

.user-avatar-small {
  width: 2rem;
  height: 2rem;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1rem;
  color: white;
  border: 2px solid #8B4513;
  overflow: hidden;
}

.user-avatar-small.anonimo {
  background: #8B4513;
}

.star-rating-container {
  background: #f8fafc;
  padding: 1.5rem;
  border-radius: 12px;
  border: 1px solid #e2e8f0;
}

.star-rating {
  display: flex;
  justify-content: center;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.star {
  font-size: 3rem;
  color: #e2e8f0;
  cursor: pointer;
  transition: all 0.2s ease;
  text-shadow: 0 2px 4px rgba(0,0,0,0.1);
  position: relative;
}

.star:hover {
  transform: scale(1.1);
}

.star.filled {
  color: #FFD700;
}

.star.selected {
  color: #FFD700;
  text-shadow: 0 0 20px rgba(255, 215, 0, 0.5);
}

.star.animated {
  animation: starPulse 0.6s ease;
}

@keyframes starPulse {
  0% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.3);
    text-shadow: 0 0 30px rgba(255, 215, 0, 0.8);
  }
  100% {
    transform: scale(1);
  }
}

.rating-feedback {
  text-align: center;
  min-height: 1.5rem;
}

.rating-text {
  color: #8B4513;
  font-weight: 600;
  font-size: 1.1rem;
}

.rating-hint {
  color: #64748b;
  font-size: 0.875rem;
}

.form-group {
  margin-bottom: 1.5rem;
}

.form-label {
  display: block;
  color: #374151;
  font-weight: 600;
  font-size: 0.875rem;
  margin-bottom: 0.5rem;
}

.comment-textarea {
  width: 100%;
  padding: 1rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 1rem;
  font-family: inherit;
  resize: vertical;
  min-height: 120px;
  transition: all 0.2s ease;
  background: white;
}

.comment-textarea:focus {
  outline: none;
  border-color: #8B4513;
  box-shadow: 0 0 0 3px rgba(139, 69, 19, 0.1);
}

.comment-textarea.error {
  border-color: #ef4444;
  box-shadow: 0 0 0 3px rgba(239, 68, 68, 0.1);
}

.textarea-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 0.5rem;
}

.char-count {
  font-size: 0.75rem;
  color: #64748b;
}

.char-count.warning {
  color: #ef4444;
  font-weight: 600;
}

.error-message {
  color: #ef4444;
  font-size: 0.875rem;
  font-weight: 500;
}

.form-actions {
  display: flex;
  gap: 1rem;
  justify-content: flex-end;
}

.submit-btn {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 1rem 2rem;
  border: none;
  border-radius: 12px;
  font-weight: 600;
  font-size: 1rem;
  cursor: pointer;
  transition: all 0.3s ease;
}

.submit-btn.primary {
  background: linear-gradient(135deg, #8B4513, #654321);
  color: white;
}

.submit-btn.primary:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(139, 69, 19, 0.4);
}

.submit-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  transform: none !important;
}

.loading-spinner {
  width: 1.25rem;
  height: 1.25rem;
  border: 2px solid transparent;
  border-top: 2px solid white;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.comments-section {
  background: white;
  border-radius: 16px;
  padding: 2rem;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
}

.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2rem;
  padding-bottom: 1rem;
  border-bottom: 1px solid #f1f5f9;
}

.section-header h2 {
  color: #1e293b;
  font-size: 1.75rem;
  font-weight: 700;
  margin: 0;
}

.section-controls {
  display: flex;
  align-items: center;
  gap: 1.5rem;
}

.results-info {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.results-count {
  font-weight: 600;
  color: #1e293b;
}

.results-filter {
  color: #64748b;
  font-size: 0.875rem;
}

.view-controls {
  display: flex;
  background: #f8fafc;
  padding: 0.25rem;
  border-radius: 8px;
  border: 1px solid #e2e8f0;
}

.view-btn {
  background: none;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.875rem;
  color: #64748b;
  transition: all 0.2s ease;
}

.view-btn.active {
  background: white;
  color: #1e293b;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.no-comments {
  text-align: center;
  padding: 4rem 2rem;
  color: #64748b;
}

.no-comments-icon {
  font-size: 4rem;
  margin-bottom: 1rem;
  opacity: 0.5;
}

.no-comments h3 {
  color: #374151;
  margin: 0 0 1rem 0;
  font-size: 1.5rem;
}

.no-comments p {
  margin: 0 0 2rem 0;
  font-size: 1.1rem;
}

.clear-filters-btn {
  background: #8B4513;
  color: white;
  border: none;
  padding: 0.75rem 1.5rem;
  border-radius: 8px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
}

.clear-filters-btn:hover {
  background: #654321;
}

.comments-container.grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(400px, 1fr));
  gap: 1.5rem;
}

.comments-container.list {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.comment-card {
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 16px;
  padding: 1.5rem;
  transition: all 0.3s ease;
  position: relative;
}

.comment-card:hover {
  border-color: #8B4513;
  box-shadow: 0 8px 25px rgba(139, 69, 19, 0.15);
}

.comment-card.highlighted {
  border-color: #8B4513;
  background: #F5F5DC;
  animation: highlight 2s ease;
}

.comment-card.user-comment {
  border-color: #8B4513;
  background: linear-gradient(135deg, #F5F5DC, #DEB887);
}

@keyframes highlight {
  0% { background: #DEB887; }
  100% { background: #F5F5DC; }
}

.comment-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 1rem;
}

.user-info {
  display: flex;
  align-items: flex-start;
  gap: 1rem;
  flex: 1;
}

.user-avatar {
  width: 3rem;
  height: 3rem;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.25rem;
  color: white;
  flex-shrink: 0;
  border: 2px solid #8B4513;
  overflow: hidden;
}

.user-avatar.anonimo {
  background: #8B4513;
}

.user-details {
  flex: 1;
}

.user-name-row {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin-bottom: 0.5rem;
}

.user-name {
  font-weight: 600;
  color: #1e293b;
  margin: 0;
  font-size: 1.1rem;
}

.your-comment-badge {
  background: #8B4513;
  color: white;
  padding: 0.25rem 0.5rem;
  border-radius: 12px;
  font-size: 0.75rem;
  font-weight: 600;
}

.comment-meta {
  display: flex;
  align-items: center;
  gap: 1rem;
  flex-wrap: wrap;
}

.comment-star-rating {
  display: flex;
  gap: 0.125rem;
}

.comment-star {
  color: #FFD700;
  font-size: 1.1rem;
}

.comment-date {
  color: #64748b;
  font-size: 0.875rem;
}

.comment-actions-header {
  display: flex;
  align-items: flex-start;
  gap: 0.75rem;
}

.rating-badge {
  display: flex;
  align-items: center;
  gap: 0.25rem;
  background: #F5F5DC;
  padding: 0.5rem 0.75rem;
  border-radius: 20px;
  border: 1px solid #D2B48C;
  flex-shrink: 0;
}

.rating-number {
  font-weight: 700;
  color: #8B4513;
  font-size: 1.1rem;
}

.rating-icon {
  color: #FFD700;
  font-size: 0.875rem;
}

.comment-menu {
  position: relative;
  display: inline-block;
}

.menu-toggle {
  background: none;
  border: none;
  padding: 0.5rem;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s ease;
  color: #64748b;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 2rem;
  height: 2rem;
}

.menu-toggle:hover {
  background: #f1f5f9;
  color: #374151;
}

.menu-dots {
  font-size: 1.25rem;
  font-weight: bold;
  line-height: 1;
}

.menu-dropdown {
  position: absolute;
  top: 100%;
  right: 0;
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.15);
  z-index: 10;
  min-width: 180px;
  padding: 0.5rem;
  margin-top: 0.25rem;
  animation: dropdownSlide 0.2s ease;
}

@keyframes dropdownSlide {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.menu-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  width: 100%;
  padding: 0.75rem 1rem;
  border: none;
  background: none;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s ease;
  font-size: 0.875rem;
  color: #374151;
  text-align: left;
}

.menu-item:hover {
  background: #f8fafc;
}

.menu-item.delete:hover {
  background: #fef2f2;
  color: #dc2626;
}

.menu-icon {
  font-size: 1rem;
  width: 1rem;
  text-align: center;
}

.comment-content {
  margin-bottom: 1rem;
}

.comment-text {
  color: #374151;
  line-height: 1.6;
  margin: 0;
  font-size: 1rem;
}

.comment-response {
  background: #f8fafc;
  border-radius: 12px;
  padding: 1.25rem;
  margin: 1rem 0;
  border-left: 4px solid #8B4513;
}

.response-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.75rem;
}

.response-badge {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  background: #8B4513;
  color: white;
  padding: 0.375rem 0.75rem;
  border-radius: 20px;
  font-size: 0.875rem;
  font-weight: 600;
}

.response-avatar {
  width: 1.5rem;
  height: 1.5rem;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.875rem;
  border: 2px solid white;
  overflow: hidden;
}

.response-avatar.anonimo {
  background: white;
  color: #8B4513;
}

.response-date {
  color: #64748b;
  font-size: 0.75rem;
}

.response-text {
  color: #374151;
  line-height: 1.5;
  margin: 0;
  font-size: 0.95rem;
}

.comment-actions {
  display: flex;
  gap: 0.75rem;
  padding-top: 1rem;
  border-top: 1px solid #f1f5f9;
}

.action-btn {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  background: none;
  border: 1px solid #e2e8f0;
  color: #64748b;
  cursor: pointer;
  padding: 0.5rem 1rem;
  border-radius: 20px;
  transition: all 0.2s ease;
  font-size: 0.875rem;
  font-weight: 500;
  position: relative;
}

.action-btn:hover:not(:disabled) {
  background: #f8fafc;
  border-color: #8B4513;
  color: #654321;
}

.action-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.action-btn:disabled:hover {
  background: none;
  border-color: #e2e8f0;
  color: #64748b;
}

.action-btn.like-btn.pulse-effect .action-icon {
  animation: pulseLike 0.6s ease;
}

@keyframes pulseLike {
  0% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.3);
    color: #8B4513;
  }
  100% {
    transform: scale(1);
  }
}

.action-icon.liked {
  color: #8B4513;
  animation: bounce 0.6s ease;
}

.action-icon.disabled {
  opacity: 0.5;
}

@keyframes bounce {
  0%, 20%, 60%, 100% {
    transform: scale(1);
  }
  40% {
    transform: scale(1.2);
  }
  80% {
    transform: scale(1.1);
  }
}

.action-count {
  font-weight: 600;
  min-width: 1.5rem;
  text-align: center;
  transition: all 0.3s ease;
}

.action-btn.pulse-effect .action-count {
  animation: countBounce 0.6s ease;
}

@keyframes countBounce {
  0%, 20%, 60%, 100% {
    transform: scale(1);
  }
  40% {
    transform: scale(1.3);
    color: #8B4513;
  }
  80% {
    transform: scale(1.1);
  }
}

.pagination-container {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 2rem;
  padding-top: 1.5rem;
  border-top: 1px solid #e2e8f0;
}

.pagination-info {
  color: #64748b;
  font-size: 0.875rem;
}

.pagination {
  display: flex;
  gap: 0.5rem;
}

.pagination-btn {
  padding: 0.5rem 1rem;
  border: 1px solid #e2e8f0;
  background: white;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s ease;
  font-weight: 500;
  font-size: 0.875rem;
  color: #374151;
}

.pagination-btn:hover:not(:disabled) {
  border-color: #8B4513;
  color: #654321;
}

.pagination-btn.active {
  background: #8B4513;
  color: white;
  border-color: #8B4513;
}

.pagination-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.pagination-btn.prev,
.pagination-btn.next {
  font-weight: 600;
}

.success-notification {
  position: fixed;
  top: 2rem;
  right: 2rem;
  background: white;
  border-radius: 12px;
  padding: 1rem 1.5rem;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.15);
  border: 1px solid #10b981;
  z-index: 1001;
  animation: slideInRight 0.3s ease;
  max-width: 400px;
}

.delete-notification {
  border-color: #ef4444;
}

.share-notification {
  border-color: #3b82f6;
}

@keyframes slideInRight {
  from {
    transform: translateX(100%);
    opacity: 0;
  }
  to {
    transform: translateX(0);
    opacity: 1;
  }
}

.notification-content {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.notification-icon {
  font-size: 1.5rem;
  flex-shrink: 0;
}

.notification-text {
  display: flex;
  flex-direction: column;
  flex: 1;
}

.notification-text strong {
  color: #065f46;
  font-weight: 600;
}

.delete-notification .notification-text strong {
  color: #dc2626;
}

.share-notification .notification-text strong {
  color: #1d4ed8;
}

.notification-text span {
  color: #047857;
  font-size: 0.875rem;
}

.delete-notification .notification-text span {
  color: #b91c1c;
}

.share-notification .notification-text span {
  color: #1e40af;
}

.notification-close {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 0.25rem;
  border-radius: 4px;
  transition: all 0.2s ease;
  width: 1.5rem;
  height: 1.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
}

.notification-close:hover {
  background: #f1f5f9;
  color: #374151;
}

/* NUEVOS ESTILOS PARA LA SECCIÓN DE AUTENTICACIÓN - MANTIENEN EL DISEÑO ORIGINAL */
.auth-required-section {
  display: flex;
  align-items: center;
  gap: 1.5rem;
  padding: 2rem;
  background: #f8fafc;
  border-radius: 12px;
  border: 1px solid #e2e8f0;
}

.auth-icon {
  font-size: 3rem;
  opacity: 0.7;
}

.auth-content {
  flex: 1;
}

.auth-content h4 {
  color: #8B4513;
  font-size: 1.25rem;
  margin: 0 0 0.5rem 0;
  font-weight: 600;
}

.auth-content p {
  color: #64748b;
  margin: 0 0 1.5rem 0;
  font-size: 1rem;
}

.auth-buttons {
  display: flex;
  gap: 1rem;
}

.auth-btn {
  padding: 0.75rem 1.5rem;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
  font-size: 0.875rem;
}

.auth-btn.login {
  background: linear-gradient(135deg, #8B4513, #654321);
  color: white;
}

.auth-btn.login:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(139, 69, 19, 0.3);
}

.auth-btn.register {
  background: white;
  color: #8B4513;
  border: 1px solid #8B4513;
}

.auth-btn.register:hover {
  background: #F5F5DC;
}

/* EL RESTO DE LOS ESTILOS PARA RESPONDER COMENTARIOS SE MANTIENEN IGUAL */
.user-responses {
  margin: 1rem 0;
  padding-left: 1rem;
  border-left: 3px solid #e2e8f0;
}

.user-response {
  background: #f8fafc;
  border-radius: 12px;
  padding: 1rem;
  margin-bottom: 0.75rem;
  position: relative;
  border: 1px solid #e2e8f0;
}

.user-response.own-response {
  background: #F5F5DC;
  border-color: #D2B48C;
}

.user-response-badge {
  background: #64748b !important;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.response-own-badge {
  background: #8B4513;
  color: white;
  padding: 0.125rem 0.375rem;
  border-radius: 8px;
  font-size: 0.7rem;
  font-weight: 600;
}

.response-menu {
  position: absolute;
  top: 0.5rem;
  right: 0.5rem;
}

.menu-toggle.small {
  width: 1.5rem;
  height: 1.5rem;
  padding: 0.25rem;
}

.menu-toggle.small .menu-dots {
  font-size: 1rem;
}

.reply-section {
  margin-top: 1rem;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
}

.reply-form {
  background: #f8fafc;
  border-radius: 12px;
  padding: 1rem;
}

.user-badge.small {
  background: #e2e8f0;
  padding: 0.5rem 0.75rem;
  margin-bottom: 0.75rem;
}

.user-badge.small .user-avatar-small {
  width: 1.5rem;
  height: 1.5rem;
  font-size: 0.8rem;
}

.reply-textarea {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.875rem;
  font-family: inherit;
  resize: vertical;
  min-height: 60px;
  margin-bottom: 0.75rem;
  transition: all 0.2s ease;
}

.reply-textarea:focus {
  outline: none;
  border-color: #8B4513;
  box-shadow: 0 0 0 2px rgba(139, 69, 19, 0.1);
}

.reply-actions {
  display: flex;
  gap: 0.75rem;
  justify-content: flex-end;
}

.reply-cancel {
  background: #64748b;
  color: white;
  padding: 0.5rem 1rem;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 600;
  border: none;
  cursor: pointer;
  transition: all 0.2s ease;
}

.reply-cancel:hover {
  background: #475569;
}

.reply-submit {
  background: #8B4513;
  color: white;
  padding: 0.5rem 1rem;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 600;
  border: none;
  cursor: pointer;
  transition: all 0.2s ease;
}

.reply-submit:hover:not(:disabled) {
  background: #654321;
}

.reply-submit:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}
</style>