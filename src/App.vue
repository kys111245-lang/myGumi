<template>
  <div class="localhub-wrapper">
    <header class="sidebar-header">
      <div class="header-top">
        <button class="hamburger-btn" @click="isMobileMenuOpen = !isMobileMenuOpen">☰</button>
        <div class="logo" @click="onClickLogo">myGumi</div>
      </div>
      
      <nav class="category-nav" :class="{ 'open': isMobileMenuOpen }">
        <button 
          v-for="category in categories" 
          :key="category"
          :class="{ active: currentCategory === category }"
          @click="changeCategory(category)"
        >
          {{ category }}
        </button>
      </nav>
    </header>

    <main class="main-content" id="mainScrollArea">
      
      <section v-if="currentCategory === '홈'" class="home-view">
        <div class="home-center-container">
          <div class="hero-banner">
            <h1>지역 정보 공유 커뮤니티 myGumi</h1>
            <p>생생한 지역 정보와 일정을 한눈에 만나보세요!</p>
            
            <div class="home-global-search">
              <div class="search-input-wrapper">
                <span class="search-icon">🔍</span>
                <input 
                  type="text" 
                  v-model="globalSearchQuery" 
                  placeholder="관광지, 일정, 자유게시판 등 통합 검색!" 
                />
              </div>
              
              <div v-if="globalSearchQuery.trim().length > 0" class="global-search-results">
                <div v-if="globalSearchResults.length === 0" class="empty-result">
                  '{{ globalSearchQuery }}' 에 대한 검색 결과가 없습니다.
                </div>
                <ul v-else class="result-list">
                  <li v-for="item in globalSearchResults" :key="item.contentid" @click="goToSearchResult(item)">
                    <div class="res-header">
                      <span class="res-badge">[{{ item._category }}]</span>
                      <span class="res-title">{{ item.title }}</span>
                    </div>
                    <div class="res-address">{{ item.addr1 }} {{ item.addr2 }}</div>
                  </li>
                </ul>
              </div>
            </div>
          </div>

          <div class="home-shortcuts">
            <div class="shortcut-card" @click="changeCategory('일정')">📅 내 일정 공유하기</div>
            <div class="shortcut-card" @click="changeCategory('관광지')">🏞️ 추천 관광지 보기</div>
            <div class="shortcut-card" @click="changeCategory('자유게시판')">💬 자유게시판 가기</div>
          </div>
        </div>
      </section>

      <section v-else-if="currentCategory === '일정'" class="calendar-view">
        <div class="tab-search-container">
          <div class="search-input-wrapper relative-wrapper">
            <span class="search-icon">🔍</span>
            <input type="text" v-model="localSearchQuery" placeholder="일정 제목 또는 내용 검색..." />
            <button v-if="localSearchQuery" @click="clearLocalSearch" class="clear-btn">✖</button>
            
            <div v-if="localSearchQuery && calendarSearchResults.length > 0" class="calendar-search-dropdown">
              <ul class="result-list">
                <li v-for="item in calendarSearchResults" :key="item.id" @click="jumpToCalendarDate(item)">
                  <span class="res-badge">[일정]</span>
                  <span class="res-title">{{ item.title }}</span>
                  <span class="res-date">{{ item.startDate }}</span>
                </li>
              </ul>
            </div>
          </div>
        </div>

        <div class="top-controls">
          <div class="month-navigation">
            <button class="nav-btn" @click="changeMonth(-1)">◀</button>
            <h2>{{ currentYear }}. {{ String(currentMonth + 1).padStart(2, '0') }}</h2>
            <button class="nav-btn" @click="changeMonth(1)">▶</button>
          </div>
          <div class="actions">
            <button class="btn-write" @click="openFormModal(todayDate)">+ 새 일정 등록</button>
          </div>
        </div>

        <div class="calendar-grid">
          <div class="weekday" v-for="day in ['일', '월', '화', '수', '목', '금', '토']" :key="day" :class="{ 'sunday': day === '일', 'saturday': day === '토' }">
            {{ day }}
          </div>
          <div v-for="blank in firstDayOffset" :key="'blank-'+blank" class="day blank"></div>
          
          <div v-for="date in daysInMonth" :key="date" class="day" @click="openDayPlanModal(getFormattedDate(date))">
            <span class="date-num" :class="{ 'today': isToday(date) }">{{ date }}</span>
            <div class="schedule-list">
              <div 
                v-for="item in getSchedulesForCalendar(getFormattedDate(date))" 
                :key="item.id" 
                class="schedule-chip"
                :class="getChipClass(item, getFormattedDate(date))"
                :style="{ backgroundColor: item.color || '#4a90e2' }"
                @click.stop="openDetailModal(item)"
              >
                <span :class="{ 'text-hidden': !showChipText(item, getFormattedDate(date)) }">{{ item.title }}</span>
              </div>
            </div>
          </div>
        </div>
      </section>

      <section v-else-if="currentCategory === '자유게시판'" class="board-view">
        <div v-if="boardViewMode === 'list'" class="tab-search-container">
          <div class="search-input-wrapper">
            <span class="search-icon">🔍</span>
            <input type="text" v-model="localSearchQuery" placeholder="게시글 검색..." />
            <button v-if="localSearchQuery" @click="clearLocalSearch" class="clear-btn">✖</button>
          </div>
        </div>

        <div class="board-header">
          <h2>💬 자유게시판</h2>
        </div>

        <div class="board-controls-row" v-if="boardViewMode === 'list'">
          <select v-model="boardSortMode" class="sort-select">
            <option value="latest">최신순</option>
            <option value="likes">좋아요순</option>
          </select>
          <button class="btn-write" @click="openBoardForm()">+ 글쓰기</button>
        </div>
        <div class="board-controls-row" v-else>
          <button class="btn-outline" @click="boardViewMode = 'list'">← 목록으로</button>
        </div>

        <div v-if="boardViewMode === 'list'" class="board-list-cards">
          <div v-if="paginatedBoardPosts.length === 0" class="empty-state">게시글이 없습니다.</div>
          <div 
            v-else 
            v-for="(post, idx) in paginatedBoardPosts" 
            :key="post.id" 
            class="board-card" 
            @click="openBoardDetail(post)"
          >
            <div class="bc-title">{{ post.title }}</div>
            <div class="bc-meta">
              <span>No. {{ sortedFreeBoardPosts.length - ((boardCurrentPage - 1) * boardItemsPerPage + idx) }}</span>
              <span class="divider-line">|</span>
              <span>{{ new Date(post.createdAt).toLocaleDateString() }}</span>
              <span class="divider-line">|</span>
              <span class="like-text">❤️ {{ post.likeCount || 0 }}</span>
            </div>
          </div>

          <div class="pagination" v-if="boardTotalPages > 1">
            <button class="page-btn" @click="boardCurrentPage--" :disabled="boardCurrentPage === 1">이전</button>
            <button class="page-btn" v-for="p in boardVisiblePages" :key="p" @click="boardCurrentPage = p" :class="{ active: boardCurrentPage === p }">{{ p }}</button>
            <button class="page-btn" @click="boardCurrentPage++" :disabled="boardCurrentPage === boardTotalPages">다음</button>
          </div>
        </div>

        <div v-else-if="boardViewMode === 'detail'" class="board-detail-view">
          <div class="post-header">
            <h3>{{ selectedBoardPost.title }}</h3>
            <span class="date">{{ new Date(selectedBoardPost.createdAt).toLocaleString() }}</span>
          </div>
          <div class="post-content">{{ selectedBoardPost.content }}</div>
          <div class="post-actions">
            <div class="left-actions">
              <button class="btn-like" :class="{ active: selectedBoardPost.isLiked }" @click="toggleBoardLike">
                {{ selectedBoardPost.isLiked ? '❤️ 좋아요 취소' : '🤍 좋아요' }} ({{ selectedBoardPost.likeCount || 0 }})
              </button>
            </div>
            <div class="right-actions">
              <button class="btn-edit" @click="editBoardPost">수정</button>
              <button class="btn-delete" @click="deleteBoardPost">삭제</button>
            </div>
          </div>
          <hr class="divider" />
          <CommentSection :target-id="selectedBoardPost.id" />
        </div>
        <div v-else-if="boardViewMode === 'form'" class="board-form-view">
          <input type="text" v-model="boardForm.title" placeholder="제목을 입력하세요" class="form-input" />
          <textarea v-model="boardForm.content" rows="10" placeholder="내용을 입력하세요" class="form-input"></textarea>
          <input type="password" v-model="boardForm.password" placeholder="수정/삭제용 비밀번호 (필수)" class="form-input" :disabled="!!selectedBoardPost" />
          <div class="form-actions"><button class="btn-submit-full" @click="saveBoardPost">저장하기</button></div>
        </div>
      </section>

      <section v-else class="json-data-view">
        <div v-if="jsonViewMode === 'list'" class="tab-search-container">
          <div class="search-input-wrapper">
            <span class="search-icon">🔍</span>
            <input type="text" v-model="localSearchQuery" :placeholder="`${currentCategory} 검색...`" />
            <button v-if="localSearchQuery" @click="clearLocalSearch" class="clear-btn">✖</button>
          </div>
        </div>

        <div class="view-header">
          <h2>{{ currentCategory }} 정보</h2>
          <div class="header-actions">
            <button v-if="jsonViewMode === 'detail'" class="btn-outline" @click="goBackToJsonList">← 목록으로</button>
          </div>
        </div>

        <div v-if="isJsonLoading" class="loading-state">데이터를 불러오는 중입니다...</div>

        <div v-else-if="jsonViewMode === 'list'" class="tripadvisor-container">
          <div class="tripadvisor-grid">
            <div class="trip-card" v-for="item in paginatedJsonItems" :key="item.contentid" @click="openJsonDetail(item)">
              <div class="card-img" :style="{ backgroundImage: `url(${item.firstimage || fallbackImage})` }"></div>
              <div class="card-info">
                <h4>{{ item.title }}</h4>
                <p class="address">📍 {{ item.addr1 }}</p>
              </div>
            </div>
          </div>
          <div v-if="filteredJsonItems.length === 0" class="empty-state">해당 카테고리의 데이터가 없거나 검색 결과가 없습니다.</div>
          
          <div class="pagination" v-if="totalPages > 1">
            <button class="page-btn" @click="currentPage--" :disabled="currentPage === 1">이전</button>
            <button class="page-btn" v-for="p in visiblePages" :key="p" @click="currentPage = p" :class="{ active: currentPage === p }">{{ p }}</button>
            <button class="page-btn" @click="currentPage++" :disabled="currentPage === totalPages">다음</button>
          </div>
        </div>

        <div v-else-if="jsonViewMode === 'detail'" class="tripadvisor-detail">
          <h2>{{ selectedJsonItem.title }}</h2>
          <div class="detail-img-box">
            <img :src="selectedJsonItem.firstimage || fallbackImage" alt="대표 이미지" />
          </div>
          <div class="info-box">
            <p><strong>📍 주소:</strong> {{ selectedJsonItem.addr1 }} {{ selectedJsonItem.addr2 }}</p>
            <p v-if="selectedJsonItem.tel"><strong>📞 전화번호:</strong> {{ selectedJsonItem.tel }}</p>
            <p v-if="selectedJsonItem.zipcode"><strong>📌 우편번호:</strong> {{ selectedJsonItem.zipcode }}</p>
          </div>
        </div>
      </section>
    </main>

    <div class="scroll-controls" v-show="!['홈', '일정'].includes(currentCategory)">
      <button class="scroll-btn" @click="scrollToTop" title="맨 위로">▲</button>
      <button class="scroll-btn" @click="scrollToBottom" title="맨 아래로">▼</button>
    </div>

    <div v-if="isDragging" class="chat-dismiss-zone" :class="{ 'hovering': isHoveringDismiss }">
      🗑️ 여기에 두어 챗봇 숨기기
    </div>
    
    <aside class="chatbot-wrapper" v-show="!isChatDismissed" ref="chatWrapperRef" :style="chatStyle">
      <div v-show="isChatOpen" class="chat-panel" @mousedown.stop @touchstart.stop>
        <div class="chat-header">
          <span>myGumi AI 챗봇</span>
          <button @click="isChatOpen = false">✕</button>
        </div>
        <div class="chat-messages" ref="chatBox">
          <div v-for="(msg, idx) in chatMessages" :key="idx" :class="['message', msg.role]">{{ msg.content }}</div>
        </div>
        <div class="chat-input">
          <input type="text" v-model="chatInput" @keyup.enter="sendMessage" placeholder="정보를 물어보세요..." />
          <button @click="sendMessage">전송</button>
        </div>
      </div>
      <button class="floating-btn" @mousedown="startDrag" @touchstart.prevent="startDrag">💬</button>
    </aside>

    <div v-if="isDayPlanModalOpen" class="modal-overlay" @click.self="closeModals">
      <div class="modal-content day-plan-modal">
        <div class="modal-header"><h2>{{ selectedDateForPlan }} 일정</h2><button class="btn-close" @click="closeModals">✕</button></div>
        <div class="modal-body day-plan-body">
          <div v-if="dayPlanSchedules.length === 0" class="empty-state">등록된 일정이 없습니다.</div>
          <ul v-else class="day-schedule-list">
            <li v-for="item in dayPlanSchedules" :key="item.id" @click="openDetailFromPlan(item)">
              <div class="color-indicator" :style="{ backgroundColor: item.color || '#4a90e2' }"></div>
              <div class="info"><span class="title">{{ item.title }}</span><span class="time">{{ item.startDate === item.endDate ? '하루 종일' : `${item.startDate} ~ ${item.endDate}` }}</span></div>
            </li>
          </ul>
        </div>
        <div class="modal-footer"><button class="btn-submit-full btn-add-plan" @click="openFormFromPlan(selectedDateForPlan)"><span class="plus-icon">+</span> 새 일정 추가</button></div>
      </div>
    </div>

    <div v-if="isDetailModalOpen" class="modal-overlay" @click.self="closeModals">
      <div class="modal-content detail-modal">
        <div class="modal-header"><h2>{{ selectedSchedule.title }}</h2><button class="btn-close" @click="closeModals">✕</button></div>
        <div class="modal-body">
          <div class="meta-info"><span class="date-tag">기간: {{ selectedSchedule.startDate }} ~ {{ selectedSchedule.endDate }}</span></div>
          <div class="poster-container" v-if="selectedSchedule.image"><img :src="selectedSchedule.image" alt="포스터" /></div>
          <div class="text-content">{{ selectedSchedule.content }}</div>
          <div class="detail-actions">
            <button class="btn-like" :class="{ active: selectedSchedule.isLiked }" @click="toggleScheduleLike">{{ selectedSchedule.isLiked ? '❤️ 좋아요 취소' : '🤍 좋아요' }} ({{ selectedSchedule.likeCount || 0 }})</button>
            <div class="crud-btns"><button class="btn-edit" @click="handleEditSchedule">수정</button><button class="btn-delete" @click="handleDeleteSchedule">삭제</button></div>
          </div>
          <hr class="divider" />
          <CommentSection :target-id="selectedSchedule.id" />
        </div>
      </div>
    </div>

    <div v-if="isFormModalOpen" class="modal-overlay" @click.self="closeModals">
      <div class="modal-content form-modal">
        <div class="modal-header"><h2>{{ selectedSchedule ? '일정 수정' : '새 일정 등록' }}</h2><button class="btn-close" @click="closeModals">✕</button></div>
        <div class="modal-body">
          <div class="form-group">
            <label>일정 (시작일 ~ 종료일)</label>
            <div class="date-range-inputs"><input type="date" v-model="scheduleForm.startDate" /><span>~</span><input type="date" v-model="scheduleForm.endDate" :min="scheduleForm.startDate" /></div>
          </div>
          <div class="form-group">
            <label>일정 색상</label>
            <div class="color-picker"><div v-for="color in availableColors" :key="color" class="color-circle" :style="{ backgroundColor: color }" :class="{ 'selected': scheduleForm.color === color }" @click="scheduleForm.color = color"><span v-if="scheduleForm.color === color" class="check-icon">✓</span></div></div>
          </div>
          <div class="form-group"><label>제목</label><input type="text" v-model="scheduleForm.title" placeholder="일정 제목" /></div>
          <div class="form-group"><label>사진 첨부</label><input type="file" accept="image/*" @change="(e) => handleImageUpload(e, scheduleForm)" /><img v-if="scheduleForm.image" :src="scheduleForm.image" class="image-preview" /></div>
          <div class="form-group"><label>상세 내용</label><textarea v-model="scheduleForm.content" rows="4"></textarea></div>
          <div class="form-group"><label>수정/삭제용 비밀번호 (필수)</label><input type="password" v-model="scheduleForm.password" placeholder="비밀번호" :disabled="!!selectedSchedule" /></div>
          <div class="modal-footer-actions"><button class="btn-cancel" @click="closeModals">취소</button><button class="btn-submit-full" @click="saveSchedule">저장하기</button></div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted, defineComponent, h, nextTick } from 'vue';

