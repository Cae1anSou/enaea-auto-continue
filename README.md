# 学习公社云平台倍速与跨课程连播

用于学习公社云平台（`study.enaea.edu.cn`）的课程表页。脚本读取当前课程表的全部分页，在一个独立的播放标签页中按顺序播放未完成的视频课，默认使用 2 倍速，并自动处理“让眼睛休息一下吧”的暂停提醒。一门课的所有分集在服务端达到 100% 后，自动进入下一门课。如果视频已到片尾但累计学时不足，会自动从头重播未完成分集以补足记录。

课程表页是控制页，脚本始终运行在这里，通过同源窗口访问播放页，因此播放页更换课程后仍然有效。不需要安装插件，也不需要在每个播放页重新粘贴代码。

## 怎么用

在 Chrome 中登录平台，进入班级的“课程学习”课程表页，地址应包含 `circleIndexRedirect.do?action=toNewMyClass&type=course`。先关闭或暂停其他正在播放的课程，平台会检测多门课程同时计时。

在课程表页按 `⌘ + Option + J`（macOS）或 `Ctrl + Shift + J`（Windows / Linux）打开 Console（控制台），粘贴下面的完整代码并执行。页面右下角会出现“课程连播”面板。点击“读取课程”，等待列表加载后，选择倍速，再点击“开始连播”。如果 Chrome 拦截新标签页，需要允许当前网站打开弹出式窗口后重试。

保持课程表页和脚本打开的播放页都处于打开状态。可以关闭开发者工具。脚本会确认视频实际开始播放；初次启动失败时每隔至少 5 秒重试，最多 3 轮。如果 Chrome 拦截有声自动播放，会尝试静音启动，面板会提示当前静音，可以在播放页手动开启声音。仍无法启动时，面板会保留错误原因，需要手动播放。正常播放之后的普通暂停由你手动恢复；休息提醒中的“继续学习”会自动点击。运行期间可在面板切换 1、1.5、2、3、4 倍速，新分集和新课程都会使用所选速度。

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
    <label>倍速 <select data-rate><option value="1">1×</option><option value="1.5">1.5×</option><option value="2" selected>2×</option><option value="3">3×</option><option value="4">4×</option></select></label>
    <div style="margin-top:10px;display:flex;gap:8px;flex-wrap:wrap">
      <button data-load>读取课程</button><button data-start disabled>开始连播</button>
      <button data-stop>停止</button><button data-close>移除面板</button>
    </div>`;
  document.body.append(panel);
  const status = panel.querySelector("[data-status]");
  const loadButton = panel.querySelector("[data-load]");
  const startButton = panel.querySelector("[data-start]");
  const rateSelect = panel.querySelector("[data-rate]");
  let queue = [];
  let player = null;
  let run = null;
  let loading = false;
  let destroyed = false;

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
    current.loadedAt = Date.now();
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
    if (run) clearInterval(run.timer);
    run = null;
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
    video.playbackRate = Number(rateSelect.value);
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
      const items = [...doc.querySelectorAll(".cvtb-MCK-course-content")];
      const video = doc.querySelector("video");
      if (!items.length || !video) {
        if (Date.now() - current.loadedAt > 60000) throw new Error("播放器加载超时。");
        return;
      }
      continueAfterRest(doc);
      video.playbackRate = Number(rateSelect.value);
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
        const playbackMessage = await ensureStarted(current, video, selected?.dataset.id);
        if (run !== current) return;
        status.textContent = `${current.index + 1}/${queue.length} · ${queue[current.index].title}\n${rateSelect.value} 倍速 · 剩余 ${unfinished.length} 个未完成分集`;
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
    startButton.disabled = true;
    loadButton.disabled = true;
    try {
      navigate(current);
      current.timer = setInterval(() => void tick(current), 2000);
    } catch (error) {
      stop(`无法打开播放页：${error.message}`);
    }
  }

  /** 清理定时器和事件；重复粘贴时先销毁上一个实例。 */
  function destroy() {
    destroyed = true;
    stop();
    panel.remove();
    window.removeEventListener("pagehide", onPageHide);
    delete window.enaeaPlayer;
  }

  function onPageHide() { stop("控制页已离开，播放已停止。"); }
  loadButton.addEventListener("click", () => void loadCourses());
  startButton.addEventListener("click", start);
  panel.querySelector("[data-stop]").addEventListener("click", () => stop());
  panel.querySelector("[data-close]").addEventListener("click", destroy);
  window.addEventListener("pagehide", onPageHide);
  window.enaeaPlayer = { stop, destroy };
})();
```

