<template>
  <div class="running-player" :style="backgroundStyle">
    <div class="overlay">
      <div class="container">
        <h1 class="title">🏃‍♂️ ランニング音楽プレーヤー 🎵</h1>

        <!-- タイマー表示 -->
        <div class="timer-section">
          <div class="timer-display">
            <div class="time">{{ formattedTime }}</div>
            <div class="timer-label">ランニング時間</div>
          </div>
          <div class="progress-bar">
            <div
              class="progress"
              :style="{ width: progressPercentage + '%' }"
            ></div>
          </div>
          <div class="timer-controls">
            <button @click="toggleTimer" class="timer-btn">
              {{ isRunning ? '⏸ 一時停止' : '▶ 開始' }}
            </button>
            <button
              @click="resetTimer"
              class="timer-btn reset-btn"
              :disabled="remainingSeconds === totalSeconds"
            >
              🔄 リセット
            </button>
          </div>
        </div>

        <!-- 音楽コントロール -->
        <div class="music-section">
          <div class="music-info" v-if="currentMusic">
            <h3>現在再生中</h3>
            <p>{{ currentMusic.name }}</p>
            <p class="track-info">
              {{ currentTrackIndex + 1 }} / {{ playlist.length }}
            </p>
          </div>

          <div class="music-controls">
            <button
              @click="prevTrack"
              class="control-btn"
              :disabled="playlist.length === 0"
            >
              ⏮️
            </button>
            <button
              @click="togglePlay"
              class="control-btn play-btn"
              :disabled="playlist.length === 0"
            >
              {{ isPlaying ? '⏸️' : '▶️' }}
            </button>
            <button
              @click="nextTrack"
              class="control-btn"
              :disabled="playlist.length === 0"
            >
              ⏭️
            </button>
            <button
              @click="stopMusic"
              class="control-btn"
              :disabled="playlist.length === 0"
            >
              ⏹️
            </button>
            <button
              class="control-btn"
              @click="toggleShuffleMode"
              :disabled="playlist.length === 0"
              :class="{ active: isShuffleMode }"
              :title="
                isShuffleMode ? 'シャッフル再生をオフ' : 'シャッフル再生をオン'
              "
            >
              {{ isShuffleMode ? '🔀' : '🔁' }}
            </button>
          </div>

          <!-- 音量コントロールバー -->
          <div class="volume-control" v-if="playlist.length > 0">
            <span class="volume-icon">🔊</span>
            <input
              type="range"
              min="0"
              max="1"
              step="0.1"
              v-model="volume"
              @input="updateVolume"
              class="volume-slider"
            />
            <span class="volume-value">{{ Math.round(volume * 100) }}%</span>
          </div>

          <input
            type="file"
            @change="handleFileUpload"
            accept="audio/*"
            class="file-input"
            id="musicUpload"
            style="display: none"
            multiple
          />
          <label for="musicUpload" class="upload-btn">
            📁 音楽をアップロード（複数選択可）
          </label>

          <div class="playlist" v-if="playlist.length > 0">
            <h4>🎵 プレイリスト ({{ playlist.length }}曲)</h4>
            <div
              class="playlist-item"
              v-for="(track, index) in playlist"
              :key="track.url"
              :class="{ active: index === currentTrackIndex }"
              @click="selectTrack(index)"
            >
              <span class="track-number">{{ index + 1 }}.</span>
              <span class="track-name">{{ track.name }}</span>
              <button @click.stop="removeTrack(index)" class="remove-track-btn">
                ❌
              </button>

              <!-- ホバーで詳細情報を表示 -->
              <div class="track-tooltip">
                <div class="tooltip-content">
                  <strong>ファイル名:</strong> {{ track.name }}<br />
                  <strong>サイズ:</strong> {{ formatFileSize(track.size)
                  }}<br />
                  <strong>タイプ:</strong> {{ track.type }}<br />
                  <strong>再生時間:</strong>
                  {{ track.duration || '読み込み中...' }}
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- 背景設定 -->
        <div class="background-section">
          <h3>🎨 カスタム背景</h3>
          <div class="upload-section">
            <label for="background-upload" class="upload-btn">
              📁 背景画像をアップロード
            </label>
            <input
              type="file"
              id="background-upload"
              accept="image/*"
              @change="handleBackgroundUpload"
              style="display: none"
            />
            <div v-if="selectedBackground" class="current-background">
              <img :src="selectedBackground" alt="現在の背景" />
              <button @click="removeBackground" class="remove-btn">
                ❌ 背景を削除
              </button>
            </div>
          </div>
        </div>

        <!-- ステータスメッセージ -->
        <div
          class="status-message"
          :class="`status-${statusMessage?.type || 'info'}`"
          v-if="statusMessage"
        >
          {{ statusMessage.text }}
        </div>
      </div>
    </div>
  </div>

  <!-- 隐藏的音频元素 -->
  <audio ref="audio" style="display: none"></audio>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