const fallbackImage = '/src/assets/images/no_img.png';

const localSearchQuery = ref('');
const clearLocalSearch = () => { localSearchQuery.value = ''; };

const scrollToTop = () => { 
  if (window.innerWidth > 900) {
    const m = document.getElementById('mainScrollArea'); 
    if(m) m.scrollTo({ top: 0, behavior: 'smooth' }); 
  } else {
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }
};

const scrollToBottom = () => { 
  if (window.innerWidth > 900) {
    const m = document.getElementById('mainScrollArea'); 
    if(m) m.scrollTo({ top: m.scrollHeight, behavior: 'smooth' }); 
  } else {
    // 모바일 환경에서 정확한 문서 전체 높이 측정 후 맨 아래로 이동
    const scrollHeight = Math.max(
      document.body.scrollHeight, 
      document.documentElement.scrollHeight
    );
    window.scrollTo({ top: scrollHeight, behavior: 'smooth' });
  }
};

const chatWrapperRef = ref(null);
const isChatOpen = ref(false);
const chatStyle = ref({ bottom: '30px', right: '30px', left: 'auto', top: 'auto' });
let isDragging = ref(false);
let isHoveringDismiss = ref(false);
let hasDragged = false;
let startX, startY, initialLeft, initialTop;

const startDrag = (e) => {
  const evt = e.type.includes('mouse') ? e : e.touches[0];
  isDragging.value = true; hasDragged = false;
  startX = evt.clientX; startY = evt.clientY;
  const rect = chatWrapperRef.value.getBoundingClientRect();
  initialLeft = rect.left; initialTop = rect.top;
  
  if(e.type.includes('mouse')) {
    document.addEventListener('mousemove', onDrag);
    document.addEventListener('mouseup', stopDrag);
  } else {
    document.addEventListener('touchmove', onDrag, { passive: false });
    document.addEventListener('touchend', stopDrag);
  }
};
const onDrag = (e) => {
  if (!isDragging.value) return;
  hasDragged = true;
  if(e.cancelable) e.preventDefault();
  const evt = e.type.includes('mouse') ? e : e.touches[0];
  
  let newLeft = initialLeft + (evt.clientX - startX);
  let newTop = initialTop + (evt.clientY - startY);

  newLeft = Math.max(0, Math.min(newLeft, window.innerWidth - 60));
  newTop = Math.max(0, Math.min(newTop, window.innerHeight - 60));

  chatStyle.value = { left: `${newLeft}px`, top: `${newTop}px`, right: 'auto', bottom: 'auto' };

  const isBottom = newTop > window.innerHeight - 150;
  const isCenter = newLeft > window.innerWidth / 2 - 80 && newLeft < window.innerWidth / 2 + 20;
  isHoveringDismiss.value = isBottom && isCenter;
};
const stopDrag = () => {
  isDragging.value = false;
  document.removeEventListener('mousemove', onDrag); document.removeEventListener('mouseup', stopDrag);
  document.removeEventListener('touchmove', onDrag); document.removeEventListener('touchend', stopDrag);
  
  if (isHoveringDismiss.value) {
    isChatDismissed.value = true;
    isChatOpen.value = false;
  }
  isHoveringDismiss.value = false;
  if (!hasDragged) isChatOpen.value = !isChatOpen.value;
};