## 停止与恢复

点击面板中的“停止”会停止调度、暂停脚本管理的播放页，并将其恢复为 1 倍速；如果启动时使用了静音兜底，还会恢复该视频原来的静音状态。“移除面板”会同时清理脚本。也可以在课程表页的控制台执行 `enaeaPlayer.stop()` 或 `enaeaPlayer.destroy()`。

停止后建议点击“读取课程”更新进度，再开始连播。刷新或离开控制页会停止当前播放；恢复时重新运行脚本并读取课程即可，已完成的课程和分集会被跳过。关闭播放页会停止队列。

## 范围与验证

视频到片尾后，脚本先等待至少 5 秒，再读取服务端分集记录。全部完成就换课；尚未完成则播放未完成分集，当前分集到片尾但学时不足时从头重播。重播后连续两次记录没有增加会停止，并提示检查平台计时或网络。这个过程只通过真实播放累计学时，不修改上报数据。

脚本读取当前班级、当前课程表的全部分页，按平台返回顺序播放，忽略页面上临时选择的分类或完成状态筛选。已完成、无学习权限、外链、第三方播放器等非普通视频课程不会进入队列。课程中的测验需要手动处理。脚本不直接提交或修改学时记录，学时由平台播放器正常上报。

以课程表服务端返回的“已学”总分钟数为准，当前课程短时测试中：3 倍速约 55 秒使已学从 75 增至 78 分钟；4 倍速约 35 秒使已学从 78 增至 80 分钟。汇总只显示整数分钟，存在取整误差，不能保证其他课程或项目使用相同计时规则。页面上的“已学”不会实时刷新，复核时需重新读取服务端汇总。

当前课程的短时实测中，2 倍速约 53 秒播放了 106 秒视频，服务端分集进度从 85% 增至 92%，记录提交间隔约 30 秒；恢复 1 倍速后约 60 秒提交一次。这只能说明当时该课程的分集进度接受了倍速，不能保证所有项目的总学时折算方式一致。休息提醒点击逻辑通过了平台原生弹窗组件的模拟测试，尚未等待真实的 20 分钟提醒完成验证。

新版已在真实页面验证全部分页读取、受控播放标签页打开、已完成分集跳过、2 倍速启动、跨窗口处理模拟休息提醒，以及停止后暂停并恢复 1 倍速。跨课程跳转、服务端未完成时等待、队列结束、弹窗等待和停止后的异步竞态通过了模拟测试；尚未连续播放完一整门课来完成真实的跨课程切换验证。

跨课程控制依赖课程表页保持运行，以及播放页与它处于同源。电脑休眠、Chrome 冻结控制页、登录失效或平台页面结构变化，都可能中断连播。仅切换标签页并不能保证浏览器不会限制后台定时器。

自动播放使用视频元素的 `play()` 并检查返回结果。Chrome 对有声自动播放有限制，因此脚本在遇到 `NotAllowedError` 时使用静音启动兜底，参见 [Chrome 自动播放策略](https://developer.chrome.com/blog/autoplay/)。不会通过模拟点击保证有声自动播放。启动失败重试、静音兜底、错误提示保留和停止后恢复静音状态已通过模拟测试；修复版也已在当前真实课程页面验证无需再次点击即可自动启动。

片尾处理已通过模拟测试，覆盖界面进度过期、记录不足后重播、重播无增量时停止，以及片尾核对期间用户停止后不得继续换课。真实课程的片尾完整流程仍需自然播放到结束后验证。
