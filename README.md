# 学习公社云平台倍速与跨课程连播

用于学习公社云平台（`study.enaea.edu.cn`）的课程表页。脚本读取当前课程表的全部分页，在一个独立的播放标签页中顺序播放未完成的视频课，并自动处理休息提醒。默认 1 倍速，上限 3 倍速，可选 1、1.5、2、3 倍速。

课程表页是控制页，脚本一直运行在这里，通过同源窗口访问播放页，因此更换课程后仍然有效。一门课的全部分集在服务端达到 100% 后进入下一门；视频到片尾但记录未满时，会从头重播未完成分集以补足学时。

## 怎么用

在 Chrome 中登录平台，进入班级的“课程学习”课程表页，地址应包含 `circleIndexRedirect.do?action=toNewMyClass&type=course`。先关闭或暂停其他正在播放的课程，平台会检测多门课程同时计时。

在课程表页按 `⌘ + Option + J`（macOS）或 `Ctrl + Shift + J`（Windows / Linux）打开 Console（控制台），粘贴下面的完整代码并执行。右下角出现面板后，点击“读取课程”，选择倍速，再点击“开始连播”。如果新标签页被拦截，允许本网站打开弹出式窗口后重试。

面板里的“设定倍速”是你选择的目标值，“实际倍速”与播放/暂停/加载状态由播放页的媒体事件即时更新，并有每秒采样兜底；切回控制页时会立即重新读取。分集切换或播放器刚启动时，两者可能暂时不同。正常播放后的手动暂停不会被反复恢复；首次启动失败最多重试三轮，Chrome 拦截有声自动播放时会尝试静音启动，可在播放页手动开启声音。

“服务端已学”来自重新读取的课程表汇总，每分钟同步一次，也可点击“刷新学时”。“本次连播增量”是最新汇总与本次开播前基准的差值，跨课程或切换倍速不重置，停止后重新开始会重新建立基准。同步失败会保留上次数据并提示；如果开播时读取失败，则在首次成功同步时建立基准，面板会显示基准时间。这个增量不是根据播放速度估算的，也不等于刚切换到某个倍速后的单独收益。

保持控制页和播放页都打开，可以关闭开发者工具。课程中的测验需要手动处理。

