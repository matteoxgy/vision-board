<template>
  <div class="vision-board">
    <div class="toolbar">
      <label class="upload-button">
        <input
          type="file"
          @change="handleImageUpload"
          accept="image/*"
          :disabled="imageLoading"
          ref="fileInput"
        />
        <span>上传图片</span>
      </label>
      <span v-if="imageLoading" class="loading-text">
        <i class="loading-icon">⟳</i>图片加载中...
      </span>
    </div>

    <div class="board-container">
      <!-- 愿景板主区域 -->
      <div class="board" ref="boardRef">
        <vue-draggable-resizable
          v-for="(item, index) in boardItems"
          :key="index"
          :x="item.x"
          :y="item.y"
          :w="item.width"
          :h="item.height"
          :draggable="true"
          :resizable="true"
          :lock-aspect-ratio="true"
          :min-width="50"
          :min-height="50"
          :aspect-ratio="item.originalWidth / item.originalHeight"
          @activated="selectedIndex = index"
          @deactivated="selectedIndex = -1"
          @dragstop="(x, y) => updatePosition(index, x, y)"
          @resizestop="
            (x, y, width, height) => updateSize(index, x, y, width, height)
          "
          :style="{
            transform: `rotate(${item.rotation}deg)`,
            transformOrigin: 'center center',
          }"
        >
          <div
            class="image-container"
            :style="{
              transform: `rotate(${item.rotation}deg), transformOrigin: 'center center'`,
            }"
          >
            <img
              :src="item.imageUrl"
              :alt="'Vision item ' + index"
              class="draggable-item"
            />
            <div
              class="rotate-handle"
              @mousedown.stop="(e) => startRotating(e, index)"
              @touchstart.stop="(e) => startRotating(e, index)"
            >
              ⟲
            </div>
          </div>
        </vue-draggable-resizable>
      </div>
    </div>

    <!-- 裁剪弹窗 -->
    <div v-if="showCropper" class="cropper-modal">
      <div class="cropper-container">
        <vue-cropper
          ref="cropperRef"
          :img="selectedImage"
          :aspect-ratio="NaN"
          :auto-crop="true"
          :center-box="true"
          :output-size="1"
          :output-type="'png'"
          :info="true"
          :full="false"
          :can-move="true"
          :can-scale="true"
          :high="true"
          :mode="'contain'"
          style="width: 500px; height: 400px"
          @ready="cropperReady"
          @error="(e) => console.error('裁剪器错误:', e)"
        />
        <div class="cropper-buttons">
          <button @click="confirmCrop">确认</button>
          <button @click="cancelCrop">取消</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { reactive, ref } from "vue";
import { VueCropper } from "vue-cropper";
import "vue-cropper/dist/index.css";
import VueDraggableResizable from "vue3-draggable-resizable";
import "vue3-draggable-resizable/dist/Vue3DraggableResizable.css";

const boardItems = reactive([]);
const selectedImage = ref(null);
const showCropper = ref(false);
const cropperRef = ref(null);
const boardRef = ref(null);
const imageLoading = ref(false);
const selectedIndex = ref(-1);

// 添加旋转相关的响应式变量
const isRotating = ref(false);
const rotationStartAngle = ref(0);

// 修改 cropperReady 方法
const cropperReady = () => {
  console.log("裁剪器已就绪");
  // 给裁剪器一些时间初始化
  setTimeout(() => {
    if (cropperRef.value) {
      cropperRef.value.refresh();
    }
  }, 100);
};

// 修改图片上传处理方法
const fileInput = ref(null);

const handleImageUpload = (event) => {
  const file = event.target.files[0];
  if (!file) return;

  imageLoading.value = true;
  const reader = new FileReader();

  reader.onload = () => {
    selectedImage.value = reader.result;
    showCropper.value = true;
    imageLoading.value = false;

    // 清除 input 的值，这样相同的图片可以重复上传
    if (fileInput.value) {
      fileInput.value.value = '';
    }

    // 给一些时间让模态框显示
    setTimeout(() => {
      if (cropperRef.value) {
        cropperRef.value.refresh();
      }
    }, 200);
  };

  reader.onerror = (error) => {
    console.error("图片读取失败:", error);
    imageLoading.value = false;
    // 发生错误时也清除 input 的值
    if (fileInput.value) {
      fileInput.value.value = '';
    }
  };

  reader.readAsDataURL(file);
};

