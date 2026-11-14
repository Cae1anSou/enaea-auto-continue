# 学习公社云平台倍速与跨课程连播

用于学习公社云平台（`study.enaea.edu.cn`）的课程表页。脚本读取当前课程表的全部分页，在一个独立的播放标签页中按顺序播放未完成的视频课，默认使用 2 倍速，并自动处理“让眼睛休息一下吧”的暂停提醒。一门课的所有分集在服务端达到 100% 后，自动进入下一门课。

课程表页是控制页，脚本始终运行在这里，通过同源窗口访问播放页，因此播放页更换课程后仍然有效。不需要安装插件，也不需要在每个播放页重新粘贴代码。

## 怎么用

在 Chrome 中登录平台，进入班级的“课程学习”课程表页，地址应包含 `circleIndexRedirect.do?action=toNewMyClass&type=course`。先关闭或暂停其他正在播放的课程，平台会检测多门课程同时计时。

在课程表页按 `⌘ + Option + J`（macOS）或 `Ctrl + Shift + J`（Windows / Linux）打开 Console（控制台），粘贴下面的完整代码并执行。页面右下角会出现“课程连播”面板。点击“读取课程”，等待列表加载后，选择倍速，再点击“开始连播”。如果 Chrome 拦截新标签页，需要允许当前网站打开弹出式窗口后重试。

保持课程表页和脚本打开的播放页都处于打开状态。可以关闭开发者工具。播放页若提示浏览器限制自动播放，手动点击一次播放即可；脚本会为每个新视频尝试启动一次，之后的普通暂停由你手动恢复；休息提醒中的“继续学习”会自动点击。运行期间可在面板切换 1、1.5、2 倍速，新分集和新课程都会使用所选速度。

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
    <label>倍速 <select data-rate><option value="1">1×</option><option value="1.5">1.5×</option><option value="2" selected>2×</option></select></label>
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
    current.completeSince = 0;
    current.endedSince = 0;
    current.attemptedSources = new Set();
    current.switchingUntil = 0;
    status.textContent = `${current.index + 1}/${queue.length} · ${queue[current.index].title}\n正在打开课程…`;
    const url = new URL("/viewerforccvideo.do", location.origin);
    url.search = new URLSearchParams({ courseId: queue[current.index].id, circleId });
    player.location.href = url.href;
  }

  /** 仅停止本控制页管理的窗口，恢复正常速度，避免留下后台播放。 */
  function stop(message = "已停止，可重新读取课程后继续。") {
    if (run) clearInterval(run.timer);
    run = null;
    try {
      if (player && !player.closed) {
        for (const video of player.document.querySelectorAll("video")) {
          video.pause();
          video.playbackRate = 1;
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
      if (unfinished.length) {
        current.completeSince = 0;
        // 平台常从第一集打开；已完成分集直接切到尚未完成的一集。
        if ((!selected || !unfinished.includes(selected)) && Date.now() > current.switchingUntil) {
          current.switchingUntil = Date.now() + 8000;
          unfinished[0].click();
          return;
        }
        // 每个视频源仅尝试启动一次；弹窗已在上方拦截，不反复覆盖普通暂停。
        const source = `${selected?.dataset.id}:${video.currentSrc}`;
        if (video.readyState >= 2 && video.currentSrc && !current.attemptedSources.has(source)) {
          current.attemptedSources.add(source);
          if (video.paused && !video.ended) {
            void video.play().catch(() => {
              if (run === current) status.textContent = "浏览器限制自动播放，请在播放页手动点击一次播放。";
            });
          }
        }
        status.textContent = `${current.index + 1}/${queue.length} · ${queue[current.index].title}\n${rateSelect.value} 倍速 · 剩余 ${unfinished.length} 个未完成分集`;
        if (video.paused && !video.ended) status.textContent += "\n已暂停，请在播放页手动恢复。";
        if (video.ended && selected === items.at(-1)) {
          current.endedSince ||= Date.now();
          if (Date.now() - current.endedSince > 90000) throw new Error("视频已结束但进度未满，请检查学时记录。");
        } else current.endedSince = 0;
        return;
      }
      // 服务端确认所有分集完成后才导航，避免丢失最后一次记录。
      current.completeSince ||= Date.now();
      if (Date.now() - current.lastCheck < 10000) return;
      current.lastCheck = Date.now();
      const data = await readJSON("/course.do", {
        action: "getCourseContentList", courseId: queue[current.index].id, circleId,
      });
      if (run !== current) return;
      const ids = items.map(item => item.dataset.id);
      const complete = data.result.list.length === ids.length && data.result.list.every(item =>
        ids.includes(String(item.id)) && Number(item.studyProgress) >= 100
      );
      if (!complete) {
        if (Date.now() - current.completeSince > 90000) throw new Error("服务端尚未确认课程完成。");
        status.textContent = "正在等待服务端确认本课程完成…";
        return;
      }
      current.index++;
      if (current.index >= queue.length) return stop("队列中的视频课程已全部完成。");
      navigate(current);
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

点击面板中的“停止”会停止调度、暂停脚本管理的播放页，并将其恢复为 1 倍速。“移除面板”会同时清理脚本。也可以在课程表页的控制台执行 `enaeaPlayer.stop()` 或 `enaeaPlayer.destroy()`。

停止后建议点击“读取课程”更新进度，再开始连播。刷新或离开控制页会停止当前播放；恢复时重新运行脚本并读取课程即可，已完成的课程和分集会被跳过。关闭播放页会停止队列。

## 范围与验证

脚本读取当前班级、当前课程表的全部分页，按平台返回顺序播放，忽略页面上临时选择的分类或完成状态筛选。已完成、无学习权限、外链、第三方播放器等非普通视频课程不会进入队列。课程中的测验需要手动处理。脚本不直接提交或修改学时记录，学时由平台播放器正常上报。

当前课程的短时实测中，2 倍速约 53 秒播放了 106 秒视频，服务端分集进度从 85% 增至 92%，记录提交间隔约 30 秒；恢复 1 倍速后约 60 秒提交一次。这只能说明当时该课程的分集进度接受了倍速，不能保证所有项目的总学时折算方式一致。休息提醒点击逻辑通过了平台原生弹窗组件的模拟测试，尚未等待真实的 20 分钟提醒完成验证。

新版已在真实页面验证全部分页读取、受控播放标签页打开、已完成分集跳过、2 倍速启动、跨窗口处理模拟休息提醒，以及停止后暂停并恢复 1 倍速。跨课程跳转、服务端未完成时等待、队列结束、弹窗等待和停止后的异步竞态通过了模拟测试；尚未连续播放完一整门课来完成真实的跨课程切换验证。

跨课程控制依赖课程表页保持运行，以及播放页与它处于同源。电脑休眠、Chrome 冻结控制页、登录失效或平台页面结构变化，都可能中断连播。仅切换标签页并不能保证浏览器不会限制后台定时器。