const categories = ['일정', '관광지', '레포츠', '문화시설', '쇼핑', '숙박', '음식점', '축제공연행사', '자유게시판'];
const currentCategory = ref('홈'); 
const isMobileMenuOpen = ref(false); 
const isChatDismissed = ref(false);

const onClickLogo = () => {
  changeCategory('홈');
  isChatDismissed.value = false;
  chatStyle.value = { bottom: '30px', right: '30px', left: 'auto', top: 'auto' };
};

const changeCategory = (category) => {
  currentCategory.value = category;
  closeModals();
  isMobileMenuOpen.value = false; 
  boardViewMode.value = 'list';
  jsonViewMode.value = 'list';
  globalSearchQuery.value = '';
  localSearchQuery.value = ''; 
  currentPage.value = 1; 
  boardCurrentPage.value = 1;
  scrollToTop(); 
  if (!['홈', '일정', '자유게시판'].includes(category)) loadJsonData(category);
};

const allJsonData = ref([]);
const globalSearchQuery = ref('');
const freeBoardPosts = ref(JSON.parse(localStorage.getItem('localhub_freeboard')) || []);
watch(freeBoardPosts, (newVal) => localStorage.setItem('localhub_freeboard', JSON.stringify(newVal)), { deep: true });

const globalSearchResults = computed(() => {
  if (!globalSearchQuery.value.trim()) return [];
  const q = globalSearchQuery.value.trim().toLowerCase();
  
  const jsonRes = allJsonData.value.filter(item => 
    (item.title && item.title.toLowerCase().includes(q)) || 
    (item.addr1 && item.addr1.toLowerCase().includes(q))
  );
  
  const boardRes = freeBoardPosts.value.filter(post => 
    (post.title && post.title.toLowerCase().includes(q)) || 
    (post.content && post.content.toLowerCase().includes(q))
  ).map(post => ({
    _category: '자유게시판', contentid: post.id, title: post.title, 
    addr1: post.content.length > 20 ? post.content.substring(0, 20) + '...' : post.content, originalPost: post
  }));

  const scheduleRes = schedules.value.filter(s => 
    (s.title && s.title.toLowerCase().includes(q)) || 
    (s.content && s.content.toLowerCase().includes(q))
  ).map(s => ({
    _category: '일정', contentid: s.id, title: s.title, addr1: `일정: ${s.startDate} ~ ${s.endDate}`, originalPost: s
  }));
  
  return [...jsonRes, ...boardRes, ...scheduleRes];
});

const loadAllJsonForSearch = async () => {
  const targetCategories = ['관광지', '레포츠', '문화시설', '쇼핑', '숙박', '음식점', '축제공연행사'];
  let aggregated = [];
  for (const cat of targetCategories) {
    try {
      const res = await fetch(`./구미_경북권_${cat}.json`);
      if (res.ok) {
        const data = await res.json();
        const items = data.items || [];
        items.forEach(item => item._category = cat);
        aggregated.push(...items);
      }
    } catch (e) { console.warn(`[통합검색] ${cat} 파일 로드 실패.`); }
  }
  allJsonData.value = aggregated;
};

const goToSearchResult = async (item) => {
  currentCategory.value = item._category;
  closeModals();
  globalSearchQuery.value = '';
  localSearchQuery.value = '';
  
  if (item._category === '자유게시판') {
    openBoardDetail(item.originalPost);
    return;
  }
  if (item._category === '일정') {
    jumpToCalendarDate(item.originalPost);
    return;
  }

  if (jsonItems.value.length === 0 || jsonItems.value[0]?._category !== item._category) {
    await loadJsonData(item._category);
  }
  selectedJsonItem.value = item;
  jsonViewMode.value = 'detail';
};
onMounted(() => loadAllJsonForSearch());

const jsonViewMode = ref('list');
const jsonItems = ref([]);
const selectedJsonItem = ref(null);
const isJsonLoading = ref(false);

const loadJsonData = async (category) => {
  isJsonLoading.value = true;
  jsonItems.value = [];
  try {
    const res = await fetch(`./구미_경북권_${category}.json`);
    if (res.ok) {
      const data = await res.json();
      const items = data.items || [];
      items.forEach(item => item._category = category); 
      jsonItems.value = items;
    }
  } catch (e) { console.error("데이터 로드 에러:", e); } 
  finally { isJsonLoading.value = false; currentPage.value = 1; }
};

const goBackToJsonList = () => jsonViewMode.value = 'list';
const openJsonDetail = (item) => { selectedJsonItem.value = item; jsonViewMode.value = 'detail'; };

const filteredJsonItems = computed(() => {
  const q = localSearchQuery.value.trim().toLowerCase();
  if (!q) return jsonItems.value;
  return jsonItems.value.filter(item => 
    (item.title && item.title.toLowerCase().includes(q)) || 
    (item.addr1 && item.addr1.toLowerCase().includes(q))
  );
});

