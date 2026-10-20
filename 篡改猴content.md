```js
// ==UserScript==
// @name         小红书网页版 · 一键回顶/触底 + 首赞速查
// @namespace    xhs-scroll-buttons
// @version      1.3.0
// @description  ①返回顶部 ②滚动触底 ③首赞速查：自动滚动让页面自然加载，同时在启动前挂钩旁路收集每页接口响应，以接口的 has_more=false 精确判断真正到底，锁定第一条点赞（虚拟滚动下"高度不变"不可靠，已弃用该判据）。
// @match        https://www.xiaohongshu.com/*
// @match        https://xiaohongshu.com/*
// @run-at       document-start
// @grant        none
// @noframes
// ==/UserScript==

(function () {
  'use strict';

  const VERSION = '1.3.0';
  const LOG = (...a) => console.log('%c[XHS首赞]', 'color:#ff2d55;font-weight:bold', ...a);

  /* ====== 可调参数 ====== */
  const WAIT_MS = 1500;      // 每轮最多等新内容的时间（有新内容会立即进入下一轮）
  const POLL_MS = 150;       // 等待期间轮询间隔
  const MAX_IDLE_ROUNDS = 4; // 连续几轮"既无新页、高度也不变"才认为结束
  const MAX_ROUNDS = 10000;  // 安全上限

  const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

  /* ================================================================
   * 响应嗅探：必须在页面启动前挂钩（@run-at document-start）。
   * 页面会把启动时的 fetch/XHR 引用保存起来，晚挂就拦不到——
   * 这正是 v1.2 抓不到包、直连 404 的原因。
   * 思路：不猜接口地址、不直连，只旁听页面自己加载到的分页数据，
   * 用响应里的 has_more 精确判断是否真正到底。
   * ================================================================ */
  const sniff = { active: false, pages: [], total: 0, done: false, lastUrl: '' };

  function pageArrayOf(d) {
    if (Array.isArray(d.notes) && d.notes.length) return d.notes;
    if (Array.isArray(d.items) && d.items.length) return d.items;
    return null;
  }

  function looksLikePagedList(j) {
    const d = j && j.data;
    if (!d || typeof d !== 'object') return false;
    if (d.has_more === undefined) return false; // 必须是带翻页语义的列表接口
    const arr = pageArrayOf(d);
    if (!arr) return false;
    const it = arr[0];
    return !!(it && (it.note_id || it.id || it.display_title || it.title));
  }

  function isDoneFlag(v) {
    return v === false || v === 0 || v === 'false';
  }

  function handleResponse(url, text) {
    let j = null;
    try { j = JSON.parse(text); } catch (e) { return; }
    if (!looksLikePagedList(j)) return;
    sniff.lastUrl = url;
    if (!sniff.active) { LOG('（旁听）发现分页列表接口：' + url); return; }
    if (!/liked/i.test(location.href)) { LOG('捕获到分页接口，但当前不在「赞」页，忽略：' + url); return; }

    const d = j.data;
    const arr = pageArrayOf(d);
    sniff.pages.push(arr);
    sniff.total += arr.length;
    if (isDoneFlag(d.has_more)) sniff.done = true;
    LOG(`捕获第 ${sniff.pages.length} 页：+${arr.length} 条，累计 ${sniff.total} 条，has_more=${d.has_more}`);
  }

  // ---- fetch 挂钩（返回原 promise，仅旁路 clone 读取，不影响页面） ----
  const origFetch = window.fetch.bind(window);
  function sniffFetch(input, init) {
    const p = origFetch(input, init);
    try {
      const url = typeof input === 'string' ? input : (input && input.url) || '';
      if (/xiaohongshu\.com\/api/i.test(url)) {
        p.then((resp) => {
          try { resp.clone().text().then((t) => handleResponse(url, t)).catch(() => {}); } catch (e) {}
        }).catch(() => {});
      }
    } catch (e) {}
    return p;
  }
  try { sniffFetch.toString = () => 'function fetch() { [native code] }'; } catch (e) {}
  window.fetch = sniffFetch;

  // ---- XHR 挂钩 ----
  const origOpen = XMLHttpRequest.prototype.open;
  const origSend = XMLHttpRequest.prototype.send;
  XMLHttpRequest.prototype.open = function (method, url) {
    this.__sniffUrl = url;
    return origOpen.apply(this, arguments);
  };
  XMLHttpRequest.prototype.send = function () {
    const u = this.__sniffUrl;
    if (u && /xiaohongshu\.com\/api/i.test(u)) {
      this.addEventListener('load', () => {
        try { if (this.responseText) handleResponse(u, this.responseText); } catch (e) {}
      });
    }
    return origSend.apply(this, arguments);
  };

  LOG(`响应嗅探已就位（v${VERSION}，document-start 挂钩）`);

  /* ====== 滚动基础 ====== */
  let scrollStop = false; // 手动滚动 / Esc 置位

  const isWindowScroller = (c) =>
    c === document.scrollingElement || c === document.documentElement || c === document.body;

  function getScrollContainer() {
    let node = document.elementFromPoint(innerWidth / 2, innerHeight * 0.4);
    while (node && node !== document.body && node !== document.documentElement) {
      const s = getComputedStyle(node);
      if (/(auto|scroll)/.test(s.overflowY) && node.scrollHeight > node.clientHeight + 10) return node;
      node = node.parentElement;
    }
    return document.scrollingElement || document.documentElement;
  }

  function scrollToTop(c) {
    if (isWindowScroller(c)) window.scrollTo({ top: 0, behavior: 'smooth' });
    else c.scrollTo({ top: 0, behavior: 'smooth' });
  }

  function jumpToBottom(c) {
    if (isWindowScroller(c)) window.scrollTo(0, c.scrollHeight);
    else c.scrollTop = c.scrollHeight;
  }

  // 「底部」按钮用的通用循环（按高度判据）
  async function scrollLoop(tag, isStopped, onRound) {
    const c = getScrollContainer();
    LOG(tag + '：开始自动滚动（想停：按 Esc 或手动滚动）');
    let lastHeight = -1, idleRounds = 0, round = 0;
    while (!isStopped() && round < MAX_ROUNDS) {
      round++;
      jumpToBottom(c);
      await sleep(WAIT_MS);
      const h = c.scrollHeight;
      LOG(tag + `：第 ${round} 轮，页面高度 ${h}` + (h === lastHeight ? '（无新内容）' : ''));
      if (h === lastHeight) {
        if (++idleRounds >= MAX_IDLE_ROUNDS) break;
      } else {
        idleRounds = 0;
      }
      lastHeight = h;
      if (onRound) onRound(round, h);
    }
    jumpToBottom(c);
    LOG(tag + '：滚动结束，共 ' + round + ' 轮');
    return round;
  }

  /* ====== 首赞速查 ====== */
  const walk = { running: false, stop: false };

  // 兜底：从 DOM 提取笔记链接（列表最后一排 = 已加载的最早一批）
  function extractFromDom() {
    const out = [];
    const seen = new Set();
    document.querySelectorAll('a[href]').forEach((a) => {
      const m = (a.getAttribute('href') || '').match(/(?:explore|discovery\/item|search_result)\/([0-9a-fA-F]{16,})/);
      if (!m || seen.has(m[1])) return;
      seen.add(m[1]);
      const title = (a.textContent || '').trim().replace(/\s+/g, ' ').slice(0, 50) || a.getAttribute('aria-label') || '(标题见页面)';
      out.push({ note_id: m[1], display_title: title });
    });
    return out;
  }

  async function findFirstLike() {
    if (walk.running) { walk.stop = true; LOG('收到停止指令…'); return; }
    walk.running = true;
    walk.stop = false;
    scrollStop = false;
    sniff.active = true;
    sniff.pages = [];
    sniff.total = 0;
    sniff.done = false;
    setBtnRunning(firstBtn, true, '运行');
    closeResult();
    LOG('开始：自动滚动 + 旁路收集页面自己的加载结果（真正的每页数据都会被记录，标题/时间齐全）');

    const c = getScrollContainer();
    let lastHeight = -1, lastPages = 0, idle = 0, round = 0;

    try {
      while (!walk.stop && !scrollStop && round < MAX_ROUNDS) {
        round++;
        jumpToBottom(c);
        // 等待新内容：一旦"捕获页数增加或高度变化"立即进入下一轮，不等满 WAIT_MS
        let changed = false;
        for (let t = 0; t < WAIT_MS && !changed && !walk.stop && !scrollStop; t += POLL_MS) {
          await sleep(POLL_MS);
          changed = sniff.pages.length !== lastPages || c.scrollHeight !== lastHeight;
        }
        const h = c.scrollHeight;
        const pages = sniff.pages.length;
        firstBtn.querySelector('span').textContent = '停止 ' + sniff.total + '条';
        if (changed) {
          idle = 0;
          LOG(`第 ${round} 轮 | 高度 ${h} | 已捕获 ${pages} 页 / 共 ${sniff.total} 条`);
        } else {
          idle++;
          LOG(`第 ${round} 轮 | 无新内容（${idle}/${MAX_IDLE_ROUNDS}）| 高度 ${h} | ${pages} 页 / ${sniff.total} 条`);
          if (idle >= MAX_IDLE_ROUNDS) { LOG('连续多轮无新内容，结束'); break; }
        }
        lastHeight = h;
        lastPages = pages;
        if (sniff.done) { LOG('接口返回 has_more=false，已到真正的最后一页 ✔'); break; }
      }

      if (sniff.total) {
        const lastPage = sniff.pages[sniff.pages.length - 1];
        const earliest = [...lastPage].reverse();
        showResult(earliest, sniff.total, sniff.pages.length, sniff.done);
        if (!sniff.done) toast('注意：加载提前停止，结果可能不是最早的（详见面板提示）');
        return;
      }

      // 一页接口数据都没旁路到（页面可能用了别的加载方式）→ DOM 提取兜底
      LOG('未能旁路收集到接口数据，改用 DOM 提取兜底');
      const dom = extractFromDom();
      if (!dom.length) { toast('没找到笔记链接，详情看控制台日志'); return; }
      showResult([...dom].reverse(), dom.length, 0, false);
    } finally {
      sniff.active = false;
      walk.running = false;
      walk.stop = false;
      setBtnRunning(firstBtn, false, '首赞');
    }
  }

  /* ================= UI ================= */
  let topBtn = null, bottomBtn = null, firstBtn = null;
  let setBtnRunning = function () {}; // initUI 里赋真正实现

  function initUI() {
    const style = document.createElement('style');
    style.textContent = `
      .xhs-sb-btn:hover{transform:translateY(-2px) !important;filter:brightness(1.06)}
      .xhs-sb-btn:active{transform:translateY(0) !important}
    `;
    (document.head || document.documentElement).appendChild(style);

    const SVG_UP = '<svg viewBox="0 0 24 24"><path d="M12 4l-8 8h5v8h6v-8h5z"/></svg>';
    const SVG_DOWN = '<svg viewBox="0 0 24 24"><path d="M12 20l8-8h-5V4H9v8H4z"/></svg>';
    const SVG_HEART = '<svg viewBox="0 0 24 24"><path d="M12 21s-7.6-4.9-10-9.3C.3 8 2.4 4.4 6.4 4.4c2.2 0 3.9 1.2 4.6 2.6.7-1.4 2.4-2.6 4.6-2.6 4 0 6.1 3.6 4.4 7.3C19.6 16.1 12 21 12 21z"/></svg>';
    const SVG_STOP = '<svg viewBox="0 0 24 24"><rect x="6" y="6" width="12" height="12" rx="2"/></svg>';

    function makeBtn(svg, label) {
      const b = document.createElement('button');
      b.className = 'xhs-sb-btn';
      b.innerHTML = svg + '<span></span>';
      b.querySelector('span').textContent = label;
      return b;
    }

    function paintBtn(b, svg, bg) {
      b.style.cssText =
        'width:50px;height:50px;border:none;border-radius:50%;cursor:pointer;display:flex;flex-direction:column;align-items:center;justify-content:center;gap:2px;background:' + bg + ';color:#fff;font-size:11px;line-height:1;box-shadow:0 4px 14px rgba(0,0,0,.2)';
      b.querySelector('svg').outerHTML = svg;
      const svgEl = b.querySelector('svg');
      if (svgEl) svgEl.style.cssText = 'width:15px;height:15px;fill:currentColor';
    }

    const panel = document.createElement('div');
    panel.id = 'xhs-sb-panel';
    panel.style.cssText =
      'position:fixed;right:20px;bottom:150px;z-index:2147483000;display:flex;flex-direction:column;gap:10px;user-select:none;font-family:system-ui,-apple-system,"PingFang SC","Microsoft YaHei",sans-serif';

    topBtn = makeBtn(SVG_UP, '顶部');
    bottomBtn = makeBtn(SVG_DOWN, '底部');
    firstBtn = makeBtn(SVG_HEART, '首赞');
    for (const b of [topBtn, bottomBtn, firstBtn]) paintBtn(b, b.querySelector('svg').outerHTML, 'rgba(255,45,85,.92)');

    topBtn.title = '返回顶部';
    bottomBtn.title = '滚动到底部并持续等待加载（再点一次 / 滚动 / Esc 停止）';
    firstBtn.title = '自动滚动并旁路收集每页数据，精确找出第一条点赞（再点一次 / Esc 停止）';

    let scrollRunning = false;
    topBtn.addEventListener('click', () => scrollToTop(getScrollContainer()));
    bottomBtn.addEventListener('click', async () => {
      if (scrollRunning) { scrollStop = true; return; }
      scrollRunning = true;
      scrollStop = false;
      setBtnRunning(bottomBtn, true, '停止');
      await scrollLoop('[XHS触底]', () => scrollStop,
        (r) => { bottomBtn.querySelector('span').textContent = '停止 ' + r; });
      scrollRunning = false;
      setBtnRunning(bottomBtn, false, '底部');
      toast(scrollStop ? '已停止（随时可继续）' : '🎉 已到底部，最下面的就是最早的记录');
    });
    firstBtn.addEventListener('click', findFirstLike);
    panel.append(topBtn, bottomBtn, firstBtn);

    window.addEventListener('wheel', () => { scrollStop = true; }, { passive: true });
    window.addEventListener('touchmove', () => { scrollStop = true; }, { passive: true });
    window.addEventListener('keydown', (e) => {
      if (e.key !== 'Escape') return;
      scrollStop = true;
      if (walk.running) walk.stop = true;
    });

    setBtnRunning = function (btn, on, label) {
      const svg = btn === firstBtn ? SVG_HEART : SVG_DOWN;
      paintBtn(btn, on ? SVG_STOP : svg, on ? '#333333' : 'rgba(255,45,85,.92)');
      btn.querySelector('span').textContent = label;
    };

    function ensurePanel() {
      if (!document.body.contains(panel)) document.body.appendChild(panel);
    }
    setInterval(ensurePanel, 3000);
    ensurePanel();
    console.log(`[XHS滚动按钮] 界面已就绪 v${VERSION}`);
  }

  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', initUI);
  } else {
    initUI();
  }

  /* ---- toast 与结果面板挂在 document.body 上，惰性创建 ---- */
  let toastTimer = null;
  function toast(msg) {
    if (!document.body) return;
    let t = document.getElementById('xhs-sb-toast');
    if (!t) {
      t = document.createElement('div');
      t.id = 'xhs-sb-toast';
      t.style.cssText =
        'position:fixed;top:24px;left:50%;transform:translateX(-50%);background:rgba(0,0,0,.78);color:#fff;padding:9px 18px;border-radius:20px;font-size:13px;z-index:2147483000;opacity:0;pointer-events:none;transition:opacity .3s;font-family:system-ui,-apple-system,"PingFang SC","Microsoft YaHei",sans-serif';
      document.body.appendChild(t);
    }
    t.textContent = msg;
    t.style.opacity = '1';
    clearTimeout(toastTimer);
    toastTimer = setTimeout(() => (t.style.opacity = '0'), 3200);
  }

  function closeResult() {
    const old = document.getElementById('xhs-sb-result');
    if (old) old.remove();
  }

  const noteTitle = (it) => it.display_title || it.title || it.name || '(无标题)';
  const noteId = (it) => it.note_id || it.id || it.noteId || '';
  const noteTime = (it) => {
    const t = it.liked_time ?? it.timestamp ?? it.time ?? it.create_time;
    if (!t) return '';
    const ms = t > 1e12 ? t : t > 1e9 ? t * 1000 : null;
    return ms ? new Date(ms).toLocaleString('zh-CN') : '';
  };
  const noteLink = (it) => noteId(it) ? 'https://www.xiaohongshu.com/explore/' + noteId(it) : '';

  function showResult(earliest, total, pages, complete) {
    closeResult();
    const first = earliest[0];

    const box = document.createElement('div');
    box.id = 'xhs-sb-result';
    box.style.cssText =
      'position:fixed;right:20px;bottom:150px;width:300px;max-height:65vh;overflow:auto;background:#fff;border-radius:14px;box-shadow:0 8px 30px rgba(0,0,0,.2);z-index:2147483000;padding:14px;font-size:13px;color:#333;line-height:1.6;font-family:system-ui,-apple-system,"PingFang SC","Microsoft YaHei",sans-serif';

    const close = document.createElement('div');
    close.textContent = '×';
    close.style.cssText = 'position:absolute;top:6px;right:12px;font-size:18px;cursor:pointer;color:#999';
    close.addEventListener('click', closeResult);
    box.appendChild(close);

    const add = (html) => { const d = document.createElement('div'); d.innerHTML = html; box.appendChild(d); return d; };

    add('<div style="font-weight:bold;font-size:14px;margin-bottom:6px">🎯 你第一条点赞的笔记</div>');
    const firstDiv = add(
      '<div style="color:#ff2d55;font-weight:bold;margin:4px 0">' + noteTitle(first) + '</div>' +
      (noteTime(first) ? '<div style="color:#999;font-size:12px">点赞时间：' + noteTime(first) + '</div>' : '')
    );
    const link = noteLink(first);
    if (link) {
      const a = document.createElement('a');
      a.href = link; a.target = '_blank'; a.rel = 'noopener';
      a.textContent = '打开这篇笔记 ↗';
      a.style.cssText = 'color:#06c;display:inline-block;margin:6px 0 2px';
      firstDiv.appendChild(a);
      const copy = document.createElement('button');
      copy.textContent = '复制链接';
      copy.style.cssText = 'margin-left:8px;border:1px solid #ddd;background:#fafafa;border-radius:6px;padding:2px 10px;cursor:pointer;font-size:12px;color:#333';
      copy.addEventListener('click', () => {
        navigator.clipboard.writeText(link).then(() => toast('链接已复制')).catch(() => toast('复制失败：' + link));
      });
      firstDiv.appendChild(copy);
    }

    add('<div style="margin-top:10px;font-weight:bold">最早的点赞（倒序）</div>');
    const list = document.createElement('div');
    earliest.slice(0, 10).forEach((it, i) => {
      const row = document.createElement('div');
      row.style.cssText = 'padding:3px 0;border-bottom:1px dashed #f0f0f0;white-space:nowrap;overflow:hidden;text-overflow:ellipsis';
      const t = document.createElement('span');
      t.textContent = (i + 1) + '. ' + noteTitle(it);
      row.appendChild(t);
      const lk = noteLink(it);
      if (lk && i > 0) {
        const a = document.createElement('a');
        a.href = lk; a.target = '_blank'; a.rel = 'noopener';
        a.textContent = ' ↗'; a.style.color = '#06c';
        row.appendChild(a);
      }
      list.appendChild(row);
    });
    box.appendChild(list);
    add(
      '<div style="color:#999;font-size:12px;margin-top:8px">共 ' + total + ' 条' +
      (pages ? ' / 依据 ' + pages + ' 页接口数据' : '') +
      (complete ? '' : '<br><b style="color:#e67e22">注意：加载提前停止，上面可能还不是最早的</b>') +
      '</div>'
    );

    document.body.appendChild(box);
    toast(complete ? '🎉 完成！已确认翻到最后一页' : '完成（但未确认到最后一页）');
    LOG('第一条点赞：' + noteTitle(first) + ' ' + link);
    if (link && navigator.clipboard) navigator.clipboard.writeText(link).catch(() => {});
  }
})();
```