// タイマー関連
const totalSeconds = 3600 // 1時間
const remainingSeconds = ref(totalSeconds)
const isRunning = ref(false)
let timerInterval = null

// 音楽関連
const audio = ref(null)
const playlist = ref([])
const currentTrackIndex = ref(0)
const isPlaying = ref(false)
const currentMusic = computed(() => {
  return playlist.value[currentTrackIndex.value] || null
})
const isShuffleMode = ref(false)
const playedIndexes = ref([])
const volume = ref(1) // 音量コントロール、デフォルト100%

// 背景関連
const selectedBackground = ref('')
const statusMessage = ref('')

// 時間表示のフォーマット
const formattedTime = computed(() => {
  const hours = Math.floor(remainingSeconds.value / 3600)
  const minutes = Math.floor((remainingSeconds.value % 3600) / 60)
  const seconds = remainingSeconds.value % 60
  return `${hours.toString().padStart(2, '0')}:${minutes
    .toString()
    .padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`
})

// 進捗パーセンテージ
const progressPercentage = computed(() => {
  return ((totalSeconds - remainingSeconds.value) / totalSeconds) * 100
})

// 背景スタイル
const backgroundStyle = computed(() => {
  return {
    backgroundImage: selectedBackground.value
      ? `url(${selectedBackground.value})`
      : 'linear-gradient(135deg, #667eea 0%, #764ba2 100%)',
    backgroundSize: 'cover',
    backgroundPosition: 'center',
    backgroundAttachment: 'fixed'
  }
})

// タイマーの開始/一時停止
const toggleTimer = () => {
  if (isRunning.value) {
    pauseTimer()
  } else {
    startTimer()
  }
}

const startTimer = () => {
  isRunning.value = true
  timerInterval = setInterval(() => {
    if (remainingSeconds.value > 0) {
      remainingSeconds.value--
    } else {
      pauseTimer()
      showStatus('ランニング時間終了！🎉', 'success')
      if (audio.value) {
        audio.value.pause()
        isPlaying.value = false
      }
    }
  }, 1000)
}

const pauseTimer = () => {
  isRunning.value = false
  if (timerInterval) {
    clearInterval(timerInterval)
    timerInterval = null
  }
}

const resetTimer = () => {
  pauseTimer()
  remainingSeconds.value = totalSeconds
  showStatus('タイマーをリセットしました', 'info')
}