```javascript
(() => {
  "use strict";
  window.enaeaPlayer?.destroy();
  const params = new URL(location.href).searchParams;
  const circleId = params.get("circleId");
  const syllabusId = params.get("syllabusId");
  if (location.hostname !== "study.enaea.edu.cn" ||
      location.pathname !== "/circleIndexRedirect.do" ||
      params.get("type") !== "course" || !circleId || !syllabusId) {
    console.error("请在班级的课程表页运行此脚本。");
    return;
  }

  const panel = document.createElement("section");
  panel.id = "enaea-controller";
  panel.style.cssText = "position:fixed;right:20px;bottom:20px;z-index:2147483647;width:300px;padding:16px;background:white;color:#222;border:1px solid #ccc;border-radius:10px;box-shadow:0 4px 24px #0003;font:14px/1.6 sans-serif;";
  panel.innerHTML = `<strong>学习公社 · 课程连播</strong>
    <p data-status style="margin:8px 0;white-space:pre-wrap">请先读取课程列表。</p>
    <label>设定倍速 <select data-rate><option value="1" selected>1×</option><option value="1.5">1.5×</option><option value="2">2×</option><option value="3">3×</option></select></label>
    <p data-video style="margin:8px 0">实际倍速：— · 尚未开始</p>
    <p data-study style="margin:8px 0;white-space:pre-wrap">服务端已学：尚未读取</p>
    <div style="margin-top:10px;display:flex;gap:8px;flex-wrap:wrap">
      <button data-load>读取课程</button><button data-start disabled>开始连播</button>
      <button data-stop>停止</button><button data-close>移除面板</button>
      <button data-refresh>刷新学时</button>
    </div>`;
  document.body.append(panel);
  const status = panel.querySelector("[data-status]");
  const loadButton = panel.querySelector("[data-load]");
  const startButton = panel.querySelector("[data-start]");
  const rateSelect = panel.querySelector("[data-rate]");
  const videoStatus = panel.querySelector("[data-video]");
  const studyStatus = panel.querySelector("[data-study]");
  const refreshButton = panel.querySelector("[data-refresh]");
  const allowedRates = new Set([1, 1.5, 2, 3]);
  let queue = [];
  let player = null;
  let run = null;
  let loading = false;
  let destroyed = false;
  let mediaBinding = null;
  const study = { value: null, baseline: null, baselineAt: 0, updatedAt: 0,
    error: "", requestId: 0, controller: null, pending: false };

  /** 限制实际设置值，避免旧面板或异常选项把倍速带出支持范围。 */
  function selectedRate() {
    const value = Number(rateSelect.value);
    if (allowedRates.has(value)) return value;
    rateSelect.value = "1";
    return 1;
  }

  /** 解绑旧文档，避免换课或停止后旧视频事件继续更新面板。 */
  function detachPlayerEvents() {
    if (!mediaBinding) return;
    for (const event of mediaBinding.events) {
      mediaBinding.doc.removeEventListener(event, mediaBinding.handler, true);
    }
    mediaBinding.observer?.disconnect();
    mediaBinding = null;
  }

  /** 在播放文档捕获媒体事件，视频节点被替换后也能同步；不依赖控制页定时器。 */
  function bindPlayerEvents(doc) {
    if (!run || mediaBinding?.doc === doc) return;
    detachPlayerEvents();
    const current = run;
    const events = ["play", "playing", "pause", "ended", "waiting", "canplay",
      "loadedmetadata", "emptied", "ratechange", "volumechange", "timeupdate"];
    const isActive = () => {
      try { return !destroyed && run === current && doc === player.document; }
      catch { return false; }
    };
    const handler = event => {
      if (!isActive() || event.target !== doc.querySelector("video")) return;
      if (event.type === "playing") {
        // 平台的 play 事件可能重置为1倍；等本轮处理完成后再落实用户选择，不主动恢复暂停。
        queueMicrotask(() => {
          if (!isActive() || event.target !== doc.querySelector("video")) return;
          event.target.playbackRate = selectedRate();
          renderVideoState();
        });
      }
      renderVideoState();
    };
    for (const event of events) doc.addEventListener(event, handler, true);
    const observer = typeof player.MutationObserver === "function"
      ? new player.MutationObserver(() => {
        if (isActive()) renderVideoState();
      }) : null;
    observer?.observe(doc.documentElement, { childList: true, subtree: true });
    mediaBinding = { doc, events, handler, observer };
  }

  function setVideoStatus(text) {
    if (videoStatus.textContent !== text) videoStatus.textContent = text;
  }

  /** 媒体事件即时更新，定时采样与控制页重新可见时的读取仅作兜底。 */
  function renderVideoState() {
    try {
      if (!player || player.closed) {
        setVideoStatus("实际倍速：— · 未连接播放页");
        return;
      }
      const doc = player.document;
      bindPlayerEvents(doc);
      const video = doc.querySelector("video");
      if (!video) {
        setVideoStatus("实际倍速：— · 正在加载或切换分集");
        return;
      }
      const state = video.ended ? "已到片尾" : video.readyState < 2 ? "正在加载" :
        video.paused ? "已暂停，可在播放页恢复" : video.readyState < 3 ? "正在缓冲" : "播放中";
      setVideoStatus(`实际倍速：${video.playbackRate}× · ${state}${video.muted ? " · 静音" : ""}`);
    } catch {
      setVideoStatus("实际倍速：— · 暂时无法访问播放页");
    }
  }

  /** 增量只取服务端汇总之差，读取失败时保留上次值并明确显示数据已过期。 */
  function renderStudy() {
    const lines = [study.value === null ? "服务端已学：尚未读取" : `服务端已学：${study.value} 分钟`];
    if (study.baseline !== null && study.value !== null) {
      const delta = study.value - study.baseline;
      lines.push(delta < 0 ? "汇总值回退，本次增量暂不计算" : `本次连播增量：${delta} 分钟`);
      lines.push(`基准时间：${new Date(study.baselineAt).toLocaleTimeString()}`);
    }
    if (study.updatedAt) lines.push(`上次同步：${new Date(study.updatedAt).toLocaleTimeString()}`);
    if (study.error) lines.push(`同步失败：${study.error}；以上为上次成功数据。`);
    else if (study.pending) lines.push("正在同步…");
    studyStatus.textContent = lines.join("\n");
    refreshButton.disabled = study.pending;
  }

  /** 取消旧会话的只读请求，防止停止/重启后的旧结果覆盖新基准。 */
  function cancelStudyRead() {
    study.requestId++;
    study.controller?.abort();
    study.controller = null;
    study.pending = false;
  }

  /** 每分钟读取课程表汇总 HTML，不推算或写入平台学时。 */
  async function refreshStudy(current = run, resetBaseline = false) {
    if (destroyed || (study.pending && !resetBaseline)) return;
    cancelStudyRead();
    const requestId = study.requestId;
    const controller = new AbortController();
    study.controller = controller;
    study.pending = true;
    const timeout = setTimeout(() => controller.abort(), 15000);
    renderStudy();
    try {
      const url = new URL(location.href);
      url.searchParams.set("ct", String(Date.now()));
      const response = await fetch(url, {
        credentials: "same-origin", cache: "no-store", signal: controller.signal,
      });
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      const html = await response.text();
      if (destroyed || requestId !== study.requestId || run !== current) return;
      const doc = new DOMParser().parseFromString(html, "text/html");
      const label = [...doc.querySelectorAll(".class-survey .require")]
        .map(node => node.textContent.trim()).find(text => /^已学[：:]/.test(text));
      const match = label?.match(/^已学[：:]\s*(\d+)\s*分钟\s*$/);
      if (!match) throw new Error("未找到已学总分钟数，请检查登录状态或页面变化");
      const value = Number(match[1]);
      if (!Number.isSafeInteger(value)) throw new Error("已学分钟数异常");
      study.value = value;
      study.updatedAt = Date.now();
      study.error = "";
      if (current && (resetBaseline || study.baseline === null)) {
        study.baseline = value;
        study.baselineAt = study.updatedAt;
      }
    } catch (error) {
      if (!destroyed && requestId === study.requestId && run === current) {
        study.error = error.name === "AbortError" ? "读取超时" : error.message;
      }
    } finally {
      clearTimeout(timeout);
      if (requestId === study.requestId) {
        study.pending = false;
        study.controller = null;
        if (!destroyed) renderStudy();
      }
    }
  }

  /** 只读列表与完成状态；超时或登录失效时停止，不重放学时上报请求。 */
  async function readJSON(path, values) {
    const url = new URL(path, location.origin);
    url.search = new URLSearchParams({ ...values, _: String(Date.now()) });
    const response = await fetch(url, {
      credentials: "same-origin", cache: "no-store",
      signal: AbortSignal.timeout(15000),
    });
    if (!response.ok) throw new Error(`读取失败：HTTP ${response.status}`);
    // 平台部分 JSON 接口返回 text/html，按正文解析，登录页等 HTML 会在这里失败。
    let data;
    try { data = await response.json(); }
    catch { throw new Error("未返回课程数据，请检查登录状态。"); }
    if (data.success === false) throw new Error(data.message || "平台拒绝读取课程。");
    if (!Array.isArray(data.result?.list)) throw new Error("课程数据格式发生变化。");
    return data;
  }

  /** 沿用课程表的分页和排序，去重后保留有权限、未完成的普通视频课。 */
  async function loadCourses() {
    if (loading || run) return;
    loading = true;
    queue = [];
    startButton.disabled = true;
    loadButton.disabled = true;
    const rows = [];
    try {
      let total = Infinity;
      for (let start = 0; start < total;) {
        status.textContent = `正在读取课程：${rows.length} 门…`;
        const data = await readJSON("/circleIndex.do", {
          action: "getMyClass", start: String(start), limit: "20",
          circleId, syllabusId, isCompleted: "", categoryRemark: "all",
        });
        if (destroyed) return;
        total = Number(data.totalCount);
        if (!Number.isInteger(total) || total < 0 || total > 5000) {
          throw new Error("课程总数异常。");
        }
        if (!data.result.list.length && start < total) throw new Error("课程分页提前结束。");
        rows.push(...data.result.list);
        start += data.result.list.length;
        if (!data.result.list.length) break;
      }
      const seen = new Set();
      for (const row of rows) {
        const course = row.studyCenterDTO;
        const id = String(course?.courseId || "");
        const progress = Number(course?.studyProgress ?? 0);
        if (!/^\d+$/.test(id) || seen.has(id)) continue;
        seen.add(id);
        if (row.canView !== true || course.coursecontentType !== "video" ||
            !Number.isFinite(progress) || progress >= 100) continue;
        queue.push({ id, title: course.courseTitle || id });
      }
      status.textContent = `已读取 ${rows.length} 门，待播放 ${queue.length} 门。\n已完成、无权限及外链等课程不加入队列。`;
      startButton.disabled = queue.length === 0;
    } catch (error) {
      queue = [];
      status.textContent = `读取失败：${error.message}`;
    } finally {
      loading = false;
      loadButton.disabled = false;
    }
  }

  /** 只匹配休息提醒，不自动关闭测验、登录异常等其他弹窗。 */
  function continueAfterRest(doc) {
    for (const button of doc.querySelectorAll("button,a,input[type=button],input[type=submit]")) {
      if ((button.textContent || button.value || "").trim() !== "继续学习" ||
          !button.getClientRects().length) continue;
      let container = button.parentElement;
      for (let depth = 0; container && container !== doc.body && depth < 8; depth++) {
        const text = container.textContent || "";
        if (text.includes("您已学习") && text.includes("让眼睛休息一下吧")) {
          button.click();
          return;
        }
        container = container.parentElement;
      }
    }
  }

  /** 始终复用同一个播放标签页，避免同时播放多门课程。 */
  function navigate(current) {
    detachPlayerEvents();
    current.loadedAt = Date.now();
    current.playerMissingSince = null;
    current.lastCheck = 0;
    current.endedSince = 0;
    current.playback = null;
    current.replayProgress = new Map();
    current.originalMutes = new Map();
    current.switchingUntil = 0;
    status.textContent = `${current.index + 1}/${queue.length} · ${queue[current.index].title}\n正在打开课程…`;
    const url = new URL("/viewerforccvideo.do", location.origin);
    url.search = new URLSearchParams({ courseId: queue[current.index].id, circleId });
    player.location.href = url.href;
  }

  /** 仅停止本控制页管理的窗口，恢复正常速度，避免留下后台播放。 */
  function stop(message = "已停止，可重新读取课程后继续。") {
    const previous = run;
    if (run) {
      clearInterval(run.timer);
      clearInterval(run.studyTimer);
    }
    run = null;
    detachPlayerEvents();
    cancelStudyRead();
    try {
      if (player && !player.closed) {
        for (const video of player.document.querySelectorAll("video")) {
          video.pause();
          video.playbackRate = 1;
          if (previous?.originalMutes.has(video)) video.muted = previous.originalMutes.get(video);
        }
      }
    } catch (error) {
      // 登录跳转可能离开同源范围，此时不能再控制该窗口。
      console.warn("[课程连播] 请手动检查播放页：", error.message);
    }
    startButton.disabled = !queue.length;
    loadButton.disabled = false;
    status.textContent = message;
    renderVideoState();
    renderStudy();
  }

  /** 播放失败不记为成功；仅在首次启动阶段重试，播放过后的手动暂停保持不变。 */
  async function ensureStarted(current, video, contentId) {
    const source = `${contentId}:${video.currentSrc}`;
    let state = current.playback;
    if (!state || state.video !== video || state.source !== source) {
      state = { video, source, position: video.currentTime, started: false,
        attempts: 0, lastAttempt: 0, message: "" };
      current.playback = state;
    }
    if (!video.paused && video.currentTime > state.position + 0.25) {
      state.started = true;
      state.message = "";
    }
    if (state.started) return video.paused && !video.ended ? "已暂停，请在播放页手动恢复。" : "";
    if (!video.paused) return "正在确认播放已启动…";
    if (video.ended || video.readyState < 2 || !video.currentSrc) return "正在等待视频加载…";
    if (state.attempts >= 3) return state.message || "自动启动失败，请在播放页手动播放。";
    if (Date.now() - state.lastAttempt < 5000) return state.message || "正在等待重试自动播放…";
    state.lastAttempt = Date.now();
    state.attempts++;
    try {
      try {
        await video.play();
      } catch (error) {
        // 有声自动播放被拦截时改为静音启动，不通过伪造点击绕过浏览器策略。
        if (error.name !== "NotAllowedError" || video.muted || run !== current) throw error;
        if (!current.originalMutes.has(video)) current.originalMutes.set(video, video.muted);
        video.muted = true;
        await video.play();
      }
      if (run !== current) {
        if (!run) video.pause();
        return "";
      }
      state.message = "正在确认播放已启动…";
    } catch (error) {
      if (run !== current) return "";
      state.message = `自动播放失败（${error.name || "Error"}）：${error.message || "未知原因"}。`;
      if (state.attempts >= 3) state.message += " 请在播放页手动播放。";
      else state.message += " 稍后自动重试。";
      console.warn("[课程连播]", state.message);
    }
    return state.message;
  }

  /** 以服务端记录校正界面，避免最后一次上报和页面更新的先后顺序影响换课。 */
  async function confirmProgress(current, items) {
    const data = await readJSON("/course.do", {
      action: "getCourseContentList", courseId: queue[current.index].id, circleId,
    });
    if (run !== current) return null;
    const records = new Map(data.result.list.map(item => [String(item.id), Number(item.studyProgress)]));
    if (records.size !== items.length || records.size !== data.result.list.length ||
        items.some(item => !records.has(item.dataset.id)) ||
        [...records.values()].some(progress => !Number.isFinite(progress) || progress < 0 || progress > 100)) {
      throw new Error("服务端分集数据与播放页不一致。");
    }
    for (const item of items) {
      item.querySelector('[id^="J_studyProgress_"]').textContent = `${records.get(item.dataset.id)}%`;
    }
    return records;
  }

  /** 全部分集已确认完成后复用窗口进入下一门；队列结束则停止播放。 */
  function nextCourse(current) {
    current.index++;
    if (current.index >= queue.length) return stop("队列中的视频课程已全部完成。");
    navigate(current);
  }

  /** 片尾先留出最后一次上报时间；时长不足时重播，连续无增量则停止避免空转。 */
  async function handleEnded(current, items, video, selected) {
    current.endedSince ||= Date.now();
    status.textContent = "视频已结束，正在核对最后一次学时记录…";
    if (Date.now() - current.endedSince < 5000 || Date.now() - current.lastCheck < 5000) return;
    current.lastCheck = Date.now();
    const records = await confirmProgress(current, items);
    if (!records) return;
    const target = items.find(item => records.get(item.dataset.id) < 100);
    if (!target) return nextCourse(current);
    current.endedSince = 0;
    current.playback = null;
    if (target !== selected) {
      current.switchingUntil = Date.now() + 8000;
      target.click();
      return;
    }
    const progress = records.get(target.dataset.id);
    const previous = current.replayProgress.get(target.dataset.id);
    const stalls = previous && progress <= previous.progress ? previous.stalls + 1 : 0;
    if (stalls >= 2) throw new Error("该分集重播后连续两次学时未增加，请检查平台计时或网络。");
    current.replayProgress.set(target.dataset.id, { progress, stalls });
    // 断点续播可能到达片尾但累计时长仍不足；从头真实播放补足，不修改上报数据。
    video.currentTime = 0;
    const message = await ensureStarted(current, video, target.dataset.id);
    if (run !== current) return;
    video.playbackRate = selectedRate();
    status.textContent = `该分集已到片尾，学时进度 ${progress}%，正在从头补足播放。\n${message}`;
  }

  /** 异步检查串行执行，停止或重启后旧请求不得继续导航。 */
  async function tick(current) {
    if (run !== current || current.busy) return;
    current.busy = true;
    try {
      if (!player || player.closed) throw new Error("播放标签页已关闭。");
      const url = new URL(player.location.href);
      if (url.pathname !== "/viewerforccvideo.do") {
        if (url.pathname.includes("login") || Date.now() - current.loadedAt > 30000) {
          throw new Error("播放页未能打开，请检查登录状态或课程权限。");
        }
        return;
      }
      if (url.searchParams.get("courseId") !== queue[current.index].id) {
        if (Date.now() - current.loadedAt > 30000) throw new Error("播放页已离开当前课程。");
        return;
      }
      const doc = player.document;
      bindPlayerEvents(doc);
      const items = [...doc.querySelectorAll(".cvtb-MCK-course-content")];
      const video = doc.querySelector("video");
      if (!items.length || !video) {
        // 分集切换会暂时移除视频节点；只计算本次连续缺失，不能沿用整门课的打开时间。
        current.playerMissingSince ??= Date.now();
        if (Date.now() - current.playerMissingSince > 60000) throw new Error("播放器加载超时。");
        status.textContent = "正在等待播放器初始化或切换分集…";
        return;
      }
      current.playerMissingSince = null;
      continueAfterRest(doc);
      video.playbackRate = selectedRate();
      // 休息提醒之外的弹窗留给用户，不切分集，也不尝试恢复播放。
      if ([...doc.querySelectorAll(".dialog-header")].some(el => el.getClientRects().length)) {
        status.textContent = "播放页有提示或测验，请手动处理后继续。";
        return;
      }
      const unfinished = items.filter(item => {
        const progress = item.querySelector('[id^="J_studyProgress_"]')?.textContent.trim();
        if (!/^\d+(\.\d+)?%$/.test(progress || "")) throw new Error("无法识别分集进度。");
        return Number.parseFloat(progress) < 100;
      });
      const selected = items.find(item => item.classList.contains("current"));
      if (video.ended) {
        await handleEnded(current, items, video, selected);
        return;
      }
      current.endedSince = 0;
      if (unfinished.length) {
        // 平台常从第一集打开；已完成分集直接切到尚未完成的一集。
        if (!selected || !unfinished.includes(selected)) {
          if (Date.now() > current.switchingUntil) {
            current.switchingUntil = Date.now() + 8000;
            unfinished[0].click();
          }
          return;
        }
        const source = video.currentSrc;
        let playbackMessage = await ensureStarted(current, video, selected?.dataset.id);
        // 播放承诺返回时可能已经切了分集，不能把旧视频的暂停/失败状态写进新分集面板。
        if (run !== current || doc !== player.document || doc.querySelector("video") !== video ||
            video.currentSrc !== source || !selected.classList.contains("current")) return;
        // 暂停状态由独立监视器即时显示，避免两处文案给出不同结论。
        if (playbackMessage === "已暂停，请在播放页手动恢复。") playbackMessage = "";
        status.textContent = `${current.index + 1}/${queue.length} · ${queue[current.index].title}\n剩余 ${unfinished.length} 个未完成分集`;
        if (video.muted) status.textContent += "\n当前静音，可在播放页手动开启声音。";
        if (playbackMessage) status.textContent += `\n${playbackMessage}`;
        if (current.replayProgress.has(selected?.dataset.id)) status.textContent += "\n正在重播以补足该分集学时。";
        return;
      }
      // 服务端确认所有分集完成后才导航，避免丢失最后一次记录。
      if (Date.now() - current.lastCheck < 10000) return;
      current.lastCheck = Date.now();
      const records = await confirmProgress(current, items);
      if (!records) return;
      if ([...records.values()].some(progress => progress < 100)) {
        status.textContent = "已同步服务端进度，将继续未完成分集…";
        return;
      }
      nextCourse(current);
    } catch (error) {
      if (run === current) stop(`已停止：${error.message}`);
    } finally {
      current.busy = false;
    }
  }

  /** 开播前读取新基准；汇总不可读时仍可播放，待后续同步成功再建立基准。 */
  async function begin(current) {
    try {
      await refreshStudy(current, true);
      if (run !== current) return;
      navigate(current);
      current.timer = setInterval(() => void tick(current), 2000);
      current.studyTimer = setInterval(() => void refreshStudy(current), 60000);
    } catch (error) {
      if (run === current) stop(`无法打开播放页：${error.message}`);
    }
  }

  /** 由按钮点击同步开窗，避免跨页面时被弹窗拦截。 */
  function start() {
    if (run || loading || !queue.length) return;
    // 沿用课程表的“达标后锁课”限制，不打开已被项目锁定的新课。
    if (String(window.closedCourseTag) === "true" && Number(window.assessmentRequire) > 0 &&
        Number(window.assessmentComplete) >= Number(window.assessmentRequire)) {
      status.textContent = "项目已达标并启用锁课，请在平台确认后手动选择课程。";
      return;
    }
    if (!player || player.closed) player = window.open("about:blank", "enaea-managed-player");
    if (!player) {
      status.textContent = "新标签页被拦截，请允许本网站打开弹出式窗口后重试。";
      return;
    }
    run = { index: 0, busy: false, endedSince: 0 };
    const current = run;
    study.baseline = null;
    study.baselineAt = 0;
    startButton.disabled = true;
    loadButton.disabled = true;
    status.textContent = "正在读取本次连播的学时基准…";
    void begin(current);
  }

  /** 清理定时器和事件；重复粘贴时先销毁上一个实例。 */
  function destroy() {
    destroyed = true;
    stop();
    clearInterval(videoTimer);
    panel.remove();
    window.removeEventListener("pagehide", onPageHide);
    window.removeEventListener("focus", renderVideoState);
    document.removeEventListener("visibilitychange", renderVideoState);
    delete window.enaeaPlayer;
  }

  function onPageHide() { stop("控制页已离开，播放已停止。"); }
  loadButton.addEventListener("click", () => void loadCourses());
  startButton.addEventListener("click", start);
  panel.querySelector("[data-stop]").addEventListener("click", () => stop());
  panel.querySelector("[data-close]").addEventListener("click", destroy);
  refreshButton.addEventListener("click", () => void refreshStudy());
  rateSelect.addEventListener("change", () => {
    const rate = selectedRate();
    try {
      const video = run && player && !player.closed ? player.document.querySelector("video") : null;
      if (video) video.playbackRate = rate;
    } catch (error) {
      console.warn("[课程连播] 暂时无法设置倍速：", error.message);
    }
    renderVideoState();
  });
  window.addEventListener("pagehide", onPageHide);
  window.addEventListener("focus", renderVideoState);
  document.addEventListener("visibilitychange", renderVideoState);
  const videoTimer = setInterval(renderVideoState, 1000);
  window.enaeaPlayer = { stop, destroy, refreshStudy: () => refreshStudy() };
  void refreshStudy();
})();
```

