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
// 补全Vue核心API和必要组件导入（避免运行报错）
import { ref, watch, onMounted, onBeforeUnmount, h } from "vue";
import { mainStore } from "@/store";
import { Error } from "@icon-park/vue-next";
import { ElMessage } from "element-plus";

const store = mainStore();
const bgUrl = ref(null);
const imgTimeout = ref(null);
const emit = defineEmits(["loadComplete"]);

// 【安全核心】从Vite环境变量读取Token，代码中无硬编码（部署时注入）
const IMG_BED_TOKEN = import.meta.env.VITE_IMGBED_TOKEN || "";
// 适配你的图床域名：https://tu.fqzlr.top/
const IMG_BED_BASE_URL = "https://tu.fqzlr.top";
// 适配API文档的参数：type=img直接返回图片、auto自适应方向
const IMG_API_PARAMS = {
  type: "img",        // 必选：直接返回图片流（无需解析JSON）
  orientation: "auto",// 自适应设备方向（桌面横图/手机竖图）
  content: "image"    // 只返回图片（过滤视频）
};

// 拼接带参数的API请求地址
const getRandomImgUrl = () => {
  const params = new URLSearchParams(IMG_API_PARAMS);
  return `${IMG_BED_BASE_URL}/random?${params.toString()}`;
};

// 本地壁纸随机数
const bgRandom = Math.floor(Math.random() * 9 + 1);

// 更换壁纸逻辑（异步+鉴权+容错）
const changeBg = async (type) => {
  store.setImgLoadStatus(false);
  
  if (type === 0) {
    bgUrl.value = `/images/background${bgRandom}.jpg`;
  } else if (type === 1) {
    bgUrl.value = "https://api.dujin.org/bing/1920.php";
  } else if (type === 2) {
    bgUrl.value = "https://api.btstu.cn/sjbz/?lx=fengjing&format=images";
  } else if (type === 3) {
    bgUrl.value = "https://api.btstu.cn/sjbz/?lx=dongman&format=images";
  } else if (type === 4) {
    // 无Token时兜底（避免报错）
    if (!IMG_BED_TOKEN) {
      ElMessage({
        message: "未配置图片床Token，已切换默认壁纸",
        type: "warning",
        icon: h(Error, { theme: "filled", fill: "#efefef" }),
      });
      bgUrl.value = `/images/background${bgRandom}.jpg`;
      return;
    }

    try {
      // 带鉴权请求你的随机图API
      const response = await fetch(getRandomImgUrl(), {
        method: "GET",
        headers: {
          "Authorization": IMG_BED_TOKEN, // 按API文档设置鉴权头
          // 携带视口信息，让API精准判断设备方向
          "Sec-CH-Viewport-Width": window.innerWidth.toString(),
          "Sec-CH-Viewport-Height": window.innerHeight.toString(),
        },
      });

      if (!response.ok) throw new Error(`请求失败：${response.status}`);
      
      // 转为Blob URL（替代Base64，更高效）
      const blob = await response.blob();
      bgUrl.value = URL.createObjectURL(blob);
    } catch (error) {
      console.error("自定义随机图加载失败：", error);
      // 失败兜底：切回本地壁纸
      bgUrl.value = `/images/background${bgRandom}.jpg`;
      ElMessage({
        message: "自定义壁纸加载失败，已切换回默认",
        icon: h(Error, { theme: "filled", fill: "#efefef" }),
      });
    }
  }
};

// 图片加载完成处理
const imgLoadComplete = () => {
  imgTimeout.value = setTimeout(
    () => store.setImgLoadStatus(true),
    Math.floor(Math.random() * (600 - 300 + 1)) + 300
  );
};

// 图片动画完成
const imgAnimationEnd = () => {
  console.log("壁纸加载且动画完成");
  emit("loadComplete");
};

// 图片加载失败兜底
const imgLoadError = () => {
  console.error("壁纸加载失败：", bgUrl.value);
  ElMessage({
    message: "壁纸加载失败，已临时切换回默认",
    icon: h(Error, { theme: "filled", fill: "#efefef" }),
  });
  bgUrl.value = `/images/background${bgRandom}.jpg`;
};

// 监听壁纸类型切换
watch(() => store.coverType, (value) => changeBg(value));
// 监听窗口大小变化，重新加载自适应方向的图片
watch([() => window.innerWidth, () => window.innerHeight], () => {
  if (store.coverType === 4) changeBg(4);
}, { deep: true });

onMounted(() => changeBg(store.coverType));
onBeforeUnmount(() => clearTimeout(imgTimeout.value));
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
    transition: filter 0.3s, transform 0.3s;
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
    background-image: 
      radial-gradient(rgba(0, 0, 0, 0) 0, rgba(0, 0, 0, 0.5) 100%),
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
    display: flex;
    justify-content: center;
    align-items: center;
    width: 120px;
    height: 30px;
    padding: 20px 26px;
    border-radius: 8px;
    background-color: #00000030;
    text-decoration: none;

    &:hover {
      transform: scale(1.05);
      background-color: #00000060;
    }

    &:active {
      transform: scale(1);
    }
  }
}

// 补全动画定义（避免样式失效）
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