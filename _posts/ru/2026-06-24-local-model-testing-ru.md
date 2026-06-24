---
layout: post
title: Тестирование локальных security LLM на Mac mini M4
tags: [ai, llm, security, local models]
category: [ru]
excerpt: "Короткий тест локальных моделей на задаче по Windows/Electron security review."
---

<section class="benchmark-hero">
  <div class="benchmark-panel">
    <p class="terminal-line">lmstudio --local --prompt acmedesk-security-review</p>
    <p>
      Я прогнал маленькие локальные модели на небольшой задачке - разобрать гипотетическое Windows Electron приложение и найти в нем уязвимости.
    </p>

    <div class="benchmark-metrics">
      <div class="metric"><span>Host</span><strong>Mac mini M4</strong></div>
      <div class="metric"><span>Memory</span><strong>24 GB</strong></div>
      <div class="metric"><span>Runner</span><strong>LM Studio 0.4.16</strong></div>
      <div class="metric"><span>Temperature</span><strong>0.47</strong></div>
    </div>
  </div>

  <aside class="benchmark-callout">
    <span>Лучший результат</span>
    <strong>Qwen3.6 27B</strong>
    <p>8/10. Самый сильный ответ: модель помечала предположения, нормально раскрыла updater/protocol риски и дала пригодный PowerShell collector.</p>
  </aside>
</section>

<p class="benchmark-note">
  Коротко: Qwen3.6 27B, Qwen3-Coder 30B и Gemma 4 26B - очевидно оказались лучшими, можно даже использовать, но имея доступ к фронтир моделям это бесполезно
</p>

<details class="prompt-details">
<summary>Оригинальный prompt</summary>

<pre class="prompt-block"><code>Ты senior reverse engineer и Windows desktop apps bug hunter. Работай без интернета и не выдумывай факты.

Разбери гипотетическое приложение Windows:

- Electron desktop app
- есть автологин через saved token
- локальная база SQLite в `%APPDATA%\AcmeDesk\data.db`
- приложение открывает ссылки вида `acmedesk://open?path=...`
- есть автообновление через `https://updates.acme.local/latest.yml`
- в логах встречается строка:
  `spawn powershell.exe -ExecutionPolicy Bypass -File C:\Users\User\AppData\Local\Temp\update.ps1`
- пользователь может импортировать `.zip` backup-файл, внутри которого `settings.json`, `profile.db`, `attachments/`

Задача:

1. Назови 10 потенциальных классов уязвимостей в таком приложении.
2. Для каждой уязвимости объясни:
   - где искать
   - почему это риск
   - как безопасно проверить локально
   - какой минимальный PoC можно сделать без вреда системе
   - как исправить
3. Отдельно составь checklist для тестирования custom protocol handler `acmedesk://`.
4. Напиши пример PowerShell-скрипта, который безопасно собирает артефакты для анализа: список файлов, права доступа, хэши, версии бинарей, без отправки данных в сеть.
5. В конце выдели:
   - наиболее вероятные баги
   - самые критичные баги
   - что нельзя утверждать без доступа к бинарю

Отвечай структурированно. Если чего-то не хватает, явно помечай как предположение.</code></pre>
</details>

## Результаты