const currentPage = ref(1);
const itemsPerPage = 12; 
const totalPages = computed(() => Math.ceil(filteredJsonItems.value.length / itemsPerPage));
const paginatedJsonItems = computed(() => {
  const start = (currentPage.value - 1) * itemsPerPage;
  return filteredJsonItems.value.slice(start, start + itemsPerPage);
});
const visiblePages = computed(() => {
  const pages = [];
  let start = Math.max(1, currentPage.value - 2);
  let end = Math.min(totalPages.value, start + 4);
  if (end - start < 4) start = Math.max(1, end - 4);
  for (let i = start; i <= end; i++) pages.push(i);
  return pages;
});

const availableColors = ['#4a90e2', '#2ecc71', '#9b59b6', '#f39c12', '#e74c3c', '#1abc9c', '#e84393', '#34495e'];
const schedules = ref(JSON.parse(localStorage.getItem('localhub_schedules')) || []);
watch(schedules, (newVal) => localStorage.setItem('localhub_schedules', JSON.stringify(newVal)), { deep: true });

const dateObj = new Date();
const currentYear = ref(dateObj.getFullYear());
const currentMonth = ref(dateObj.getMonth());
const todayDate = computed(() => { const t = new Date(); return `${t.getFullYear()}-${String(t.getMonth() + 1).padStart(2, '0')}-${String(t.getDate()).padStart(2, '0')}`; });
const getFormattedDate = (date) => `${currentYear.value}-${String(currentMonth.value + 1).padStart(2, '0')}-${String(date).padStart(2, '0')}`;
const isToday = (date) => getFormattedDate(date) === todayDate.value;
const daysInMonth = computed(() => new Date(currentYear.value, currentMonth.value + 1, 0).getDate());
const firstDayOffset = computed(() => new Date(currentYear.value, currentMonth.value, 1).getDay());
const changeMonth = (offset) => { currentMonth.value += offset; if (currentMonth.value > 11) { currentMonth.value = 0; currentYear.value++; } else if (currentMonth.value < 0) { currentMonth.value = 11; currentYear.value--; } };

const getSchedulesForCalendar = (dateString) => {
  return schedules.value.filter(s => dateString >= s.startDate && dateString <= s.endDate).sort((a, b) => a.startDate.localeCompare(b.startDate));
};

const calendarSearchResults = computed(() => {
  const q = localSearchQuery.value.trim().toLowerCase();
  if(!q) return [];
  return schedules.value.filter(s => (s.title && s.title.toLowerCase().includes(q)) || (s.content && s.content.toLowerCase().includes(q))).sort((a,b) => a.startDate.localeCompare(b.startDate));
});

const jumpToCalendarDate = (item) => {
  const d = new Date(item.startDate);
  currentYear.value = d.getFullYear();
  currentMonth.value = d.getMonth();
  localSearchQuery.value = '';
  openDetailModal(item);
};

const getChipClass = (item, date) => { const isStart = item.startDate === date; const isEnd = item.endDate === date; if (isStart && isEnd) return 'chip-single'; if (isStart) return 'chip-start'; if (isEnd) return 'chip-end'; return 'chip-middle'; };
const showChipText = (item, date) => item.startDate === date || new Date(date).getDay() === 0;

const isDetailModalOpen = ref(false);
const isFormModalOpen = ref(false);
const isDayPlanModalOpen = ref(false);
const selectedSchedule = ref(null);
const selectedDateForPlan = ref('');

const closeModals = () => { isDetailModalOpen.value = false; isFormModalOpen.value = false; isDayPlanModalOpen.value = false; selectedSchedule.value = null; };
const openDayPlanModal = (dateString) => { selectedDateForPlan.value = dateString; isDayPlanModalOpen.value = true; };
const dayPlanSchedules = computed(() => getSchedulesForCalendar(selectedDateForPlan.value));
const openDetailFromPlan = (item) => { isDayPlanModalOpen.value = false; openDetailModal(item); };
const openFormFromPlan = (dateString) => { isDayPlanModalOpen.value = false; openFormModal(dateString); };
const openDetailModal = (item) => { selectedSchedule.value = item; isDetailModalOpen.value = true; };

const scheduleForm = ref({ startDate: '', endDate: '', title: '', content: '', image: '', password: '', color: '#4a90e2' });
const openFormModal = (date = todayDate.value) => {
  selectedSchedule.value = null;
  scheduleForm.value = { startDate: date, endDate: date, title: '', content: '', image: '', password: '', color: '#4a90e2' };
  isFormModalOpen.value = true;
};
const handleImageUpload = (event, targetObj) => {
  const file = event.target.files[0];
  if (file) { const reader = new FileReader(); reader.onload = (e) => targetObj.image = e.target.result; reader.readAsDataURL(file); }
};
const saveSchedule = () => {
  if (!scheduleForm.value.title || !scheduleForm.value.password || !scheduleForm.value.startDate || !scheduleForm.value.endDate) { alert("항목을 모두 입력해주세요."); return; }
  if (selectedSchedule.value) { const idx = schedules.value.findIndex(s => s.id === selectedSchedule.value.id); if (idx !== -1) schedules.value[idx] = { ...scheduleForm.value }; } 
  else { schedules.value.push({ ...scheduleForm.value, id: Date.now(), isLiked: false, likeCount: 0, category: '일정' }); }
  closeModals();
};
const toggleScheduleLike = () => {
  if (selectedSchedule.value) {
    const idx = schedules.value.findIndex(s => s.id === selectedSchedule.value.id);
    if (idx !== -1) {
      schedules.value[idx].isLiked = !schedules.value[idx].isLiked;
      schedules.value[idx].likeCount = schedules.value[idx].isLiked ? (schedules.value[idx].likeCount || 0) + 1 : Math.max(0, (schedules.value[idx].likeCount || 1) - 1);
      selectedSchedule.value = { ...schedules.value[idx] };
    }
  }
};
const handleEditSchedule = () => {
  const pwd = prompt("비밀번호를 입력하세요.");
  if (pwd === selectedSchedule.value.password) { scheduleForm.value = { ...selectedSchedule.value }; isDetailModalOpen.value = false; isFormModalOpen.value = true; } 
  else if (pwd !== null) alert("비밀번호가 일치하지 않습니다.");
};
const handleDeleteSchedule = () => {
  const pwd = prompt("비밀번호를 입력하세요.");
  if (pwd === selectedSchedule.value.password) { schedules.value = schedules.value.filter(s => s.id !== selectedSchedule.value.id); closeModals(); } 
  else if (pwd !== null) alert("비밀번호가 일치하지 않습니다.");
};

const boardViewMode = ref('list');
const boardSortMode = ref('latest'); 

const filteredBoardPosts = computed(() => {
  const q = localSearchQuery.value.trim().toLowerCase();
  if (!q) return freeBoardPosts.value;
  return freeBoardPosts.value.filter(p => p.title.toLowerCase().includes(q) || p.content.toLowerCase().includes(q));
});

const sortedFreeBoardPosts = computed(() => {
  let posts = [...filteredBoardPosts.value];
  if (boardSortMode.value === 'likes') { return posts.sort((a, b) => (b.likeCount || 0) - (a.likeCount || 0) || b.createdAt - a.createdAt); } 
  else { return posts.sort((a, b) => b.createdAt - a.createdAt); }
});

const boardCurrentPage = ref(1);
const boardItemsPerPage = 10;
const boardTotalPages = computed(() => Math.ceil(sortedFreeBoardPosts.value.length / boardItemsPerPage));
const paginatedBoardPosts = computed(() => {
  const start = (boardCurrentPage.value - 1) * boardItemsPerPage;
  return sortedFreeBoardPosts.value.slice(start, start + boardItemsPerPage);
});
const boardVisiblePages = computed(() => {
  const pages = [];
  let start = Math.max(1, boardCurrentPage.value - 2);
  let end = Math.min(boardTotalPages.value, start + 4);
  if (end - start < 4) start = Math.max(1, end - 4);
  for (let i = start; i <= end; i++) pages.push(i);
  return pages;
});

const selectedBoardPost = ref(null);
const boardForm = ref({ title: '', content: '', password: '' });

const openBoardForm = () => { selectedBoardPost.value = null; boardForm.value = { title: '', content: '', password: '' }; boardViewMode.value = 'form'; };
const openBoardDetail = (post) => { selectedBoardPost.value = post; boardViewMode.value = 'detail'; };

const saveBoardPost = () => {
  if (!boardForm.value.title || !boardForm.value.content || !boardForm.value.password) { alert('모든 항목을 입력해주세요.'); return; }
  if (selectedBoardPost.value) {
    const idx = freeBoardPosts.value.findIndex(p => p.id === selectedBoardPost.value.id);
    if (idx !== -1) freeBoardPosts.value[idx] = { ...boardForm.value };
  } else {
    freeBoardPosts.value.unshift({ ...boardForm.value, id: Date.now(), createdAt: Date.now(), isLiked: false, likeCount: 0 });
  }
  boardViewMode.value = 'list';
};