## 停止与恢复

“停止”会停止调度和自动学时同步，暂停受控播放页，并恢复 1 倍速；如果脚本使用了静音兜底，也会恢复该视频原来的静音状态。“移除面板”还会清理监视定时器和事件。控制台也可以执行 `enaeaPlayer.stop()`、`enaeaPlayer.destroy()` 或 `enaeaPlayer.refreshStudy()`。

停止后建议重新读取课程再开始。刷新或离开控制页会停止当前播放；恢复时重新运行脚本即可，已完成课程和分集会被跳过。关闭播放页会停止队列。停止后的汇总显示保留上次成功同步值，可以手动刷新查看最终记录。

## 范围与计时方式

队列使用当前班级、当前课程表的全部分页，按平台返回顺序播放，忽略页面上临时选择的分类或完成状态筛选。已完成、无学习权限、外链和其他非普通视频课程不会加入队列。

播放器具备按配置生成倍速选项的能力，但检查过的课程下发配置只有 1 倍速。脚本通过视频元素设置播放速度，不修改平台计时回调，不补算视频跨过的秒数，也不直接提交或重放 `studyLog.do` 请求。学时增量只读自服务端汇总，最终计时以平台实际记录为准。

平台播放器按播放进度事件的整数秒变化次数计数，并非直接累计视频前进的秒数。此前短时实测中，3 倍速约 55 秒增加了 3 分钟“已学”；4 倍速约 35 秒增加了 2 分钟；10 倍速约 30 秒虽然播放了约 300 秒内容，却只增加了 2 分钟。服务端汇总有整数取整和上报批次影响，这些样本不保证其他课程或项目使用相同规则。本脚本当前最高提供 3 倍速。