// 音楽コントロール - 複数の曲をプレイリストに追加
const handleFileUpload = event => {
  const files = Array.from(event.target.files)
  if (files.length === 0) return

  const audioFiles = files.filter(
    file =>
      file.type.startsWith('audio/') ||
      file.name.toLowerCase().endsWith('.mp3') ||
      file.name.toLowerCase().endsWith('.wav') ||
      file.name.toLowerCase().endsWith('.m4a') ||
      file.name.toLowerCase().endsWith('.flac')
  )

  if (audioFiles.length === 0) {
    showStatus('有効なオーディオファイルを選択してください', 'error')
    return
  }

  let addedCount = 0
  let skippedCount = 0

  for (const file of audioFiles) {
    try {
      // 同じファイルが既に存在するかチェック（ファイル名とサイズで判断）
      const isDuplicate = playlist.value.some(
        track =>
          track.name === file.name.replace(/\.[^/.]+$/, '') &&
          track.size === file.size
      )

      if (isDuplicate) {
        console.log(`重複ファイルをスキップ: ${file.name}`)
        skippedCount++
        continue
      }

      const url = URL.createObjectURL(file)
      const newTrack = {
        id: Date.now() + Math.random(),
        name: file.name.replace(/\.[^/.]+$/, ''),
        url: url,
        file: file,
        type: file.type,
        size: file.size,
        duration: 0,
        currentTime: 0
      }

      // オーディオの再生時間を取得（変数名の衝突を避けるため別の変数名を使用）
      const tempAudio = new Audio()
      tempAudio.addEventListener('loadedmetadata', () => {
        newTrack.duration = tempAudio.duration
        URL.revokeObjectURL(tempAudio.src)
      })
      tempAudio.addEventListener('error', () => {
        newTrack.duration = 0
        URL.revokeObjectURL(tempAudio.src)
      })
      tempAudio.src = url

      playlist.value.push(newTrack)
      addedCount++
    } catch (error) {
      console.error(`ファイル ${file.name} の処理中にエラー:`, error)
      showStatus(`ファイル ${file.name} の処理に失敗しました`, 'error')
    }
  }

  if (addedCount > 0) {
    let message = `${addedCount} 曲の新しい音楽をプレイリストに追加しました`
    if (skippedCount > 0) {
      message += `、${skippedCount} 曲の重複をスキップしました`
    }
    showStatus(message, 'success')

    // 現在再生中の曲がない場合、最初の曲を自動選択
    if (!currentMusic.value && playlist.value.length > 0) {
      currentTrackIndex.value = 0
      // オーディオソースを強制的に設定
      setTimeout(() => {
        if (audio.value && currentMusic.value) {
          console.log('オーディオソースを初期化:', currentMusic.value.name)
          audio.value.src = currentMusic.value.url
        }
      }, 100)
    }
  } else if (skippedCount > 0) {
    showStatus(`${skippedCount} 曲の重複をスキップしました`, 'info')
  }

  event.target.value = ''
}

// オーディオイベントリスナーの設定
// オーディオイベントリスナー参照の保存
let audioEventListeners = {
  loadeddata: null,
  error: null,
  ended: null
}

const setupAudioEvents = () => {
  if (!audio.value) return

  // 音量を初期化
  audio.value.volume = volume.value

  // 以前のイベントリスナーをクリア
  if (audioEventListeners.loadeddata) {
    audio.value.removeEventListener(
      'loadeddata',
      audioEventListeners.loadeddata
    )
  }
  if (audioEventListeners.error) {
    audio.value.removeEventListener('error', audioEventListeners.error)
  }
  if (audioEventListeners.ended) {
    audio.value.removeEventListener('ended', audioEventListeners.ended)
  }

  // 新しいイベントリスナーを作成して保存
  audioEventListeners.loadeddata = () => {
    showStatus(`現在再生中: ${currentMusic.value?.name}`, 'info')
    console.log('音楽の読み込み完了:', currentMusic.value?.name)
  }

  audioEventListeners.error = e => {
    showStatus('音楽の読み込みに失敗しました、再試行してください', 'error')
    console.error('Audio error:', e)
    console.error('Error details:', {
      src: audio.value?.src,
      error: audio.value?.error,
      networkState: audio.value?.networkState,
      readyState: audio.value?.readyState,
      code: audio.value?.error?.code,
      message: audio.value?.error?.message
    })
  }

  audioEventListeners.ended = () => {
    // 自動的に次の曲を再生
    if (playlist.value.length > 0) {
      nextTrack()
    } else {
      // プレイリスト終了
      isPlaying.value = false
      showStatus('プレイリストが終了しました', 'info')
    }
  }

  // 新しいイベントリスナーを追加
  audio.value.addEventListener('loadeddata', audioEventListeners.loadeddata)
  audio.value.addEventListener('error', audioEventListeners.error)
  audio.value.addEventListener('ended', audioEventListeners.ended)
}

const togglePlay = () => {
  console.log('togglePlay clicked:', {
    hasAudio: !!audio.value,
    hasCurrentMusic: !!currentMusic.value,
    playlistLength: playlist.value.length,
    currentTrackIndex: currentTrackIndex.value
  })

  if (playlist.value.length === 0) {
    showStatus('プレイリストに音楽をアップロードしてください', 'warning')
    return
  }

  // 現在の音楽がない場合、最初の曲を選択
  if (!currentMusic.value) {
    console.log('現在の音楽がない、最初の曲を選択')
    currentTrackIndex.value = 0
  }

  if (isPlaying.value) {
    audio.value.pause()
    isPlaying.value = false
    showStatus('音楽を一時停止しました', 'info')
  } else {
    // selectTrack の完全なロジックを使用
    selectTrack(currentTrackIndex.value)
  }
}