// 修改确认裁剪方法
const confirmCrop = () => {
  if (!cropperRef.value) return;

  cropperRef.value.getCropData((data) => {
    if (data) {
      const img = new Image();
      img.onload = () => {
        const maxSize = 200;
        let width = img.width;
        let height = img.height;

        // 保持宽高比的同时缩放到合适大小
        if (width > height) {
          if (width > maxSize) {
            height = (height * maxSize) / width;
            width = maxSize;
          }
        } else {
          if (height > maxSize) {
            width = (width * maxSize) / height;
            height = maxSize;
          }
        }

        const boardWidth = 800;
        const boardHeight = 600;
        const x = (boardWidth - width) / 2;
        const y = (boardHeight - height) / 2;

        boardItems.push({
          imageUrl: data,
          width: Math.round(width),
          height: Math.round(height),
          x,
          y,
          rotation: 0,
          scale: 1,
          originalWidth: img.width, // 保存原始宽度
          originalHeight: img.height, // 保存原始高度
        });
      };
      img.src = data;
    }
    showCropper.value = false;
    selectedImage.value = null;
  });
};

// 修改取消裁剪方法
const cancelCrop = () => {
  // 清理 URL 对象
  if (selectedImage.value && selectedImage.value.startsWith("blob:")) {
    URL.revokeObjectURL(selectedImage.value);
  }
  showCropper.value = false;
  selectedImage.value = null;
};

// 拖拽处理
const updatePosition = (index, x, y) => {
  const item = boardItems[index];
  boardItems[index] = {
    ...item,
    x: Math.max(0, Math.min(x, 800 - item.width)), // 限制在画布范围内
    y: Math.max(0, Math.min(y, 600 - item.height)),
  };
};

// 调整大小处理
const updateSize = (index, x, y, width, height) => {
  const item = boardItems[index];

  // 确保位置和尺寸在画布范围内
  const newX = Math.max(0, Math.min(x, 800 - width));
  const newY = Math.max(0, Math.min(y, 600 - height));

  boardItems[index] = {
    ...item,
    x: newX,
    y: newY,
    width,
    height,
  };
};

// 修改开始旋转的处理函数
const startRotating = (event, index) => {
  event.stopPropagation();
  isRotating.value = true;
  const item = boardItems[index];

  // 获取触摸或鼠标事件的坐标
  const clientX = event.touches ? event.touches[0].clientX : event.clientX;
  const clientY = event.touches ? event.touches[0].clientY : event.clientY;

  // 获取元素中心点
  const rect = event.target.parentElement.getBoundingClientRect();
  const centerX = rect.left + rect.width / 2;
  const centerY = rect.top + rect.height / 2;

  // 计算初始角度
  const startAngle = (Math.atan2(clientY - centerY, clientX - centerX) * 180) / Math.PI;
  rotationStartAngle.value = startAngle - (item.rotation || 0);

  // 处理移动事件的函数
  const handleMove = (e) => {
    e.preventDefault();
    if (!isRotating.value) return;

    const moveX = e.touches ? e.touches[0].clientX : e.clientX;
    const moveY = e.touches ? e.touches[0].clientY : e.clientY;
    
    const angle = (Math.atan2(moveY - centerY, moveX - centerX) * 180) / Math.PI;
    let newRotation = angle - rotationStartAngle.value;
    newRotation = ((newRotation % 360) + 360) % 360;

    boardItems[index] = {
      ...item,
      rotation: newRotation,
    };
  };

  // 处理结束事件的函数
  const handleEnd = () => {
    isRotating.value = false;
    document.removeEventListener("mousemove", handleMove);
    document.removeEventListener("mouseup", handleEnd);
    document.removeEventListener("touchmove", handleMove);
    document.removeEventListener("touchend", handleEnd);
  };

  // 添加事件监听器
  document.addEventListener("mousemove", handleMove);
  document.addEventListener("mouseup", handleEnd);
  document.addEventListener("touchmove", handleMove, { passive: false });
  document.addEventListener("touchend", handleEnd);
};
</script>

