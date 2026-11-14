# 学习公社云平台自动继续播放

用于学习公社云平台（`study.enaea.edu.cn`）课程播放页的浏览器脚本。当页面弹出“您已学习……分钟了，让眼睛休息一下吧”的休息提醒时，自动点击“继续学习”，恢复播放。下一集由平台自带的自动连播功能切换。

## 使用方法

在 Chrome 中登录学习公社云平台，打开课程并开始播放。按 `⌘ + Option + J`（macOS）或 `Ctrl + Shift + J`（Windows / Linux）打开开发者工具的 Console（控制台），粘贴下面的完整代码并按回车执行。看到“已启用自动继续”的提示后即可关闭开发者工具，保持课程页面打开。

```javascript
(() => {
  // 重复运行时先清理旧任务，避免多个定时器同时点击。
  window.courseAutoContinue?.stop();

  /** 只处理休息提醒，避免误点测验、登录异常等其他弹窗。 */
  function continueAfterRest() {
    const buttons = document.querySelectorAll(
      'button, a, input[type="button"], input[type="submit"]'
    );

    for (const button of buttons) {
      const text = (button.textContent || button.value || "").trim();
      if (text !== "继续学习" || !button.getClientRects().length) continue;

      let container = button.parentElement;
      for (let depth = 0; container && depth < 8; depth++) {
        // 不检查整个页面，避免其他区域的文字导致误判。
        if (container === document.body) break;

        const message = container.textContent || "";
        if (
          message.includes("您已学习") &&
          message.includes("让眼睛休息一下吧")
        ) {
          button.click();
          console.info("[课程播放] 已关闭休息提醒并继续播放");
          return;
        }
        container = container.parentElement;
      }
    }
  }

  const timer = window.setInterval(continueAfterRest, 2000);

  window.courseAutoContinue = {
    /** 停止自动处理提醒，不影响当前播放。 */
    stop() {
      window.clearInterval(timer);
      console.info("[课程播放] 自动继续已停止");
    },
  };

  console.info(
    "[课程播放] 已启用自动继续；停止请运行 courseAutoContinue.stop()"
  );
})();
```

需要停止时，在同一页面的控制台运行：

```javascript
courseAutoContinue.stop();
```

## 使用说明

脚本每两秒检查一次休息提醒。刷新、关闭页面或跳转到新的播放页后，需要重新运行；在同一页面重复运行会自动替换旧任务。

脚本只点击休息提醒中的“继续学习”，不修改播放速度和学时记录，不自动答题，也不处理登录失效、网络异常等提示。学时是否计入以平台实际记录为准。电脑休眠或浏览器冻结标签页时，自动继续可能无法执行。

当前根据学习公社云平台播放页的提醒文案和自动连播逻辑编写，尚未在实际休息提醒弹出时完成验证。平台调整页面结构或文案后，脚本可能需要同步更新。