const nextTrack = () => {
  if (playlist.value.length === 0) return

  // シャッフルモードの場合、現在の曲が再生履歴にあることを確認
  if (
    isShuffleMode.value &&
    !playedIndexes.value.includes(currentTrackIndex.value)
  ) {
    playedIndexes.value.push(currentTrackIndex.value)
  }

  currentTrackIndex.value = getNextTrackIndex()
  if (audio.value && currentMusic.value) {
    try {
      console.log('次の曲に切り替え:', currentMusic.value.name)

      // オーディオ状態をリセット
      audio.value.pause()
      audio.value.currentTime = 0

      // 新しいBlob URLを作成、URLの無効化を防ぐ
      let newUrl = currentMusic.value.url
      if (
        currentMusic.value.file &&
        currentMusic.value.url.startsWith('blob:')
      ) {
        URL.revokeObjectURL(currentMusic.value.url)
        newUrl = URL.createObjectURL(currentMusic.value.file)
        currentMusic.value.url = newUrl
      }

      audio.value.src = newUrl
      setupAudioEvents()
      audio.value.load()

      audio.value
        .play()
        .then(() => {
          isPlaying.value = true
          showStatus(`次の曲: ${currentMusic.value.name}`, 'info')
        })
        .catch(e => {
          console.error('再生失敗:', e)
          showStatus('再生失敗、オーディオファイルを確認してください', 'error')
          isPlaying.value = false
        })
    } catch (error) {
      console.error('音楽の切り替えに失敗:', error)
      showStatus('音楽の切り替えに失敗しました、再試行してください', 'error')
    }
  }
}

const prevTrack = () => {
  if (playlist.value.length === 0) return

  // シャッフルモードの場合、現在の曲が再生履歴にあることを確認
  if (
    isShuffleMode.value &&
    !playedIndexes.value.includes(currentTrackIndex.value)
  ) {
    playedIndexes.value.push(currentTrackIndex.value)
  }

  currentTrackIndex.value = getPrevTrackIndex()
  if (audio.value && currentMusic.value) {
    try {
      console.log('前の曲に切り替え:', currentMusic.value.name)

      // オーディオ状態をリセット
      audio.value.pause()
      audio.value.currentTime = 0

      // 新しいBlob URLを作成、URLの無効化を防ぐ
      let newUrl = currentMusic.value.url
      if (
        currentMusic.value.file &&
        currentMusic.value.url.startsWith('blob:')
      ) {
        URL.revokeObjectURL(currentMusic.value.url)
        newUrl = URL.createObjectURL(currentMusic.value.file)
        currentMusic.value.url = newUrl
      }

      audio.value.src = newUrl
      setupAudioEvents()
      audio.value.load()

      audio.value
        .play()
        .then(() => {
          isPlaying.value = true
          showStatus(`再生中: ${currentMusic.value.name}`, 'info')
        })
        .catch(e => {
          console.error('再生失敗:', e)
          showStatus('再生失敗、オーディオファイルを確認してください', 'error')
          isPlaying.value = false
        })
    } catch (error) {
      console.error('音楽の切り替えに失敗:', error)
      showStatus('音楽の切り替えに失敗しました、再試行してください', 'error')
    }
  }
}

// 音楽停止
const stopMusic = () => {
  if (audio.value) {
    audio.value.pause()
    audio.value.currentTime = 0
    isPlaying.value = false
  }
}

// 音量調整
const updateVolume = () => {
  if (audio.value) {
    audio.value.volume = volume.value
  }
}

// シャッフル再生モードの切り替え
const toggleShuffleMode = () => {
  isShuffleMode.value = !isShuffleMode.value

  if (isShuffleMode.value) {
    // シャッフルモードをオンにする時、現在の曲を含む再生履歴を初期化
    if (
      currentTrackIndex.value >= 0 &&
      currentTrackIndex.value < playlist.value.length
    ) {
      playedIndexes.value = [currentTrackIndex.value]
    } else {
      playedIndexes.value = []
    }
  } else {
    // シャッフルモードをオフにする時、再生履歴をクリア
    playedIndexes.value = []
  }

  showStatus(
    isShuffleMode.value
      ? 'シャッフル再生をオンにしました'
      : 'シャッフル再生をオフにしました',
    'info'
  )
}

