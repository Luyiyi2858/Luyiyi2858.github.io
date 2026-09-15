# 留言板

欢迎在这里留下你的足迹～不管是建议、吐槽，还是单纯来打个招呼，我都会看的。

!!! tip "使用说明"
    - **不需要注册账号**，填个昵称就能发言，邮箱只有我能看到，不会公开。
    - 请保持友善，广告和垃圾信息会被清理。
    - 如果评论区一直转圈加载不出来，多半是网络波动，刷新一下就好。

<div id="twikoo-container"></div>

<script src="https://cdn.jsdelivr.net/npm/twikoo@1.7.23/dist/twikoo.min.js"></script>

<script>
  function initTwikoo() {
    if (typeof twikoo === 'undefined') return;
    twikoo.init({
      envId: 'https://luyiyi-comment.netlify.app/.netlify/functions/twikoo',
      el: '#twikoo-container',
      lang: 'zh-CN',
      path: window.location.pathname
    });
  }
  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', initTwikoo);
  } else {
    initTwikoo();
  }
</script>
