---
layout: post
title: Local security LLM testing on Mac mini M4
tags: [ai, llm, security, local models]
category: [en]
excerpt: "A compact benchmark of local models on a Windows/Electron security-review prompt."
---

<section class="benchmark-hero">
  <div class="benchmark-panel">
    <p class="terminal-line">lmstudio --local --prompt acmedesk-security-review</p>
    <p>
      I tested local models on a practical task: analyzing a hypothetical Windows Electron application offline and producing a safe bug-hunting plan.
      I evaluated concrete Windows and Electron reasoning, safe local PoCs, and the ability to distinguish evidence from claims that cannot be verified without the binary, rather than sheer verbosity.
    </p>

    <div class="benchmark-metrics">
      <div class="metric"><span>Host</span><strong>Mac mini M4</strong></div>
      <div class="metric"><span>Memory</span><strong>24 GB</strong></div>
      <div class="metric"><span>Runner</span><strong>LM Studio 0.4.16</strong></div>
      <div class="metric"><span>Temperature</span><strong>0.47</strong></div>
    </div>
  </div>

  <aside class="benchmark-callout">
    <span>Best overall</span>
    <strong>Qwen3.6 27B</strong>
    <p>8/10. The strongest answer: it clearly marked assumptions, addressed updater and protocol risks in concrete terms, and produced a usable PowerShell collector.</p>
  </aside>
</section>

<p class="benchmark-note">
  In short, Qwen3.6 27B, Qwen3-Coder 30B, and Gemma 4 26B were clearly the best. They are usable, but largely pointless when frontier models are available.
</p>

<details class="prompt-details">
<summary>Original prompt</summary>

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

## Results

