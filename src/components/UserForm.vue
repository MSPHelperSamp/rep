<template>
  <form @submit.prevent="convertYouTubeVideo">
    <section>
      <label for="link">Посилання на медіа:</label>
      <input id="link" name="link" type="url" placeholder="url" v-model="mediaLink">
    </section>
    <section>
      <label for="selectedLanguage">Вкажіть мову медіа:</label>
      <select v-model="selectedLanguage" id="selectedLanguage">
        <option v-for="language in languages" :key="language" :value="language">
          {{ language }}
        </option>
      </select>
    </section>
    <section class="buttonBlock">
      <button>Конвертувати текст</button>
    </section>
    <section class="transcription" v-if="transcription">
      <h3>Транскрипція</h3>
      <pre>{{ transcription }}</pre>
      <button @click="copyText(transcription)">Копіювати</button>
      <h5 class="textCopied" v-if="textCopied">Text is copied!</h5>
    </section>
  </form>
</template>

<script>
import axios from "axios";

export default {
  data() {
    return {
      mediaLink: '',
      selectedLanguage: 'nl',
      languages: ['en', 'nl', 'ru', 'ua'],
      API_KEY: '',//api key
      textCopied:false,
      mediaId: null,
      transcription: ''
    }
  },
  methods: {
    changeCopiedActive(){
      this.textCopied=true
      setTimeout(()=>{
        this.textCopied=false
      },3000)
    },
    async copyText(text){
      try {
        await navigator.clipboard.writeText(text);
        console.log('Текст успешно скопирован в буфер обмена!');
        this.changeCopiedActive()
      } catch (err) {
        console.error('Ошибка:', err);
      }
    },
    async convertYouTubeVideo() {
      try {
        // Отправляем видео на конвертацию
        const response = await axios.post('https://api.sonix.ai/v1/media', {
          file_url: this.mediaLink,
          language: this.selectedLanguage
        }, {
          headers: {
            'Authorization': `Bearer ${this.API_KEY}`,
            'Content-Type': 'application/json'
          }
        });

        if (response.data && response.data.id) {
          this.mediaId = response.data.id;
          console.log(`Видео успешно отправлено на конвертацию. Media ID: ${this.mediaId}`);

          // Периодически проверяем статус конвертации и получаем транскрипцию
          this.transcription = await this.checkStatusAndGetTranscription(this.mediaId);
          console.log('Транскрипция видео:', this.transcription);
        } else {
          console.log('Не удалось отправить видео на конвертацию.');
        }
      } catch (error) {
        console.error('Ошибка при отправке видео на конвертацию:', error.response ? error.response.data : error.message);
      }
    },
    async checkStatusAndGetTranscription(mediaId) {
      try {
        let status = 'in_progress';

        while (status === 'in_progress' || status === 'queued' || status === 'preparing' || status === 'transcribing') {
          const response = await axios.get(`https://api.sonix.ai/v1/media/${mediaId}`, {
            headers: {
              'Authorization': `Bearer ${this.API_KEY}`,
              'Content-Type': 'application/json'
            }
          });

          status = response.data.status;
          console.log(`Текущий статус: ${status}`);
          //|| status === 'duplicate'
          if (status === 'completed' ) {
            // Получаем транскрипцию
            const transcriptResponse = await axios.get(`https://api.sonix.ai/v1/media/${mediaId}/transcript`, {
              headers: {
                'Authorization': `Bearer ${this.API_KEY}`,
                'Content-Type': 'application/json'
              }
            });

            console.log('Ответ транскрипции:', transcriptResponse.data);

            if (typeof transcriptResponse.data === 'string') {
              return transcriptResponse.data;
            } else if (transcriptResponse.data && transcriptResponse.data.text) {
              return transcriptResponse.data.text;
            } else {
              console.warn('Транскрипция пуста или имеет неверный формат:', transcriptResponse.data);
              return '';
            }
          } else if (status === 'failed') {
            throw new Error('Конвертация не удалась.');
          }

          // Ждем 10 секунд перед следующей проверкой статуса
          await new Promise(resolve => setTimeout(resolve, 10000));
        }
      } catch (error) {
        console.error('Ошибка при получении транскрипции:', error.response ? error.response.data : error.message);
      }
    }
  }
}
</script>

<style scoped>
form {
  display: flex;
  flex-direction: column;
  align-items: start;
  width: 300px;
}

section,
select {
  display: flex;
  flex-direction: column;
  align-items: start;
  width: 100%;
}
.transcription h3{
  width: 100%;
  text-align: center;
}
.transcription{
  padding: 5px;
  margin: 5px;
  width: 100%;
  height: 100%;
  max-width: 300px;
  max-height: 300px;
  background-color: #1a037e50;
  border-radius: 10px;

}
pre{
  margin-top: -15px;
  padding: 5px;
  overflow: auto;
}
.transcription button{
  margin: 0 auto;
}
.textCopied{
  width: 100%;
  margin-top: 3px;
  margin-bottom: 3px;
}
input {
  width: 100%;
}

.buttonBlock {
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-top: 5px;
}

button {
  margin: 0;
}

pre {
  white-space: pre-wrap;
  word-wrap: break-word;
}
</style>