<style scoped>
.vision-board {
  width: 800px;
  height: 600px;
  position: relative;
  overflow: hidden;
  border: 1px solid #ddd;
  border-radius: 8px;
  background: white;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
}

.toolbar {
  padding: 16px;
  border-bottom: 1px solid #eee;
  background: #f8f9fa;
}

.upload-button {
  display: inline-block;
  padding: 8px 16px;
  background: #4a90e2;
  color: white;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.3s ease;
}

.upload-button:hover {
  background: #357abd;
  transform: translateY(-1px);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.upload-button input {
  display: none;
}

.loading-text {
  margin-left: 12px;
  color: #666;
  display: inline-flex;
  align-items: center;
  gap: 8px;
}

.loading-icon {
  display: inline-block;
  animation: spin 1s linear infinite;
}

.board-container {
  width: 100%;
  height: calc(100vh - 100px);
  border: 2px dashed #ccc;
  position: relative;
  overflow: hidden;
}

.board {
  width: 100%;
  height: 100%;
  position: relative;
}

.cropper-modal {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  animation: fadeIn 0.3s ease;
}

.cropper-container {
  background: white;
  padding: 24px;
  border-radius: 12px;
  min-width: 600px;
  min-height: 500px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
  box-shadow: 0 4px 24px rgba(0, 0, 0, 0.2);
  animation: slideUp 0.3s ease;
}

.cropper-buttons {
  margin-top: 20px;
  display: flex;
  gap: 16px;
}

.cropper-buttons button {
  padding: 10px 24px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 500;
  transition: all 0.2s ease;
}

.cropper-buttons button:first-child {
  background: #4a90e2;
  color: white;
}

.cropper-buttons button:last-child {
  background: #f0f0f0;
  color: #666;
}

.cropper-buttons button:hover {
  transform: translateY(-1px);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* 添加拖拽项的样式限制 */
.draggable-item {
  max-width: 100%;
  max-height: 100%;
  position: absolute;
  cursor: move;
  user-select: none;
}

.draggable-item img {
  width: 100%;
  height: 100%;
  object-fit: contain;
}

.vdr {
  border: 1px solid transparent;
  position: absolute;
  transform-origin: center center;
  transition: border-color 0.2s ease;
}

.vdr.active {
  border: 2px solid #4a90e2;
  box-shadow: 0 0 0 1px rgba(74, 144, 226, 0.3);
}

.vdr .handle {
  background-color: #4a90e2;
  border: 2px solid white;
  box-shadow: 0 0 4px rgba(0, 0, 0, 0.2);
  transition: transform 0.2s ease;
}

.vdr .handle:hover {
  transform: scale(1.2);
}

/* 调整手柄样式 */
.handle {
  background-color: #00a8ff;
  border: 1px solid white;
  box-shadow: 0 0 2px rgba(0, 0, 0, 0.2);
}

/* 确保图片在容器内正确显示 */
.draggable-item {
  width: 100%;
  height: 100%;
  overflow: hidden;
}

.draggable-item img {
  width: 100%;
  height: 100%;
  object-fit: contain;
  pointer-events: none;
}

/* 旋转控制手柄样式 */
.rotate-handle {
  position: absolute;
  top: -30px;
  left: 50%;
  transform: translateX(-50%);
  width: 24px;
  height: 24px;
  background-color: #4a90e2;
  border-radius: 50%;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  z-index: 100;
  transition: all 0.2s ease;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
}

.rotate-handle:hover {
  background-color: #357abd;
  transform: translateX(-50%) scale(1.1);
}

/* 确保旋转时图片容器样式正确 */
.draggable-item {
  width: 100%;
  height: 100%;
  position: relative;
  transform-origin: center center;
}

.draggable-item img {
  width: 100%;
  height: 100%;
  object-fit: contain;
  pointer-events: none;
}

/* 添加选中状态的视觉效果 */
.vdr.active .draggable-item::after {
  content: "";
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  border: 2px solid #00a8ff;
  pointer-events: none;
}

.image-container {
  width: 100%;
  height: 100%;
  position: relative;
  transform-origin: center center;
}
</style>
