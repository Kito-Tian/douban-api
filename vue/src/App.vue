<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';
import icRatingS from './assets/ic_rating_s.png';
import axios from 'axios';

// 根据评分生成样式对象，用于设置评分图标的背景位置
const getRatingStyle = (rating) => {
  return {
    backgroundImage: `url(${icRatingS})`, // 设置背景图片
    backgroundPositionY: `-${(5 - rating) * 22}px`, // 根据评分调整背景图片的位置
  };
};

// 定义 movies 和 isMobile 的响应式变量
const movies = ref([]); // 电影列表
const isMobile = ref(false); // 判断是否为移动端

// 打开电影详情页
const openMovieDetail = (url) => {
  if (url && url !== '#') {
    window.open(url, '_blank'); // 在新标签页中打开链接
  }
};

// 从电影简介中提取信息（如上映日期、时长）
const extractMovieInfo = (intro, isMobile = false) => {
  const dateRegex = /\d{4}-\d{2}-\d{2}\([^)]+\)/g; // 匹配上映日期的正则表达式
  const releaseDates = intro.match(dateRegex) || []; // 提取所有上映日期

  const durationRegex = /\d+分钟/; // 匹配时长的正则表达式
  const durationMatch = intro.match(durationRegex); // 提取时长
  const duration = durationMatch ? durationMatch[0] : null;

  // 如果是移动设备，返回简化的信息（如果都保留，会过长，不好看）
  if (isMobile) {
    const year = releaseDates.length > 0 ? releaseDates[0].substring(0, 4) + '年上映' : '';
    return `${year}${duration ? ' / ' + duration : ''}`;
  }

  // 返回完整的信息
  let info = releaseDates.join(' / ');
  if (duration) {
    info += ` / ${duration}`;
  }

  return info;
};

// 提取电影主标题（如“星际穿越 / Interstellar”只保留“星际穿越”）
const extractMainTitle = (title) => {
  const index = title.indexOf('/'); // 找到副标题的分隔符
  return index !== -1 ? title.substring(0, index).trim() : title;
};

// 检查是否为移动设备
const checkIsMobile = () => {
  isMobile.value = window.innerWidth < 768; // 根据窗口宽度判断
};

// 组件挂载时执行
onMounted(async () => {
  checkIsMobile(); // 初始化设备类型
  window.addEventListener('resize', checkIsMobile); // 监听窗口大小变化

  try {
    // 通过代理服务获取豆瓣电影列表页面的 HTML 内容（避免跨域问题，在 README 中已经说明）
    const response = await axios.get('https://api.allorigins.win/get?url=https://movie.douban.com/people/178287633/collect');
    const htmlContent = response.data.contents; // 获取 HTML 内容

    // 使用 DOMParser 解析 HTML
    const parser = new DOMParser();
    const doc = parser.parseFromString(htmlContent, 'text/html');

    // 获取所有电影条目
    const items = doc.querySelectorAll('.comment-item');

    // 遍历每个电影条目，提取信息并存入 movies 数组
    items.forEach(item => {
      const rawTitle = item.querySelector('.title em')?.textContent.trim() || '未知电影'; // 电影标题
      const title = rawTitle;
      const mainTitle = extractMainTitle(title); // 提取主标题
      const cover = item.querySelector('.pic img')?.src || ''; // 电影封面
      const rating = item.querySelector('.rating1-t, .rating2-t, .rating3-t, .rating4-t, .rating5-t')?.className.replace('rating', '').replace('-t', '') || '无评分'; // 评分
      const comment = item.querySelector('.comment')?.textContent.trim() || '无评价'; // 用户评价
      const detailUrl = item.querySelector('.title a')?.href || '#'; // 详情页链接
      const intro = item.querySelector('.intro')?.textContent.trim() || ''; // 电影简介
      const commentDate = item.querySelector('.date')?.textContent.trim() || ''; // 评价日期

      const movieInfo = extractMovieInfo(intro); // 提取电影信息

      // 使用代理服务加载封面图片，避免跨域问题
      const proxiedCover = cover ? `https://images.weserv.nl/?url=${encodeURIComponent(cover)}` : '';

      // 将电影信息存入 movies 数组
      movies.value.push({
        title,
        mainTitle,
        cover: proxiedCover,
        rating,
        comment,
        detailUrl,
        movieInfo,
        commentDate,
      });
    });

    console.log(movies.value); // 打印电影列表
  } catch (error) {
    console.error('Error fetching movies:', error);
  }
});

