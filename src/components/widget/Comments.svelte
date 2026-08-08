<div id="tcomment"></div>

<script>
import { onMount } from 'svelte'

const TWIKOO_ENV_ID = 'https://histcattwikoo.netlify.app/.netlify/functions/twikoo'

const DARK_STYLES = `
  /* twikoo dark mode — injected by Comments.svelte */
  html.dark .tk-comments,
  html.dark .tk-comments-container,
  html.dark .tk-comment,
  html.dark .tk-content,
  html.dark .tk-input,
  html.dark .tk-preview,
  html.dark .tk-comments-title,
  html.dark .tk-comments-count,
  html.dark .tk-nick,
  html.dark .tk-nick-link,
  html.dark .tk-meta,
  html.dark .tk-submit,
  html.dark .tk-action-link,
  html.dark .tk-avatar,
  html.dark .tk-row,
  html.dark .tk-replies,
  html.dark .tk-expand,
  html.dark .tk-owo,
  html.dark .tk-extras,
  html.dark .tk-label,
  html.dark .el-input__inner,
  html.dark .el-input__wrapper,
  html.dark .el-textarea__inner,
  html.dark .OwO .OwO-logo,
  html.dark .vemo,
  html.dark .vcontent,
  html.dark .vreply {
    color: #d1d5db !important;
  }
  html.dark .tk-time,
  html.dark .tk-row-actions,
  html.dark .tk-action,
  html.dark .tk-action-count,
  html.dark .tk-admin,
  html.dark .tk-badge {
    color: #9ca3af !important;
  }
  html.dark .el-input__inner,
  html.dark .el-textarea__inner {
    background-color: #1f2937 !important;
    border-color: #4b5563 !important;
  }
  html.dark .el-textarea__inner:focus,
  html.dark .el-input__inner:focus {
    border-color: #6366f1 !important;
  }
  html.dark .tk-comment {
    border-bottom-color: #374151 !important;
  }
`

let styleEl = null

function updateDarkStyles() {
  const isDark = document.documentElement.classList.contains('dark')
  if (isDark && !styleEl) {
    styleEl = document.createElement('style')
    styleEl.id = 'twikoo-dark-styles'
    styleEl.textContent = DARK_STYLES
    document.head.appendChild(styleEl)
  } else if (!isDark && styleEl) {
    styleEl.remove()
    styleEl = null
  }
}

onMount(() => {
  const script = document.createElement('script')
  script.src = 'https://cdn.jsdelivr.net/npm/twikoo@1.7.15/dist/twikoo.all.min.js'
  script.onload = () => {
    window.twikoo?.init({
      envId: TWIKOO_ENV_ID,
      el: '#tcomment',
    })
  }
  document.body.appendChild(script)

  // apply dark styles on mount
  updateDarkStyles()

  // observe theme changes
  const observer = new MutationObserver(() => updateDarkStyles())
  observer.observe(document.documentElement, {
    attributes: true,
    attributeFilter: ['class'],
  })
})
</script>