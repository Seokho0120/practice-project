<script setup lang="ts">
import { computed, onMounted, ref, toRefs, watch } from 'vue';

export interface ImageItemsType {
  link: string;
  id: number;
  name: string;
}

const props = withDefaults(
  defineProps<{
    imageItems: ImageItemsType[];
    showPrevButton?: boolean;
    showNextButton?: boolean;
    pagination?: boolean | { dynamicBullets?: boolean };
    scrollbar?: boolean;
    autoPlay?: boolean;
    autoPlayDuration?: number;
    effectFade?: boolean;
    keyboardControl?: boolean;
    parallax?: boolean;
    contents?: {
      title: string;
      subTitle: string;
      content: string;
    }[];
    lazy?: boolean;
  }>(),
  {
    autoPlayDuration: 3000,
  },
);

const { imageItems, scrollbar, autoPlay, pagination } = toRefs(props);

const showScrollbar = computed(() => scrollbar.value ?? false);
const carouselRef = ref<HTMLElement | null>(null);
const currentIndex = ref(0);
const isDragging = ref(false);
const startX = ref(0);
const startTranslateX = ref(0);

function nextHandler() {
  if (currentIndex.value < imageItems.value.length - 1) {
    currentIndex.value += 1;
  } else {
    currentIndex.value = 0;
  }
}

function prevHandler() {
  if (currentIndex.value > 0) {
    currentIndex.value -= 1;
  } else {
    currentIndex.value = imageItems.value.length - 1;
  }
}

// 드래그 시작
function dragStart(event: MouseEvent) {
  if (carouselRef.value) {
    isDragging.value = true;
    startX.value = event.pageX;
    startTranslateX.value = currentIndex.value * -100; // 현재 위치 저장
  }
}

// 드래그 중
function dragging(event: MouseEvent) {
  if (!isDragging.value || !carouselRef.value) return;

  const dragX = event.pageX;
  const walk = ((dragX - startX.value) / window.innerWidth) * 100; // 드래그 거리 계산

  carouselRef.value.style.transition = 'none'; // 드래그 중에는 애니메이션 비활성화
  carouselRef.value.style.transform = `translateX(${startTranslateX.value + walk}%)`; // 이동
}

const autoPlayInterval = ref<number | null>(null);

// 드래그 종료
function dragStop(event: MouseEvent) {
  if (!isDragging.value) return;

  const dragStandardDistance = 4; // 드래그 기준 거리, 현재 %로 계산됨
  const dragX = event.pageX;
  const walk = ((dragX - startX.value) / window.innerWidth) * 100; // 드래그 거리 계산

  // 드래그 거리 기준으로 이전/다음 이미지로 이동
  if (
    walk < -dragStandardDistance &&
    currentIndex.value < imageItems.value.length - 1
  ) {
    nextHandler(); // 다음 이미지로 이동
  } else if (walk > dragStandardDistance && currentIndex.value > 0) {
    prevHandler(); // 이전 이미지로 이동
  }

  if (carouselRef.value) {
    carouselRef.value.style.transition = 'transform 0.4s ease'; // 이미지 드래그할때 여러 애니메이션을 줄 수 있음 일반적으로 ease
    // 드래그 종료 시 최종 위치를 정해줘야 그 이미지를 볼 수 있기 떄문에
    carouselRef.value.style.transform = `translateX(${currentIndex.value * -100}%)`; // 최종 위치 설정
    isDragging.value = false;
  }
}

function goToImage(index: number) {
  currentIndex.value = index;
}

function startAutoPlay() {
  if (autoPlay.value) {
    autoPlayInterval.value = setInterval(() => {
      nextHandler();
    }, props.autoPlayDuration);
  }
}

watch(
  autoPlay,
  (newVal) => {
    if (newVal) {
      startAutoPlay();
    } else {
      if (autoPlayInterval.value) {
        clearInterval(autoPlayInterval.value);
      }
    }
  },
  { immediate: true },
);

function getPaginationClass(idx: number) {
  const isDynamicBullets =
    typeof pagination.value === 'object' && pagination.value.dynamicBullets;
  const isActive = idx === currentIndex.value;
  const isNextOrPrev =
    idx === currentIndex.value - 1 || idx === currentIndex.value + 1;

  if (isDynamicBullets) {
    return isActive
      ? 'opacity-100 scale-125' // 현재 idx
      : isNextOrPrev
        ? 'opacity-70 scale-100' // 현재 idx 앞뒤
        : 'opacity-30 scale-75'; // 나머지 idx
  } else {
    return isActive
      ? 'opacity-100'
      : isNextOrPrev
        ? 'opacity-40'
        : 'opacity-30';
  }
}