// 次に再生する曲のインデックスを取得（シャッフル対応）
const getNextTrackIndex = () => {
  if (playlist.value.length === 0) return 0

  if (!isShuffleMode.value) {
    // 順番再生
    return (currentTrackIndex.value + 1) % playlist.value.length
  } else {
    // シャッフル再生
    // 現在の曲が再生履歴にあることを確認
    if (!playedIndexes.value.includes(currentTrackIndex.value)) {
      playedIndexes.value.push(currentTrackIndex.value)
    }

    // 未再生の曲のインデックスを見つける
    const allIndexes = Array.from(
      { length: playlist.value.length },
      (_, i) => i
    )
    const availableIndexes = allIndexes.filter(
      index =>
        index !== currentTrackIndex.value &&
        !playedIndexes.value.includes(index)
    )

    if (availableIndexes.length > 0) {
      // 未再生の曲からランダムに選択
      const randomIndex = Math.floor(Math.random() * availableIndexes.length)
      const nextIndex = availableIndexes[randomIndex]
      playedIndexes.value.push(nextIndex)
      return nextIndex
    } else {
      // すべての曲が再生済み
      if (playlist.value.length > 1) {
        // ポップアップで通知し、プロセスを終了
        alert('すべての曲が再生完了しました！')
        // プロセスを終了
        if (typeof window !== 'undefined' && window.close) {
          // ウィンドウを閉じられない場合、少なくとも再生を停止
          stopMusic()
          isPlaying.value = false
        } else return currentTrackIndex.value
      } else {
        // 曲が1つだけの場合、再生を維持
        return currentTrackIndex.value
      }
    }
  }
}

// 前に再生する曲のインデックスを取得（シャッフル対応）
const getPrevTrackIndex = () => {
  if (playlist.value.length === 0) return 0

  if (!isShuffleMode.value) {
    // 順番再生
    return (
      (currentTrackIndex.value - 1 + playlist.value.length) %
      playlist.value.length
    )
  } else {
    // シャッフル再生の場合、前の曲に戻る
    if (playedIndexes.value.length > 1) {
      // 現在のインデックスを削除し、前のものを返す
      playedIndexes.value.pop()
      return playedIndexes.value[playedIndexes.value.length - 1] || 0
    } else {
      // 履歴がない場合、現在の曲を保持
      return currentTrackIndex.value
    }
  }
}

const selectTrack = index => {
  if (index < 0 || index >= playlist.value.length) return

  currentTrackIndex.value = index

  // シャッフルモードの場合、再生履歴を更新
  if (isShuffleMode.value) {
    // このインデックスが既に再生履歴にあるかチェック
    const existingIndex = playedIndexes.value.indexOf(index)
    if (existingIndex === -1) {
      // 履歴にない場合、末尾に追加
      playedIndexes.value.push(index)
    } else {
      // 履歴にある場合、その位置まで切り詰める（後ろを重複再生しないように）
      playedIndexes.value = playedIndexes.value.slice(0, existingIndex + 1)
    }
  }
  if (audio.value && currentMusic.value) {
    try {
      console.log('再生曲を選択:', currentMusic.value.name)
      console.log('オーディオURL:', currentMusic.value.url)

      // オーディオ状態をリセット
      audio.value.pause()
      audio.value.currentTime = 0

      // 新しいBlob URLを作成、URLの無効化を防ぐ
      let newUrl = currentMusic.value.url
      if (
        currentMusic.value.file &&
        currentMusic.value.url.startsWith('blob:')
      ) {
        // 古いBlob URLをクリア
        URL.revokeObjectURL(currentMusic.value.url)
        // 新しいBlob URLを作成
        newUrl = URL.createObjectURL(currentMusic.value.file)
        currentMusic.value.url = newUrl
      }

      // 新しいオーディオソースを設定
      audio.value.src = newUrl

      // イベントリスナーを再設定
      setupAudioEvents()

      // イベントリスナーが設定されたことを確認してから読み込み
      audio.value.load()

      // 選択した曲を自動再生
      audio.value
        .play()
        .then(() => {
          isPlaying.value = true
          if (!isRunning.value) {
            startTimer()
          }
          showStatus(`再生開始: ${currentMusic.value.name}`, 'info')
        })
        .catch(e => {
          console.error('再生失敗:', e)
          showStatus('再生失敗、オーディオファイルを確認してください', 'error')
          isPlaying.value = false
        })
    } catch (error) {
      console.error('音楽の選択に失敗:', error)
      showStatus('音楽の選択に失敗しました、再試行してください', 'error')
    }
  }
}