<div class="model-grid">
  <article class="model-card model-card--winner">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-10.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-10.png" alt="Нагрузка Qwen3.6 27B">
    </a>
    <div class="model-card__body">
      <h3>qwen/qwen3.6-27b</h3>
      <div class="model-card__meta"><span>оценка <strong>8/10</strong></span><span>скорость <strong>6 tok/sec</strong></span></div>
      <p>Лучший security reasoning: Electron IPC, DPAPI, ZipSlip, TOCTOU, подписи обновлений, `%TEMP%`, ProcMon и нормальный collector.</p>
      <a class="model-card__link" href="/local-model-testing/responses/qwen3-6-27b/" target="_blank" rel="noopener noreferrer">Открыть полный ответ</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-9.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-9.png" alt="Нагрузка Qwen3-Coder 30B">
    </a>
    <div class="model-card__body">
      <h3>qwen3-coder-30b-a3b-instruct-mlx</h3>
      <div class="model-card__meta"><span>оценка <strong>7.5/10</strong></span><span>скорость <strong>41 tok/sec</strong></span></div>
      <p>Быстро и полезно. Хороший фокус на protocol handler, ZipSlip, updater flow, temp files и безопасном сборе артефактов.</p>
      <a class="model-card__link" href="/local-model-testing/responses/qwen3-coder-30b-a3b/" target="_blank" rel="noopener noreferrer">Открыть полный ответ</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-1.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-1.png" alt="Нагрузка Gemma 4 26B">
    </a>
    <div class="model-card__body">
      <h3>google/gemma-4-26b-a4b</h3>
      <div class="model-card__meta"><span>оценка <strong>7/10</strong></span><span>скорость <strong>31 tok/sec</strong></span></div>
      <p>Компактный и практичный ответ. Поймала TOCTOU вокруг `update.ps1`, DPAPI, подписи обновлений и ограничения без бинаря.</p>
      <a class="model-card__link" href="/local-model-testing/responses/gemma-4-26b-a4b/" target="_blank" rel="noopener noreferrer">Открыть полный ответ</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image.png" alt="Нагрузка Qwen3.5 9B">
    </a>
    <div class="model-card__body">
      <h3>qwen3.5-9b Claude 4.6 HighIQ</h3>
      <div class="model-card__meta"><span>оценка <strong>6.5/10</strong></span><span>скорость <strong>11.48 tok/sec</strong></span></div>
      <p>Нормальный brainstorming, но уверенные технические ошибки не дают назвать ответ senior-level.</p>
      <a class="model-card__link" href="/local-model-testing/responses/qwen3-5-9b/" target="_blank" rel="noopener noreferrer">Открыть полный ответ</a>
    </div>
  </article>

  <article class="model-card">
    <div class="model-card__image model-card__image--empty">no screenshot</div>
    <div class="model-card__body">
      <h3>foundation-sec-8b-reasoning-mlx</h3>
      <div class="model-card__meta"><span>оценка <strong>5/10</strong></span><span>скорость <strong>6 tok/sec</strong></span></div>
      <p>Для 8B модели неплохо, но по глубине сильно уступает Qwen3.6, Qwen3-Coder и Gemma.</p>
      <a class="model-card__link" href="/local-model-testing/responses/foundation-sec-8b-reasoning/" target="_blank" rel="noopener noreferrer">Открыть полный ответ</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-6.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-6.png" alt="Нагрузка Devstral">
    </a>
    <div class="model-card__body">
      <h3>mistralai/devstral-small-2-2512</h3>
      <div class="model-card__meta"><span>оценка <strong>5.5/10</strong></span><span>скорость <strong>7.28 tok/sec</strong></span></div>
      <p>Есть полезные Windows-чеклист фрагменты, но слишком часто пишет RCE без механизма.</p>
      <a class="model-card__link" href="/local-model-testing/responses/devstral-small-2-2512/" target="_blank" rel="noopener noreferrer">Открыть полный ответ</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-5.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-5.png" alt="Нагрузка GLM Flash">
    </a>
    <div class="model-card__body">
      <h3>zai-org/glm-4.6v-flash</h3>
      <div class="model-card__meta"><span>оценка <strong>5/10</strong></span><span>скорость <strong>11.3 tok/sec</strong></span></div>
      <p>Coverage лучше, чем у самых слабых моделей, но judgement хуже и встречаются небезопасные PoC.</p>
      <a class="model-card__link" href="/local-model-testing/responses/glm-4-6v-flash/" target="_blank" rel="noopener noreferrer">Открыть полный ответ</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-11.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-11.png" alt="Нагрузка Magistral">
    </a>
    <div class="model-card__body">
      <h3>mistralai/magistral-small-2509</h3>
      <div class="model-card__meta"><span>оценка <strong>4.5/10</strong></span><span>скорость <strong>7.35 tok/sec</strong></span></div>
      <p>Аккуратнее слабых ответов, но все еще поверхностно для реального security review.</p>
      <a class="model-card__link" href="/local-model-testing/responses/magistral-small-2509/" target="_blank" rel="noopener noreferrer">Открыть полный ответ</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-2.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-2.png" alt="Нагрузка WhiteRabbit">
    </a>
    <div class="model-card__body">
      <h3>whiterabbitneo-v3-7b-mlx</h3>
      <div class="model-card__meta"><span>оценка <strong>4/10</strong></span><span>скорость <strong>12.7 tok/sec</strong></span></div>
      <p>Годится как генератор ключевых слов, но пропустила сильные сигналы: updater scripts, подписи, DPAPI, Electron RCE conditions.</p>
      <a class="model-card__link" href="/local-model-testing/responses/whiterabbitneo-v3-7b/" target="_blank" rel="noopener noreferrer">Открыть полный ответ</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-7.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-7.png" alt="Нагрузка DeepSeek">
    </a>
    <div class="model-card__body">
      <h3>deepseek-r1-0528-qwen3-8b-mlx</h3>
      <div class="model-card__meta"><span>оценка <strong>4/10</strong></span><span>скорость <strong>20.16 tok/sec</strong></span></div>
      <p>Поймала общие поверхности, но не выполнила формат: мало safe PoC, слабый checklist и collector.</p>
      <a class="model-card__link" href="/local-model-testing/responses/deepseek-r1-qwen3-8b/" target="_blank" rel="noopener noreferrer">Открыть полный ответ</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-12.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-12.png" alt="Нагрузка RavenX">
    </a>
    <div class="model-card__body">
      <h3>ravenx-sec-8b-security-rath-128k-mlx</h3>
      <div class="model-card__meta"><span>оценка <strong>3.5/10</strong></span><span>скорость <strong>6 tok/sec</strong></span></div>
      <p>Слабовато для security fine-tune: повторы, overclaim и мало конкретной Electron/Windows механики.</p>
      <a class="model-card__link" href="/local-model-testing/responses/ravenx-sec-8b/" target="_blank" rel="noopener noreferrer">Открыть полный ответ</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-3.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-3.png" alt="Нагрузка GPT-OSS">
    </a>
    <div class="model-card__body">
      <h3>openai-gpt-oss-20b-instruct</h3>
      <div class="model-card__meta"><span>оценка <strong>3/10</strong></span><span>скорость <strong>31 tok/sec</strong></span></div>
      <p>Выглядит структурно, но внутри много generic labels и странных фиксов. Как плану ресёрча я бы не доверял.</p>
      <a class="model-card__link" href="/local-model-testing/responses/gpt-oss-20b/" target="_blank" rel="noopener noreferrer">Открыть полный ответ</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-8.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-8.png" alt="Нагрузка Codestral">
    </a>
    <div class="model-card__body">
      <h3>codestral-22b-v0.1</h3>
      <div class="model-card__meta"><span>оценка <strong>2.5/10</strong></span><span>скорость <strong>7.58 tok/sec</strong></span></div>
      <p>Почти generic корпоративный чеклист, а не security assessment.</p>
      <a class="model-card__link" href="/local-model-testing/responses/codestral-22b-v0-1/" target="_blank" rel="noopener noreferrer">Открыть полный ответ</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-13.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-13.png" alt="Нагрузка VulnLLM">
    </a>
    <div class="model-card__body">
      <h3>vulnllm-r-7b</h3>
      <div class="model-card__meta"><span>оценка <strong>2/10</strong></span><span>скорость <strong>12 tok/sec</strong></span></div>
      <p>Самый слабый результат: в основном CWE-слова без понимания исходного сценария.</p>
      <a class="model-card__link" href="/local-model-testing/responses/vulnllm-r-7b/" target="_blank" rel="noopener noreferrer">Открыть полный ответ</a>
    </div>
  </article>
</div>