function effectFadeStyle(index: number) {
  return `${currentIndex.value === index ? 'opacity-100 transition-opacity duration-1000 ease-in-out' : 'opacity-0'}`;
}

function handleKeyDown(event: KeyboardEvent) {
  if (props.keyboardControl) {
    if (event.key === 'ArrowRight') {
      nextHandler();
    } else if (event.key === 'ArrowLeft') {
      prevHandler();
    }
  }
}

onMounted(() => {
  if (carouselRef.value) {
    carouselRef.value.focus(); // 자동 포커스
  }
});

const getParallaxStyle = (index: number, offset: number) => {
  const progress = currentIndex.value - index;
  const translateXValue = offset * progress;

  return {
    transform: `translateX(${translateXValue}px)`,
    transition: 'transform 0.6s ease',
  };
};
</script>

<template>
  <div class="overflow-hidden relative flex-shrink-0">
    <ul
      ref="carouselRef"
      class="flex transition-transform duration-400"
      :style="{ transform: `translateX(${currentIndex * -100}%)` }"
      @mousedown="dragStart"
      @mouseup="dragStop"
      @mouseleave="dragStop"
      @mousemove="dragging"
      @keydown="handleKeyDown"
      tabindex="0"
    >
      <li
        v-for="(image, index) in imageItems"
        :key="index"
        :class="`flex-shrink-0 w-full h-full ${effectFade && effectFadeStyle(index)}`"
      >
        <div v-if="props.contents" class="relative">
          <h1
            class="absolute top-5 left-5 text-xl font-bold"
            :style="getParallaxStyle(index, -300)"
          >
            {{ props.contents[index]?.title }}
          </h1>
          <h2
            class="absolute top-20 left-5 text-lg"
            :style="getParallaxStyle(index, -200)"
          >
            {{ props.contents[index]?.subTitle }}
          </h2>
          <div
            class="absolute top-32 left-5 text-sm"
            :style="getParallaxStyle(index, -100)"
          >
            {{ props.contents[index]?.content }}
          </div>

          <img
            :src="image.link"
            :alt="image.name"
            draggable="false"
            class="w-full h-full object-cover"
          />
        </div>

        <div v-else>
          <img
            :src="image.link"
            :alt="image.name"
            draggable="false"
            class="w-full h-full object-cover"
          />
        </div>
      </li>
    </ul>

    <slot
      name="prev-btn"
      :goToPrev="prevHandler"
      :defaultClass="'absolute left-4 top-1/2'"
    >
      <button
        v-if="showPrevButton"
        type="button"
        @click="prevHandler"
        :class="`absolute left-4 top-1/2 h-[1.8rem] w-[1.8rem] flex items-center justify-center rounded-full bg-white opacity-40 hover:opacity-60 ${currentIndex === 0 ? 'cursor-not-allowed' : ''}`"
        :disabled="currentIndex === 0"
      >
        <i class="pi pi-angle-left text-gray-800" />
      </button>
    </slot>

    <slot
      name="next-btn"
      :goToNext="nextHandler"
      :defaultClass="'absolute right-4 top-1/2'"
    >
      <button
        v-if="showNextButton"
        type="button"
        @click="nextHandler"
        :class="`absolute right-4 top-1/2 h-[1.8rem] w-[1.8rem] flex items-center justify-center rounded-full bg-white opacity-40 hover:opacity-60 ${currentIndex === imageItems.length - 1 ? 'cursor-not-allowed' : ''}`"
        :disabled="currentIndex === imageItems.length - 1"
      >
        <i class="pi pi-angle-right text-gray-800" />
      </button>
    </slot>

    <ul
      v-if="pagination && imageItems.length > 1"
      class="absolute bottom-4 flex w-full justify-center gap-2"
    >
      <li
        v-for="(_, idx) in imageItems"
        :key="idx"
        :class="`${getPaginationClass(idx)} h-[0.5rem] w-[0.5rem] rounded-full 
        bg-white transition-all duration-300 cursor-pointer`"
        @click="goToImage(idx)"
      />
    </ul>

    <div
      v-if="showScrollbar"
      class="absolute bottom-0 left-0 right-0 h-1 bg-gray-400 transition-opacity duration-300"
      :class="{ 'opacity-100': showScrollbar, 'opacity-0': !showScrollbar }"
    >
      <div
        :style="{
          width: `${((currentIndex + 1) / props.imageItems.length) * 100}%`,
        }"
        class="h-full bg-gray-200 transition-all duration-300"
      />
    </div>
  </div>
</template>

<style lang="scss" scoped>
ul:focus {
  outline: none;
}

.blur-load {
  background-size: cover;
  background-position: center;
}
</style>
