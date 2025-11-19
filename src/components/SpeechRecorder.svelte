<script lang="ts">
  let isRecording = false;
  let recognizedText = '';
  let recognition = null;
  let error = '';

  // Проверяем поддержку SpeechRecognition в браузере
  const isSupported = typeof window !== 'undefined' && 
    (window.SpeechRecognition || window.webkitSpeechRecognition);

  function startRecording() {
    if (!isSupported) {
      error = 'Распознавание речи не поддерживается в вашем браузере';
      return;
    }

    // Инициализируем SpeechRecognition
    const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
    recognition = new SpeechRecognition();
    
    recognition.lang = 'ru-RU'; // Русский язык
    recognition.continuous = false; // Остановиться после первого результата
    recognition.interimResults = false; // Только финальные результаты

    recognition.onstart = () => {
      isRecording = true;
      error = '';
      recognizedText = 'Слушаю...';
    };

    recognition.onresult = (event) => {
      const transcript = event.results[0][0].transcript;
      recognizedText = transcript;
    };

    recognition.onerror = (event) => {
      error = `Ошибка: ${event.error}`;
      isRecording = false;
    };

    recognition.onend = () => {
      isRecording = false;
    };

    recognition.start();
  }

  function stopRecording() {
    if (recognition) {
      recognition.stop();
      isRecording = false;
    }
  }
</script>

<div class="speech-recognition">
  <h2>Распознавание речи (Русский)</h2>
  
  {#if !isSupported}
    <div class="error">
      Ваш браузер не поддерживает API распознавания речи.
      <br>Попробуйте Chrome, Edge или Safari.
    </div>
  {:else}
    <button 
      class="record-button {isRecording ? 'recording' : ''}"
      on:click={isRecording ? stopRecording : startRecording}
    >
      <svg 
        width="20" 
        height="20" 
        viewBox="0 0 24 24" 
        fill="none" 
        stroke="currentColor" 
        stroke-width="2"
      >
        <path d="M12 1a3 3 0 0 0-3 3v8a3 3 0 0 0 6 0V4a3 3 0 0 0-3-3z" />
        <path d="M19 10v2a7 7 0 0 1-14 0v-2" />
        <path d="M12 19v4" />
        <path d="M8 23h8" />
      </svg>
      {isRecording ? 'Стоп' : 'Говорить'}
    </button>

    {#if recognizedText}
      <div class="result">
        <h3>Распознанный текст:</h3>
        <p>{recognizedText}</p>
      </div>
    {/if}

    {#if error}
      <div class="error">{error}</div>
    {/if}
  {/if}
</div>

<style>
  .speech-recognition {
    font-family: Arial, sans-serif;
    max-width: 400px;
    margin: 0 auto;
    padding: 20px;
    text-align: center;
  }

  .record-button {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 12px 24px;
    background: #007acc;
    color: white;
    border: none;
    border-radius: 8px;
    font-size: 16px;
    cursor: pointer;
    transition: all 0.3s ease;
  }

  .record-button:hover {
    background: #005a9e;
  }

  .record-button.recording {
    background: #dc3545;
  }

  .record-button.recording:hover {
    background: #c82333;
  }

  .result {
    margin-top: 20px;
    padding: 15px;
    background: #f8f9fa;
    border-radius: 8px;
    border-left: 4px solid #007acc;
  }

  .result h3 {
    margin: 0 0 10px 0;
    color: #333;
  }

  .result p {
    margin: 0;
    color: #666;
    font-size: 18px;
    line-height: 1.4;
  }

  .error {
    margin-top: 15px;
    padding: 10px;
    background: #ffe6e6;
    color: #dc3545;
    border-radius: 4px;
    border: 1px solid #dc3545;
  }
</style>