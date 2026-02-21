<template>
  <div :class="store.backgroundShow ? 'cover show' : 'cover'">
    <img
      v-show="store.imgLoadStatus"
      :src="bgUrl"
      class="bg"
      alt="cover"
      @load="imgLoadComplete"
      @error.once="imgLoadError"
      @animationend="imgAnimationEnd"
    />
    <div :class="store.backgroundShow ? 'gray hidden' : 'gray'" />
    <Transition name="fade" mode="out-in">
      <a
        v-if="store.backgroundShow && store.coverType != '3'"
        class="down"
        :href="bgUrl"
        target="_blank"
      >
        下载壁纸
      </a>
    </Transition>
  </div>
</template>

<script setup>
// 补全所有必要的导入
import { ref, watch, onMounted, onBeforeUnmount, h } from "vue";
import { mainStore } from "@/store";
import { Error } from "@icon-park/vue-next";
import { ElMessage } from "element-plus";

const store = mainStore();
const bgUrl = ref(null);
const imgTimeout = ref(null);
const emit = defineEmits(["loadComplete"]);

// 读取环境变量中的 Token（部署时注入，本地写在 .env.local）
const IMG_BED_TOKEN = import.meta.env.VITE_IMGBED_TOKEN || "";

// 壁纸随机数
const bgRandom = Math.floor(Math.random() * 9 + 1);

// 更换壁纸链接（异步函数，支持 API 鉴权）
const changeBg = async (type) => {
  // 重置加载状态，优化体验
  store.setImgLoadStatus(false);
  
  if (type == 0) {
    bgUrl.value = `/images/background${bgRandom}.jpg`;
  } else if (type == 1) {
    // 必应壁纸接口（稳定）
    bgUrl.value = "https://api.dujin.org/bing/1920.php";
  } else if (type == 2) {
    // 彼岸壁纸-风景类
    bgUrl.value = "https://api.btstu.cn/sjbz/?lx=fengjing&format=images";
  } else if (type == 3) {
    // 彼岸壁纸-动漫类
    bgUrl.value = "https://api.btstu.cn/sjbz/?lx=dongman&format=images";
  } else if (type == 4) {
    // 无 Token 时兜底提示
    if (!IMG_BED_TOKEN) {
      ElMessage({
        message: "未配置图片床 Token，已切换默认壁纸",
        type: "warning",
        icon: h(Error, { theme: "filled", fill: "#efefef" }),
      });
      bgUrl.value = `/images/background${bgRandom}.jpg`;
      return;
    }

    // 带鉴权调用自定义随机图 API
    try {
      const response = await fetch("https://tu.fqzlr.top/random", {
        method: "GET",
        headers: {
          // 按 CloudFlare ImgBed API 文档设置鉴权头
          "Authorization": IMG_BED_TOKEN,
          "Accept": "image/*",
        },
      });

      if (!response.ok) {
        throw new Error(`API 请求失败：${response.status}`);
      }

      // 将图片转为 Base64（解决 img 标签无法设置请求头的问题）
      const blob = await response.blob();
      const reader = new FileReader();
      reader.onload = () => {
        bgUrl.value = reader.result; // Base64 字符串作为 img.src
      };
      reader.readAsDataURL(blob);
    } catch (error) {
      console.error("自定义随机图加载失败：", error);
      // 失败兜底
      bgUrl.value = `/images/background${bgRandom}.jpg`;
      ElMessage({
        message: "自定义壁纸加载失败，已切换回默认",
        icon: h(Error, { theme: "filled", fill: "#efefef" }),
      });
    }
  }
};

// 图片加载完成
const imgLoadComplete = () => {
  imgTimeout.value = setTimeout(
    () => {
      store.setImgLoadStatus(true);
    },
    Math.floor(Math.random() * (600 - 300 + 1)) + 300,
  );
};

// 图片动画完成
const imgAnimationEnd = () => {
  console.log("壁纸加载且动画完成");
  // 加载完成事件
  emit("loadComplete");
};

// 图片显示失败
const imgLoadError = () => {
  console.error("壁纸加载失败：", bgUrl.value);
  ElMessage({
    message: "壁纸加载失败，已临时切换回默认",
    icon: h(Error, {
      theme: "filled",
      fill: "#efefef",
    }),
  });
  bgUrl.value = `/images/background${bgRandom}.jpg`;
};

// 监听壁纸切换
watch(
  () => store.coverType,
  (value) => {
    changeBg(value);
  },
);

onMounted(() => {
  // 加载壁纸
  changeBg(store.coverType);
});

onBeforeUnmount(() => {
  clearTimeout(imgTimeout.value);
});
</script>

<style lang="scss" scoped>
.cover {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  transition: 0.25s;
  z-index: -1;

  &.show {
    z-index: 1;
  }

  .bg {
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
    backface-visibility: hidden;
    filter: blur(20px) brightness(0.3);
    transition:
      filter 0.3s,
      transform 0.3s;
    animation: fade-blur-in 0.8s cubic-bezier(0.25, 0.46, 0.45, 0.94) forwards;
    animation-delay: 0.45s;
  }
  .gray {
    opacity: 1;
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    background-image: radial-gradient(rgba(0, 0, 0, 0) 0, rgba(0, 0, 0, 0.5) 100%),
      radial-gradient(rgba(0, 0, 0, 0) 33%, rgba(0, 0, 0, 0.3) 166%);

    transition: 1.5s;
    &.hidden {
      opacity: 0;
      transition: 1.5s;
    }
  }
  .down {
    font-size: 16px;
    color: white;
    position: absolute;
    bottom: 30px;
    left: 0;
    right: 0;
    margin: 0 auto;
    display: block;
    padding: 20px 26px;
    border-radius: 8px;
    background-color: #00000030;
    width: 120px;
    height: 30px;
    display: flex;
    justify-content: center;
    align-items: center;
    &:hover {
      transform: scale(1.05);
      background-color: #00000060;
    }
    &:active {
      transform: scale(1);
    }
  }
}

// 补全缺失的动画定义
@keyframes fade-blur-in {
  from {
    filter: blur(40px) brightness(0);
    opacity: 0;
  }
  to {
    filter: blur(20px) brightness(0.3);
    opacity: 1;
  }
}
</style>