const removeTrack = index => {
  if (index < 0 || index >= playlist.value.length) return

  const removedTrack = playlist.value[index]

  // リソースをクリア
  if (removedTrack.url.startsWith('blob:')) {
    URL.revokeObjectURL(removedTrack.url)
  }

  // 現在再生中の曲を削除する場合
  if (index === currentTrackIndex.value) {
    // 再生を停止
    if (audio.value) {
      audio.value.pause()
      isPlaying.value = false
    }

    // インデックスを調整
    if (playlist.value.length === 1) {
      // 最後の1曲が削除された
      currentTrackIndex.value = 0
    } else if (index >= playlist.value.length - 1) {
      // 最後の曲が削除された場合、最初に戻る
      currentTrackIndex.value = 0
    }
    // その他の場合、インデックスは自動的に調整される
  } else if (index < currentTrackIndex.value) {
    // 削除された曲が現在再生中の曲より前にある場合、インデックスを調整
    currentTrackIndex.value--
  }

  // プレイリストから削除
  playlist.value.splice(index, 1)

  if (playlist.value.length === 0) {
    showStatus('プレイリストが空になりました', 'info')
  } else {
    showStatus(`削除しました: ${removedTrack.name}`, 'info')
  }
}

// ファイルサイズのフォーマット
const formatFileSize = bytes => {
  if (bytes === 0) return '0 Bytes'
  const k = 1024
  const sizes = ['Bytes', 'KB', 'MB', 'GB']
  const i = Math.floor(Math.log(bytes) / Math.log(k))
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i]
}

// 背景画像のアップロード
const handleBackgroundUpload = event => {
  const file = event.target.files[0]
  console.log('アップロードされた画像:', file)

  if (file && file.type.startsWith('image/')) {
    // 古い背景URLをクリア
    if (selectedBackground.value.startsWith('blob:')) {
      URL.revokeObjectURL(selectedBackground.value)
    }

    const url = URL.createObjectURL(file)
    console.log('作成された画像URL:', url)

    // 画像が読み込めるか検証
    const img = new Image()
    img.onload = () => {
      selectedBackground.value = url
      showStatus('背景画像の読み込み成功', 'success')
      console.log('背景画像の設定成功')
    }
    img.onerror = () => {
      showStatus(
        '背景画像の読み込みに失敗しました、再試行してください',
        'error'
      )
      console.error('画像の読み込みに失敗')
    }
    img.src = url
  } else {
    showStatus('有効な画像ファイルを選択してください', 'info')
    console.log('無効な画像タイプ:', file?.type)
  }

  // ファイル入力ボックスをクリア、同じファイルを繰り返し選択可能に
  event.target.value = ''
}

// 背景画像を削除
const removeBackground = () => {
  if (selectedBackground.value.startsWith('blob:')) {
    URL.revokeObjectURL(selectedBackground.value)
  }
  selectedBackground.value = ''
  showStatus('背景画像を削除しました', 'info')
}

// ステータスメッセージ
const showStatus = (message, type = 'info') => {
  statusMessage.value = { text: message, type: type }
  setTimeout(() => {
    statusMessage.value = null
  }, 3000)
}

// コンポーネントマウント時の初期化
onMounted(() => {
  showStatus('ランニング音楽プレーヤーへようこそ！', 'info')
})

// コンポーネントアンマウント時のクリーンアップ
onUnmounted(() => {
  pauseTimer()

  // オーディオリソースをクリア
  if (audio.value) {
    audio.value.pause()
    if (audio.value.src && audio.value.src.startsWith('blob:')) {
      URL.revokeObjectURL(audio.value.src)
    }
  }

  // 背景画像リソースをクリア
  if (selectedBackground.value.startsWith('blob:')) {
    URL.revokeObjectURL(selectedBackground.value)
  }
})
</script>