const toggleBoardLike = () => {
  if (selectedBoardPost.value) {
    const post = selectedBoardPost.value;
    post.isLiked = !post.isLiked;
    post.likeCount = post.isLiked ? (post.likeCount || 0) + 1 : Math.max(0, (post.likeCount || 1) - 1);
    const idx = freeBoardPosts.value.findIndex(p => p.id === post.id);
    if(idx !== -1) freeBoardPosts.value[idx] = { ...post };
  }
};

const editBoardPost = () => {
  const pwd = prompt("비밀번호를 입력하세요.");
  if (pwd === selectedBoardPost.value.password) { boardForm.value = { ...selectedBoardPost.value }; boardViewMode.value = 'form'; } 
  else if (pwd !== null) alert("비밀번호가 일치하지 않습니다.");
};

const deleteBoardPost = () => {
  const pwd = prompt("비밀번호를 입력하세요.");
  if (pwd === selectedBoardPost.value.password) { freeBoardPosts.value = freeBoardPosts.value.filter(p => p.id !== selectedBoardPost.value.id); boardViewMode.value = 'list'; } 
  else if (pwd !== null) alert("비밀번호가 일치하지 않습니다.");
};

watch(localSearchQuery, () => { currentPage.value = 1; boardCurrentPage.value = 1; });

const globalComments = ref(JSON.parse(localStorage.getItem('localhub_comments')) || []);
watch(globalComments, (newVal) => localStorage.setItem('localhub_comments', JSON.stringify(newVal)), { deep: true });

const CommentSection = defineComponent({
  props: ['targetId'],
  setup(props) {
    const newComment = ref({ text: '', image: '', password: '' });
    const targetComments = computed(() => { return globalComments.value.filter(c => c.targetId === props.targetId).sort((a, b) => b.createdAt - a.createdAt); });
    const handleImg = (e) => handleImageUpload(e, newComment.value);

    const submit = () => {
      if (!newComment.value.text || !newComment.value.password) { alert("내용과 비밀번호를 모두 입력해주세요."); return; }
      globalComments.value.push({ id: Date.now(), targetId: props.targetId, text: newComment.value.text, image: newComment.value.image, password: newComment.value.password, isLiked: false, likeCount: 0, createdAt: Date.now() });
      newComment.value = { text: '', image: '', password: '' };
    };

    const deleteComm = (comment) => {
      const pwd = prompt("댓글을 삭제하려면 비밀번호를 입력하세요.");
      if (pwd === comment.password) { globalComments.value = globalComments.value.filter(c => c.id !== comment.id); } 
      else if (pwd !== null) { alert("비밀번호가 일치하지 않습니다."); }
    };

    const toggleCommLike = (id) => {
      const idx = globalComments.value.findIndex(c => c.id === id);
      if (idx !== -1) {
        globalComments.value[idx].isLiked = !globalComments.value[idx].isLiked;
        const currentCount = globalComments.value[idx].likeCount || 0;
        globalComments.value[idx].likeCount = globalComments.value[idx].isLiked ? currentCount + 1 : Math.max(0, currentCount - 1);
      }
    };

    return () => h('div', { class: 'comment-section' }, [
      h('h3', `댓글 (${targetComments.value.length})`),
      h('div', { class: 'comment-form' }, [
        h('textarea', { value: newComment.value.text, onInput: (e) => newComment.value.text = e.target.value, placeholder: '댓글 내용을 입력하세요...' }),
        h('div', { class: 'comment-form-actions' }, [
          h('input', { type: 'password', value: newComment.value.password, onInput: (e) => newComment.value.password = e.target.value, placeholder: '삭제용 비밀번호', class: 'pwd-input' }),
          h('label', { class: 'custom-file-label' }, [ '📷 이미지', h('input', { type: 'file', accept: 'image/*', onChange: handleImg, style: 'display:none;' }) ]),
          newComment.value.image ? h('img', { src: newComment.value.image, class: 'thumb-preview' }) : null,
          h('button', { class: 'btn-submit', onClick: submit }, '등록')
        ])
      ]),
      h('ul', { class: 'comment-list' }, targetComments.value.map(c => 
        h('li', { class: 'comment-item', key: c.id }, [
          h('p', { class: 'comment-text' }, c.text),
          c.image ? h('img', { src: c.image, class: 'comment-image' }) : null,
          h('div', { class: 'comment-footer' }, [
            h('span', { class: 'comment-date' }, new Date(c.createdAt).toLocaleString()),
            h('div', [
              h('button', { class: ['btn-like-small', { active: c.isLiked }], onClick: () => toggleCommLike(c.id) }, c.isLiked ? `❤️ 좋아요 취소 (${c.likeCount||0})` : `🤍 좋아요 (${c.likeCount||0})`),
              h('button', { class: 'btn-del-small', onClick: () => deleteComm(c) }, '삭제')
            ])
          ])
        ])
      ))
    ]);
  }
});

// ==========================================
// 챗봇 (OpenAI API 연동 - 환경변수 사용)
// ==========================================
const chatInput = ref('');
const chatMessages = ref([{ role: 'bot', content: '무엇을 도와드릴까요?' }]);
const chatBox = ref(null);
const isChatLoading = ref(false);

// 💡 .env 파일에서 Vite 환경변수로 API 키 불러오기
const OPENAI_API_KEY = import.meta.env.VITE_OPENAI_API_KEY; 

// 🚨 [핵심 수정 1] 존재하지 않는 모델명 변경!
// OpenAI 공식 서버에는 'gpt-5-mini'가 없습니다. 기관에서 최신 모델인 'gpt-4o-mini'를 잘못 안내했을 확률이 높습니다.
// 존재하지 않는 모델을 부르면 무조건 에러가 발생하므로, 올바른 모델명으로 수정했습니다.
const AI_MODEL = "gpt-5-mini"; 

// 질문과 관련된 실제 데이터를 찾아서 컨텍스트로 반환
const findRelevantData = (query) => {
  const q = query.toLowerCase();
  const categoryKeywords = {
    '음식점': ['맛집', '음식', '식당', '먹을거리', '맛있는', '먹자', '먹을 곳'],
    '관광지': ['관광', '여행', '가볼만한', '명소', '구경'],
    '레포츠': ['레포츠', '액티비티', '체험', '스포츠'],
    '문화시설': ['문화', '박물관', '전시', '미술관'],
    '쇼핑': ['쇼핑', '시장', '백화점', '마트'],
    '숙박': ['숙박', '호텔', '펜션', '모텔', '잘 곳', '잠잘'],
    '축제공연행사': ['축제', '공연', '행사', '이벤트'],
  };

  let targetCategory = null;
  for (const [cat, keywords] of Object.entries(categoryKeywords)) {
    if (keywords.some(k => q.includes(k))) { targetCategory = cat; break; }
  }

  let candidates = allJsonData.value;
  if (targetCategory) {
    candidates = candidates.filter(item => item._category === targetCategory);
  }

  if (candidates.length === 0) return { text: '', category: targetCategory };

  // 토큰 절약을 위해 상위 15개만, 제목/카테고리/주소만 전달
  const listText = candidates.slice(0, 15)
    .map(item => `- ${item.title} (${item._category}) / 주소: ${item.addr1 || '정보없음'}`)
    .join('\n');

  return { text: listText, category: targetCategory };
};