视频到片尾后至少等待 5 秒再核对服务端记录；未完成则继续相应分集，连续两次重播都没有增加记录时停止并报错。分集切换时只按视频节点本次连续缺失的时长判断加载超时，避免把整门课的运行时间误当成新分集的加载时间。

电脑休眠、Chrome 冻结控制页、登录失效或平台结构变化都可能中断连播。有声自动播放限制参见 [Chrome 自动播放策略](https://developer.chrome.com/blog/autoplay/)。

## 验证情况

真实平台页面曾验证过分页读取、倍速启动、分集切换、休息提醒模拟弹窗和停止恢复。模拟回归覆盖跨课程完成确认、片尾补足、无进展停止、播放失败重试、旧异步请求取消以及分集切换加载超时。

本次新增验证覆盖默认 1 倍与最高 3 倍限制、设定值与实际值分别显示、服务端增量计算、读取失败保留旧数据、跨课程保留基准、重新开始重建基准，以及停止后旧汇总结果不得覆盖面板。

播放状态监视绑定在播放文档上，视频节点替换后仍能捕获事件。视频真正进入播放状态后，会在平台本轮事件处理结束后重新落实所选倍速，避免启动时被平台重置为 1 倍。旧文档、旧视频和停止后的事件都不会恢复播放或覆盖面板。上述同步行为通过了不运行控制页定时器的事件回归测试，也在真实播放器上验证了暂停后立即显示暂停、恢复后立即显示播放中，以及恢复播放后重新落实 3 倍速。