<style scoped>
.running-player {
  min-height: 100vh;
  position: relative;
}

.overlay {
  min-height: 100vh;
  background: rgba(0, 0, 0, 0.4);
  backdrop-filter: blur(2px);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
}

.container {
  max-width: 800px;
  width: 100%;
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  border-radius: 20px;
  padding: 40px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.title {
  text-align: center;
  color: white;
  font-size: 2.5em;
  margin-bottom: 30px;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5);
}

.timer-section {
  text-align: center;
  margin-bottom: 40px;
}

.timer-display {
  margin-bottom: 20px;
}

.time {
  font-size: 4em;
  font-weight: bold;
  color: white;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5);
  font-family: 'Courier New', monospace;
}

.timer-label {
  font-size: 1.2em;
  color: rgba(255, 255, 255, 0.8);
  margin-top: 10px;
}

.progress-bar {
  width: 100%;
  height: 8px;
  background: rgba(255, 255, 255, 0.2);
  border-radius: 4px;
  overflow: hidden;
}

.progress {
  height: 100%;
  background: linear-gradient(90deg, #ff6b6b, #4ecdc4);
  transition: width 1s linear;
}

.music-section {
  text-align: center;
  margin-bottom: 40px;
}

.music-info h3 {
  color: white;
  margin-bottom: 10px;
}

.music-info p {
  color: rgba(255, 255, 255, 0.9);
  font-size: 1.1em;
}

.music-controls {
  margin: 20px 0;
}

.control-btn {
  background: none;
  border: none;
  color: white;
  font-size: 2em;
  padding: 10px 15px;
  margin: 0 15px;
  cursor: pointer;
  transition: all 0.3s ease;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5);
}

.control-btn:hover {
  transform: scale(1.2);
  text-shadow: 3px 3px 6px rgba(0, 0, 0, 0.7);
}

.control-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  transform: none;
}