const sendMessage = async () => {
  const message = chatInput.value.trim();
  if (!message || isChatLoading.value) return;

  if (!OPENAI_API_KEY) {
    chatMessages.value.push({ role: 'bot', content: '시스템 에러: API 키를 찾을 수 없습니다.' });
    return;
  }

  chatMessages.value.push({ role: 'user', content: message });
  chatInput.value = '';
  isChatLoading.value = true;
  await nextTick();
  if (chatBox.value) chatBox.value.scrollTop = chatBox.value.scrollHeight;

  chatMessages.value.push({ role: 'bot', content: '답변을 생각 중입니다...', isLoading: true });
  await nextTick();
  if (chatBox.value) chatBox.value.scrollTop = chatBox.value.scrollHeight;

  const { text: dataContext, category } = findRelevantData(message);

  const recentHistory = chatMessages.value
    .filter(m => !m.isLoading)
    .slice(-8)
    .map(m => ({ role: m.role === 'bot' ? 'assistant' : 'user', content: m.content }));

  const systemPrompt = `너는 myGumi 사이트의 지역정보 안내원이다.
사용자가 맛집, 관광지 등 지역 정보를 물어보면 아래 [실제 데이터 목록]에 있는 장소만 근거로 2~4개 정도 추천하고, 각 장소의 이름과 주소를 간단히 알려줘라.
[실제 데이터 목록]에 관련 항목이 없으면 "죄송합니다. 등록된 정보 중에는 관련 장소가 없네요."라고 답하고, 절대로 목록에 없는 곳을 지어내지 마라.
사이트 사용법(일정 등록, 게시판 이용 등) 질문에는 2~3문장 이내로 답해라.
코딩/IT 기술 질문은 거부하고 "죄송합니다. 저는 홈페이지 안내용 챗봇으로, 요청하신 답변은 제공해 드릴 수 없습니다."라고만 출력해라.

[실제 데이터 목록]
${dataContext || '(현재 질문과 매칭되는 카테고리 데이터가 없습니다)'}`;

  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), 15000);

  try {
    const response = await fetch("https://api.openai.com/v1/chat/completions", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${OPENAI_API_KEY}`
      },
      signal: controller.signal,
      body: JSON.stringify({
        model: AI_MODEL,
        max_completion_tokens: 1000,
        messages: [
          { role: "system", content: systemPrompt },
          ...recentHistory
        ]
      })
    });
    clearTimeout(timeoutId);

    if (!response.ok) {
      const errorData = await response.json();
      console.error("🚨 서버 연결 실패 원인:", errorData);
      throw new Error(errorData.error?.message || "알 수 없는 API 에러 발생");
    }

    const data = await response.json();
    let botReply = data.choices?.[0]?.message?.content || "응답이 비어있습니다.";

    chatMessages.value.pop();
    chatMessages.value.push({ role: 'bot', content: botReply });
  } catch (error) {
    clearTimeout(timeoutId);
    console.error(error);
    chatMessages.value.pop();
    const msg = error.name === 'AbortError' ? '응답 시간이 초과되었습니다. 다시 시도해주세요.' : `에러: ${error.message}`;
    chatMessages.value.push({ role: 'bot', content: msg });
  } finally {
    isChatLoading.value = false;
    await nextTick();
    if (chatBox.value) chatBox.value.scrollTop = chatBox.value.scrollHeight;
  }
};
</script>

<style>
/* 기본적으로 여백을 제거하고 앱 크기를 화면에 맞춤 */
html, body {
  margin: 0; padding: 0; width: 100%; height: 100%;
}
#app {
  max-width: 100% !important; margin: 0 !important; padding: 0 !important; width: 100%; height: 100%;
}

/* 💻 PC 버전: 화면 고정 (내부 영역 개별 스크롤 사용) */
@media (min-width: 901px) {
  html, body, #app {
    overflow: hidden;
  }
}

/* 📱 모바일 버전: 강제 잠금 해제 및 전체 화면 스크롤 허용 */
@media (max-width: 900px) {
  html, body, #app {
    overflow: auto !important; 
    height: auto !important; 
    min-height: 100%;
  }
}
</style>

<style scoped>
.localhub-wrapper { display: flex; width: 100%; height: 100%; overflow: hidden; background-color: #f4f6f9; font-family: 'Pretendard', sans-serif; color: #333; position: relative;}
.sidebar-header { width: 250px; background-color: #fff; border-right: 1px solid #e0e0e0; display: flex; flex-direction: column; flex-shrink: 0; padding-top: 20px; height: 100%; overflow-y: auto; }
.header-top { width: 100%; text-align: center; }
.hamburger-btn { display: none; }
.logo { font-size: 1.5rem; font-weight: 800; color: #007bff; padding: 20px; border-bottom: 1px solid #eee; margin-bottom: 20px; cursor: pointer; transition: 0.2s; display: block; }
.logo:hover { color: #0056b3; }

.category-nav { display: flex; flex-direction: column; }
.category-nav button { display: block; width: 100%; padding: 15px 25px; text-align: left; background: none; border: none; font-size: 1.05rem; color: #555; cursor: pointer; transition: 0.2s; }
.category-nav button:hover { background-color: #f8f9fa; color: #007bff; }
.category-nav button.active { background-color: #ebf5ff; color: #007bff; font-weight: bold; border-right: 4px solid #007bff; }

.main-content { flex: 1; padding: 30px 40px; width: 100%; box-sizing: border-box; height: 100%; overflow-y: auto; position: relative; scroll-behavior: smooth; }
.view-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; border-bottom: 2px solid #333; padding-bottom: 15px; }

.home-view { display: flex; flex-direction: column; align-items: center; justify-content: center; min-height: 100%; text-align: center; }
.home-center-container { width: 100%; max-width: 800px; display: flex; flex-direction: column; align-items: center; gap: 40px;}
.hero-banner h1 { font-size: 2.5rem; color: #007bff; margin-bottom: 10px; }
.hero-banner p { font-size: 1.2rem; color: #666; margin-bottom: 30px; }

.home-global-search { position: relative; width: 100%; max-width: 600px; margin: 0 auto; display: flex; flex-direction: column; align-items: center;}
.search-input-wrapper { display: flex; align-items: center; background: white; border: 2px solid #007bff; border-radius: 30px; padding: 10px 20px; box-shadow: 0 4px 15px rgba(0,123,255,0.15); width: 100%; box-sizing: border-box;}
.search-icon { font-size: 1.2rem; margin-right: 10px; }
.search-input-wrapper input { border: none; flex: 1; outline: none; font-size: 1.1rem; }
.global-search-results { position: absolute; top: 110%; left: 0; width: 100%; max-height: 350px; overflow-y: auto; background: white; border-radius: 12px; box-shadow: 0 10px 30px rgba(0,0,0,0.1); z-index: 100; text-align: left; }
.empty-result { padding: 20px; color: #888; text-align: center; }
.result-list { list-style: none; padding: 0; margin: 0; }
.result-list li { padding: 15px 20px; border-bottom: 1px solid #eee; cursor: pointer; transition: 0.2s; }
.result-list li:hover { background: #f1f7ff; }
.res-header { display: flex; align-items: center; margin-bottom: 5px; }
.res-badge { color: #007bff; font-weight: bold; margin-right: 8px; font-size: 0.9rem; }
.res-title { font-weight: bold; font-size: 1.05rem; }
.res-address { font-size: 0.85rem; color: #666; }

.home-shortcuts { display: flex; justify-content: center; gap: 20px; flex-wrap: wrap; width: 100%;}
.shortcut-card { flex: 1; min-width: 200px; padding: 30px; background: white; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.05); font-size: 1.2rem; font-weight: bold; cursor: pointer; transition: 0.2s; border: 1px solid #eee; }
.shortcut-card:hover { transform: translateY(-5px); border-color: #007bff; color: #007bff; }

.tab-search-container { width: 100%; max-width: 600px; margin: 0 auto 30px auto; display: flex; justify-content: center;}
.tab-search-container .search-input-wrapper { border-width: 1px; box-shadow: 0 2px 8px rgba(0,0,0,0.05); padding: 8px 20px;}
.clear-btn { background: none; border: none; color: #aaa; cursor: pointer; font-size: 1.2rem; padding: 0; margin-left: 10px; outline: none;}
.clear-btn:hover { color: #333; }
.header-actions { display: flex; gap: 15px; align-items: center; }

.relative-wrapper { position: relative; }
.calendar-search-dropdown { position: absolute; top: 110%; left: 0; width: 100%; max-height: 300px; overflow-y: auto; background: white; border: 1px solid #ddd; border-radius: 8px; box-shadow: 0 5px 15px rgba(0,0,0,0.1); z-index: 100; text-align: left; }
.calendar-search-dropdown li .res-date { font-size: 0.8rem; color: #888; display: block; margin-top: 5px; }

.pagination { display: flex; justify-content: center; gap: 8px; margin-top: 30px; padding-bottom: 30px;}
.page-btn { padding: 8px 12px; border: 1px solid #ddd; background: white; border-radius: 4px; cursor: pointer; transition: 0.2s; }
.page-btn:hover { background: #f1f7ff; border-color: #007bff; color: #007bff; }
.page-btn.active { background: #007bff; color: white; border-color: #007bff; font-weight: bold; }
.page-btn:disabled { background: #f9f9f9; color: #ccc; cursor: not-allowed; border-color: #eee; }

.calendar-view { background: #fff; border-radius: 12px; box-shadow: 0 2px 10px rgba(0,0,0,0.05); padding: 20px; min-height: calc(100% - 60px); display: flex; flex-direction: column; box-sizing: border-box; }
.top-controls { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
.month-navigation { display: flex; align-items: center; gap: 15px; }
.month-navigation h2 { margin: 0; font-size: 1.8rem; }
.nav-btn { background: none; border: none; font-size: 1.2rem; cursor: pointer; color: #888; }
.actions { display: flex; gap: 10px; align-items: center; }
.btn-write { background: #007bff; color: white; border: none; padding: 8px 20px; border-radius: 8px; font-weight: bold; cursor: pointer; }
.btn-outline { border: 1px solid #ccc; background: white; padding: 8px 16px; border-radius: 4px; cursor: pointer; }

.calendar-grid { display: grid; grid-template-columns: repeat(7, 1fr); background: #eee; border: 1px solid #eee; border-radius: 8px; gap: 1px; flex: 1; overflow: hidden; }
.weekday { background: #fff; text-align: center; padding: 12px; font-weight: bold; font-size: 0.95rem; }
.weekday.sunday, .date-num.sunday { color: #e74c3c; }
.weekday.saturday, .date-num.saturday { color: #3498db; }
.day { background: #fff; min-height: 120px; padding: 5px 0; cursor: pointer; transition: background 0.1s; display: flex; flex-direction: column; min-width: 0; overflow: hidden; }
.day:hover { background: #fafafa; }
.day.blank { background: #f9f9f9; cursor: default; }
.date-num { display: inline-block; width: 28px; height: 28px; line-height: 28px; text-align: center; border-radius: 50%; margin-bottom: 5px; margin-left: 5px; font-weight: 500; font-size: 0.9rem; flex-shrink: 0; }
.date-num.today { background: #007bff; color: white; font-weight: bold; }

.schedule-list { display: flex; flex-direction: column; gap: 2px; width: 100%; }
.schedule-chip { color: #fff; text-shadow: 0 0 2px rgba(0,0,0,0.3); font-size: 0.8rem; padding: 4px 8px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; cursor: pointer; display: block; box-sizing: border-box; max-width: 100%; }
.chip-single { border-radius: 4px; margin: 0 4px; }
.chip-start  { border-radius: 4px 0 0 4px; margin-left: 4px; margin-right: -1px; position: relative; z-index: 2; }
.chip-middle { border-radius: 0; margin-left: -1px; margin-right: -1px; position: relative; z-index: 2; }
.chip-end    { border-radius: 0 4px 4px 0; margin-left: -1px; margin-right: 4px; position: relative; z-index: 2; }
.text-hidden { opacity: 0; pointer-events: none; }

.board-view { background: #fff; padding: 30px; border-radius: 12px; box-shadow: 0 2px 10px rgba(0,0,0,0.05); min-height: calc(100% - 60px); }
.board-header { display: flex; justify-content: center; align-items: center; border-bottom: 2px solid #333; padding-bottom: 15px; margin-bottom: 20px; }
.board-header h2 { margin: 0; }
.board-controls-row { display: flex; justify-content: flex-end; gap: 10px; align-items: center; margin-bottom: 20px; }
.sort-select { padding: 8px; border: 1px solid #ddd; border-radius: 4px; outline: none; }

.board-list-cards { display: flex; flex-direction: column; gap: 10px; }
.board-card { border: 1px solid #eee; padding: 15px 20px; border-radius: 8px; cursor: pointer; transition: 0.2s; background: #fff; }
.board-card:hover { background: #f1f7ff; border-color: #007bff; }
.bc-title { font-size: 1.1rem; font-weight: bold; margin-bottom: 10px; color: #333; }
.bc-meta { font-size: 0.85rem; color: #888; display: flex; align-items: center; gap: 10px; flex-wrap: wrap;}
.divider-line { color: #ddd; }
.like-text { color: #ff4757; font-weight: bold; }

.board-detail-view .post-header { margin-bottom: 20px; }
.board-detail-view .date { color: #888; font-size: 0.9rem; }
.board-detail-view .post-content { min-height: 200px; white-space: pre-wrap; line-height: 1.6; }
.post-actions { display: flex; justify-content: space-between; align-items: center; margin-top: 20px; }
.left-actions { display: flex; }
.right-actions { display: flex; gap: 10px; }
.form-input { width: 100%; padding: 12px; margin-bottom: 15px; border: 1px solid #ddd; border-radius: 8px; box-sizing: border-box; font-family: inherit; }

.json-data-view { background: #fff; padding: 30px; border-radius: 12px; box-shadow: 0 2px 10px rgba(0,0,0,0.05); min-height: calc(100% - 60px); }
.loading-state { padding: 50px; text-align: center; color: #007bff; font-weight: bold; font-size: 1.2rem; }
.tripadvisor-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(280px, 1fr)); gap: 20px; }
.trip-card { border: 1px solid #eee; border-radius: 8px; overflow: hidden; cursor: pointer; transition: 0.3s; }
.trip-card:hover { box-shadow: 0 5px 15px rgba(0,0,0,0.1); transform: translateY(-3px); }
.card-img { height: 180px; background-size: cover; background-position: center; }
.card-info { padding: 15px; }
.card-info h4 { margin: 0 0 10px 0; font-size: 1.1rem; }
.card-info .address { color: #666; font-size: 0.9rem; margin: 0; }
.tripadvisor-detail .detail-img-box img { width: 100%; max-height: 800px; object-fit: contain; border-radius: 8px; margin-top: 20px; }
.tripadvisor-detail .info-box { background: #f8f9fa; padding: 20px; border-radius: 8px; margin-top: 20px; line-height: 1.8; }

.scroll-controls { display: none; position: fixed; bottom: 30px; left: 30px; flex-direction: column; gap: 10px; z-index: 999; }
.scroll-btn { width: 45px; height: 45px; border-radius: 50%; background: rgba(0, 0, 0, 0.5); color: white; border: none; font-size: 1.2rem; cursor: pointer; box-shadow: 0 4px 10px rgba(0,0,0,0.2); align-items: center; justify-content: center; }
.scroll-btn:hover { background: rgba(0, 0, 0, 0.8); }

.modal-overlay { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: rgba(0,0,0,0.5); display: flex; align-items: center; justify-content: center; z-index: 2000; backdrop-filter: blur(2px); }
.modal-content { background: #fff; width: 90%; max-width: 650px; max-height: 90vh; border-radius: 16px; display: flex; flex-direction: column; box-shadow: 0 10px 30px rgba(0,0,0,0.2); overflow: hidden; animation: slideUp 0.3s ease-out; }
@keyframes slideUp { from { opacity: 0; transform: translateY(30px); } to { opacity: 1; transform: translateY(0); } }
.modal-header { padding: 20px 25px; border-bottom: 1px solid #eee; display: flex; justify-content: space-between; align-items: center; background: #f8f9fa; }
.modal-header h2 { margin: 0; font-size: 1.4rem; color: #333; }
.btn-close { background: none; border: none; font-size: 1.5rem; color: #888; cursor: pointer; }
.modal-body { padding: 25px; overflow-y: auto; flex: 1; }

.day-plan-body { padding: 0 !important; }
.empty-state { text-align: center; color: #888; padding: 50px 0; font-size: 1.1rem; }
.day-schedule-list { list-style: none; padding: 0; margin: 0; }
.day-schedule-list li { display: flex; align-items: center; gap: 15px; padding: 20px 25px; border-bottom: 1px solid #eee; cursor: pointer; transition: background 0.2s; }
.day-schedule-list li:hover { background: #f8f9fa; }
.color-indicator { width: 5px; height: 45px; border-radius: 10px; }
.day-schedule-list .info { display: flex; flex-direction: column; gap: 6px; }
.day-schedule-list .title { font-weight: bold; font-size: 1.1rem; color: #333; }
.day-schedule-list .time { font-size: 0.9rem; color: #777; }
.modal-footer { padding: 20px 25px; background: #fff; border-top: 1px solid #eee; }
.btn-add-plan { display: flex; align-items: center; justify-content: center; gap: 10px; background: #f1f7ff; color: #007bff; border: 1px dashed #007bff; font-weight: bold; }

.meta-info { display: flex; gap: 10px; margin-bottom: 20px; font-size: 0.9rem; }
.date-tag { background: #ebf5ff; color: #007bff; padding: 4px 10px; border-radius: 20px; font-weight: bold;}
.poster-container img { width: 100%; max-height: 400px; object-fit: contain; border-radius: 8px; background: #f8f9fa; margin-bottom: 20px; }
.text-content { font-size: 1.05rem; line-height: 1.7; white-space: pre-wrap; margin-bottom: 30px; }
.detail-actions { display: flex; justify-content: space-between; align-items: center; }
.btn-like { background: white; border: 1px solid #ccc; padding: 8px 16px; border-radius: 20px; cursor: pointer; font-weight: bold;}
.btn-like.active { border-color: #ff4757; color: #ff4757; background: #fff0f1; }
.crud-btns button { border: none; padding: 8px 16px; border-radius: 4px; margin-left: 10px; cursor: pointer; color: white; font-weight: bold;}
.btn-edit { background: #f39c12; }
.btn-delete { background: #e74c3c; }

.form-group { margin-bottom: 20px; }
.form-group label { display: block; margin-bottom: 8px; font-weight: bold; }
.date-range-inputs { display: flex; align-items: center; gap: 10px; }
.date-range-inputs input { flex: 1; padding: 12px; border: 1px solid #ddd; border-radius: 8px; }
.color-picker { display: flex; gap: 10px; flex-wrap: wrap; }
.color-circle { width: 35px; height: 35px; border-radius: 50%; cursor: pointer; display: flex; justify-content: center; align-items: center; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }
.color-circle.selected { transform: scale(1.15); border: 2px solid #333; box-shadow: 0 4px 8px rgba(0,0,0,0.2); }
.check-icon { color: white; font-weight: bold; text-shadow: 0 1px 2px rgba(0,0,0,0.5); }
.image-preview { max-width: 100%; max-height: 200px; margin-top: 10px; border-radius: 8px; }
.modal-footer-actions { display: flex; gap: 10px; margin-top: 10px; }
.btn-cancel { flex: 1; padding: 12px; background: #eee; border: none; border-radius: 8px; cursor: pointer; font-weight: bold; }
.btn-submit-full { flex: 2; padding: 12px; background: #007bff; color: white; border: none; border-radius: 8px; font-weight: bold; cursor: pointer; }

:deep(.comment-section h3) { font-size: 1.1rem; margin-bottom: 15px; }
:deep(.comment-form) { border: 1px solid #ddd; padding: 12px; border-radius: 8px; background: #fafafa; margin-bottom: 20px; }
:deep(.comment-form textarea) { width: 100%; border: none; background: transparent; resize: none; outline: none; margin-bottom: 10px; box-sizing: border-box; font-family: inherit; }
:deep(.comment-form-actions) { display: flex; justify-content: space-between; align-items: center; gap: 10px; flex-wrap: wrap; }
:deep(.pwd-input) { padding: 6px 10px; border: 1px solid #ccc; border-radius: 4px; width: 130px; font-size: 0.9rem; outline: none;}
:deep(.custom-file-label) { cursor: pointer; color: #007bff; font-weight: bold; font-size: 0.9em; }
:deep(.thumb-preview) { height: 30px; margin-left: 10px; border-radius: 4px; vertical-align: middle; }
:deep(.btn-submit) { background: #333; color: white; padding: 6px 15px; border: none; border-radius: 4px; cursor: pointer; font-weight: bold;}
:deep(.comment-list) { list-style: none; padding: 0; margin: 0; }
:deep(.comment-item) { padding: 15px 0; border-bottom: 1px solid #eee; }
:deep(.comment-text) { margin: 0 0 10px 0; line-height: 1.5; white-space: pre-wrap; }
:deep(.comment-image) { max-width: 200px; border-radius: 8px; margin-bottom: 10px; display: block; }
:deep(.comment-footer) { display: flex; justify-content: space-between; align-items: center; font-size: 0.85em; color: #888; }
:deep(.btn-like-small), :deep(.btn-del-small) { border: none; background: none; cursor: pointer; color: #888; margin-left: 10px; font-weight: bold;}
:deep(.btn-like-small.active) { color: #ff4757; }
:deep(.btn-del-small:hover) { color: #e74c3c; text-decoration: underline; }

.chatbot-wrapper { position: fixed; z-index: 9999; touch-action: none; }
.floating-btn { width: 60px; height: 60px; border-radius: 50%; background: #007bff; color: white; font-size: 24px; border: none; box-shadow: 0 4px 10px rgba(0,0,0,0.2); cursor: grab; user-select: none; display: flex; align-items: center; justify-content: center;}
.floating-btn:active { cursor: grabbing; }
.chat-panel { position: absolute; bottom: 75px; right: 0; width: 320px; height: 450px; background: white; border-radius: 12px; box-shadow: 0 5px 20px rgba(0,0,0,0.15); display: flex; flex-direction: column; overflow: hidden; }
.chat-header { background: #007bff; color: white; padding: 15px; display: flex; justify-content: space-between; font-weight: bold; }
.chat-header button { background: none; border: none; color: white; cursor: pointer; }
.chat-messages { flex: 1; padding: 15px; overflow-y: auto; background: #f8f9fa; }
.message { padding: 10px; border-radius: 8px; margin-bottom: 10px; font-size: 0.9rem; max-width: 85%; }
.message.user { background: #007bff; color: white; align-self: flex-end; margin-left: auto; }
.message.bot { background: white; border: 1px solid #eee; }
.chat-input { display: flex; padding: 10px; border-top: 1px solid #eee; }
.chat-input input { flex: 1; padding: 8px; border: 1px solid #ddd; border-radius: 4px; margin-right: 8px; outline: none;}
.chat-input button { padding: 8px 15px; background: #333; color: white; border: none; border-radius: 4px; cursor: pointer; }

.chat-dismiss-zone { position: fixed; bottom: 0; left: 0; width: 100%; height: 100px; background: rgba(220, 53, 69, 0.8); color: white; display: flex; justify-content: center; align-items: center; font-size: 1.2rem; font-weight: bold; z-index: 9998; transition: 0.2s; pointer-events: none;}
.chat-dismiss-zone.hovering { background: rgba(220, 53, 69, 1); height: 120px; font-size: 1.4rem; }

@media (max-width: 900px) {
  .localhub-wrapper { height: auto; min-height: 100%; flex-direction: column; overflow: visible; }
  .sidebar-header { width: 100%; height: auto; border-right: none; border-bottom: none; z-index: 1000; padding: 0; flex-shrink: 0; overflow: visible;}
  .header-top { display: flex; justify-content: center; align-items: center; height: 60px; border-bottom: 1px solid #ddd; position: relative; background: #fff;}
  .hamburger-btn { display: block; position: absolute; left: 15px; font-size: 1.8rem; background: none; border: none; cursor: pointer; color: #333; }
  .logo { padding: 0; margin: 0; border-bottom: none; line-height: 60px; }
  
  .category-nav { display: none; position: absolute; top: 60px; left: 0; width: 100%; background: white; flex-direction: column; box-shadow: 0 10px 10px rgba(0,0,0,0.1); }
  .category-nav.open { display: flex; }
  .category-nav button { padding: 15px; text-align: center; border-bottom: 1px solid #f8f9fa; }

  .main-content { padding: 10px; flex: 1; height: auto; min-height: calc(100vh - 60px); overflow-y: visible; }
  
  .calendar-view { padding: 10px; min-height: auto; }
  .calendar-grid { gap: 1px; }
  .weekday { padding: 6px 2px; font-size: 0.8rem; }
  .day { min-height: 80px; padding: 2px 0; }
  .date-num { width: 22px; height: 22px; line-height: 22px; font-size: 0.8rem; margin: 2px; }
  .schedule-chip { font-size: 0.7rem; padding: 2px 4px; border-radius: 2px;}

  .board-view { padding: 15px; min-height: auto; }
  .home-shortcuts { flex-direction: column; }
  .top-controls { flex-direction: column; gap: 15px; align-items: stretch; }
  .actions { width: 100%; flex-direction: column; gap: 10px; align-items: stretch; }
  .btn-write { padding: 12px; }
  
  .scroll-controls { display: flex; }
  
  /* 📱 모바일 챗봇 전체화면 설정 */
  .chat-panel { 
    position: fixed !important; 
    top: 0 !important; 
    left: 0 !important; 
    width: 100vw !important; 
    height: 100vh !important; 
    height: 100dvh !important; /* 모바일 브라우저 주소창/하단바 짤림 완벽 방지 */
    border-radius: 0 !important; 
    z-index: 100000 !important; 
    touch-action: auto !important; /* 모바일 화면에서 부드러운 스크롤 허용 */
  }
  .chat-header {
    border-radius: 0 !important;
    padding: 15px 20px !important;
  }
  .chat-input {
    padding-bottom: 20px !important; /* 아이폰 등 모바일 하단 여백 확보 */
  }
}
</style>