<div class="model-grid">
  <article class="model-card model-card--winner">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-10.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-10.png" alt="Qwen3.6 27B system load">
    </a>
    <div class="model-card__body">
      <h3>qwen/qwen3.6-27b</h3>
      <div class="model-card__meta"><span>score <strong>8/10</strong></span><span>speed <strong>6 tok/sec</strong></span></div>
      <p>The strongest security reasoning, covering Electron IPC, DPAPI, Zip Slip, TOCTOU, updater signatures, `%TEMP%`, ProcMon, and a solid artifact collector.</p>
      <a class="model-card__link" href="/local-model-testing/responses/qwen3-6-27b/" target="_blank" rel="noopener noreferrer">Open full answer</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-9.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-9.png" alt="Qwen3-Coder 30B system load">
    </a>
    <div class="model-card__body">
      <h3>qwen3-coder-30b-a3b-instruct-mlx</h3>
      <div class="model-card__meta"><span>score <strong>7.5/10</strong></span><span>speed <strong>41 tok/sec</strong></span></div>
      <p>Fast and useful, with a strong focus on protocol handling, Zip Slip, the updater flow, temporary files, and safe PowerShell collection.</p>
      <a class="model-card__link" href="/local-model-testing/responses/qwen3-coder-30b-a3b/" target="_blank" rel="noopener noreferrer">Open full answer</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-1.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-1.png" alt="Gemma 4 26B system load">
    </a>
    <div class="model-card__body">
      <h3>google/gemma-4-26b-a4b</h3>
      <div class="model-card__meta"><span>score <strong>7/10</strong></span><span>speed <strong>31 tok/sec</strong></span></div>
      <p>Concise and practical. It identified the `update.ps1` TOCTOU angle, DPAPI, signed updates, and the limits imposed by not having the binary.</p>
      <a class="model-card__link" href="/local-model-testing/responses/gemma-4-26b-a4b/" target="_blank" rel="noopener noreferrer">Open full answer</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image.png" alt="Qwen3.5 9B system load">
    </a>
    <div class="model-card__body">
      <h3>qwen3.5-9b Claude 4.6 HighIQ</h3>
      <div class="model-card__meta"><span>score <strong>6.5/10</strong></span><span>speed <strong>11.48 tok/sec</strong></span></div>
      <p>Good brainstorming, but several confident technical mistakes kept it below the top tier.</p>
      <a class="model-card__link" href="/local-model-testing/responses/qwen3-5-9b/" target="_blank" rel="noopener noreferrer">Open full answer</a>
    </div>
  </article>

  <article class="model-card">
    <div class="model-card__image model-card__image--empty">no screenshot</div>
    <div class="model-card__body">
      <h3>foundation-sec-8b-reasoning-mlx</h3>
      <div class="model-card__meta"><span>score <strong>5/10</strong></span><span>speed <strong>6 tok/sec</strong></span></div>
      <p>Respectable for an 8B model, but not deep enough compared with Qwen3.6, Qwen3-Coder, or Gemma.</p>
      <a class="model-card__link" href="/local-model-testing/responses/foundation-sec-8b-reasoning/" target="_blank" rel="noopener noreferrer">Open full answer</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-6.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-6.png" alt="Devstral system load">
    </a>
    <div class="model-card__body">
      <h3>mistralai/devstral-small-2-2512</h3>
      <div class="model-card__meta"><span>score <strong>5.5/10</strong></span><span>speed <strong>7.28 tok/sec</strong></span></div>
      <p>Some useful Windows checklist items, but too many unsupported RCE claims with no plausible mechanism.</p>
      <a class="model-card__link" href="/local-model-testing/responses/devstral-small-2-2512/" target="_blank" rel="noopener noreferrer">Open full answer</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-5.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-5.png" alt="GLM Flash system load">
    </a>
    <div class="model-card__body">
      <h3>zai-org/glm-4.6v-flash</h3>
      <div class="model-card__meta"><span>score <strong>5/10</strong></span><span>speed <strong>11.3 tok/sec</strong></span></div>
      <p>Broader coverage than the weakest models, but poorer judgment and several unsafe PoC suggestions.</p>
      <a class="model-card__link" href="/local-model-testing/responses/glm-4-6v-flash/" target="_blank" rel="noopener noreferrer">Open full answer</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-11.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-11.png" alt="Magistral system load">
    </a>
    <div class="model-card__body">
      <h3>mistralai/magistral-small-2509</h3>
      <div class="model-card__meta"><span>score <strong>4.5/10</strong></span><span>speed <strong>7.35 tok/sec</strong></span></div>
      <p>Cleaner than the weakest answers, but still too shallow for a real security review.</p>
      <a class="model-card__link" href="/local-model-testing/responses/magistral-small-2509/" target="_blank" rel="noopener noreferrer">Open full answer</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-2.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-2.png" alt="WhiteRabbit system load">
    </a>
    <div class="model-card__body">
      <h3>whiterabbitneo-v3-7b-mlx</h3>
      <div class="model-card__meta"><span>score <strong>4/10</strong></span><span>speed <strong>12.7 tok/sec</strong></span></div>
      <p>Readable, but largely a collection of keywords. It missed strong signals in the prompt, including updater scripts, signing, DPAPI, and Electron-specific RCE preconditions.</p>
      <a class="model-card__link" href="/local-model-testing/responses/whiterabbitneo-v3-7b/" target="_blank" rel="noopener noreferrer">Open full answer</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-7.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-7.png" alt="DeepSeek system load">
    </a>
    <div class="model-card__body">
      <h3>deepseek-r1-0528-qwen3-8b-mlx</h3>
      <div class="model-card__meta"><span>score <strong>4/10</strong></span><span>speed <strong>20.16 tok/sec</strong></span></div>
      <p>Identified broad attack surfaces, but ignored the requested format and failed to provide safe minimal PoCs or a useful protocol checklist.</p>
      <a class="model-card__link" href="/local-model-testing/responses/deepseek-r1-qwen3-8b/" target="_blank" rel="noopener noreferrer">Open full answer</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-12.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-12.png" alt="RavenX system load">
    </a>
    <div class="model-card__body">
      <h3>ravenx-sec-8b-security-rath-128k-mlx</h3>
      <div class="model-card__meta"><span>score <strong>3.5/10</strong></span><span>speed <strong>6 tok/sec</strong></span></div>
      <p>Disappointing for a security fine-tune: repetitive, overconfident, and weak on Electron and Windows internals.</p>
      <a class="model-card__link" href="/local-model-testing/responses/ravenx-sec-8b/" target="_blank" rel="noopener noreferrer">Open full answer</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-3.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-3.png" alt="GPT-OSS system load">
    </a>
    <div class="model-card__body">
      <h3>openai-gpt-oss-20b-instruct</h3>
      <div class="model-card__meta"><span>score <strong>3/10</strong></span><span>speed <strong>31 tok/sec</strong></span></div>
      <p>Superficially structured, but full of generic labels and questionable fixes. I would not trust it as a research plan.</p>
      <a class="model-card__link" href="/local-model-testing/responses/gpt-oss-20b/" target="_blank" rel="noopener noreferrer">Open full answer</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-8.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-8.png" alt="Codestral system load">
    </a>
    <div class="model-card__body">
      <h3>codestral-22b-v0.1</h3>
      <div class="model-card__meta"><span>score <strong>2.5/10</strong></span><span>speed <strong>7.58 tok/sec</strong></span></div>
      <p>The answer was mostly a generic enterprise checklist rather than a security assessment.</p>
      <a class="model-card__link" href="/local-model-testing/responses/codestral-22b-v0-1/" target="_blank" rel="noopener noreferrer">Open full answer</a>
    </div>
  </article>

  <article class="model-card">
    <a class="model-card__preview" href="/assets/images/local-model-testing/image-13.png" target="_blank" rel="noopener noreferrer">
      <img class="model-card__image" src="/assets/images/local-model-testing/image-13.png" alt="VulnLLM system load">
    </a>
    <div class="model-card__body">
      <h3>vulnllm-r-7b</h3>
      <div class="model-card__meta"><span>score <strong>2/10</strong></span><span>speed <strong>12 tok/sec</strong></span></div>
      <p>The weakest result: mostly CWE-like words with little understanding of the scenario.</p>
      <a class="model-card__link" href="/local-model-testing/responses/vulnllm-r-7b/" target="_blank" rel="noopener noreferrer">Open full answer</a>
    </div>
  </article>
</div>

## Takeaway

For local security work, the strongest models stayed close to the artifacts: `acmedesk://`, SQLite, saved token storage, `latest.yml`, `update.ps1`, and backup ZIP imports. The weaker models sounded fluent in security terminology but skipped the engineering steps needed to turn a signal into a verifiable finding.
