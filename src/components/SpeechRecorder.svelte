<script lang="ts">
  let isRecording = false;
  let recognizedText = '';
  let interimText = '';
  let recognition = null;
  let error = '';
  let isLoading = false;
  let apiResponse = null;

  // Проверяем поддержку SpeechRecognition в браузере
  const isSupported = typeof window !== 'undefined' && 
    (window.SpeechRecognition || window.webkitSpeechRecognition);

  function startRecording() {
    if (!isSupported) {
      error = 'Распознавание речи не поддерживается в вашем браузере';
      return;
    }

    // Сбрасываем предыдущие результаты
    apiResponse = null;
    recognizedText = '';
    interimText = '';
    
    // Инициализируем SpeechRecognition
    const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
    recognition = new SpeechRecognition();
    
    recognition.lang = 'ru-RU'; // Русский язык
    recognition.continuous = true; // Продолжать слушать
    recognition.interimResults = true; // Показывать промежуточные результаты

    recognition.onstart = () => {
      isRecording = true;
      error = '';
      interimText = 'Слушаю...';
    };

    recognition.onresult = (event) => {
      let finalTranscript = '';
      let interimTranscript = '';

      for (let i = event.resultIndex; i < event.results.length; i++) {
        const transcript = event.results[i][0].transcript;
        
        if (event.results[i].isFinal) {
          finalTranscript += transcript;
        } else {
          interimTranscript += transcript;
        }
      }

      if (finalTranscript) {
        recognizedText = finalTranscript;
        interimText = '';
        // Вызываем API только когда распознавание завершено
        callAPI(finalTranscript);
      } else if (interimTranscript) {
        interimText = interimTranscript;
      }
    };

    recognition.onerror = (event) => {
      error = `Ошибка распознавания: ${event.error}`;
      isRecording = false;
      interimText = '';
    };

    recognition.onend = () => {
      isRecording = false;
      if (!recognizedText && interimText) {
        // Если распознавание закончилось, но есть промежуточный текст
        recognizedText = interimText;
        interimText = '';
        callAPI(recognizedText);
      }
    };

    recognition.start();
  }

  async function callAPI(text: string) {
    isLoading = true;
    error = '';
    
    try {
      const response = await fetch('https://callcenter.thoughtspile.tech/rag/answer', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          query: text,
          use_colbert: false,
          semantic_weight: 0.0,
          keyword_weight: 0.8,
        }),
      });

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const data = await response.json();
      apiResponse = data;
    } catch (err) {
      error = `Ошибка при обращении к API: ${err.message}`;
      console.error('API Error:', err);
    } finally {
      isLoading = false;
    }
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
      disabled={isLoading}
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

    <!-- Промежуточный текст -->
    {#if interimText}
      <div class="interim-result">
        <h3>Распознается:</h3>
        <p class="interim-text">{interimText}</p>
      </div>
    {/if}

    <!-- Финальный распознанный текст -->
    {#if recognizedText}
      <div class="final-result">
        <h3>Распознанный текст:</h3>
        <p>{recognizedText}</p>
      </div>
    {/if}

    {#if isLoading}
      <div class="loading">
        <div class="spinner"></div>
        <p>Обработка запроса...</p>
      </div>
    {/if}

    {#if apiResponse}
      <div class="api-result">
        <div class="answer-section">
          <h3>Ответ:</h3>
          <p class="answer-text">{apiResponse.answer}</p>
        </div>

        {#if apiResponse.documents && apiResponse.documents.length > 0}
          <div class="documents-section">
            <h3>Документы ({apiResponse.documents.length}):</h3>
            <div class="documents-list">
              {#each apiResponse.documents as document, index}
                <div class="document-item">
                  <details>
                    <summary>
                      <span class="document-title">Документ #{index + 1}</span>
                      <svg class="expand-icon" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                        <path d="M6 9l6 6 6-6" />
                      </svg>
                    </summary>
                    <div class="document-content">
                      {document.text}
                    </div>
                  </details>
                </div>
              {/each}
            </div>
          </div>
        {/if}
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
    max-width: 600px;
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

  .record-button:hover:not(:disabled) {
    background: #005a9e;
  }

  .record-button:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }

  .record-button.recording {
    background: #dc3545;
  }

  .record-button.recording:hover:not(:disabled) {
    background: #c82333;
  }

  .interim-result {
    margin-top: 20px;
    padding: 15px;
    background: #fff3cd;
    border-radius: 8px;
    border-left: 4px solid #ffc107;
    animation: pulse 2s infinite;
  }

  @keyframes pulse {
    0% { opacity: 1; }
    50% { opacity: 0.8; }
    100% { opacity: 1; }
  }

  .interim-result h3 {
    margin: 0 0 10px 0;
    color: #856404;
    font-size: 16px;
  }

  .interim-text {
    margin: 0;
    color: #856404;
    font-size: 18px;
    line-height: 1.4;
    font-style: italic;
  }

  .final-result {
    margin-top: 20px;
    padding: 15px;
    background: #d4edda;
    border-radius: 8px;
    border-left: 4px solid #28a745;
  }

  .final-result h3 {
    margin: 0 0 10px 0;
    color: #155724;
    font-size: 16px;
  }

  .final-result p {
    margin: 0;
    color: #155724;
    font-size: 18px;
    line-height: 1.4;
    font-weight: 500;
  }

  .loading {
    margin: 20px 0;
    padding: 15px;
    background: #f8f9fa;
    border-radius: 8px;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 10px;
  }

  .spinner {
    width: 40px;
    height: 40px;
    border: 4px solid #f3f3f3;
    border-top: 4px solid #007acc;
    border-radius: 50%;
    animation: spin 1s linear infinite;
  }

  @keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }

  .api-result {
    margin-top: 20px;
    text-align: left;
  }

  .answer-section {
    padding: 20px;
    background: #e8f4fd;
    border-radius: 8px;
    border-left: 4px solid #007acc;
    margin-bottom: 20px;
  }

  .answer-section h3 {
    margin: 0 0 10px 0;
    color: #0c5460;
    font-size: 18px;
  }

  .answer-text {
    margin: 0;
    color: #333;
    font-size: 16px;
    line-height: 1.5;
  }

  .documents-section {
    background: #f8f9fa;
    border-radius: 8px;
    padding: 20px;
  }

  .documents-section h3 {
    margin: 0 0 15px 0;
    color: #333;
    font-size: 18px;
  }

  .documents-list {
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  .document-item details {
    background: white;
    border-radius: 6px;
    border: 1px solid #dee2e6;
    overflow: hidden;
  }

  .document-item summary {
    padding: 15px;
    display: flex;
    justify-content: space-between;
    align-items: center;
    cursor: pointer;
    list-style: none;
    transition: background 0.2s ease;
  }

  .document-item summary:hover {
    background: #f8f9fa;
  }

  .document-item summary::-webkit-details-marker {
    display: none;
  }

  .document-title {
    font-weight: 500;
    color: #495057;
  }

  .expand-icon {
    transition: transform 0.3s ease;
  }

  .document-item details[open] .expand-icon {
    transform: rotate(180deg);
  }

  .document-content {
    padding: 15px;
    border-top: 1px solid #dee2e6;
    background: #f8f9fa;
    color: #666;
    line-height: 1.5;
    font-size: 14px;
    white-space: pre-wrap;
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