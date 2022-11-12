<template>
  <div class="container">
    <div class="download-logo fr ac jc">
      <img src="../assets/images/logo.png" alt="">
    </div>
    <div class="download-box">
      <a-textarea v-model:value="videoUrl" placeholder="请输入视频地址" :auto-size="{ minRows: 2 }" />
      <center>
        <a-button :loading="loading" type="primary" @click="download()" style="margin-top: 30px;">下载</a-button>
      </center>
    </div>
    <div class="setting">
      <SettingOutlined :style="{ fontSize: '18px' }" @click="settingDrawer.open()" />
    </div>
    <div class="user">
      <UserOutlined :style="{ fontSize: '18px' }" @click="userModal.toogleVisible()" />
    </div>
  </div>
  <UserModal ref="userModal" />
  <SettingDrawer ref="settingDrawer" />
  <LoginModal ref="loginModal" />
  <VideoModal ref="videoModal" />
</template>

<script lang="ts" setup>
import { message } from 'ant-design-vue'
import { UserOutlined, ArrowDownOutlined, SettingOutlined, LoadingOutlined } from '@ant-design/icons-vue'
import { ref, toRaw } from 'vue'
import { store } from '../store'
import { checkUrl, checkLogin, checkUrlRedirect, parseHtml, getDownloadList, addDownload } from '../core/bilibili'
import UserModal from '../components/UserModal/index.vue'
import SettingDrawer from '../components/SettingDrawer/index.vue'
import LoginModal from '../components/LoginModal/index.vue'
import VideoModal from '../components/VideoModal/index.vue'
import { sleep } from '@/utils/sleep'
import router from '@/router'
import { userQuality } from '@/assets/data/quality'
import { VideoData } from '@/type'

const videoUrl = ref<string | null>(null)
const loading = ref<boolean>(false)
const userModal = ref<any>(null)
const settingDrawer = ref<any>(null)
const loginModal = ref<any>(null)
const videoModal = ref<any>(null)

const showContextmenu = () => {
  window.electron.showContextmenu('home')
}

const handleDownload = async (videoInfo: VideoData, selected: number[], quality: number) => {
  // 获取当前选中视频的下载数据
  const list = await getDownloadList(toRaw(videoInfo), toRaw(selected), quality)
  console.log(list)
  const taskList = addDownload(list)
  store.taskStore().setTask(taskList)
  let count = 0
  let selectedTask = ''
  for (const key in taskList) {
    const task = taskList[key]
    if (task.status === 1) {
      window.electron.downloadVideo(task)
      count += 1
      if (!selectedTask) selectedTask = task.id
    }
    await sleep(300)
  }
  store.baseStore().addDownloadingTaskCount(count)
  store.taskStore().setRightTaskId(selectedTask)
}

const open = async (data: VideoData) => {
  const qualities = userQuality[store.baseStore().loginStatus]
  data.qualityOptions.filter((item: any) => qualities.includes(item.value))
  // visible.value = true
  // 如果是单p，则默认选中
  // if (videoInfo.value.page.length === 1) {
  //   selected.value.push(videoInfo.value.page[0].page)
  // }

  let quality = data.qualityOptions[0].value
  for (let i = 0; i < data.qualityOptions.length; i++) {
    if (data.qualityOptions[i].label.includes('720')) {
      quality = data.qualityOptions[i].value
    }
  }
  const videoInfo = data
  const selected: number[] = []
  // 默认全选中
  videoInfo.page.forEach(element => {
    selected.push(element.page)
  })

  await handleDownload(videoInfo, selected, quality)
}

const downloadOne = async (videoUrl: string) => {
  console.log('downloadOne')

  const videoType = checkUrl(videoUrl)
  if (!videoType) {
    message.error('请输入正确的视频地址')
    loading.value = false
    return
  }
  // 不检查登陆状态
  // if (store.baseStore().allowLogin) {
  //   const status = await checkLogin(store.settingStore().SESSDATA)
  //   store.baseStore().setLoginStatus(status)
  //   if (status === 0) {
  //     loginModal.value.open()
  //     loading.value = false
  //     return
  //   }
  // }
  // 检查是否有重定向
  const { body, url } = await checkUrlRedirect(videoUrl)
  // 解析html
  try {
    const videoInfo = await parseHtml(body, videoType, url)
    if (videoInfo === -1) {
      throw videoInfo
    }
    // loading.value = false
    await open(videoInfo)
  } catch (error: any) {
    // loading.value = false
    if (error === -1) {
      message.error('解析错误或者不支持当前视频')
    } else {
      message.error(`解析错误：${error}`)
    }
  }
}

const download = async () => {
  console.log('download')
  loading.value = true
  if (!videoUrl.value) {
    message.warn('请输入视频地址')
    loading.value = false
    return
  }

  const urls = videoUrl.value.split('\n')
  urls.forEach(async element => {
    await downloadOne(element)
  })

  loading.value = false
  router.push({ name: 'download' })
}
</script>

<style lang="less" scoped>
.container {
  box-sizing: border-box;
  padding: 16px;
  position: relative;
  height: calc(100% - 28px);

  .download-logo {
    margin: 130px 0px 50px 0px;

    img {
      transform: scale(.6);
    }
  }

  .download-box {
    padding: 0px 64px;

    :deep(.ant-input-group-addon) {
      background: @primary-color;
      border: none;
    }

    .icon {
      color: #ffffff;
      font-size: 18px;
    }
  }

  .setting {
    position: absolute;
    left: 16px;
    bottom: 16px;
    z-index: 100;
    color: @primary-color;
    font-size: 16px;
  }

  .user {
    position: absolute;
    right: 16px;
    bottom: 16px;
    z-index: 100;
    color: @primary-color;
    font-size: 16px;
  }
}
</style>
