#!/usr/bin/env node
/**
 * Yukecil 预警监控 —— 链接/网站打不开，自动发 Telegram 消息
 * 需要 Node.js 18+（自带 fetch），无需安装任何依赖。
 *
 * 必填环境变量：
 *   BOT_TOKEN  在 @BotFather 创建机器人拿到的 token
 *   CHAT_ID    接收预警的 Telegram 账号 ID（数字）
 *   SITE_URL   你的网站地址，例如 https://example.com
 * 可选：
 *   CHECK_INTERVAL_MIN 检查间隔(分钟)，默认 5
 *   FAIL_THRESHOLD     连续失败几次才报警，默认 2（避免误报）
 *   REMIND_MIN         持续故障时的重复提醒间隔(分钟)，默认 60
 *   TIMEOUT_SEC        单个链接超时(秒)，默认 12
 *   EXTRA_URLS         额外要监控的链接，逗号分隔
 *   STATE_FILE         状态文件路径，默认 ./monitor-state.json
 * 用法：
 *   node monitor.js           常驻运行，按间隔循环检查
 *   node monitor.js --once    只检查一次（适合 crontab）
 *   node monitor.js --test    发送一条测试消息
 */
const fs = require('fs');
const env = process.env;
const BOT = env.BOT_TOKEN, CHAT = env.CHAT_ID;
const SITE = (env.SITE_URL || '').replace(/\/+$/, '');
const EVERY = Number(env.CHECK_INTERVAL_MIN || 5) * 60000;
const TIMEOUT = Number(env.TIMEOUT_SEC || 12) * 1000;
const FAILS = Math.max(1, Number(env.FAIL_THRESHOLD || 2));
const REMIND = Number(env.REMIND_MIN || 60) * 60000;
const STATE_FILE = env.STATE_FILE || './monitor-state.json';
const TG_API = env.TG_API || 'https://api.telegram.org';
const UA = 'Mozilla/5.0 (compatible; YukecilMonitor/1.0)';

const esc = s => String(s).replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');
const now = () => new Date().toLocaleString('zh-CN', { timeZone: env.TZ_NAME || 'Asia/Shanghai', hour12: false });

async function tg(text) {
  const r = await fetch(`${TG_API}/bot${BOT}/sendMessage`, {
    method: 'POST', headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ chat_id: CHAT, text, parse_mode: 'HTML', disable_web_page_preview: true })
  });
  if (!r.ok) throw new Error('Telegram 发送失败 HTTP ' + r.status + ' ' + (await r.text().catch(() => '')));
}

// 能收到服务器响应就算“活着”；404/410/5xx、超时、DNS/连接失败才算打不开。
// 401/403/405/429 多半是对方拦截机器人，不算故障。
async function check(url) {
  const ctl = new AbortController();
  const t = setTimeout(() => ctl.abort(), TIMEOUT);
  try {
    const r = await fetch(url, { signal: ctl.signal, redirect: 'follow', headers: { 'user-agent': UA } });
    r.body?.cancel?.().catch(() => {});
    if (r.status === 404 || r.status === 410 || r.status >= 500) return { ok: false, detail: 'HTTP ' + r.status };
    return { ok: true };
  } catch (e) {
    const code = e.name === 'AbortError' ? '超时' : (e.cause?.code || e.message || '连接失败');
    return { ok: false, detail: String(code) };
  } finally { clearTimeout(t); }
}

async function targets() {
  const list = [];
  if (SITE) list.push({ key: 'site', name: '网站首页', url: SITE + '/' }, { key: 'api', name: '网站接口', url: SITE + '/api/site' });
  for (const u of (env.EXTRA_URLS || '').split(',').map(s => s.trim()).filter(Boolean)) list.push({ key: 'x:' + u, name: '额外链接', url: u });
  if (SITE) {
    try {
      const r = await fetch(SITE + '/api/site', { headers: { 'user-agent': UA }, signal: AbortSignal.timeout(TIMEOUT) });
      const d = await r.json();
      for (const g of d.games || []) {
        if (g.enabled !== undefined && !Number(g.enabled)) continue;       // 已下架不检查
        if (!/^https?:\/\//i.test(g.game_url || '')) continue;              // 空链接 / # 跳过
        list.push({ key: 'g:' + g.id, name: g.name || ('游戏#' + g.id), url: g.game_url, game: true });
      }
    } catch { /* 接口本身打不开会在 api 目标里报警 */ }
  }
  return list;
}

const load = () => { try { return JSON.parse(fs.readFileSync(STATE_FILE, 'utf8')); } catch { return {}; } };
const save = s => fs.writeFileSync(STATE_FILE, JSON.stringify(s, null, 2));

async function run() {
  const state = load(), list = await targets(), msgs = [], t0 = Date.now();
  const results = [];
  for (let i = 0; i < list.length; i += 8)                                   // 8 个一组并发
    results.push(...await Promise.all(list.slice(i, i + 8).map(async x => [x, await check(x.url)])));

  for (const [x, res] of results) {
    const s = state[x.key] || { fails: 0, down: false, lastAlert: 0 };
    const label = `${x.game ? '🎮 ' : '🌐 '}<b>${esc(x.name)}</b>\n${esc(x.url)}`;
    if (res.ok) {
      if (s.down) msgs.push(`✅ <b>已恢复</b>\n${label}`);
      s.fails = 0; s.down = false;
    } else {
      s.fails++; s.detail = res.detail;
      if (s.fails >= FAILS && !s.down) { s.down = true; s.lastAlert = t0; msgs.push(`🚨 <b>打不开</b>（${esc(res.detail)}）\n${label}`); }
      else if (s.down && t0 - s.lastAlert >= REMIND) { s.lastAlert = t0; msgs.push(`⚠️ <b>仍未恢复</b>（${esc(res.detail)}）\n${label}`); }
    }
    state[x.key] = s;
  }
  const alive = new Set(list.map(x => x.key));                               // 清理已删除/下架的目标
  for (const k of Object.keys(state)) if (!alive.has(k)) delete state[k];
  save(state);

  const bad = results.filter(([, r]) => !r.ok).length;
  console.log(`[${now()}] 检查 ${list.length} 个，异常 ${bad} 个，通知 ${msgs.length} 条`);
  if (msgs.length) {
    try { await tg(`${msgs.join('\n\n')}\n\n🕐 ${now()}`); }
    catch (e) { console.error(e.message); for (const [k, v] of Object.entries(state)) if (v.down) v.lastAlert = 0; save(state); }
  }
}

(async () => {
  if (!BOT || !CHAT) { console.error('请设置 BOT_TOKEN 和 CHAT_ID'); process.exit(1); }
  if (process.argv.includes('--test')) { await tg('✅ Yukecil 预警测试：Telegram 通知已连通\n🕐 ' + now()); console.log('测试消息已发送'); return; }
  if (!SITE && !env.EXTRA_URLS) { console.error('请设置 SITE_URL（或 EXTRA_URLS）'); process.exit(1); }
  if (process.argv.includes('--once')) return run();
  for (;;) { try { await run(); } catch (e) { console.error(e); } await new Promise(r => setTimeout(r, EVERY)); }
})();