.control-btn.active {
  background: linear-gradient(135deg, #ff6b6b, #ee5a24);
  color: white;
}

.play-btn {
  font-size: 2em;
  padding: 15px 25px;
}

.file-input {
  display: none;
}

.upload-btn {
  display: inline-block;
  background: linear-gradient(135deg, #667eea, #764ba2);
  color: white;
  padding: 15px 30px;
  border-radius: 25px;
  cursor: pointer;
  transition: all 0.3s ease;
  margin: 10px 5px;
  border: none;
  font-size: 1.1em;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.upload-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 16px rgba(102, 126, 234, 0.4);
  background: linear-gradient(135deg, #5a6fd8, #6a4190);
}

.background-section h3 {
  color: white;
  text-align: center;
  margin-bottom: 20px;
}

.upload-section {
  text-align: center;
}

.upload-btn {
  display: inline-block;
  background: linear-gradient(135deg, #667eea, #764ba2);
  color: white;
  padding: 15px 30px;
  border-radius: 25px;
  cursor: pointer;
  transition: all 0.3s ease;
  font-size: 1.1em;
  margin-bottom: 20px;
  border: none;
}

.upload-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
}

.current-background {
  margin-top: 20px;
  text-align: center;
}

.current-background img {
  width: 200px;
  height: 120px;
  object-fit: cover;
  border-radius: 10px;
  margin-bottom: 10px;
  border: 2px solid rgba(255, 255, 255, 0.3);
}

.remove-btn {
  background: rgba(255, 107, 107, 0.8);
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 15px;
  cursor: pointer;
  font-size: 0.9em;
}

.playlist {
  margin-top: 30px;
}

.playlist h4 {
  color: white;
  margin-bottom: 15px;
  font-size: 1.3em;
}

.playlist-item {
  display: flex;
  align-items: center;
  padding: 12px 15px;
  margin: 8px 0;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 10px;
  cursor: pointer;
  transition: all 0.3s ease;
  position: relative;
}

.playlist-item:hover {
  background: rgba(255, 255, 255, 0.2);
  transform: translateY(-1px);
}

.playlist-item.active {
  background: rgba(102, 126, 234, 0.3);
  border-left: 4px solid #667eea;
}

.track-number {
  color: rgba(255, 255, 255, 0.7);
  margin-right: 10px;
  font-weight: bold;
  min-width: 30px;
}

.track-name {
  flex: 1;
  color: white;
  text-align: left;
  font-size: 1.1em;
}

.remove-track-btn {
  background: none;
  border: none;
  color: rgba(255, 255, 255, 0.7);
  cursor: pointer;
  font-size: 1.2em;
  padding: 5px;
  transition: all 0.3s ease;
}

.remove-track-btn:hover {
  color: #ff6b6b;
  transform: scale(1.2);
}

.track-tooltip {
  position: absolute;
  top: -5px;
  left: 100%;
  margin-left: 10px;
  background: rgba(0, 0, 0, 0.9);
  color: white;
  padding: 10px;
  border-radius: 5px;
  font-size: 0.9em;
  white-space: nowrap;
  z-index: 1000;
  opacity: 0;
  visibility: hidden;
  transition: all 0.3s ease;
  pointer-events: none;
}

.playlist-item:hover .track-tooltip {
  opacity: 1;
  visibility: visible;
}

.timer-controls {
  display: flex;
  justify-content: center;
  gap: 15px;
  margin-top: 20px;
}

.timer-btn {
  background: rgba(255, 255, 255, 0.2);
  border: none;
  color: white;
  padding: 12px 24px;
  border-radius: 25px;
  cursor: pointer;
  font-size: 1.1em;
  transition: all 0.3s ease;
  backdrop-filter: blur(5px);
}

.timer-btn:hover {
  background: rgba(255, 255, 255, 0.3);
  transform: translateY(-2px);
}

.timer-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  transform: none;
}

.reset-btn {
  background: rgba(255, 107, 107, 0.3);
}

.reset-btn:hover {
  background: rgba(255, 107, 107, 0.5);
}

.status-message {
  position: fixed;
  top: 20px;
  right: 20px;
  padding: 15px 20px;
  border-radius: 10px;
  color: white;
  font-weight: bold;
  z-index: 1000;
  animation: slideIn 0.3s ease;
}

.status-success {
  background: linear-gradient(135deg, #4ecdc4, #44a08d);
}

.status-error {
  background: linear-gradient(135deg, #ff6b6b, #ee5a24);
}

.status-info {
  background: linear-gradient(135deg, #667eea, #764ba2);
}

.status-warning {
  background: linear-gradient(135deg, #ffa726, #ff7043);
}

@keyframes slideIn {
  from {
    transform: translateX(100%);
    opacity: 0;
  }
  to {
    transform: translateX(0);
    opacity: 1;
  }
}

/* 音量コントロールバーのスタイル */
.volume-control {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-top: 15px;
  padding: 15px 20px;
  background: rgba(255, 255, 255, 0.8);
  backdrop-filter: blur(10px);
  border-radius: 25px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.volume-icon {
  font-size: 1.2rem;
  color: #667eea;
}

.volume-slider {
  flex: 1;
  height: 6px;
  border-radius: 5px;
  background: rgba(102, 126, 234, 0.2);
  outline: none;
  cursor: pointer;
  -webkit-appearance: none;
}

.volume-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: linear-gradient(135deg, #667eea, #764ba2);
  cursor: pointer;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.2);
  transition: all 0.3s ease;
}

.volume-slider::-webkit-slider-thumb:hover {
  transform: scale(1.2);
  box-shadow: 0 3px 10px rgba(0, 0, 0, 0.5);
}

.volume-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: linear-gradient(135deg, #667eea, #764ba2);
  cursor: pointer;
  border: none;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.2);
}

.volume-value {
  font-size: 0.9rem;
  font-weight: bold;
  color: #667eea;
  min-width: 40px;
  text-align: center;
}

@media (max-width: 600px) {
  .container {
    padding: 20px;
    margin: 10px;
  }

  .title {
    font-size: 2em;
  }

  .time {
    font-size: 3em;
  }

  .music-controls {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    gap: 10px;
  }

  .control-btn {
    margin: 5px;
    font-size: 2em;
  }

  .volume-control {
    flex-direction: column;
    gap: 8px;
  }

  .volume-slider {
    width: 100%;
    max-width: 200px;
  }
}
</style>