// 组件卸载时执行
onBeforeUnmount(() => {
  window.removeEventListener('resize', checkIsMobile); // 移除窗口大小变化监听器
});
</script>

<template>
  <div class="movie-list">
    <!-- 遍历 movies 数组，渲染每个电影卡片 -->
    <div v-for="movie in movies" :key="movie.title" class="movie-card">
      <!-- 电影封面 -->
      <div class="movie-cover-wrapper" @click="openMovieDetail(movie.detailUrl)">
        <img :src="movie.cover" :alt="movie.title" class="movie-cover" />
      </div>
      <!-- 电影信息 -->
      <div class="movie-info">
        <!-- 电影标题 -->
        <div class="movie-title">
          <span @click="openMovieDetail(movie.detailUrl)">
            {{ isMobile ? movie.mainTitle : movie.title }}
          </span>
        </div>
        <!-- 电影信息（上映日期、时长） -->
        <p class="movie-info-text" v-if="movie.movieInfo">
          {{ isMobile ? extractMovieInfo(movie.movieInfo, true) : movie.movieInfo }}
        </p>
        <!-- 电影评分 -->
        <p class="movie-rating">
          <span
            v-if="movie.rating !== '无评分'"
            class="rating-icon"
            :style="getRatingStyle(movie.rating)"
          ></span>
          <span v-else>无评分</span>
          <span class="rating-date" v-if="movie.commentDate">{{ movie.commentDate }}</span>
        </p>
        <!-- 用户评价 -->
        <div class="movie-comment">
          <span>{{ movie.comment }}</span>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* 全局样式 */
body {
  font-family: -apple-system, BlinkMacSystemFont, Segoe UI, Roboto, Helvetica Neue, Arial, Noto Sans, Liberation Sans, sans-serif, Apple Color Emoji, Segoe UI Emoji, Segoe UI Symbol, Noto Color Emoji;
  background-color: #f9f9f9;
  margin: 0;
  padding: 0;
}

/* 电影列表容器 */
.movie-list {
  max-width: 800px;
  margin: 20px auto;
}

/* 电影卡片样式 */
.movie-card {
  display: flex;
  border-radius: 4px;
  box-shadow: rgba(0, 0, 0, 0.05) 0px 0px 0px 1px;
  margin-bottom: 20px;
  overflow: hidden;
  width: 100%;
}

/* 电影封面容器 */
.movie-cover-wrapper {
  width: 18%;
  height: 200px;
  display: flex;
  cursor: pointer;
  background-color: #f0f0f0;
}

/* 电影封面图片 */
.movie-cover {
  width: 100%;
  height: auto;
  object-fit: cover;
  border-radius: 4px;
}

/* 电影信息容器 */
.movie-info {
  flex: 1;
  padding: 0 0 10px 10px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

/* 电影标题 */
.movie-title {
  margin-bottom: 10px;
  padding: 5px;
  font-size: 15px;
  color: #3377AA;
  border-radius: 4px;
  text-align: start;
  background-color: #F2F4F6;
}

/* 电影标题悬停效果 */
.movie-title span:hover {
  cursor: pointer;
  color: #fff;
  background-color: #3377AA;
}

/* 电影信息文本 */
.movie-info-text {
  margin: 0 0 6px;
  font-size: 13px;
  color: #666;
  text-align: start;
}

/* 电影评分 */
.movie-rating {
  margin: 0 0 6px;
  font-size: 15px;
  color: #666;
  text-align: start;
  display: flex;
  align-items: center;
  gap: 6px;
}

/* 评分日期 */
.movie-rating .rating-date {
  font-size: 13px;
  color: rgb(102, 102, 102);
}

/* 评分星级 */
.rating-icon {
  display: inline-block;
  width: 55px;
  height: 11px;
  background-repeat: no-repeat;
  background-position-x: 0;
}

/* 评价 */
.movie-comment {
  margin: 0 10px 0 0;
  font-size: 15px;
  color: #444;
  line-height: 1;
  text-align: start;
  border-left: #4870AC solid 2px;
  background-color: #F5F8FE;
  padding: 10px;
}

/* 移动端样式 */
@media (max-width: 768px) {
  .movie-comment {
    margin-top: 7px;
    font-size: 13px;
    line-height: 1.5;
  }
  .movie-cover-wrapper {
    width: 25%;
    height: 150px;
  }
  .movie-info-text {
    font-size: 12px;
  }
}
</style>