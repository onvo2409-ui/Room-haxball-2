/* =====================================================================
VN HAXBALL ROOM  —  1v1 • 2v2 • 5v5 • Practice  —  Full hệ thống Admin (Đã tối ưu)

CÁCH CHẠY:
Mở https://www.haxball.com/headless
Nhấn F12 → tab Console → dán toàn bộ file này → Enter
===================================================================== */

"use strict";
/* ═══════════════════════ 1. CẤU HÌNH ═══════════════════════ */

const CONFIG = {
ROOM_NAME: "🏟️ VN HaxBall Pro",
MAX_PLAYERS: 20,
PUBLIC: true,
ROOM_PASSWORD: null,
TOKEN: "",
GEO: { code: "VN", lat: 21.0285, lon: 105.8542 },
PASSWORDS: { SUPER: "tuatdz", ADMIN: "chan" },   // để trống = tự sinh ngẫu nhiên

// ── Đăng nhập / chống dò mật khẩu
LOGIN_MAX_FAILS: 3,            // sai quá số lần → khoá đăng nhập
LOGIN_LOCK_MS: 10 * 60e3,
LOGIN_BAN_FAILS: 6,            // sai quá số lần → ban tạm
LOGIN_BAN_MIN: 30,

// ── Bảo vệ quyền admin
ADMIN_ACTIONS_PER_MIN: 5,      // admin thường chỉ được kick/ban/mute tối đa X lần / phút
ADMIN_MAX_BAN_MIN: 120,
ADMIN_MAX_MUTE_MIN: 60,
ADMIN_MAX_BUFF_SEC: 300,
SUPER_MAX_BUFF_SEC: 3600,
MAX_PER_IP: 1,                 // tối đa kết nối / IP
JOIN_FLOOD_COUNT: 8,           // >= 8 lượt vào trong 10s = flood
JOIN_FLOOD_WINDOW_MS: 10e3,

// ── Chat
MAX_MSG_LEN: 120,
CHAT_WINDOW_MS: 5e3,
CHAT_MAX_IN_WINDOW: 4,
CHAT_MIN_GAP_MS: 500,
DUP_WINDOW_MS: 20e3,
STRIKES_TO_MUTE: 3,
AUTO_MUTE_MS: 3 * 60e3,
AUTO_MUTES_TO_KICK: 3,
CMD_COOLDOWN_MS: 400,

// ── AFK / bóng đứng yên
AFK_WARN_MS: 12e3,
AFK_MOVE_MS: 22e3,
AFK_FLAG_KICK_MS: 30 * 60e3,   // AFK quá lâu trong phòng → kick (0 = tắt)
PAUSE_MAX_MS: 90e3,
HOLD_MAX_MS: 5 * 60e3,

// ── Luân chuyển đội
MAX_WIN_STREAK: 3,             // thắng liên tiếp X trận → nhường chỗ (0 = không giới hạn)
NEXT_MATCH_DELAY_MS: 3e3,
MIN_RATED_SECONDS: 20,         // trận ngắn hơn không tính Elo

// ── Chọn người (đội trưởng)
PICK_TURN_MS: 10e3,            // thời gian mỗi lượt chọn; hết giờ tự chọn người đầu hàng chờ
PICK_CONFIRM_MS: 1500,         // đồng hồ canh khi room không xác nhận việc đổi đội
PEN_OT_SEC: 30,
PEN_KICK_TIMEOUT_MS: 10e3,
PEN_RESOLVE_MS: 5e3,
PEN_BETWEEN_MS: 2500,
PEN_MAX_ROUNDS:12,                                                                                                        };

/* ═══════════════════════ 2. TIỆN ÍCH ═══════════════════════ */

const Now = () => Date.now();
const clamp = (v, a, b) => Math.min(b, Math.max(a, v));
const norm = s => String(s).normalize("NFD").replace(/[\u0300-\u036f]/g, "")
.replace(/đ/g, "d").replace(/Đ/g, "D").toLowerCase();

const fmtDur = ms => {
const s = Math.max(0, Math.round(ms / 1000));
if (s < 60) return s + "s";
const m = Math.floor(s / 60);
if (m < 60) return m + "p" + (s % 60 ? (s % 60) + "s" : "");
return Math.floor(m / 60) + "h" + (m % 60) + "p";
};

const genPass = (n = 10) => {
const chars = "ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnpqrstuvwxyz23456789";
const arr = new Uint32Array(n);
try { (globalThis.crypto || window.crypto).getRandomValues(arr); }
catch (e) { for (let i = 0; i < n; i++) arr[i] = Math.floor(Math.random() * 4e9); }
return Array.from(arr, x => chars[x % chars.length]).join("");
};

const C = {
super: 0xFFD700, admin: 0x00E5FF, info: 0x9EE7FF, ok: 0x7CFC98,
warn: 0xFFB347, err: 0xFF6B6B, sys: 0xC9D1FF, goal: 0xFFFFFF, team: 0xB8FFB0,
};

const ROLE = { USER: 0, ADMIN: 1, SUPER: 2 };
const ROLE_NAME = ["Thành viên", "Admin", "Super Admin"];
const BADGE = ["", "[ADMIN] ", "[SUPER] "];

/* ═══════════════ 3. BẢN ĐỒ: KÍCH THƯỚC & BỘ DỰNG ═══════════════ */

const MODE_INFO = {
  "1v1": { size: 1, min: 1, dim: "1v1", barrier: true,  score: 3, time: 3, label: "1v1" },
  "2v2": { size: 2, min: 2, dim: "2v2", barrier: true,  score: 3, time: 3, label: "2v2" },
  "3v3": { size: 3, min: 3, dim: "3v3", barrier: true,  score: 3, time: 4, label: "3v3" },
  "5v5": { size: 5, min: 4, dim: "5v5", barrier: true,  score: 4, time: 4, label: "5v5" },
  "practice": { size: 0, min: 0, dim: "2v2", barrier: false, score: 0, time: 0, label: "Luyện tập" },
};


const CUSTOM_MAPS = {
"1v1": `{"name":"Futsal 1v1 by Luchooo from HaxMaps","width":420,"height":200,"spawnDistance":180,"bg":{"type":"hockey","width":368,"height":171,"kickOffRadius":50,"cornerRadius":0},"vertexes":[{"x":-368,"y":171,"bCoef":1,"cMask":["ball"],"trait":"ballArea"},{"x":-368,"y":50,"bCoef":1,"cMask":["ball"],"trait":"ballArea"},{"x":-368,"y":-50,"bCoef":1,"cMask":["ball"],"trait":"ballArea"},{"x":-368,"y":-171,"bCoef":1,"cMask":["ball"],"trait":"ballArea"},{"x":368,"y":171,"bCoef":1,"cMask":["ball"],"trait":"ballArea"},{"x":368,"y":50,"bCoef":1,"cMask":["ball"],"trait":"ballArea"},{"x":368,"y":-50,"bCoef":1,"cMask":["ball"],"trait":"ballArea"},{"x":368,"y":-171,"bCoef":1,"cMask":["ball"],"trait":"ballArea"},{"x":0,"y":176,"trait":"kickOffBarrier"},{"x":0,"y":50,"trait":"kickOffBarrier"},{"x":0,"y":-50,"trait":"line"},{"x":0,"y":-176,"trait":"kickOffBarrier"},{"x":-384,"y":-50,"bCoef":0.1,"cMask":["all"],"trait":"goalNet"},{"x":384,"y":-50,"bCoef":0.1,"cMask":["all"],"trait":"goalNet"},{"x":-384,"y":50,"bCoef":0.1,"cMask":["all"],"trait":"goalNet"},{"x":384,"y":50,"bCoef":0.1,"cMask":["all"],"trait":"goalNet"},{"x":-368,"y":-127,"trait":"line"},{"x":368,"y":-127,"trait":"line"},{"x":-368,"y":127,"trait":"line"},{"x":368,"y":127,"trait":"line"},{"x":-350,"y":-171,"bCoef":0,"trait":"line"},{"x":-368,"y":-163,"bCoef":0,"trait":"line"},{"x":350,"y":-171,"bCoef":0,"trait":"line"},{"x":368,"y":-163,"bCoef":0,"trait":"line"},{"x":-350,"y":171,"bCoef":0,"trait":"line"},{"x":-368,"y":163,"bCoef":0,"trait":"line"},{"x":350,"y":171,"bCoef":0,"trait":"line"},{"x":368,"y":163,"bCoef":0,"trait":"line"},{"x":368,"y":171,"bCoef":1,"trait":"ballArea"},{"x":368,"y":-171,"bCoef":1,"trait":"ballArea"},{"x":0,"y":171,"bCoef":0,"trait":"line"},{"x":0,"y":-171,"bCoef":0,"trait":"line"},{"x":0,"y":50,"trait":"kickOffBarrier"},{"x":0,"y":-50,"trait":"kickOffBarrier"},{"x":377,"y":-50,"bCoef":1,"cMask":["red"],"trait":"line"},{"x":377,"y":-171,"bCoef":1,"cMask":["ball"],"trait":"ballArea"},{"x":-377,"y":-50,"bCoef":1,"cMask":["blue"],"trait":"line"},{"x":-377,"y":-171,"bCoef":1,"cMask":["ball"],"trait":"ballArea"},{"x":-377,"y":50,"bCoef":1,"cMask":["blue"],"trait":"line"},{"x":-377,"y":171,"bCoef":1,"cMask":["ball"],"trait":"ballArea"},{"x":377,"y":50,"bCoef":1,"cMask":["red"],"trait":"line"},{"x":377,"y":171,"bCoef":1,"cMask":["ball"],"trait":"ballArea"}],"segments":[{"v0":0,"v1":1,"trait":"ballArea"},{"v0":2,"v1":3,"trait":"ballArea"},{"v0":4,"v1":5,"trait":"ballArea"},{"v0":6,"v1":7,"trait":"ballArea"},{"v0":8,"v1":9,"trait":"kickOffBarrier"},{"v0":9,"v1":10,"curve":180,"cGroup":["blueKO"],"trait":"kickOffBarrier"},{"v0":9,"v1":10,"curve":-180,"cGroup":["redKO"],"trait":"kickOffBarrier"},{"v0":10,"v1":11,"trait":"kickOffBarrier"},{"v0":2,"v1":12,"curve":-35,"vis":true,"color":"FFFFFF","bCoef":0.1,"cMask":["all"],"trait":"goalNet"},{"v0":6,"v1":13,"curve":35,"vis":true,"color":"FFFFFF","bCoef":0.1,"cMask":["all"],"trait":"goalNet"},{"v0":1,"v1":14,"curve":35,"vis":true,"color":"FFFFFF","bCoef":0.1,"cMask":["all"],"trait":"goalNet"},{"v0":5,"v1":15,"curve":-35,"vis":true,"color":"FFFFFF","bCoef":0.1,"cMask":["all"],"trait":"goalNet"},{"v0":12,"v1":14,"curve":-35,"vis":true,"color":"FFFFFF","bCoef":0.1,"cMask":["all"],"trait":"goalNet","x":-585},{"v0":13,"v1":15,"curve":35,"vis":true,"color":"FFFFFF","bCoef":0.1,"cMask":["all"],"trait":"goalNet","x":585},{"v0":1,"v1":0,"vis":true,"color":"FFFFFF","bCoef":1,"cMask":["ball"],"trait":"ballArea","x":-368},{"v0":5,"v1":4,"vis":true,"color":"FFFFFF","bCoef":1,"cMask":["ball"],"trait":"ballArea","x":368},{"v0":2,"v1":3,"vis":true,"color":"FFFFFF","bCoef":1,"cMask":["ball"],"trait":"ballArea","x":-368},{"v0":6,"v1":7,"vis":true,"color":"FFFFFF","bCoef":1,"cMask":["ball"],"trait":"ballArea","x":368},{"v0":0,"v1":28,"vis":true,"color":"FFFFFF","bCoef":1,"trait":"ballArea","y":171},{"v0":3,"v1":29,"vis":true,"color":"FFFFFF","bCoef":1,"trait":"ballArea","y":-171},{"v0":30,"v1":31,"curve":0,"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line"},{"v0":10,"v1":9,"curve":-180,"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line"},{"v0":33,"v1":32,"curve":180,"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line"},{"v0":2,"v1":1,"curve":0,"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line"},{"v0":6,"v1":5,"curve":0,"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line"},{"v0":34,"v1":35,"vis":false,"color":"FFFFFF","bCoef":1,"cMask":["ball"],"trait":"ballArea","x":330},{"v0":36,"v1":37,"vis":false,"color":"FFFFFF","bCoef":1,"cMask":["ball"],"trait":"ballArea","x":-330},{"v0":38,"v1":39,"vis":false,"color":"FFFFFF","bCoef":1,"cMask":["ball"],"trait":"ballArea","x":-330},{"v0":40,"v1":41,"vis":false,"color":"FFFFFF","bCoef":1,"cMask":["ball"],"trait":"ballArea","x":330},{"v0":34,"v1":40,"curve":60,"vis":false,"color":"FFFFFF","bCoef":1,"cMask":["red"],"trait":"line"},{"v0":38,"v1":36,"curve":60,"vis":false,"color":"FFFFFF","bCoef":1,"cMask":["blue"],"trait":"line"}],"goals":[{"p0":[-372,-52],"p1":[-372,48],"team":"red"},{"p0":[372,50],"p1":[372,-50],"team":"blue"}],"discs":[{"radius":5,"pos":[-368,50],"color":"FFFFFF","trait":"goalPost"},{"radius":5,"pos":[-368,-50],"color":"FFFFFF","trait":"goalPost"},{"radius":5,"pos":[368,50],"color":"FFFFFF","trait":"goalPost"},{"radius":5,"pos":[368,-50],"color":"FFFFFF","trait":"goalPost"},{"radius":3,"invMass":0,"pos":[383,51],"color":"FFFFFF","bCoef":0,"trait":"line"},{"radius":3,"invMass":0,"pos":[-383,51],"color":"FFFFFF","bCoef":0,"trait":"line"},{"radius":3,"invMass":0,"pos":[383,-51],"color":"FFFFFF","bCoef":0,"trait":"line"},{"radius":3,"invMass":0,"pos":[-383,-51],"color":"FFFFFF","bCoef":0,"trait":"line"}],"planes":[{"normal":[0,1],"dist":-171,"trait":"ballArea"},{"normal":[0,-1],"dist":-171,"trait":"ballArea"},{"normal":[0,1],"dist":-200,"bCoef":0.2,"cMask":["all"]},{"normal":[0,-1],"dist":-200,"bCoef":0.2,"cMask":["all"]},{"normal":[1,0],"dist":-420,"bCoef":0.2,"cMask":["all"]},{"normal":[-1,0],"dist":-420,"bCoef":0.2,"cMask":["all"]}],"traits":{"ballArea":{"vis":false,"bCoef":1,"cMask":["ball"]},"goalPost":{"radius":8,"invMass":0,"bCoef":1},"goalNet":{"vis":true,"bCoef":0.1,"cMask":["all"]},"kickOffBarrier":{"vis":false,"bCoef":0.1,"cGroup":["redKO","blueKO"],"cMask":["red","blue"]},"line":{"vis":true,"bCoef":0,"cMask":[""]},"arco":{"radius":2,"cMask":["n\/d"],"color":"cccccc"}},"playerPhysics":{"acceleration":0.11,"kickingAcceleration":0.1,"kickStrength":6.5},"ballPhysics":{"radius":6.4,"color":"EAFF00"}}`,

"2v2": `{"name":"Luxora training futsal [ʜᴀxᴍᴏᴅs.ᴄᴏᴍ]","width":465,"height":230,"bg":{"width":400,"height":200,"kickOffRadius":75,"cornerRadius":50,"color":"484848"},"vertexes":[{"x":-401,"y":-200,"cMask":["ball"],"cGroup":["ball"]},{"x":401,"y":-200,"cMask":["ball"],"cGroup":["ball"]},{"x":-401,"y":200,"cMask":["ball"],"cGroup":["ball"]},{"x":401,"y":200,"cMask":["ball"],"cGroup":["ball"]},{"x":-400,"y":70,"cMask":["ball"]},{"x":-400,"y":201,"cMask":["ball"],"cGroup":["ball"]},{"x":400,"y":70,"cMask":["ball"]},{"x":400,"y":201,"cMask":["ball"],"cGroup":["ball"]},{"x":-400,"y":-201,"cMask":["ball"],"cGroup":["ball"]},{"x":-400,"y":-70,"cMask":["ball"]},{"x":400,"y":-201,"cMask":["ball"],"cGroup":["ball"]},{"x":400,"y":-70,"cMask":["ball"]},{"x":-436,"y":-70,"bCoef":0.2,"cMask":["ball"],"cGroup":["ball"]},{"x":-400,"y":-70,"bCoef":0.2,"cMask":["ball"],"cGroup":["ball"]},{"x":-436,"y":70,"bCoef":0.2,"cMask":["ball"],"cGroup":["ball"]},{"x":-400,"y":70,"bCoef":0.2,"cMask":["ball"],"cGroup":["ball"]},{"x":-435,"y":-71,"bCoef":0.2,"cMask":["ball"],"cGroup":["ball"]},{"x":-435,"y":71,"bCoef":0.2,"cMask":["ball"],"cGroup":["ball"]},{"x":435,"y":-71,"bCoef":0.2,"cMask":["ball"],"cGroup":["ball"]},{"x":435,"y":71,"bCoef":0.2,"cMask":["ball"],"cGroup":["ball"]},{"x":436,"y":-70,"bCoef":0.2,"cMask":["ball"],"cGroup":["ball"]},{"x":400,"y":-70,"bCoef":0.2,"cMask":["ball"],"cGroup":["ball"]},{"x":436,"y":70,"bCoef":0.2,"cMask":["ball"],"cGroup":["ball"]},{"x":400,"y":70,"bCoef":0.2,"cMask":["ball"],"cGroup":["ball"]},{"x":-400,"y":-70,"cMask":["wall"]},{"x":-400,"y":70,"cMask":["wall"]},{"x":400,"y":-70,"cMask":["wall"]},{"x":400,"y":70,"cMask":["wall"]},{"x":0,"y":-198,"cMask":["red","blue"],"cGroup":["redKO","blueKO"]},{"x":0,"y":-80,"cMask":["red","blue"],"cGroup":["redKO","blueKO"]},{"x":0,"y":80,"cMask":["red","blue"],"cGroup":["redKO","blueKO"]},{"x":0,"y":198,"cMask":["red","blue"],"cGroup":["redKO","blueKO"]},{"x":0,"y":80,"cMask":["red","blue"],"cGroup":["blueKO"]},{"x":0,"y":-80,"cMask":["red","blue"],"cGroup":["blueKO"]},{"x":0,"y":-80,"cMask":["red","blue"],"cGroup":["redKO"]},{"x":0,"y":80,"cMask":["red","blue"],"cGroup":["redKO"]},{"x":0,"y":-81,"cMask":["wall"]},{"x":0,"y":81,"cMask":["wall"]},{"x":-402,"y":-70,"cMask":["wall"]},{"x":-402,"y":70,"cMask":["wall"]},{"x":-398,"y":-70,"cMask":["c0"],"cGroup":["c0"]},{"x":-398,"y":70,"cMask":["c0"],"cGroup":["c0"]},{"x":398,"y":-70,"cMask":["wall"]},{"x":398,"y":70,"cMask":["wall"]},{"x":402,"y":-70,"cMask":["wall"]},{"x":402,"y":70,"cMask":["wall"]},{"x":0,"y":230,"cMask":["red","blue"],"cGroup":["redKO","blueKO"]},{"x":0,"y":-230,"cMask":["red","blue"],"cGroup":["redKO","blueKO"]},{"x":-398,"y":199,"bCoef":0,"cMask":["c0"],"cGroup":["c0"]},{"x":-398,"y":155,"bCoef":0.1,"cMask":["c0"],"cGroup":["c0"]},{"x":-270,"y":60,"bCoef":0.1,"cMask":["c0"],"cGroup":["c0"]},{"x":-398,"y":-155,"bCoef":0.1,"cMask":["c0"],"cGroup":["c0"]},{"x":-270,"y":-60,"bCoef":0.1,"cMask":["c0"],"cGroup":["c0"]},{"x":398,"y":-155,"bCoef":0.1,"cMask":["c0"],"cGroup":["c0"]},{"x":270,"y":60,"bCoef":0.1,"cMask":["c0"],"cGroup":["c0"]},{"x":270,"y":-60,"bCoef":0.1,"cMask":["c0"],"cGroup":["c0"]},{"x":398,"y":155,"bCoef":0.1,"cMask":["c0"],"cGroup":["c0"]},{"x":-399,"y":198,"bCoef":0,"cMask":["c0"],"cGroup":["c0"]},{"x":399,"y":198,"bCoef":0,"cMask":["c0"],"cGroup":["c0"]},{"x":-399,"y":-198,"bCoef":0,"cMask":["c0"],"cGroup":["c0"]},{"x":399,"y":-198,"bCoef":0,"cMask":["c0"],"cGroup":["c0"]},{"x":398,"y":199,"cMask":["c0"],"cGroup":["c0"]},{"x":-398,"y":-199,"cMask":["c0"],"cGroup":["c0"]},{"x":398,"y":-199,"cMask":["c0"],"cGroup":["c0"]},{"x":0,"y":0,"cMask":["wall"]},{"x":0,"y":80,"bCoef":0,"cMask":["red","blue"],"cGroup":["redKO","blueKO"]},{"x":0,"y":-80,"bCoef":0.15,"cMask":["red","blue"],"cGroup":["redKO","blueKO"]},{"x":0,"y":-81.5,"bCoef":0.1,"cMask":["red","blue"],"cGroup":["redKO"]},{"x":0,"y":81.5,"bCoef":0.1,"cMask":["red","blue"],"cGroup":["redKO"]},{"x":0,"y":1,"bCoef":0.1,"cMask":[]},{"x":0,"y":-1,"bCoef":0.1,"cMask":[]},{"x":0,"y":3,"bCoef":0,"cMask":[]},{"x":0,"y":-3,"bCoef":0.1,"cMask":[]},{"x":0,"y":-2,"bCoef":0.1,"cMask":[]},{"x":0,"y":2,"bCoef":0,"cMask":[]},{"x":0,"y":-3.5,"bCoef":0.1,"cMask":[]},{"x":0,"y":3.5,"bCoef":0.1,"cMask":[]},{"x":0,"y":0,"cMask":["wall"]}],"segments":[{"v0":0,"v1":1,"bias":-10,"cMask":["ball"],"color":"F8F8F8"},{"v0":2,"v1":3,"bias":10,"cMask":["ball"],"color":"F8F8F8"},{"v0":4,"v1":5,"bias":10,"cMask":["ball"],"color":"F8F8F8"},{"v0":6,"v1":7,"bias":-10,"cMask":["ball"],"color":"F8F8F8"},{"v0":8,"v1":9,"bias":10,"cMask":["ball"],"color":"F8F8F8"},{"v0":10,"v1":11,"bias":-10,"cMask":["ball"],"color":"F8F8F8"},{"v0":12,"v1":13,"bias":-10,"bCoef":0.2,"cMask":["ball"],"cGroup":["ball"],"color":"FF6666"},{"v0":14,"v1":15,"bias":10,"bCoef":0.2,"cMask":["ball"],"cGroup":["ball"],"color":"FF6666"},{"v0":16,"v1":17,"bias":10,"bCoef":0.2,"cMask":["ball"],"cGroup":["ball"],"color":"FF6666"},{"v0":18,"v1":19,"bias":-10,"bCoef":0.2,"cMask":["ball"],"cGroup":["ball"],"color":"6666FF"},{"v0":20,"v1":21,"bias":10,"bCoef":0.2,"cMask":["ball"],"cGroup":["ball"],"color":"6666FF"},{"v0":22,"v1":23,"bias":-10,"bCoef":0.2,"cMask":["ball"],"cGroup":["ball"],"color":"6666FF"},{"v0":24,"v1":25,"cMask":["wall"],"color":"FF6666"},{"v0":26,"v1":27,"cMask":["wall"],"color":"6666FF"},{"v0":28,"v1":29,"cMask":["red","blue"],"cGroup":["redKO","blueKO"],"color":"626262"},{"v0":30,"v1":31,"cMask":["red","blue"],"cGroup":["redKO","blueKO"],"color":"626262"},{"v0":32,"v1":33,"curve":180,"curveF":6.123233995736766e-17,"vis":false,"cMask":["red","blue"],"cGroup":["blueKO"],"color":"626262"},{"v0":34,"v1":35,"curve":180,"curveF":6.123233995736766e-17,"vis":false,"cMask":["red","blue"],"cGroup":["redKO"],"color":"626262"},{"v0":38,"v1":39,"cMask":["wall"],"color":"484848"},{"v0":40,"v1":41,"cMask":["wall"],"color":"484848"},{"v0":42,"v1":43,"cMask":["wall"],"color":"484848"},{"v0":44,"v1":45,"cMask":["wall"],"color":"484848"},{"v0":31,"v1":46,"vis":false,"cMask":["red","blue"],"cGroup":["redKO","blueKO"],"color":"F8F8F8"},{"v0":47,"v1":28,"vis":false,"cMask":["red","blue"],"cGroup":["redKO","blueKO"],"color":"F8F8F8"},{"v0":50,"v1":49,"bCoef":0.1,"curve":89.99999999999999,"curveF":1.0000000000000002,"cMask":["c0"],"cGroup":["c0"],"color":"626262"},{"v0":51,"v1":52,"bCoef":0.1,"curve":89.99999999999999,"curveF":1.0000000000000002,"cMask":["c0"],"cGroup":["c0"],"color":"626262"},{"v0":50,"v1":52,"bCoef":0.1,"cMask":["c0"],"cGroup":["c0"],"color":"626262"},{"v0":55,"v1":53,"bCoef":0.1,"curve":89.99999999999999,"curveF":1.0000000000000002,"cMask":["c0"],"cGroup":["c0"],"color":"626262"},{"v0":56,"v1":54,"bCoef":0.1,"curve":89.99999999999999,"curveF":1.0000000000000002,"cMask":["c0"],"cGroup":["c0"],"color":"626262"},{"v0":55,"v1":54,"bCoef":0.1,"cMask":["c0"],"cGroup":["c0"],"color":"626262"},{"v0":59,"v1":60,"cMask":["c0"],"cGroup":["c0"],"color":"626262"},{"v0":57,"v1":58,"cMask":["c0"],"cGroup":["c0"],"color":"626262"},{"v0":48,"v1":41,"cMask":["c0"],"cGroup":["c0"],"color":"626262"},{"v0":61,"v1":43,"cMask":["c0"],"cGroup":["c0"],"color":"626262"},{"v0":62,"v1":40,"cMask":["c0"],"cGroup":["c0"],"color":"626262"},{"v0":63,"v1":42,"cMask":["c0"],"cGroup":["c0"],"color":"626262"},{"v0":67,"v1":68,"bCoef":0.1,"curve":180,"curveF":6.123233995736766e-17,"vis":false,"cMask":["red","blue"],"cGroup":["redKO"]},{"v0":68,"v1":67,"bCoef":0.1,"curve":180,"curveF":6.123233995736766e-17,"vis":false,"cMask":["red","blue"],"cGroup":["blueKO"]}],"planes":[{"normal":[0,1],"dist":-200,"cMask":["ball"]},{"normal":[0,-1],"dist":-200,"cMask":["ball"]},{"normal":[-1,0],"dist":-465,"bCoef":0,"cMask":["red","blue"]},{"normal":[1,0],"dist":-465,"bCoef":0,"cMask":["red","blue"]},{"normal":[0,1],"dist":-230,"bCoef":0,"cMask":["red","blue"]},{"normal":[0,-1],"dist":-230,"bCoef":0,"cMask":["red","blue"]}],"goals":[{"p0":[-405.8,-69],"p1":[-405.8,71],"team":"red"},{"p0":[405.8,70],"p1":[405.8,-70],"team":"blue"}],"discs":[{"radius":5.8,"bCoef":0.474,"invMass":1.5,"color":"FFC338","cGroup":["ball","kick","score"]},{"pos":[-400,-70],"radius":5,"bCoef":1,"invMass":0,"color":"FF6666"},{"pos":[-400,70],"radius":5,"bCoef":1,"invMass":0,"color":"FF6666"},{"pos":[400,-70],"radius":5,"bCoef":1,"invMass":0,"color":"6666FF"},{"pos":[400,70],"radius":5,"bCoef":1,"invMass":0,"color":"6666FF"}],"playerPhysics":{"bCoef":0,"acceleration":0.11,"kickingAcceleration":0.083,"kickStrength":4.545,"cGroup":["red","blue"]},"ballPhysics":"disc0","spawnDistance":201,"redSpawnPoints":[[-201,0],[-201,-56],[-201,56],[-201,111],[-201,-111],[-410,0]],"blueSpawnPoints":[[201,0],[201,-56],[201,56],[201,111],[201,-111],[410,0]],"canBeStored":true}`,

"3v3": `{"name":"Futsal 3v3 Zimska Liga from HaxMaps","width":755,"height":339,"spawnDistance":310,"bg":{"type":"hockey","width":665,"height":290,"kickOffRadius":80,"cornerRadius":0},"vertexes":[{"x":-665,"y":290,"trait":"ballArea","cMask":["ball"],"bCoef":1},{"x":-665,"y":80,"trait":"ballArea","cMask":["ball"],"bCoef":1},{"x":-665,"y":-80,"trait":"ballArea","cMask":["ball"],"bCoef":1},{"x":-665,"y":-290,"trait":"ballArea","bCoef":1,"cMask":["ball"]},{"x":665,"y":290,"trait":"ballArea","cMask":["ball"],"bCoef":1},{"x":665,"y":80,"trait":"ballArea","cMask":["ball"],"bCoef":1},{"x":665,"y":-80,"trait":"ballArea","cMask":["ball"],"bCoef":1},{"x":665,"y":-290,"trait":"ballArea","cMask":["ball"],"bCoef":1},{"x":0,"y":306,"trait":"kickOffBarrier"},{"x":0,"y":80,"trait":"kickOffBarrier"},{"x":0,"y":-80,"trait":"line"},{"x":0,"y":-306,"trait":"kickOffBarrier"},{"bCoef":0.1,"cMask":["ball"],"trait":"goalNet","x":-693,"y":-80},{"bCoef":0.1,"cMask":["ball"],"trait":"goalNet","x":693,"y":-80},{"bCoef":0.1,"cMask":["ball"],"trait":"goalNet","x":-693,"y":80},{"bCoef":0.1,"cMask":["ball"],"trait":"goalNet","x":693,"y":80},{"trait":"line","x":-665,"y":-215},{"trait":"line","x":-500,"y":-50},{"trait":"line","x":665,"y":-215},{"trait":"line","x":500,"y":-50},{"trait":"line","x":-665,"y":215},{"trait":"line","x":-500,"y":50},{"trait":"line","x":665,"y":215},{"trait":"line","x":500,"y":50},{"bCoef":1,"trait":"ballArea","x":665,"y":290},{"bCoef":1,"trait":"ballArea","x":665,"y":-290},{"bCoef":0,"trait":"line","x":0,"y":290},{"bCoef":0,"trait":"line","x":0,"y":-290},{"x":0,"y":80,"trait":"kickOffBarrier"},{"x":0,"y":-80,"trait":"kickOffBarrier"},{"x":674,"y":-80,"trait":"line","cMask":["ball"],"bCoef":1},{"x":674,"y":-290,"trait":"ballArea","cMask":["ball"],"bCoef":1},{"x":-674,"y":-80,"trait":"line","cMask":["ball"],"bCoef":1},{"x":-674,"y":-290,"trait":"ballArea","cMask":["ball"],"bCoef":1},{"x":-674,"y":80,"trait":"line","cMask":["ball"],"bCoef":1},{"x":-674,"y":290,"trait":"ballArea","cMask":["ball"],"bCoef":1},{"x":674,"y":80,"trait":"line","cMask":["ball"],"bCoef":1},{"x":674,"y":290,"trait":"ballArea","cMask":["ball"],"bCoef":1}],"segments":[{"v0":0,"v1":1,"trait":"ballArea"},{"v0":2,"v1":3,"trait":"ballArea"},{"v0":4,"v1":5,"trait":"ballArea"},{"v0":6,"v1":7,"trait":"ballArea"},{"v0":8,"v1":9,"trait":"kickOffBarrier"},{"v0":9,"v1":10,"trait":"kickOffBarrier","curve":180,"cGroup":["blueKO"]},{"v0":9,"v1":10,"trait":"kickOffBarrier","curve":-180,"cGroup":["redKO"]},{"v0":10,"v1":11,"trait":"kickOffBarrier"},{"vis":true,"bCoef":0.1,"cMask":["ball"],"trait":"goalNet","v0":2,"v1":12,"color":"FFFFFF","curve":-35},{"vis":true,"bCoef":0.1,"cMask":["ball"],"trait":"goalNet","v0":6,"v1":13,"color":"FFFFFF","curve":35},{"vis":true,"bCoef":0.1,"cMask":["ball"],"trait":"goalNet","v0":1,"v1":14,"color":"FFFFFF","curve":35},{"vis":true,"bCoef":0.1,"cMask":["ball"],"trait":"goalNet","v0":5,"v1":15,"color":"FFFFFF","curve":-35},{"vis":true,"bCoef":0.1,"cMask":["ball"],"trait":"goalNet","v0":12,"v1":14,"x":-585,"color":"FFFFFF","curve":-35},{"vis":true,"bCoef":0.1,"cMask":["ball"],"trait":"goalNet","v0":13,"v1":15,"x":585,"color":"FFFFFF","curve":35},{"color":"FFFFFF","trait":"line","v0":16,"v1":17,"curve":90},{"color":"FFFFFF","trait":"line","v0":18,"v1":19,"curve":-90},{"color":"FFFFFF","trait":"line","v0":20,"v1":21,"curve":-90},{"color":"FFFFFF","trait":"line","v0":22,"v1":23,"curve":90},{"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line","v0":17,"v1":21,"curve":0},{"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line","v0":19,"v1":23,"curve":0},{"vis":true,"color":"FFFFFF","bCoef":1,"trait":"ballArea","v0":1,"v1":0,"cMask":["ball"],"x":-665},{"vis":true,"color":"FFFFFF","bCoef":1,"trait":"ballArea","v0":5,"v1":4,"cMask":["ball"],"x":665},{"vis":true,"color":"FFFFFF","bCoef":1,"trait":"ballArea","v0":2,"v1":3,"cMask":["ball"],"x":-665},{"vis":true,"color":"FFFFFF","bCoef":1,"trait":"ballArea","v0":6,"v1":7,"cMask":["ball"],"x":665},{"vis":true,"color":"FFFFFF","bCoef":1,"trait":"ballArea","v0":0,"v1":24,"y":290},{"vis":true,"color":"FFFFFF","bCoef":1,"trait":"ballArea","v0":3,"v1":25,"y":-290},{"curve":0,"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line","v0":26,"v1":27},{"curve":-180,"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line","v0":10,"v1":9},{"curve":180,"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line","v0":29,"v1":28},{"curve":0,"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line","v0":2,"v1":1},{"curve":0,"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line","v0":6,"v1":5},{"vis":false,"color":"FFFFFF","bCoef":1,"trait":"ballArea","v0":30,"v1":31,"cMask":["ball"],"x":614},{"vis":false,"color":"FFFFFF","bCoef":1,"trait":"ballArea","v0":32,"v1":33,"cMask":["ball"],"x":-614},{"vis":false,"color":"FFFFFF","bCoef":1,"trait":"ballArea","v0":34,"v1":35,"cMask":["ball"],"x":-614},{"vis":false,"color":"FFFFFF","bCoef":1,"trait":"ballArea","v0":36,"v1":37,"cMask":["ball"],"x":614}],"goals":[{"p0":[-674,-80],"p1":[-674,80],"team":"red"},{"p0":[674,80],"p1":[674,-80],"team":"blue"}],"discs":[{"pos":[-665,80],"trait":"goalPost","color":"FFFFFF","radius":5},{"pos":[-665,-80],"trait":"goalPost","color":"FFFFFF","radius":5},{"pos":[665,80],"trait":"goalPost","color":"FFFFFF","radius":5},{"pos":[665,-80],"trait":"goalPost","color":"FFFFFF","radius":5}],"planes":[{"normal":[0,1],"dist":-290,"trait":"ballArea"},{"normal":[0,-1],"dist":-290,"trait":"ballArea"},{"normal":[0,1],"dist":-339,"bCoef":0.2,"cMask":["all"]},{"normal":[0,-1],"dist":-339,"bCoef":0.2,"cMask":["all"]},{"normal":[1,0],"dist":-755,"bCoef":0.2,"cMask":["all"]},{"normal":[-1,0],"dist":-755,"bCoef":0.2,"cMask":["all"]}],"traits":{"ballArea":{"vis":false,"bCoef":1,"cMask":["ball"]},"goalPost":{"radius":8,"invMass":0,"bCoef":1},"goalNet":{"vis":true,"bCoef":0.1,"cMask":["all"]},"kickOffBarrier":{"vis":false,"bCoef":0.1,"cGroup":["redKO","blueKO"],"cMask":["red","blue"]},"line":{"vis":true,"bCoef":0,"cMask":[""]},"arco":{"radius":2,"cMask":["n\/d"],"color":"cccccc"}},"playerPhysics":{"acceleration":0.11,"kickingAcceleration":0.1,"kickStrength":7},"ballPhysics":{"radius":6.4,"color":"9CF0E5"}}`,

"5v5" :`{"name":"QWQ'sStadium","width":900,"height":404,"spawnDistance":310,"bg":{"type":"grass","width":773,"height":336,"kickOffRadius":95,"cornerRadius":0,"color":"6C8F55"},"vertexes":[{"x":-773,"y":-336,"cMask":["ball"],"bCoef":1},{"x":773,"y":-336,"cMask":["ball"],"bCoef":1},{"x":773,"y":336,"cMask":["ball"],"bCoef":1},{"x":-773,"y":336,"cMask":["ball"],"bCoef":1},{"x":-773,"y":-100,"cMask":["ball"],"bCoef":1},{"x":-773,"y":100,"cMask":["ball"],"bCoef":1},{"x":773,"y":-100,"cMask":["ball"],"bCoef":1},{"x":773,"y":100,"cMask":["ball"],"bCoef":1},{"x":-773,"y":-100,"cMask":["ball"],"bCoef":0.1},{"x":-820,"y":-100,"cMask":["ball"],"bCoef":0.1},{"x":-820,"y":100,"cMask":["ball"],"bCoef":0.1},{"x":-773,"y":100,"cMask":["ball"],"bCoef":0.1},{"x":-780.8333333333334,"y":-100,"cMask":[]},{"x":-780.8333333333334,"y":100,"cMask":[]},{"x":-788.6666666666666,"y":-100,"cMask":[]},{"x":-788.6666666666666,"y":100,"cMask":[]},{"x":-796.5,"y":-100,"cMask":[]},{"x":-796.5,"y":100,"cMask":[]},{"x":-804.3333333333334,"y":-100,"cMask":[]},{"x":-804.3333333333334,"y":100,"cMask":[]},{"x":-812.1666666666666,"y":-100,"cMask":[]},{"x":-812.1666666666666,"y":100,"cMask":[]},{"x":-773,"y":-66,"cMask":[]},{"x":-820,"y":-66,"cMask":[]},{"x":-773,"y":-33,"cMask":[]},{"x":-820,"y":-33,"cMask":[]},{"x":-773,"y":0,"cMask":[]},{"x":-820,"y":0,"cMask":[]},{"x":-773,"y":33,"cMask":[]},{"x":-820,"y":33,"cMask":[]},{"x":-773,"y":66,"cMask":[]},{"x":-820,"y":66,"cMask":[]},{"x":773,"y":-100,"cMask":["ball"],"bCoef":0.1},{"x":820,"y":-100,"cMask":["ball"],"bCoef":0.1},{"x":820,"y":100,"cMask":["ball"],"bCoef":0.1},{"x":773,"y":100,"cMask":["ball"],"bCoef":0.1},{"x":780.8333333333334,"y":-100,"cMask":[]},{"x":780.8333333333334,"y":100,"cMask":[]},{"x":788.6666666666666,"y":-100,"cMask":[]},{"x":788.6666666666666,"y":100,"cMask":[]},{"x":796.5,"y":-100,"cMask":[]},{"x":796.5,"y":100,"cMask":[]},{"x":804.3333333333334,"y":-100,"cMask":[]},{"x":804.3333333333334,"y":100,"cMask":[]},{"x":812.1666666666666,"y":-100,"cMask":[]},{"x":812.1666666666666,"y":100,"cMask":[]},{"x":773,"y":-66,"cMask":[]},{"x":820,"y":-66,"cMask":[]},{"x":773,"y":-33,"cMask":[]},{"x":820,"y":-33,"cMask":[]},{"x":773,"y":0,"cMask":[]},{"x":820,"y":0,"cMask":[]},{"x":773,"y":33,"cMask":[]},{"x":820,"y":33,"cMask":[]},{"x":773,"y":66,"cMask":[]},{"x":820,"y":66,"cMask":[]},{"x":0,"y":-336,"cMask":[]},{"x":0,"y":336,"cMask":[]},{"x":0,"y":-95,"cMask":[]},{"x":0,"y":95,"cMask":[]},{"x":0,"y":-404,"cMask":["red","blue"],"cGroup":["redKO","blueKO"]},{"x":0,"y":-95,"cMask":["red","blue"],"cGroup":["redKO","blueKO"]},{"x":0,"y":95,"cMask":["red","blue"],"cGroup":["redKO","blueKO"]},{"x":0,"y":404,"cMask":["red","blue"],"cGroup":["redKO","blueKO"]},{"x":-773,"y":-200,"cMask":[]},{"x":-573,"y":-200,"cMask":[]},{"x":-573,"y":200,"cMask":[]},{"x":-773,"y":200,"cMask":[]},{"x":-773,"y":-140,"cMask":[]},{"x":-693,"y":-140,"cMask":[]},{"x":-693,"y":140,"cMask":[]},{"x":-773,"y":140,"cMask":[]},{"x":-573.0,"y":-67.34983296193094,"cMask":[]},{"x":-565.8583932371607,"y":-59.39715604828653,"cMask":[]},{"x":-559.6553462429176,"y":-50.692569600036556,"cMask":[]},{"x":-554.4693834869405,"y":-41.34626511422705,"cMask":[]},{"x":-550.3661541534958,"y":-31.47655760665599,"cMask":[]},{"x":-547.397601087389,"y":-21.208387860223425,"cMask":[]},{"x":-545.6013032486728,"y":-10.671740797591571,"cMask":[]},{"x":-545.0,"y":0.0,"cMask":[]},{"x":-545.6013032486728,"y":10.671740797591571,"cMask":[]},{"x":-547.397601087389,"y":21.208387860223414,"cMask":[]},{"x":-550.3661541534959,"y":31.476557606655994,"cMask":[]},{"x":-554.4693834869405,"y":41.34626511422706,"cMask":[]},{"x":-559.6553462429176,"y":50.692569600036556,"cMask":[]},{"x":-565.8583932371607,"y":59.39715604828653,"cMask":[]},{"x":-573.0,"y":67.34983296193094,"cMask":[]},{"x":773,"y":-200,"cMask":[]},{"x":573,"y":-200,"cMask":[]},{"x":573,"y":200,"cMask":[]},{"x":773,"y":200,"cMask":[]},{"x":773,"y":-140,"cMask":[]},{"x":693,"y":-140,"cMask":[]},{"x":693,"y":140,"cMask":[]},{"x":773,"y":140,"cMask":[]},{"x":573.0,"y":-67.34983296193094,"cMask":[]},{"x":565.8583932371607,"y":-59.39715604828653,"cMask":[]},{"x":559.6553462429176,"y":-50.692569600036556,"cMask":[]},{"x":554.4693834869405,"y":-41.34626511422705,"cMask":[]},{"x":550.3661541534958,"y":-31.47655760665599,"cMask":[]},{"x":547.397601087389,"y":-21.208387860223425,"cMask":[]},{"x":545.6013032486728,"y":-10.671740797591571,"cMask":[]},{"x":545.0,"y":0.0,"cMask":[]},{"x":545.6013032486728,"y":10.671740797591571,"cMask":[]},{"x":547.397601087389,"y":21.208387860223414,"cMask":[]},{"x":550.3661541534959,"y":31.476557606655994,"cMask":[]},{"x":554.4693834869405,"y":41.34626511422706,"cMask":[]},{"x":559.6553462429176,"y":50.692569600036556,"cMask":[]},{"x":565.8583932371607,"y":59.39715604828653,"cMask":[]},{"x":573.0,"y":67.34983296193094,"cMask":[]},{"x":-747.0,"y":-336.0,"cMask":[]},{"x":-747.4995827095161,"y":-330.92765162758064,"cMask":[]},{"x":-748.9791321547066,"y":-326.05023075850767,"cMask":[]},{"x":-751.3817900801338,"y":-321.55517394149035,"cMask":[]},{"x":-754.6152236891497,"y":-317.6152236891498,"cMask":[]},{"x":-758.5551739414904,"y":-314.38179008013384,"cMask":[]},{"x":-763.0502307585076,"y":-311.9791321547065,"cMask":[]},{"x":-767.9276516275806,"y":-310.499582709516,"cMask":[]},{"x":-773.0,"y":-310.0,"cMask":[]},{"x":-747.0,"y":336.0,"cMask":[]},{"x":-747.4995827095161,"y":330.92765162758064,"cMask":[]},{"x":-748.9791321547066,"y":326.05023075850767,"cMask":[]},{"x":-751.3817900801338,"y":321.55517394149035,"cMask":[]},{"x":-754.6152236891497,"y":317.6152236891498,"cMask":[]},{"x":-758.5551739414904,"y":314.38179008013384,"cMask":[]},{"x":-763.0502307585076,"y":311.9791321547065,"cMask":[]},{"x":-767.9276516275806,"y":310.499582709516,"cMask":[]},{"x":-773.0,"y":310.0,"cMask":[]},{"x":747.0,"y":-336.0,"cMask":[]},{"x":747.4995827095161,"y":-330.92765162758064,"cMask":[]},{"x":748.9791321547066,"y":-326.05023075850767,"cMask":[]},{"x":751.3817900801338,"y":-321.55517394149035,"cMask":[]},{"x":754.6152236891497,"y":-317.6152236891498,"cMask":[]},{"x":758.5551739414904,"y":-314.38179008013384,"cMask":[]},{"x":763.0502307585076,"y":-311.9791321547065,"cMask":[]},{"x":767.9276516275806,"y":-310.499582709516,"cMask":[]},{"x":773.0,"y":-310.0,"cMask":[]},{"x":747.0,"y":336.0,"cMask":[]},{"x":747.4995827095161,"y":330.92765162758064,"cMask":[]},{"x":748.9791321547066,"y":326.05023075850767,"cMask":[]},{"x":751.3817900801338,"y":321.55517394149035,"cMask":[]},{"x":754.6152236891497,"y":317.6152236891498,"cMask":[]},{"x":758.5551739414904,"y":314.38179008013384,"cMask":[]},{"x":763.0502307585076,"y":311.9791321547065,"cMask":[]},{"x":767.9276516275806,"y":310.499582709516,"cMask":[]},{"x":773.0,"y":310.0,"cMask":[]}],"segments":[{"v0":0,"v1":1,"vis":true,"color":"FFFFFF","bCoef":1,"cMask":["ball"],"bias":-10},{"v0":3,"v1":2,"vis":true,"color":"FFFFFF","bCoef":1,"cMask":["ball"],"bias":10},{"v0":0,"v1":4,"vis":true,"color":"FFFFFF","bCoef":1,"cMask":["ball"],"bias":10},{"v0":5,"v1":3,"vis":true,"color":"FFFFFF","bCoef":1,"cMask":["ball"],"bias":10},{"v0":1,"v1":6,"vis":true,"color":"FFFFFF","bCoef":1,"cMask":["ball"],"bias":-10},{"v0":7,"v1":2,"vis":true,"color":"FFFFFF","bCoef":1,"cMask":["ball"],"bias":-10},{"v0":8,"v1":9,"vis":true,"color":"FFB3B3","bCoef":0.1,"cMask":["ball"],"curve":0},{"v0":9,"v1":10,"vis":true,"color":"FFB3B3","bCoef":0.1,"cMask":["ball"],"curve":0},{"v0":10,"v1":11,"vis":true,"color":"FFB3B3","bCoef":0.1,"cMask":["ball"],"curve":0},{"v0":12,"v1":13,"vis":true,"color":"E6ECF2","bCoef":0,"cMask":[],"curve":0},{"v0":14,"v1":15,"vis":true,"color":"E6ECF2","bCoef":0,"cMask":[],"curve":0},{"v0":16,"v1":17,"vis":true,"color":"E6ECF2","bCoef":0,"cMask":[],"curve":0},{"v0":18,"v1":19,"vis":true,"color":"E6ECF2","bCoef":0,"cMask":[],"curve":0},{"v0":20,"v1":21,"vis":true,"color":"E6ECF2","bCoef":0,"cMask":[],"curve":0},{"v0":22,"v1":23,"vis":true,"color":"E6ECF2","bCoef":0,"cMask":[],"curve":0},{"v0":24,"v1":25,"vis":true,"color":"E6ECF2","bCoef":0,"cMask":[],"curve":0},{"v0":26,"v1":27,"vis":true,"color":"E6ECF2","bCoef":0,"cMask":[],"curve":0},{"v0":28,"v1":29,"vis":true,"color":"E6ECF2","bCoef":0,"cMask":[],"curve":0},{"v0":30,"v1":31,"vis":true,"color":"E6ECF2","bCoef":0,"cMask":[],"curve":0},{"v0":32,"v1":33,"vis":true,"color":"B3C6FF","bCoef":0.1,"cMask":["ball"],"curve":0},{"v0":33,"v1":34,"vis":true,"color":"B3C6FF","bCoef":0.1,"cMask":["ball"],"curve":0},{"v0":34,"v1":35,"vis":true,"color":"B3C6FF","bCoef":0.1,"cMask":["ball"],"curve":0},{"v0":36,"v1":37,"vis":true,"color":"E6ECF2","bCoef":0,"cMask":[],"curve":0},{"v0":38,"v1":39,"vis":true,"color":"E6ECF2","bCoef":0,"cMask":[],"curve":0},{"v0":40,"v1":41,"vis":true,"color":"E6ECF2","bCoef":0,"cMask":[],"curve":0},{"v0":42,"v1":43,"vis":true,"color":"E6ECF2","bCoef":0,"cMask":[],"curve":0},{"v0":44,"v1":45,"vis":true,"color":"E6ECF2","bCoef":0,"cMask":[],"curve":0},{"v0":46,"v1":47,"vis":true,"color":"E6ECF2","bCoef":0,"cMask":[],"curve":0},{"v0":48,"v1":49,"vis":true,"color":"E6ECF2","bCoef":0,"cMask":[],"curve":0},{"v0":50,"v1":51,"vis":true,"color":"E6ECF2","bCoef":0,"cMask":[],"curve":0},{"v0":52,"v1":53,"vis":true,"color":"E6ECF2","bCoef":0,"cMask":[],"curve":0},{"v0":54,"v1":55,"vis":true,"color":"E6ECF2","bCoef":0,"cMask":[],"curve":0},{"v0":4,"v1":5,"vis":true,"color":"FF8A8A","bCoef":0,"cMask":[],"curve":0},{"v0":6,"v1":7,"vis":true,"color":"8AA6FF","bCoef":0,"cMask":[],"curve":0},{"v0":56,"v1":57,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":58,"v1":59,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":180},{"v0":58,"v1":59,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":-180},{"v0":60,"v1":61,"vis":false,"bCoef":0.1,"cMask":["red","blue"],"cGroup":["redKO","blueKO"]},{"v0":62,"v1":63,"vis":false,"bCoef":0.1,"cMask":["red","blue"],"cGroup":["redKO","blueKO"]},{"v0":62,"v1":61,"vis":false,"curve":180,"bCoef":0.1,"cMask":["red","blue"],"cGroup":["blueKO"]},{"v0":62,"v1":61,"vis":false,"curve":-180,"bCoef":0.1,"cMask":["red","blue"],"cGroup":["redKO"]},{"v0":64,"v1":65,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":65,"v1":66,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":66,"v1":67,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":68,"v1":69,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":69,"v1":70,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":70,"v1":71,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":72,"v1":73,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":73,"v1":74,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":74,"v1":75,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":75,"v1":76,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":76,"v1":77,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":77,"v1":78,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":78,"v1":79,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":79,"v1":80,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":80,"v1":81,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":81,"v1":82,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":82,"v1":83,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":83,"v1":84,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":84,"v1":85,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":85,"v1":86,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":87,"v1":88,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":88,"v1":89,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":89,"v1":90,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":91,"v1":92,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":92,"v1":93,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":93,"v1":94,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":95,"v1":96,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":96,"v1":97,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":97,"v1":98,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":98,"v1":99,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":99,"v1":100,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":100,"v1":101,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":101,"v1":102,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":102,"v1":103,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":103,"v1":104,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":104,"v1":105,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":105,"v1":106,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":106,"v1":107,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":107,"v1":108,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":108,"v1":109,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":110,"v1":111,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":111,"v1":112,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":112,"v1":113,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":113,"v1":114,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":114,"v1":115,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":115,"v1":116,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":116,"v1":117,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":117,"v1":118,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":119,"v1":120,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":120,"v1":121,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":121,"v1":122,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":122,"v1":123,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":123,"v1":124,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":124,"v1":125,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":125,"v1":126,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":126,"v1":127,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":128,"v1":129,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":129,"v1":130,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":130,"v1":131,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":131,"v1":132,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":132,"v1":133,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":133,"v1":134,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":134,"v1":135,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":135,"v1":136,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":137,"v1":138,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":138,"v1":139,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":139,"v1":140,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":140,"v1":141,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":141,"v1":142,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":142,"v1":143,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":143,"v1":144,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0},{"v0":144,"v1":145,"vis":true,"color":"FFFFFF","bCoef":0,"cMask":[],"curve":0}],"goals":[{"p0":[-782,-100],"p1":[-782,100],"team":"red"},{"p0":[782,100],"p1":[782,-100],"team":"blue"}],"discs":[{"radius":6.4,"color":"FFFFFF","bCoef":0.468,"invMass":1.4,"cGroup":["ball","kick","score"]},{"pos":[-773,-100],"radius":5,"invMass":0,"bCoef":1.25,"color":"FF6B6B"},{"pos":[-773,100],"radius":5,"invMass":0,"bCoef":1.25,"color":"FF6B6B"},{"pos":[773,-100],"radius":5,"invMass":0,"bCoef":1.25,"color":"6B8CFF"},{"pos":[773,100],"radius":5,"invMass":0,"bCoef":1.25,"color":"6B8CFF"}],"planes":[{"normal":[0,1],"dist":-336,"bCoef":1,"cMask":["ball"]},{"normal":[0,-1],"dist":-336,"bCoef":1,"cMask":["ball"]},{"normal":[0,1],"dist":-404,"bCoef":0.2,"cMask":["all"]},{"normal":[0,-1],"dist":-404,"bCoef":0.2,"cMask":["all"]},{"normal":[1,0],"dist":-900,"bCoef":0.2,"cMask":["all"]},{"normal":[-1,0],"dist":-900,"bCoef":0.2,"cMask":["all"]}],"playerPhysics":{"radius":14.75,"acceleration":0.11,"kickingAcceleration":0.083,"kickStrength":4.655,"bCoef":0.25},"ballPhysics":"disc0","joints":[],"canBeStored":false,"kickOffReset":"full","cameraFollow":"ball","redSpawnPoints":[[-310,0],[-310,55],[-310,-55],[-310,110],[-310,-110]],"blueSpawnPoints":[[310,0],[310,55],[310,-55],[310,110],[310,-110]]}`,
"practice": `{"name":"Training Futsal Wall by FY [ʜᴀxᴍᴏᴅs.ᴄᴏᴍ]","width":420,"height":200,"bg":{"type":"grass","width":420,"height":200,"color":"58A380"},"vertexes":[{"x":424,"y":-2,"cMask":[],"cGroup":[]},{"x":-424,"y":-2,"cMask":[],"cGroup":[]},{"x":424,"y":0,"cMask":[],"cGroup":[]},{"x":-424,"y":0,"cMask":[],"cGroup":[]},{"x":424,"y":2,"cMask":[],"cGroup":[]},{"x":-424,"y":2,"cMask":[],"cGroup":[]},{"x":-424,"y":4,"cMask":[],"cGroup":[]},{"x":-144,"y":-204,"cMask":[],"cGroup":[]},{"x":-144,"y":204,"cMask":[],"cGroup":[]},{"x":-142,"y":-204,"cMask":[],"cGroup":[]},{"x":-142,"y":204,"cMask":[],"cGroup":[]},{"x":-140,"y":-204,"cMask":[],"cGroup":[]},{"x":-140,"y":204,"cMask":[],"cGroup":[]},{"x":-138,"y":-204,"cMask":[],"cGroup":[]},{"x":-138,"y":204,"cMask":[],"cGroup":[]},{"x":138,"y":-204,"cMask":[],"cGroup":[]},{"x":138,"y":204,"cMask":[],"cGroup":[]},{"x":140,"y":-204,"cMask":[],"cGroup":[]},{"x":140,"y":204,"cMask":[],"cGroup":[]},{"x":142,"y":-204,"cMask":[],"cGroup":[]},{"x":142,"y":204,"cMask":[],"cGroup":[]},{"x":144,"y":-204,"cMask":[],"cGroup":[]},{"x":144,"y":204,"cMask":[],"cGroup":[]},{"x":-137,"y":-200,"cMask":["ball"]},{"x":-137,"y":-4,"cMask":["ball"]},{"x":137,"y":-200,"cMask":["ball"]},{"x":137,"y":-4,"cMask":["ball"]},{"x":-137,"y":-200,"cMask":["ball"]},{"x":137,"y":-200,"cMask":["ball"]},{"x":-137,"y":-4,"cMask":["ball"]},{"x":137,"y":-4,"cMask":["ball"]},{"x":-420,"y":-200,"cMask":["ball"]},{"x":-420,"y":-4,"cMask":["ball"]},{"x":-146,"y":-200,"cMask":["ball"]},{"x":-146,"y":-4,"cMask":["ball"]},{"x":-420,"y":-200,"cMask":["ball"]},{"x":-146,"y":-200,"cMask":["ball"]},{"x":-420,"y":-4,"cMask":["ball"]},{"x":-146,"y":-4,"cMask":["ball"]},{"x":-420,"y":4,"cMask":["ball"]},{"x":-420,"y":200,"cMask":["ball"]},{"x":-146,"y":4,"cMask":["ball"]},{"x":-146,"y":200,"cMask":["ball"]},{"x":-420,"y":4,"cMask":["ball"]},{"x":-146,"y":4,"cMask":["ball"]},{"x":-420,"y":200,"cMask":["ball"]},{"x":-146,"y":200,"cMask":["ball"]},{"x":146,"y":-200,"cMask":["ball"]},{"x":146,"y":-4,"cMask":["ball"]},{"x":420,"y":-200,"cMask":["ball"]},{"x":420,"y":-4,"cMask":["ball"]},{"x":146,"y":-200,"cMask":["ball"]},{"x":420,"y":-200,"cMask":["ball"]},{"x":146,"y":-4,"cMask":["ball"]},{"x":420,"y":-4,"cMask":["ball"]},{"x":420,"y":4,"cMask":["ball"]},{"x":420,"y":200,"cMask":["ball"]},{"x":146,"y":4,"cMask":["ball"]},{"x":420,"y":4,"cMask":["ball"]},{"x":146,"y":200,"cMask":["ball"]},{"x":420,"y":200,"cMask":["ball"]},{"x":-137,"y":4,"cMask":["ball"]},{"x":-137,"y":200,"cMask":["ball"]},{"x":-137,"y":4,"cMask":["ball"]},{"x":137,"y":4,"cMask":["ball"]},{"x":-137,"y":200,"cMask":["ball"]},{"x":137,"y":200,"cMask":["ball"]},{"x":283,"y":-200,"cMask":[],"cGroup":[]},{"x":283,"y":-4,"cMask":[],"cGroup":[]},{"x":-283,"y":-200,"cMask":[],"cGroup":[]},{"x":-283,"y":-4,"cMask":[],"cGroup":[]},{"x":-283,"y":200,"cMask":[],"cGroup":[]},{"x":-283,"y":4,"cMask":[],"cGroup":[]},{"x":137,"y":4,"cMask":["ball"]},{"x":137,"y":200,"cMask":["ball"]},{"x":146,"y":4,"cMask":["ball"]},{"x":146,"y":200,"cMask":["ball"]},{"x":283,"y":200,"cMask":[],"cGroup":[]},{"x":283,"y":4,"cMask":[],"cGroup":[]}],"segments":[{"v0":0,"v1":1,"cMask":[],"cGroup":[],"color":"58A380"},{"v0":2,"v1":3,"cMask":[],"cGroup":[],"color":"58A380"},{"v0":4,"v1":5,"cMask":[],"cGroup":[],"color":"58A380"},{"v0":7,"v1":8,"cMask":[],"cGroup":[],"color":"58A380"},{"v0":9,"v1":10,"cMask":[],"cGroup":[],"color":"58A380"},{"v0":11,"v1":12,"cMask":[],"cGroup":[],"color":"58A380"},{"v0":13,"v1":14,"cMask":[],"cGroup":[],"color":"58A380"},{"v0":15,"v1":16,"cMask":[],"cGroup":[],"color":"58A380"},{"v0":17,"v1":18,"cMask":[],"cGroup":[],"color":"58A380"},{"v0":19,"v1":20,"cMask":[],"cGroup":[],"color":"58A380"},{"v0":21,"v1":22,"cMask":[],"cGroup":[],"color":"58A380"},{"v0":23,"v1":24,"cMask":["ball"],"color":"C7E6BD"},{"v0":25,"v1":26,"cMask":["ball"],"color":"C7E6BD"},{"v0":27,"v1":28,"cMask":["ball"],"color":"C7E6BD"},{"v0":29,"v1":30,"cMask":["ball"],"color":"C7E6BD"},{"v0":31,"v1":32,"cMask":["ball"],"color":"C7E6BD"},{"v0":33,"v1":34,"cMask":["ball"],"color":"C7E6BD"},{"v0":35,"v1":36,"cMask":["ball"],"color":"C7E6BD"},{"v0":37,"v1":38,"cMask":["ball"],"color":"C7E6BD"},{"v0":39,"v1":40,"cMask":["ball"],"color":"C7E6BD"},{"v0":41,"v1":42,"cMask":["ball"],"color":"C7E6BD"},{"v0":43,"v1":44,"cMask":["ball"],"color":"C7E6BD"},{"v0":45,"v1":46,"cMask":["ball"],"color":"C7E6BD"},{"v0":47,"v1":48,"cMask":["ball"],"color":"C7E6BD"},{"v0":49,"v1":50,"cMask":["ball"],"color":"C7E6BD"},{"v0":51,"v1":52,"cMask":["ball"],"color":"C7E6BD"},{"v0":53,"v1":54,"cMask":["ball"],"color":"C7E6BD"},{"v0":55,"v1":56,"cMask":["ball"],"color":"C7E6BD"},{"v0":57,"v1":58,"cMask":["ball"],"color":"C7E6BD"},{"v0":59,"v1":60,"cMask":["ball"],"color":"C7E6BD"},{"v0":61,"v1":62,"cMask":["ball"],"color":"C7E6BD"},{"v0":63,"v1":64,"cMask":["ball"],"color":"C7E6BD"},{"v0":65,"v1":66,"cMask":["ball"],"color":"C7E6BD"},{"v0":67,"v1":68,"cMask":[],"cGroup":[],"color":"C7E6BD"},{"v0":69,"v1":70,"cMask":[],"cGroup":[],"color":"C7E6BD"},{"v0":71,"v1":72,"cMask":[],"cGroup":[],"color":"C7E6BD"},{"v0":73,"v1":74,"cMask":["ball"],"color":"C7E6BD"},{"v0":75,"v1":76,"cMask":["ball"],"color":"C7E6BD"},{"v0":77,"v1":78,"cMask":[],"cGroup":[],"color":"C7E6BD"}],"planes":[{"normal":[0,1],"dist":-250,"bCoef":0.1},{"normal":[0,-1],"dist":-250,"bCoef":0.1},{"normal":[1,0],"dist":-500,"bCoef":0.1},{"normal":[-1,0],"dist":-500,"bCoef":0.1},{"normal":[0,1],"dist":-200,"cGroup":["c0","c1","c2"]},{"normal":[0,-1],"dist":-200,"cGroup":["c3"]},{"normal":[0,-1],"dist":4,"cGroup":["c0","c1","c2"]},{"normal":[0,1],"dist":4,"cGroup":["c3"]},{"normal":[1,0],"dist":-420,"cGroup":["c0","c1","c2"]},{"normal":[-1,0],"dist":-420,"cGroup":["c0","c1","c2"]},{"normal":[-1,0],"dist":146,"cGroup":["c0"]},{"normal":[1,0],"dist":-137,"cGroup":["c1"]},{"normal":[-1,0],"dist":-137,"cGroup":["c1"]},{"normal":[1,0],"dist":146,"cGroup":["c2"]},{"normal":[1,0],"dist":-420,"cGroup":["c3"]},{"normal":[-1,0],"dist":146,"cGroup":["c3"]},{"normal":[0,-1],"dist":-200,"cGroup":["kick","score"]},{"normal":[0,1],"dist":4,"cGroup":["kick","score"]},{"normal":[1,0],"dist":-137,"cGroup":["kick","score"]},{"normal":[-1,0],"dist":-420,"cGroup":["kick","score"]},{"normal":[-1,0],"dist":-137,"cGroup":["score"]},{"normal":[1,0],"dist":146,"cGroup":["kick"]}],"goals":[],"discs":[{"pos":[461,229],"radius":0.01,"invMass":1.0e-7,"cMask":["wall"],"cGroup":["ball","kick"]},{"pos":[-283,-97],"radius":6,"color":"7CFC00","cMask":["red","blue","ball","c0"],"cGroup":["ball","kick"]},{"pos":[0,-97],"radius":6,"color":"7CFC00","cMask":["red","blue","ball","c1"],"cGroup":["ball","kick"]},{"pos":[283,-97],"radius":6,"color":"7CFC00","cMask":["red","blue","ball","c2"],"cGroup":["ball","kick"]},{"pos":[-283,97],"radius":6,"color":"7CFC00","cMask":["red","blue","ball","c3"],"cGroup":["ball","kick"]},{"pos":[0,97],"radius":6,"color":"7CFC00","cMask":["red","blue","ball","score"],"cGroup":["ball","kick"]},{"pos":[283,97],"radius":6,"color":"7CFC00","cMask":["red","blue","ball","kick"],"cGroup":["ball","kick"]}],"playerPhysics":{"invMass":0.3,"acceleration":0.105,"kickingAcceleration":0.073,"kickingDamping":0.97,"kickStrength":5.7,"radius":15.9852},"ballPhysics":"disc0","cameraFollow":"player","redSpawnPoints":[[-351,-97],[-68,-97],[351,-97],[-351,97],[-68,97],[351,97]],"blueSpawnPoints":[[-214,-97],[68,-97],[214,-97],[-214,97],[68,97],[214,97]],"canBeStored":true}`,
};
const PENALTY_STADIUM = '{"name":"P | PEN","width":420,"height":200,"spawnDistance":310,"bg":{"type":"grass","width":400,"height":260,"kickOffRadius":0,"cornerRadius":0,"color":"718B5B"},"vertexes":[{"x":323,"y":260,"bCoef":1,"cMask":["ball"],"trait":"ballArea","bias":-30},{"x":323,"y":100,"bCoef":1,"cMask":["ball"],"trait":"ballArea","bias":-30},{"x":323,"y":-100,"bCoef":1,"cMask":["ball"],"trait":"ballArea","bias":30},{"x":323,"y":-260,"bCoef":1,"cMask":["ball"],"trait":"ballArea","bias":30},{"x":370,"y":-100,"bCoef":0.1,"cMask":["ball"],"trait":"goalNet","radius":0},{"x":370,"y":100,"bCoef":0.1,"cMask":["ball"],"trait":"goalNet","radius":0},{"x":332,"y":-98,"bCoef":1,"cMask":["ball"],"trait":"line"},{"x":323,"y":260,"bCoef":0,"trait":"line"},{"x":323,"y":-260,"bCoef":0,"trait":"line"},{"x":0,"y":-260,"bCoef":0,"trait":"line"},{"x":0,"y":260,"bCoef":0,"trait":"line"},{"x":323,"y":200,"bCoef":0,"trait":"line"},{"x":110,"y":200,"bCoef":0,"trait":"line"},{"x":110,"y":-200,"bCoef":0,"trait":"line"},{"x":323,"y":-200,"bCoef":0,"trait":"line"},{"x":0,"y":100,"bCoef":0,"trait":"line"},{"x":0,"y":-100,"bCoef":0,"trait":"line"},{"x":110,"y":5,"bCoef":0,"trait":"line"},{"x":110,"y":-5,"bCoef":0,"trait":"line"},{"x":55,"y":-260,"bCoef":0,"cMask":["red"],"trait":"penArea"},{"x":65,"y":260,"bCoef":0,"cMask":["red"],"trait":"penArea"},{"x":300,"y":-90,"bCoef":0.1,"cMask":["red","blue"],"cGroup":["redKO","blueKO"],"trait":"kickOffBarrier"},{"x":365,"y":-90,"bCoef":0.1,"cMask":["red","blue"],"cGroup":["redKO","blueKO"],"trait":"kickOffBarrier"},{"x":365,"y":90,"bCoef":0.1,"cMask":["red","blue"],"cGroup":["redKO","blueKO"],"trait":"kickOffBarrier"},{"x":300,"y":90,"bCoef":0.1,"cMask":["red","blue"],"cGroup":["redKO","blueKO"],"trait":"kickOffBarrier"}],"segments":[{"v0":0,"v1":1,"trait":"ballArea"},{"v0":2,"v1":3,"trait":"ballArea"},{"v0":2,"v1":4,"vis":true,"color":"FFFFFF","cMask":["ball"],"trait":"goalNet","y":-100},{"v0":1,"v1":5,"vis":true,"color":"FFFFFF","cMask":["ball"],"trait":"goalNet","y":100},{"v0":1,"v1":0,"vis":true,"color":"FFFFFF","bCoef":1,"cMask":["ball"],"trait":"ballArea","bias":-30,"x":665},{"v0":2,"v1":3,"vis":true,"color":"FFFFFF","bCoef":1,"cMask":["ball"],"trait":"ballArea","bias":30,"x":665},{"v0":2,"v1":1,"curve":0,"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line"},{"v0":10,"v1":7,"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line"},{"v0":10,"v1":9,"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line"},{"v0":9,"v1":8,"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line"},{"v0":11,"v1":12,"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line"},{"v0":12,"v1":13,"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line"},{"v0":13,"v1":14,"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line"},{"v0":15,"v1":16,"curve":150,"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line"},{"v0":18,"v1":17,"curve":180,"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line"},{"v0":17,"v1":18,"curve":180,"vis":true,"color":"FFFFFF","bCoef":0,"trait":"line"},{"v0":5,"v1":4,"vis":true,"color":"FFFFFF","cMask":["ball"],"trait":"goalNet"},{"v0":19,"v1":20,"curve":45,"vis":false,"bCoef":0,"cMask":["red"],"trait":"penArea"},{"v0":21,"v1":22,"vis":false,"color":"FFFFFF","bCoef":0.1,"cMask":["red","blue"],"cGroup":["redKO","blueKO"],"trait":"kickOffBarrier"},{"v0":22,"v1":23,"vis":false,"color":"FFFFFF","bCoef":0.1,"cMask":["red","blue"],"cGroup":["redKO","blueKO"],"trait":"kickOffBarrier"},{"v0":23,"v1":24,"vis":false,"color":"FFFFFF","bCoef":0.1,"cMask":["red","blue"],"cGroup":["redKO","blueKO"],"trait":"kickOffBarrier"},{"v0":24,"v1":21,"vis":false,"color":"FFFFFF","bCoef":0.1,"cMask":["red","blue"],"cGroup":["redKO","blueKO"],"trait":"kickOffBarrier"}],"goals":[{"p0":[332,100],"p1":[332,-100],"team":"blue"},{"p0":[325,-100],"p1":[100,0],"team":"red"},{"p0":[100,0],"p1":[320,100],"team":"red"}],"discs":[{"radius":6.5,"color":"FFDEAD","bCoef":0.465,"invMass":1.4,"pos":[110,0],"cGroup":["ball","kick","score"]},{"pos":[323,100],"trait":"goalPost"},{"pos":[323,-100],"trait":"goalPost"}],"planes":[{"normal":[0,1],"dist":-260,"bCoef":0.2,"cMask":["all"],"_data":{"extremes":{"normal":[0,1],"dist":-260,"canvas_rect":[-710,-260,710,260],"a":[-710,-260],"b":[710,-260]}}},{"normal":[0,-1],"dist":-260,"bCoef":0.2,"cMask":["all"],"_data":{"extremes":{"normal":[0,-1],"dist":-260,"canvas_rect":[-710,-260,710,260],"a":[-710,260],"b":[710,260]}}},{"normal":[1,0],"dist":-499,"bCoef":0.2,"cMask":["all"],"_data":{"extremes":{"normal":[1,0],"dist":-499,"canvas_rect":[-710,-260,710,260],"a":[-499,-260],"b":[-499,260]}}},{"normal":[-1,0],"dist":-376,"bCoef":0.2,"cMask":["all"],"_data":{"extremes":{"normal":[-1,0],"dist":-376,"canvas_rect":[-710,-260,710,260],"a":[376,-260],"b":[376,260]}}}],"traits":{"ballArea":{"vis":false,"bCoef":1,"cMask":["ball"]},"goalPost":{"radius":4.65,"invMass":0,"color":"C6D881","bCoef":1.25},"goalNet":{"vis":true,"bCoef":0.2,"cMask":["all"]},"kickOffBarrier":{"vis":false,"bCoef":0.1,"cGroup":["redKO","blueKO"],"cMask":["red","blue"]},"line":{"vis":true,"bCoef":0,"cMask":[""]},"arco":{"radius":2,"cMask":["n/d"],"color":"cccccc"},"penArea":{"vis":false,"bCoef":0,"cMask":["red"]}},"playerPhysics":{"radius":14.75,"acceleration":0.11,"kickingAcceleration":0.083,"kickStrength":4.9,"bCoef":0.26},"ballPhysics":"disc0","joints":[],"canBeStored":false,"redSpawnPoints":[[-100,0]],"blueSpawnPoints":[[323,0]],"kickOffReset":"full"}';   
function geoFromCustomMap(json) {
const w = Math.abs(json.width) || 420, h = Math.abs(json.height) || 200;
return {
outerW: w, outerH: h,
sx: Math.abs(json.spawnDistance) || Math.round(w * 0.4),
H: h * 0.8, W: w * 0.8,
};
}

const STADIUM_JSON = {}, GEO = {};
for (const k of Object.keys(MODE_INFO)) {
if (MODE_INFO[k].map) continue;   // 4v4 dùng chung map 5v5
const custom = (CUSTOM_MAPS[k] || "").trim();
if (custom) {
try {
const parsed = JSON.parse(custom);
STADIUM_JSON[k] = JSON.stringify(parsed);
GEO[k] = geoFromCustomMap(parsed);
console.log(`[VN-HAX] Đã nạp map tuỳ chỉnh cho ${k}: "${parsed.name || k}"`);
} catch (e) {
console.error(`[VN-HAX] Map tuỳ chỉnh "${k}" bị lỗi JSON, dùng map mặc định:`, e);
STADIUM_JSON[k] = JSON.stringify(buildStadium(k));
GEO[k] = deriveDims(DIMS[MODE_INFO[k].dim], 2);
}
} else {
STADIUM_JSON[k] = JSON.stringify(buildStadium(k));
GEO[k] = deriveDims(DIMS[MODE_INFO[k].dim], 2);
}
}

const BALL_RADIUS = 5.8;     // chỉ số DUY NHẤT khác mặc định HaxBall (mặc định là 10)
const WALL_BCOEF = 1;  // tường: 0.5 × 1.24 = nảy 0.62
const POST_BCOEF = 1.24;  // cột gôn: cũng 0.62 cho đồng bộ (hoặc để 1.25 như trước, chênh không đáng kể)
const DEFAULT_PLAYER = { radius: 15, invMass: 0.5, damping: 0.96, acceleration: 0.11,
                         kickingAcceleration: 0.07, kickingDamping: 0.96,
                         kickStrength: 6.4,            // bù cho bóng nặng hơn một chút
                         bCoef: 0.3 };
const DEFAULT_BALL   = { invMass: 1, bCoef: 0.5, damping: 0.99 };
const FEEL = { player: DEFAULT_PLAYER, ball: DEFAULT_BALL };
const PLAYER_RESET = { radius: FEEL.player.radius, invMass: FEEL.player.invMass, damping: FEEL.player.damping, bCoef: FEEL.player.bCoef };
const BALL_ENGINE_DEFAULT = { radius: BALL_RADIUS, invMass: FEEL.ball.invMass, bCoef: FEEL.ball.bCoef, damping: FEEL.ball.damping };

function tuneWalls(parsed, wallB, postB) {
  const T = parsed.traits || {};
  const bOf = o => (o.bCoef != null ? o.bCoef : (T[o.trait] || {}).bCoef);
  const maskOf = o => o.cMask || (T[o.trait] || {}).cMask;
  const hitsBall = o => { const m = maskOf(o); return o.trait === "ballArea" || (Array.isArray(m) && m.includes("ball")); };
  const solid = o => { const b = bOf(o); return b == null || b >= 0.9; };
  for (const o of [].concat(parsed.vertexes || [], parsed.segments || [], parsed.planes || []))
    if (o.trait !== "goalNet" && hitsBall(o) && solid(o)) o.bCoef = wallB;
  for (const d of parsed.discs || [])
    if ((d.trait === "goalPost" || d.invMass === 0) && solid(d)) d.bCoef = postB;
  if (T.ballArea) T.ballArea.bCoef = wallB;
  if (T.goalPost) T.goalPost.bCoef = postB;
}

// Ép toàn bộ chỉ số về mặc định, trừ bán kính bóng. Trả về thông số bóng gốc (dùng cho !ballbuff off)
function applyDefaults(parsed, isPractice) {
  parsed.playerPhysics = Object.assign(parsed.playerPhysics || {}, DEFAULT_PLAYER);
  let src = {};
  if (isPractice) {                                   // practice có 6 quả bóng thật (disc 1-6), disc 0 là đĩa giả
    for (const d of (parsed.discs || []).slice(1))
      if ((d.cGroup || []).includes("ball")) { Object.assign(d, DEFAULT_BALL); d.radius = BALL_RADIUS; src = d; }
  } else {
    if (parsed.ballPhysics === "disc0" || parsed.ballPhysics == null) {
      if (parsed.discs && parsed.discs[0]) src = parsed.discs[0];
    } else if (typeof parsed.ballPhysics === "object") {
      src = parsed.ballPhysics;
    }
    Object.assign(src, DEFAULT_BALL); src.radius = BALL_RADIUS;
  }
  tuneWalls(parsed, WALL_BCOEF, POST_BCOEF);
  return src;
}
/* ═══ PHÁO HOA: các đĩa "hạt" khai báo sẵn trong map ═══ */
const FW_N = 130;   // 0-109: kiếm / pháo hoa | 60-79: EZ | 110-129: tia lửa khi kiếm cắm
const FW_BASE = {};
function addFwDiscs(parsed, k) {
  parsed.discs = parsed.discs || [];
  const shift = (parsed.ballPhysics && typeof parsed.ballPhysics === "object") ? 1 : 0;
  FW_BASE[k] = parsed.discs.length + shift;
  for (let i = 0; i < FW_N; i++)
    parsed.discs.push({ pos: [0, 3000], radius: 2, invMass: 1, damping: 0.97, bCoef: 0,
                        color: "FFFFFF", cMask: [], cGroup: [] });
}const ORIGINAL_BALL = {};
for (const k of Object.keys(STADIUM_JSON)) {
  try {
    const parsed = JSON.parse(STADIUM_JSON[k]);
    const src = applyDefaults(parsed, k === "practice");
    if (k !== "practice") addFwDiscs(parsed, k);
    STADIUM_JSON[k] = JSON.stringify(parsed);
    ORIGINAL_BALL[k] = Object.assign({}, BALL_ENGINE_DEFAULT, {
      radius: src.radius != null ? src.radius : BALL_RADIUS,
      invMass: src.invMass != null ? src.invMass : BALL_ENGINE_DEFAULT.invMass,
      bCoef: src.bCoef != null ? src.bCoef : BALL_ENGINE_DEFAULT.bCoef,
      damping: src.damping != null ? src.damping : BALL_ENGINE_DEFAULT.damping,
    });
  } catch (e) { logErr("applyDefaults " + k, e); ORIGINAL_BALL[k] = BALL_ENGINE_DEFAULT; }
}

// Map luân lưu cũng dùng chỉ số mặc định
const PENALTY_JSON = (() => {
  try { const p = JSON.parse(PENALTY_STADIUM); applyDefaults(p, false); return JSON.stringify(p); }
  catch (e) { console.error("[VN-HAX] Lỗi xử lý map luân lưu:", e); return PENALTY_STADIUM; }
})();
 
                                                                                                          /* ═══════════════════ 4. LƯU TRỮ (stats / staff / ban) ═══════════════════ */

const Store = (() => {
const KEY = "vnhax_room_v1", FILE = "./vnhax_data.json";
let ls = null, fs = null, dirty = false;
try { ls = typeof localStorage !== "undefined" ? localStorage : null; } catch (e) { ls = null; }
try { if (!ls && typeof require === "function") fs = require("fs"); } catch (e) { fs = null; }

let data = { stats: {}, staff: {}, bans: [], econ: {}, codes: {}, totalMatches: 0, matchMilestones: [], mostGoalsRecord: null };

function load() {
try {
let raw = null;
if (ls) raw = ls.getItem(KEY);
else if (fs && fs.existsSync(FILE)) raw = fs.readFileSync(FILE, "utf8");
if (raw) data = Object.assign(data, JSON.parse(raw));
} catch (e) { console.warn("[Store] load lỗi:", e); }
}

function save() {
try {
const raw = JSON.stringify(data);
if (ls) ls.setItem(KEY, raw); else if (fs) fs.writeFileSync(FILE, raw);
dirty = false;
} catch (e) { console.warn("[Store] save lỗi:", e); }
}

return { get data() { return data; }, load, save, touch() { dirty = true; }, flush() { if (dirty) save(); } };
})();

Store.load();
/* ═══════════════════ 5. KHỞI TẠO PHÒNG & TRẠNG THÁI ═══════════════════ */

if (!CONFIG.PASSWORDS.SUPER) CONFIG.PASSWORDS.SUPER = genPass(12);
if (!CONFIG.PASSWORDS.ADMIN) CONFIG.PASSWORDS.ADMIN = genPass(10);
console.log("%c[VN-HAX] SUPER ADMIN password: " + CONFIG.PASSWORDS.SUPER, "color:gold;font-size:14px");
console.log("%c[VN-HAX] ADMIN password: " + CONFIG.PASSWORDS.ADMIN, "color:cyan;font-size:14px");

const roomCfg = { roomName: CONFIG.ROOM_NAME, maxPlayers: CONFIG.MAX_PLAYERS, public: CONFIG.PUBLIC, noPlayer: true, geo: CONFIG.GEO };
if (CONFIG.TOKEN) roomCfg.token = CONFIG.TOKEN;
if (CONFIG.ROOM_PASSWORD) roomCfg.password = CONFIG.ROOM_PASSWORD;

const room = HBInit(roomCfg);

const S = {
players: new Map(), gone: new Map(), queue: [], mode: "practice", modeSetting: "auto", mapKey: null,
limits: { score: null, time: null }, winnerTeam: 0, winStreak: 0, champTeam: 0, result: null, match: null,
hold: false, holdUntil: 0, planTimer: null, balTimer: null, paused: false, pausedAt: 0, kickoffWaiting: true, kickoffTeam: 1,
touches: [], buffs: new Map(), ballBuff: null, bets: { winner: {}, scorer: {} }, betsLocked: false,
mutes: new Map(), chatLocked: false, slowMs: 0, loginFails: new Map(),
joinTimes: [], captchaUntil: 0, audit: [], lastSave: Now(),
};

const running = () => room.getScores() !== null;
const list = () => room.getPlayerList().filter(p => S.players.has(p.id));
const teamIds = t => list().filter(p => p.team === t).map(p => p.id);
const rec = id => S.players.get(id);
const eligible = () => [...S.players.values()].filter(p => !p.afk);
const say = (m, color = C.sys, style = "normal", sound = 0) => room.sendAnnouncement(m, null, color, style, sound);
const tell = (id, m, color = C.info, style = "normal", sound = 0) => room.sendAnnouncement(m, id, color, style, sound);

function wrap(text, n = 125) {
const out = [];
for (const line of String(text).split("\n")) {
let cur = "";
for (const w of line.split(" ")) {
if ((cur + " " + w).trim().length > n) { out.push(cur); cur = w; } else cur = (cur + " " + w).trim();
}
out.push(cur);
}
return out;
}

const tellLines = (id, text, color) => wrap(text).forEach(l => tell(id, l, color));
function logErr(where, e) { console.error("[VN-HAX] Lỗi tại " + where + ":", e); }

function audit(text) {
const line = new Date().toLocaleTimeString("vi-VN") + " " + text;
S.audit.push(line); if (S.audit.length > 200) S.audit.shift();
// console.log("[AUDIT] " + line);
}
function alertSupers(text) {
for (const p of S.players.values()) if (p.role === ROLE.SUPER) tell(p.id, "🚨 " + text, C.err, "bold", 2);
}

function on(name, fn, onErr) {
room[name] = (...args) => { try { return fn(...args); } catch (e) { logErr(name, e); return onErr; } };
}

/* ═══════════════ 6. PHÂN QUYỀN / ĐĂNG NHẬP / BAN / MUTE ═══════════════ */

function announceStaffWelcome(p) {
const badge = p.role === 2 ? "👑 SUPER ADMIN" : "🛡️ ADMIN";
const color = p.role === 2 ? C.super : C.admin;
say("🎉 ═══════════════════════════════ 🎉", color, "bold", 2);
say(`   ${badge}  ${p.name}  ĐÃ CÓ MẶT TẠI PHÒNG!`, color, "bold");
say(`   💃 Cả nhà vỗ tay chào đón, mời ${p.name} "múa" vài đường cho không khí thêm máu lửa! 🕺🔥`, color, "bold");
say("🎉 ═══════════════════════════════ 🎉", color, "small");
}
function announceExclusiveWelcome(p) {
 const ex = econ(p.auth, p.name).exclusive;
  if (!ex) return;
  say("✨ ═══════════════════════════════ ✨", ex.color, "bold", 2);
  say(`   ✨ THE CHOSEN ONE HAS ENTERED THE PITCH!  [${ex.title}] ${p.name}`, ex.color, "bold");
  say("   Cả sân cỏ như được ban phước, vận may đã tìm đúng chủ nhân. ", ex.color, "bold");
  say("✨ ═══════════════════════════════ ✨", ex.color, "small");
}
function setRole(p, role, remember) {
p.role = role;
try { room.setPlayerAdmin(p.id, role >= 1); } catch (e) {}
try { room.setPlayerAvatar(p.id, role === 2 ? "👑" : role === 1 ? "⚡" : null); } catch (e) {}
if (p.realAuth) {
if (remember && role > 0) Store.data.staff[p.auth] = { role, name: p.name, at: Now() };
if (role === 0) delete Store.data.staff[p.auth];
Store.touch();
}
}

function findBan(auth, conn) {
const t = Now(), bans = Store.data.bans;
for (let i = bans.length - 1; i >= 0; i--) if (bans[i].until && bans[i].until <= t) bans.splice(i, 1);
return bans.find(b => (auth && b.auth === auth) || (conn && b.conn === conn));
}

function addBan(target, minutes, reason, byName, kick = true) {
Store.data.bans.push({ auth: target.auth, conn: target.conn, name: target.name, reason, by: byName, at: Now(), until: minutes > 0 ? Now() + minutes * 60e3 : 0 });
Store.save();
if (kick && S.players.has(target.id))
room.kickPlayer(target.id, `Bị ban${minutes ? " " + minutes + " phút" : " vĩnh viễn"}: ${reason}`, false);
}

function getMute(p) {
const m = S.mutes.get(p.auth);
if (!m) return null;
if (m.until && m.until <= Now()) { S.mutes.delete(p.auth); return null; }
return m;
}

function guard(p) {
if (p.role >= ROLE.SUPER) return true;
const t = Now();
p.acts = p.acts.filter(x => t - x < 60e3);
p.acts.push(t);
if (p.acts.length > CONFIG.ADMIN_ACTIONS_PER_MIN) {
tell(p.id, "⛔ Bạn thao tác kiểm duyệt quá nhanh. Chờ một chút.", C.err);
alertSupers(`${p.name} (Admin) thao tác kiểm duyệt bất thường (${p.acts.length}/phút).`);
if (p.acts.length > CONFIG.ADMIN_ACTIONS_PER_MIN + 3) {
setRole(p, ROLE.USER); audit(`Tự động hạ quyền ${p.name} do lạm dụng`);
say(`🛑 ${p.name} bị tước quyền admin do lạm dụng.`, C.err);
}
return false;
}
return true;
}

/* ═══════════════════ 7. BUFF ═══════════════════ */

const BUFFS = {
speed:  { label: "⚡ Tốc độ",     role: 1, props: { damping: 0.9685 } },
giant:  { label: "🗿 Khổng lồ",   role: 1, props: { radius: 22, invMass: 0.3 } },
tiny:   { label: "🐜 Tí hon",     role: 1, props: { radius: 10, invMass: 0.8 } },
tank:   { label: "🛡️ Xe tăng",    role: 1, props: { invMass: 0.18 } },
bouncy: { label: "🏀 Nảy",        role: 1, props: { bCoef: 1.1 } },
turbo:  { label: "🚀 Siêu tốc",   role: 2, props: { damping: 0.974 } },
god:    { label: "👑 Thần thánh", role: 2, props: { radius: 19, invMass: 0.25, damping: 0.972 } },
};

const BALL_BUFFS = {
fast: { damping: 0.996 }, slow: { damping: 0.982 }, big: { radius: 15 },
small: { radius: 7 }, heavy: { invMass: 0.55 }, light: { invMass: 1.8 }, bouncy: { bCoef: 0.95 },
};

function sanitizeProps(o) {
const r = {};
if ("radius" in o) r.radius = clamp(o.radius, 3, 30);
if ("invMass" in o) r.invMass = clamp(o.invMass, 0.1, 2.5);
if ("damping" in o) r.damping = clamp(o.damping, 0.9, 0.998);
if ("bCoef" in o) r.bCoef = clamp(o.bCoef, 0, 1.5);
return r;
}

function unrate(reason) {
if (S.match && S.match.rated) { S.match.rated = false; say(`ℹ️ Trận này KHÔNG tính thống kê (${reason}).`, C.warn); }
}

function applyPlayerBuff(id) {
const b = S.buffs.get(id); if (!b) return;
const pl = room.getPlayer(id);
if (!pl || pl.team === 0 || !running()) return;
try { room.setPlayerDiscProperties(id, sanitizeProps(BUFFS[b.type].props)); } catch (e) {}
}

function removePlayerBuff(id, resetDisc = true) {
  S.buffs.delete(id);
  if (resetDisc && running()) { try { room.setPlayerDiscProperties(id, PLAYER_RESET); } catch (e) {} }
}
function applyBallBuff() {if (!running() || !S.ballBuff) return;
try { room.setDiscProperties(0, sanitizeProps(BALL_BUFFS[S.ballBuff])); } catch (e) {}
}

function reapplyBuffs() {
setTimeout(() => { try { for (const id of S.buffs.keys()) applyPlayerBuff(id); applyBallBuff(); } catch (e) { logErr("reapplyBuffs", e); } }, 0);
}

function clearBuffs() { S.buffs.clear(); S.ballBuff = null; }

/* ═══════════════════ 8. HÀNG CHỜ & ĐỘI HÌNH — Queue + Captain Pick + Winner Stays ═══════════════════ */
/*
  THIẾT KẾ
  - Hàng chờ (S.queue)  : suy ra từ room (người ở Spectator, không AFK), giữ thứ tự FIFO.
  - Đội hình (S.lineup) : cap[t] = đội trưởng, order[t] = thứ tự đội trưởng đã pick. Tồn tại qua các ván
                          (đội thắng giữ nguyên), bị reset khi đội giải tán.
  - Phiên chọn (S.picking): active, winTeam, turn, snap, pending (thao tác đổi đội chờ xác nhận).
  - evaluate() là BỘ ĐIỀU PHỐI DUY NHẤT: mọi sự kiện (vào/ra/AFK/đổi đội/pick/hết giờ) chỉ cập nhật dữ liệu
    rồi gọi evaluate(). Lượt pick được TÍNH LẠI từ quân số (đội ít người hơn pick; bằng nhau → đội thắng
    ván trước pick), không lưu biến "lượt" riêng nên không thể nhân đôi hay lệch.
  - Pick chỉ hoàn tất khi room xác nhận người đó đã vào đội (pending → confirmPending()).
*/

// ═══ CẤU HÌNH TẬP TRUNG: bảng chế độ theo số người (đếm cả người đang ở Spec, không tính AFK) ═══
// Muốn đổi ngưỡng: sửa `min`. Muốn đổi map: sửa trường `map` của chế độ đó trong MODE_INFO (mục 3).
const AUTO_PLAN = {
    tiers: [                       // xếp từ lớn xuống nhỏ; người thừa ở lại Spec
        { min: 8, mode: "5v5" },  // 10+  → 5v5 (map 5v5)
        { min: 6,  mode: "3v3" },  // 6–7  → 3v3 (map 3v3)
        { min: 4,  mode: "2v2" },  // 4–5  → 2v2 (map 2v2)
        { min: 2,  mode: "1v1" },],
    belowMin: "practice",               // dưới mức tối thiểu: "wait" = không start, chờ người | "practice" = vào luyện tập
recoveryAllowUpgrade: true,   // sau khi có người thoát, nếu đủ người thì được nâng lên chế độ lớn hơnchỉ vì có người mới vào
    recoveryDelayMs: 1500,         // chờ sau khi trận dừng rồi mới dựng lại đội hình
    mapRetryMs: 30e3,              // tải map lỗi → thử lại sau chừng này
    watchdogMs: 6e3,               // trận không dừng được → thử dừng lại / hủy phục hồi (chống kẹt)
    resumeMatch: true,
    resumeMaxWaitMs: 180e3,        // chờ đủ người quá lâu → bỏ trận cũ, dựng trận mới
    fallbackWhenShort: true,       // admin chọn tay một chế độ (vd !mode 4v4) mà thiếu người → tự hạ xuống chế độ nhỏ hơn đủ chỗ (không bao giờ nâng lên)
};
const mapKeyOf = mode => (MODE_INFO[mode] && MODE_INFO[mode].map) || mode;   // 4v4 dùng map 5v5

const confirmMs = () => CONFIG.PICK_CONFIRM_MS || 1500;   // chỉ là "đồng hồ canh" nếu room không xác nhận, KHÔNG phải độ trễ giả định

function newPicking(winTeam) {
    return { active: false, winTeam: winTeam || 1, turn: 0, snap: [], timer: null, tok: 0,
             pending: null, confirmTimer: null, transitTimer: null, evaluating: false, again: false, promptKey: "", waitSaid: false, nudge: null, deadline: 0,
             auto: false };
}S.picking = newPicking(1);
S.lineup = { cap: { 1: null, 2: null }, order: { 1: [], 2: [] }, win: 0 };
S.leaving = new Map();
S.lastContrib = {};      // điểm đóng góp ván vừa rồi theo auth: bàn + kiến tạo − phản lưới nhà   // người đã yêu cầu ra spec nhưng room chưa xác nhận: giữ nguyên vị trí trong hàng chờ

S.recovery = null;     // phiên phục hồi sau khi người thoát giữa trận (null = không có)
S.idleSaid = false;
S.pen = null; S.curTimeMin = 0; S.otSaid = false;
const tname = t => (t === 1 ? "ĐỎ" : "XANH");
const valid = id => { const p = S.players.get(id); return !!p && !p.afk && !!room.getPlayer(id); };

// Quân số lấy từ MODE_INFO: size = tối đa mỗi đội, min = tối thiểu mỗi đội (5v5: 4–5, 3v3: 3–3)
// Thành viên đội KHÔNG tính người đang trên đường ra spec
const tm = t => teamIds(t).filter(id => !S.leaving.has(id));

function limits(key) {
    const m = MODE_INFO[key];
    const min = (S.forced && key === "5v5") ? 1 : (m.min != null ? m.min : m.size);
    return { min, max: m.size };
}

/* ───────── Hàng chờ ───────── */

function syncQueue() {
    const now = Now();
    for (const [id, at] of S.leaving) {
        const pl = room.getPlayer(id);
        if (!S.players.has(id) || !pl || pl.team === 0 || now - at > confirmMs() * 2) S.leaving.delete(id);
    }
    const specs = list().filter(p => p.team === 0 && !rec(p.id).afk).map(p => p.id);
    const set = new Set(specs);
    S.queue = S.queue.filter((id, i) => (set.has(id) || S.leaving.has(id)) && S.queue.indexOf(id) === i);   // giữ nguyên thứ tự, không xếp lại khi callback đến muộn
    for (const id of specs) if (!S.queue.includes(id)) S.queue.push(id);
}

function toSpec(id, front = false) {
    S.leaving.set(id, Now());
    room.setPlayerTeam(id, 0);
    S.queue = S.queue.filter(x => x !== id);
    if (front) S.queue.unshift(id); else S.queue.push(id);
}

function printQueueList() {
    if (S.queue.length === 0) return;
    if (S.picking.active) S.picking.snap = S.queue.slice();
    const items = S.queue.map((id, i) => {
        const r = rec(id), s = r && Store.data.stats[r.auth];
        return `【${i + 1}】${r ? r.name : "?"}${s ? `(${s.rating})` : ""}`;
    });
    const capId = S.picking.active ? S.lineup.cap[S.picking.turn] : null;
    const out = (m, c, st) => (capId != null && rec(capId)) ? tell(capId, m, c, st) : say(m, c, st);
    out(`👥 HÀNG CHỜ (${items.length}) — chat SỐ để chọn | 0 = Elo cao nhất`, C.super, "bold");
    wrap(items.join("  ›  ")).forEach(l => out("   " + l, C.info));
}
/* ───────── Đội hình & đội trưởng ───────── */

function resetLineup() { S.lineup = { cap: { 1: null, 2: null }, order: { 1: [], 2: [] }, win: 0 }; }

function pruneLineup() {
    const L = S.lineup;
    for (const t of [1, 2]) {
        const on = id => valid(id) && room.getPlayer(id).team === t && !S.leaving.has(id);
        L.order[t] = L.order[t].filter(on);
        if (L.cap[t] != null && !on(L.cap[t])) L.cap[t] = null;
    }
}

function swapLineup() {   // gọi trong !swap
    const L = S.lineup;
    [L.cap[1], L.cap[2]] = [L.cap[2], L.cap[1]];
    [L.order[1], L.order[2]] = [L.order[2], L.order[1]];
    if (S.lineup.win) S.lineup.win = 3 - S.lineup.win;
    if (S.picking.active) S.picking.winTeam = 3 - S.picking.winTeam;
}

// Đội trưởng thiếu → (1) người đội trưởng pick sớm nhất còn hợp lệ, (2) thành viên khác còn lại,
// (3) đội trống → người đứng đầu hàng chờ. Không bao giờ chọn người đã rời / thuộc đội kia.
function firstJoined(ids) {
    let best = null;
    for (const id of ids) {
        const r = rec(id);
        if (r && (best == null || r.joinedAt < rec(best).joinedAt)) best = id;
    }
    return best;
}
function ensureCaptains() {
    for (const t of [1, 2]) {
        if (S.lineup.cap[t] != null) continue;
        const members = tm(t).filter(valid);
        let c = S.lineup.order[t].find(id => members.includes(id));
        if (c == null) c = firstJoined(members);
        if (c != null) {
            S.lineup.cap[t] = c;
            say(`👑 ${rec(c).name} là đội trưởng đội ${tname(t)}.`, C.warn, "bold");
            continue;
        }
        syncQueue();
        const head = S.queue.find(id => valid(id) && room.getPlayer(id).team === 0);
        if (head != null) { placeInTeam(head, t, "captain"); return; }   // chờ xác nhận rồi evaluate chạy lại
    }
}

// Có người đã bị yêu cầu ra spec nhưng room chưa xác nhận → chưa được bắt đầu trận
function inTransit() {
    for (const id of S.leaving.keys()) { const pl = room.getPlayer(id); if (S.players.has(id) && pl && pl.team !== 0) return true; }
    return false;
}

function watchTransit() {          // phòng khi callback không đến: sau một lúc thì tính lại
    const pk = S.picking;
    if (pk.transitTimer) return;
    pk.transitTimer = setTimeout(() => {
        pk.transitTimer = null;
        try { if (S.picking === pk && pk.active) { syncQueue(); evaluate(); } } catch (e) { logErr("transitWatch", e); }
    }, confirmMs() * 2 + 50);
}

// Đội thừa người, hàng chờ trống: bỏ 1 người ÍT ĐÓNG GÓP NHẤT ra hàng chờ (không bao giờ bỏ đội trưởng).
// Đóng góp = bàn + kiến tạo − phản lưới nhà của ván vừa kết thúc (người mới được pick = 0).
// Hòa điểm → bỏ người được pick sau cùng; người ra được xếp ĐẦU hàng chờ.
function dropLeastContributor(t, why) {
    const cap = S.lineup.cap[t], order = S.lineup.order[t];
    const cands = tm(t).filter(id => id !== cap && valid(id));
    if (!cands.length) return false;
const score = id => {
    const a = (rec(id) || {}).auth, m = S.match && S.match.pl && S.match.pl[a];
    return m ? matchPoints(m) : (S.lastContrib[a] || 0);
};
    cands.sort((a, b) => score(a) - score(b) || order.indexOf(b) - order.indexOf(a) || rec(b).joinedAt - rec(a).joinedAt);
    const v = cands[0];
    say(`⚖️ Đội ${tname(t)} thừa người, ${why || "hàng chờ trống"} — ${rec(v).name} (đóng góp ${score(v)}, thấp nhất) ra hàng chờ để cân đội.`, C.warn, "bold");
    toSpec(v, true);
    S.lineup.order[t] = order.filter(x => x !== v);
    recLog(`đưa ${rec(v).name} (đội ${tname(t)}, đóng góp ${score(v)}) về Spec: ${why || "hàng chờ trống"}`);
    return true;
}

/* ───────── Đổi đội có xác nhận ───────── */

function placeInTeam(id, team, kind, capId) {
    const pk = S.picking;
    pk.pending = { id, team, kind, cap: capId };
    if (pk.timer) { clearTimeout(pk.timer); pk.timer = null; }
    if (pk.confirmTimer) clearTimeout(pk.confirmTimer);
    pk.confirmTimer = setTimeout(() => {
        try {
            if (S.picking !== pk || !pk.pending || pk.pending.id !== id) return;
            failPending("room không xác nhận việc đổi đội");
            evaluate();
        } catch (e) { logErr("confirmTimer", e); }
    }, confirmMs());
    room.setPlayerTeam(id, team);
    confirmPending();                 // nếu room đã áp dụng ngay → xác nhận luôn; nếu chưa → onPlayerTeamChange sẽ xác nhận
}

function failPending(reason) {
    const pk = S.picking, pd = pk.pending;
    if (!pd) return;
    if (pk.confirmTimer) { clearTimeout(pk.confirmTimer); pk.confirmTimer = null; }
    pk.pending = null;
    pk.promptKey = "";
    if (reason) say(`⚠️ Hủy lượt chọn: ${reason}.`, C.warn);
    syncQueue();
}

// Idempotent: gọi bao nhiêu lần (kể cả callback lặp) cũng chỉ xác nhận một lần
function confirmPending() {
    const pk = S.picking, pd = pk.pending;
    if (!pd) return false;
    const pl = room.getPlayer(pd.id);
    if (!pl || !S.players.has(pd.id)) { failPending("người được chọn đã rời phòng"); evaluate(); return false; }
    if (pl.team !== pd.team) return false;
    if (pk.confirmTimer) { clearTimeout(pk.confirmTimer); pk.confirmTimer = null; }
    pk.pending = null;
   if (pd.kind === "pick" || pd.kind === "auto") {
        S.lineup.order[pd.team].push(pd.id);
        const cap = rec(pd.cap), picked = rec(pd.id);
        if (pd.kind === "pick")
            say(`✔️ 👑${cap ? cap.name : "Đội trưởng"} (${tname(pd.team)}) đã chọn ${picked ? picked.name : "người chơi"}`, C.info, "bold");
    } else {
        S.lineup.cap[pd.team] = pd.id;
        say(`👑 ${rec(pd.id).name} lên làm đội trưởng đội ${tname(pd.team)}.`, C.warn, "bold");
    }
    syncQueue();
    evaluate();
    return true;
}

// Gọi từ onPlayerTeamChange (sau syncQueue)
function onTeamChangeEvent(ch, by) {
    if (!S.picking.active) return;
    if (S.picking.pending && S.picking.pending.id === ch.id && confirmPending()) return;
    if (by) onPickChange(ch.id);      // admin / người chơi đổi đội thủ công
    else evaluate();                  // room vừa xác nhận một thao tác của bot (vd. ra spec) → tính lại, idempotent
}

/* ───────── Bộ điều phối pick ───────── */

function evaluate() {
    const pk = S.picking;
    if (!pk.active) return;
    if (pk.evaluating) { pk.again = true; return; }       // chống gọi lồng nhau
    pk.evaluating = true;
    try {
        let n = 0;
        do { pk.again = false; evalOnce(); } while (pk.again && S.picking === pk && pk.active && ++n < 20);
    } finally { pk.evaluating = false; }
}

function evalOnce() {
    const pk = S.picking;
    if (!pk.active || pk.pending) return;
    syncQueue(); pruneLineup();
    ensureCaptains();
    if (pk.pending) return;

    if (S.recovery && S.recovery.resume && !reconsiderResumeMode()) return;   // đang giữ trận cũ: số người đổi → tính lại chế độ (cùng map)

    const lim = limits(S.mode), t1 = tm(1).length, t2 = tm(2).length, waiting = S.queue.length;

    // Không đủ người cho chế độ hiện tại (đếm cả Spec): hạ ngay xuống chế độ nhỏ hơn mà số người cho phép (4v4 thiếu → 3v3...)
    // Không tự nâng lên khi có thêm người; chế độ lớn hơn chỉ áp dụng từ ván sau.
    if (!(S.recovery && S.recovery.resume)) {
        const m = pickMode();
        if (m && MODE_INFO[m].size > 0 && MODE_INFO[m].size < MODE_INFO[S.mode].size) {
            recLog(`không đủ người cho ${S.mode} (đội ${t1}-${t2}, spec ${waiting}) → hạ xuống ${m}`);
            ensureRecovery();
            cancelPick();
            schedulePlan(0);
            return;
        }
    }

    // Đủ điều kiện: hai đội bằng nhau, trong giới hạn map. Còn chỗ & còn ≥2 người chờ → tiếp tục bổ sung luân phiên
    if (t1 === t2 && t1 >= lim.min && t1 <= lim.max && (t1 === lim.max || waiting < 2)) {
        if (inTransit()) { watchTransit(); return; }       // chờ room xác nhận người vừa ra spec rồi mới bắt đầu
        return finishPick();
    }

    // Lệch quân số mà không còn ai để bù: nếu đội ít người đã đạt tối thiểu → bỏ bớt người đóng góp ít nhất của đội thừa
    if (t1 !== t2 && waiting === 0 && Math.min(t1, t2) >= lim.min && dropLeastContributor(t1 > t2 ? 1 : 2)) {
        pk.again = true;
        return;
    }

    if (waiting === 0) {                                   // chưa đủ điều kiện mà hết người → chờ người vào
        if (pk.timer) { clearTimeout(pk.timer); pk.timer = null; }
        pk.promptKey = "";
        if (!pk.waitSaid) { pk.waitSaid = true; say(`⏳ Chưa đủ người (cần ${lim.min}–${lim.max} mỗi đội, hiện ${t1}–${t2}). Chờ thêm người vào phòng...`, C.warn); }
        return;
    }
    pk.waitSaid = false;

    // Đội ít người hơn pick; bằng nhau → đội thắng ván trước pick trước
    let turn = t1 < t2 ? 1 : t2 < t1 ? 2 : pk.winTeam;
    if (tm(turn).length >= lim.max) turn = 3 - turn;
    if (tm(turn).length >= lim.max) { if (pk.timer) clearTimeout(pk.timer); return; }
    pk.turn = turn;
    const deficit = Math.abs(t1 - t2);
    if ((pk.auto || (deficit > 0 && waiting <= deficit)) && tm(turn).length < lim.max) {
        const head = S.queue.find(id => valid(id) && room.getPlayer(id).team === 0);
        if (head != null) {
            if (!pk.auto) say(`⚡ Chỉ còn ${waiting} người chờ — tự xếp ${rec(head).name} vào đội ${tname(turn)}.`, C.info);
            placeInTeam(head, turn, "auto", S.lineup.cap[turn]);
            return;
        }
    }    const key = `${turn}|${S.lineup.cap[turn]}|${t1}|${t2}`;
    if (key !== pk.promptKey || !pk.timer) { pk.promptKey = key; promptPick(); }
    else if (JSON.stringify(pk.snap) !== JSON.stringify(S.queue)) printQueueList();
}

function promptPick() {
    const pk = S.picking, capId = S.lineup.cap[pk.turn], cap = rec(capId);
    if (!cap) return;
    const lim = limits(S.mode), cnt = tm(pk.turn).length;
    say(`🎯 Lượt đội ${tname(pk.turn)} — 👑${cap.name} chat SỐ để chọn (${cnt}/${lim.max}, ${Math.round(CONFIG.PICK_TURN_MS / 1000)}s):`, C.warn, "bold", 1);
    printQueueList();
    armPickTimer();
}

function armPickTimer() {
    const pk = S.picking;
    if (pk.timer) clearTimeout(pk.timer);
    if (pk.nudge) clearInterval(pk.nudge);
    const tok = ++pk.tok;
    pk.deadline = Now() + CONFIG.PICK_TURN_MS;
    pk.nudge = setInterval(() => {
        try {
            if (S.picking !== pk || !pk.active) { clearInterval(pk.nudge); return; }
            if (pk.pending) return;
            const capId = S.lineup.cap[pk.turn];
            syncQueue();
            if (!rec(capId) || !S.queue.length) return;
            pk.snap = S.queue.slice();
            const left = Math.max(0, Math.round((pk.deadline - Now()) / 1000));
            const txt = S.queue.map((id, i) => `[${i + 1}] ${(rec(id) || {}).name || "?"}`).join(" | ");
            wrap(`⏳ ${left}s — chọn: ${txt}`).forEach(l => tell(capId, l, C.warn, "small"));
        } catch (e) { logErr("pickNudge", e); }
    }, 4000);
    pk.timer = setTimeout(() => {
        try {
            if (S.picking !== pk || !pk.active || pk.tok !== tok || pk.pending) return;
            const capId = S.lineup.cap[pk.turn];
            syncQueue();
            if (!rec(capId) || !S.queue.length) return evaluate();
            const c = rec(capId);
            c.pickMiss = (c.pickMiss || 0) + 1;
            if (c.pickMiss >= 2) {
                c.pickMiss = 0;
                const t = pk.turn;
                say(`🚫 ${c.name} bỏ lượt 2 lần liên tiếp — mất băng đội trưởng, xuống cuối hàng chờ.`, C.err, "bold");
                S.lineup.cap[t] = null;
                S.lineup.order[t] = S.lineup.order[t].filter(x => x !== capId);
                toSpec(capId);
                return evaluate();
            }
            say(`⏳ ${c.name} hết thời gian — hệ thống tự chọn người đầu hàng chờ.`, C.warn);
            doPick(capId, 0);
        } catch (e) { logErr("pickTimer", e); cancelPick(); schedulePlan(1000); }
    }, CONFIG.PICK_TURN_MS);
}
function pickByNumber(capId, num) {
    const pk = S.picking;
    if (!pk.active || capId !== S.lineup.cap[pk.turn]) return false;
    if (pk.pending) { tell(capId, "⏳ Đang xử lý lượt chọn trước, chờ một chút.", C.warn); return false; }
    syncQueue();
    const id = pk.snap[num - 1];
    const idx = id == null ? -1 : S.queue.indexOf(id);
    if (idx < 0) {
        tell(capId, "❌ Số không hợp lệ hoặc người đó đã rời hàng chờ — xem danh sách mới:", C.warn);
        printQueueList();
        return false;
    }
    return doPick(capId, idx);
}

function doPick(capId, idx) {
    const pk = S.picking;
    if (!pk.active || pk.pending || capId !== S.lineup.cap[pk.turn]) return false;
    syncQueue();
    const id = S.queue[idx];
    if (id == null) return false;
    const pl = room.getPlayer(id);
    if (!valid(id) || !pl || pl.team !== 0 || teamIds(1).includes(id) || teamIds(2).includes(id)) { evaluate(); return false; }
    placeInTeam(id, pk.turn, "pick", capId);      // lượt chỉ chuyển sau khi confirmPending() xác nhận
    return true;
}

// Người vào/ra/AFK/bị chuyển đội trong lúc chọn
function onPickChange(id) {
    const pk = S.picking;
    if (!pk.active) return;
    if (pk.pending && pk.pending.id === id && !valid(id)) failPending("người được chọn đã rời");
    syncQueue(); pruneLineup();
    evaluate();
}

function pickBest(capId) {
    syncQueue();
    let best = -1, bi = -1;
    S.queue.forEach((id, i) => {
        if (!valid(id)) return;
        const s = Store.data.stats[rec(id).auth], v = s ? s.rating : 1000;
        if (v > best) { best = v; bi = i; }
    });
    return bi < 0 ? false : doPick(capId, bi);
}
 
function handlePickChat(id, msg) {
    const pk = S.picking;
    if (!pk.active) return false;
    const isNum = /^\d{1,3}$/.test(msg), isTag = /^@\S+/.test(msg);
    if (!isNum && !isTag) return false;
    if (id !== S.lineup.cap[1] && id !== S.lineup.cap[2]) return false;
    if (id !== S.lineup.cap[pk.turn]) { tell(id, "⏳ Chưa đến lượt đội bạn.", C.warn); return true; }
    if (pk.pending) { tell(id, "⏳ Đang xử lý lượt chọn trước, chờ một chút.", C.warn); return true; }
    const me = rec(id); if (me) me.pickMiss = 0;                 // chủ động chọn → xóa vạ bỏ lượt
    if (isTag) {
        syncQueue();
        const q = matchKey(msg.slice(1));
        const names = S.queue.filter(x => rec(x));
        const hit = names.find(x => matchKey(rec(x).name) === q) || names.find(x => matchKey(rec(x).name).includes(q));
        if (hit == null) { tell(id, "❌ Không thấy người đó trong hàng chờ.", C.warn); return true; }
        doPick(id, S.queue.indexOf(hit));
        return true;
    }
    const n = parseInt(msg, 10);
    if (n === 0) pickBest(id); else pickByNumber(id, n);
    return true;
}
function cancelPick() {
    const pk = S.picking;
    if (pk) { if (pk.timer) clearTimeout(pk.timer);if (pk.nudge) clearInterval(pk.nudge); if (pk.confirmTimer) clearTimeout(pk.confirmTimer); if (pk.transitTimer) clearTimeout(pk.transitTimer); }
    S.picking = newPicking(pk ? pk.winTeam : 1);
}

function finishPick() {
    if (!S.picking.active) return;
    if (S.recovery && S.recovery.resume) { resumeMatch(); return; }   // phục hồi: TIẾP TỤC trận cũ, KHÔNG startGame
    cancelPick();
    S.winnerTeam = 0;
    startRound();
}

/* ───────── Điều kiện bắt đầu trận ───────── */

function lineupValid(key) {
    const lim = limits(key), a = tm(1), b = tm(2);
    return a.length === b.length && a.length >= lim.min && a.length <= lim.max &&
        a.every(valid) && b.every(valid) && !S.picking.pending && !inTransit();
}

function startRound() {
    if (running()) return false;
    if (!lineupValid(S.mode)) return false;        // không bao giờ bắt đầu khi đội hình không hợp lệ
    if (!applyStadium(S.mode)) { mapFailed(); return false; }   // map lỗi → giữ trận dừng, không start với map sai
    applyLimits(S.mode);
autoKits();
    recLog(`start ${S.mode} trên map ${mapKeyOf(S.mode)}: ${tm(1).length}-${tm(2).length}, spec ${S.queue.length}`);
    room.startGame();
    return true;
}

// Dòng ĐẦU của onGameStart: chặn trận khởi động ngoài luồng (nút Start gốc) khi chưa hợp lệ
function guardGameStart() {
    if (S.mode === "practice") return false;
    if (!S.recovery && !S.picking.active && lineupValid(S.mode)) return false;
    try { room.stopGame(); } catch (e) {}
    say("⛔ Đội hình chưa hợp lệ hoặc đang chọn người — chưa thể bắt đầu trận.", C.warn);
    return true;
}

/* ───────── Chế độ, sân, giới hạn ───────── */

// Số người tối thiểu để chơi một chế độ (theo bảng AUTO_PLAN; chế độ ngoài bảng: 2 × size)
const needPlayers = mode => { const t = AUTO_PLAN.tiers.find(x => x.mode === mode); return t ? t.min : MODE_INFO[mode].size * 2; };
// Chế độ lớn nhất mà n người cho phép (null = chưa đủ để thi đấu)
function autoModeFor(n) { for (const t of AUTO_PLAN.tiers) if (n >= t.min) return t.mode; return null; }

function pickMode() {
    if (S.forced && S.modeSetting === "5v5") return "5v5";
 const n = eligible().length, set = S.modeSetting;
    if (set === "auto") return autoModeFor(n) || (AUTO_PLAN.belowMin === "practice" && n >= 1 ? "practice" : null);   // null = chưa đủ người, không start
    // Admin chọn tay = chế độ MONG MUỐN (tối đa). Thiếu người thì hạ xuống chế độ nhỏ hơn mà số người cho phép.
    if (AUTO_PLAN.fallbackWhenShort && MODE_INFO[set] && AUTO_PLAN.tiers.some(t => t.mode === set) && n < needPlayers(set)) {
        const m = autoModeFor(n);
        if (m) return m;                                   // đủ cho chế độ nhỏ hơn → hạ xuống
    }
    return set;                                            // đủ người, hoặc quá ít (<mức nhỏ nhất) → giữ đúng chế độ admin chọn
}

// Trả true nếu map đã sẵn sàng. Không tải lại map nếu đang đúng map; lỗi thì KHÔNG tự đổi sang map khác.
function applyStadium(key) {
    const mk = mapKeyOf(key);
    if (S.mapKey === mk) return true;
    try {
        room.setCustomStadium(STADIUM_JSON[mk]);
        recLog(`map ${S.mapKey || "-"} → ${mk}`);
        S.mapKey = mk;
        return true;
    } catch (e) {
        logErr("setCustomStadium", e);
        S.mapKey = null;
        say("⚠️ Không tải được map " + mk + " — giữ trận dừng, sẽ thử lại sau ít giây.", C.err);
        return false;
    }
}

function applyLimits(key) {
    const mi = MODE_INFO[key];
    room.setScoreLimit(S.limits.score != null && key !== "practice" ? S.limits.score : mi.score);
    S.curTimeMin = S.limits.time != null && key !== "practice" ? S.limits.time : mi.time;
    room.setTimeLimit(S.curTimeMin);
}
function schedulePlan(delay) {
    if (S.planTimer) clearTimeout(S.planTimer);
    S.planTimer = setTimeout(() => { try { plan(); } catch (e) { logErr("plan", e); S.planTimer = null; } }, delay);
}

function restartRound(reason) {
    if (S.pen) { abortPen("admin khởi động lại trận"); return; }                                    cancelPick();
    if (reason) say(reason, C.warn);
    S.result = null; S.winnerTeam = 0;
    if (running()) room.stopGame(); else schedulePlan(500);
}

function startPractice() {
    S.mode = "practice";
    resetLineup();
    for (const p of list()) if (!rec(p.id).afk) room.setPlayerTeam(p.id, 1);
    S.queue = [];
    if (!applyStadium("practice")) return mapFailed();
    applyLimits("practice");
    room.startGame();
}

/* ───────── Luật sau trận ───────── */

function dissolveAll() {
    for (const t of [1, 2]) for (const id of teamIds(t)) toSpec(id);
    resetLineup();
}

// Đưa người dư về Spec cho đủ giới hạn chế độ mới: ưu tiên đội ĐÔNG hơn; bằng nhau → đội pick sau
// (đối thủ của đội thắng ván trước); chọn người theo quy tắc hiện có (ít đóng góp nhất, hòa → pick sau cùng).
function trimToMax(max, win) {
    let g = 24;
    while (g-- > 0) {
        const over = [1, 2].filter(t => tm(t).length > max);
        if (!over.length) return;
        const a = tm(1).length, b = tm(2).length;
        const t = over.length === 1 ? over[0] : (a > b ? 1 : b > a ? 2 : 3 - (win || 1));
        if (!dropLeastContributor(t, "vượt giới hạn chế độ mới")) return;
    }
}

// Tiêu chí "đủ người thay thế": số người hợp lệ trong hàng chờ (TRƯỚC khi đội thua ra spec)
// >= max(quân số thực tế của đội thắng, quân số tối thiểu của map)  → xem mục F của báo cáo
const KEEP_WINNERS_MIN_QUEUE_5V5 = 3;   // 5v5: hàng chờ >= số này thì đội thắng giữ nguyên

function queueEnough(winnerCount, lim) {
    const n = S.queue.filter(valid).length;
    if (S.mode === "5v5") return n >= KEEP_WINNERS_MIN_QUEUE_5V5;
    return n >= Math.max(winnerCount, lim.min);
}
function dissolveForNext(w, lim) {
    syncQueue();                                   // chốt hàng chờ TRƯỚC khi đội thua ra spec; sau đó mọi số liệu tính lại từ đầu
    const l = 3 - w, winners = teamIds(w).filter(valid);
    if (S.lineup.cap[w] == null || !winners.includes(S.lineup.cap[w])) S.lineup.cap[w] = firstJoined(winners);
    if (queueEnough(winners.length, lim)) {
        // 4.1: đội thắng giữ nguyên; TOÀN BỘ đội thua ra spec; đội trưởng mới = đầu hàng chờ (ensureCaptains)
        teamIds(l).forEach(id => toSpec(id));
    } else {
        // 4.2: chỉ giữ đội trưởng đội thắng cũ; mọi người còn lại ra spec; đội trưởng kia = đầu hàng chờ
        const keep = S.lineup.cap[w];
        teamIds(l).forEach(id => toSpec(id));
        teamIds(w).forEach(id => { if (id !== keep) toSpec(id); });
        S.lineup.order[w] = [];
    }
    S.lineup.cap[l] = null; S.lineup.order[l] = [];
}

function plan() {
    S.planTimer = null;
    if (S.pen) return;
    if (running() || S.picking.active) return;       // khóa: không chạy khi đang có trận hoặc đang pick
    if (S.hold) {
        if (Now() > S.holdUntil) S.hold = false;
        else return schedulePlan(S.holdUntil - Now() + 50);
    }
    const autoArr = !!S.autoArrange; S.autoArrange = false;   // cờ do checkBalance đặt khi nâng chế độ
    if (eligible().length === 0) return finishRecovery("không còn người");

    const prevMode = S.mode;
    const key = S.recovery ? recoveryTargetMode() : pickMode();
    if (!key) return waitForPlayers();
    S.idleSaid = false;
    announceModeChange(prevMode, key);
    S.mode = key;
    const autoFill = (autoArr || !!S.recovery);   
    const winnerTeam = S.winnerTeam;
    S.winnerTeam = 0;

    for (const p of list()) if (rec(p.id).afk && p.team !== 0) room.setPlayerTeam(p.id, 0);
    syncQueue();

    if (key === "practice") { S.winStreak = 0; S.champTeam = 0; finishRecovery("luyện tập"); return startPractice(); }

    const lim = limits(key);
    pruneLineup();
    let winTeam = 0;

    if (prevMode === "practice") {
        dissolveAll();
    } else if (winnerTeam && teamIds(winnerTeam).filter(valid).length) {
        winTeam = winnerTeam;
        if (winnerTeam === S.champTeam) S.winStreak++; else { S.winStreak = 1; S.champTeam = winnerTeam; }
        if (CONFIG.MAX_WIN_STREAK && S.winStreak >= CONFIG.MAX_WIN_STREAK && S.queue.length) {
            say(`🔁 Đội thắng đủ ${S.winStreak} trận liên tiếp — nhường sân cho người chờ!`, C.info);
            dissolveAll(); S.winStreak = 0; S.champTeam = 0; winTeam = 0;
        } else {
            dissolveForNext(winnerTeam, lim);
        }
    }

    if (winTeam) S.lineup.win = winTeam;
    trimToMax(lim.max, winTeam || S.lineup.win || 1);

    if (S.recovery) {
        const R = S.recovery;
        recLog(`phục hồi: chế độ ${R.prevMode}→${key}, map ${R.prevMap || "-"}→${mapKeyOf(key)}, đội ${tm(1).length}-${tm(2).length}, spec ${S.queue.length}, người thoát: ${R.leavers.map(l => l.name + "(" + tname(l.team) + ")").join(", ") || "-"}`);
        finishRecovery(`dựng lại đội hình ${MODE_INFO[key].label}`);
    }

    S.picking = newPicking(winTeam || S.lineup.win || 1);
    S.picking.active = true;
    S.picking.auto = autoFill;
    const per = lim.min === lim.max ? lim.max : lim.min + "–" + lim.max;
    say(autoFill
        ? `⚡ Tự động xếp đội hình ${MODE_INFO[key].label} (mỗi đội ${per} người)...`
        : `⚔️ Chuẩn bị đội hình ${MODE_INFO[key].label} (mỗi đội ${per} người).`, C.super, "bold");
    evaluate();
}
function handleMatchEnd() {
    if (S.recovery && S.recovery.resume) {                 // trận bị dừng NGOÀI Ý MUỐN lúc đang giữ trận (admin dừng, restart...)
        const R0 = S.recovery; R0.resume = false;
        if (R0.maxWait) { clearTimeout(R0.maxWait); R0.maxWait = null; }
        cancelPick();
        recLog("trận bị dừng ngoài ý muốn trong lúc phục hồi → coi là trận bị hủy (hoàn cược, không tính Elo)");
    }
    if (!S.match) { if (S.recovery) schedulePlan(AUTO_PLAN.recoveryDelayMs); return; }   // idempotent
    try {
        S.paused = false;
        const aborted = !S.result;                       // trận bị dừng giữa chừng (người thoát / admin / restart)
        S.lastContrib = {};                              // chụp điểm đóng góp TRƯỚC khi commitMatch xóa S.match
for (const [auth, x] of Object.entries((S.match && S.match.pl) || {})) S.lastContrib[auth] = matchPoints(x);
        commitMatch();
        if (Object.keys(S.bets.winner).length) refundBets();
        const res = S.result; S.result = null;
        repRegen(res);
        S.winnerTeam = res ? res.winner : 0;
        clearBuffs();
        S.touches = [];
    } finally {
        schedulePlan(S.recovery ? AUTO_PLAN.recoveryDelayMs : CONFIG.NEXT_MATCH_DELAY_MS);
    }
}
/* ═════════════════════════════════════════════════════════════════════
   [A4] CORE LUÂN LƯU — dán SAU hàm handleMatchEnd() (mục 8)
   ═════════════════════════════════════════════════════════════════════
   Thiết kế:
   - S.pen tồn tại = "đang luân lưu". Mọi handler khác thấy S.pen thì nhường lại.
   - S.match GIỮ NGUYÊN suốt luân lưu → cuối cùng gán S.result rồi gọi
     handleMatchEnd(): Elo, thống kê, cược, đội thắng ở lại... chạy y như trận thường.
   - Mỗi lượt: dừng game → đưa hết người ra Spec → người sút vào ĐỎ(team 1),
     thủ môn vào XANH(team 2) → startGame. Map luân lưu: ĐỎ ghi bàn = vào lưới,
     "bàn" của XANH = bóng ra khỏi phễu = trượt.
   - Mọi bước có hẹn giờ dùng penLater() (1 slot, có token) nên callback cũ không chạy nhầm.
   - Thủ môn của mỗi đội = người đóng góp ÍT NHẤT trong trận; người sút xoay vòng theo
     đóng góp từ cao xuống thấp. Đội 1 người (1v1): người đó vừa sút vừa gác.
*/
const PEN_KIT_RED = 0xE84545, PEN_KIT_BLUE = 0x4F8EF7;
const PEN_BLOCKED = new Set(["swap", "move", "mv", "start", "buff", "ballbuff", "pause"]);
const cnt = a => a.filter(Boolean).length;
const nameOf = id => (rec(id) || {}).name || "?";
 
// res[0] = các quả sút của ĐỎ (đội 1 gốc), res[1] = của XANH. true = vào.
function penWinnerOf(P) {
    const a = P.res[0], b = P.res[1], sa = cnt(a), sb = cnt(b), na = a.length, nb = b.length;
    if (na >= 5 && nb >= 5) return (na === nb && sa !== sb) ? (sa > sb ? 1 : 2) : null;   // hết 5 lượt / sudden death
    if (sa > sb + (5 - nb)) return 1;                                                      // đội kia không thể đuổi kịp
    if (sb > sa + (5 - na)) return 2;
    return null;
}
 
function penLater(ms, fn) {
    const P = S.pen; if (!P) return;
    if (P.timer) clearTimeout(P.timer);
    const tok = ++P.tok;
    P.timer = setTimeout(() => {
        try { if (S.pen === P && P.tok === tok) fn(); }
        catch (e) { logErr("penalty", e); abortPen("lỗi nội bộ"); }
    }, ms);
}
 
function penBoard(note) {
    const P = S.pen, row = a => { const s = a.map(x => (x ? "🟢" : "🔴")); while (s.length < 5) s.push("⚪"); return s.join(""); };
    for (let i = 0; i < 12; i++) room.sendAnnouncement(" ", null, 0xFFFFFF, "small", 0);   // đẩy bảng cũ ra khỏi màn hình
    say(`🥅 🔴ĐỎ ${row(P.res[0])} ${cnt(P.res[0])}-${cnt(P.res[1])} ${row(P.res[1])} XANH🔵${note ? "   " + note : ""}`, C.super, "bold", 1);
} 
function penRestoreTeams(P) {
    applyKits();
    for (const t of [1, 2]) for (const id of P.roster[t]) {
        const r = rec(id);
        if (r && !r.afk && room.getPlayer(id)) room.setPlayerTeam(id, t);
    }
}
 const PEN_BOX = {
  "1v1": { gx: 368, depth: 120, hh: 127 },
  "2v2": { gx: 400, depth: 130, hh: 155 },
  "3v3": { gx: 665, depth: 165, hh: 215 },
  "5v5": { gx: 773, depth: 200, hh: 200 },
};

function inOwnBox(team, b) {
  const box = PEN_BOX[mapKeyOf(S.mode)];
  if (!box || !b) return false;
  if (Math.abs(b.y) > box.hh) return false;
  return team === 1 ? b.x <= -(box.gx - box.depth) : b.x >= (box.gx - box.depth);
}
function startShootout() {
    if (S.pen || !running() || !S.match || S.result || S.mode === "practice") return false;
    const sc = room.getScores(); if (!sc) return false;
    const roster = { 1: tm(1).filter(valid), 2: tm(2).filter(valid) };
    const contrib = id => matchPoints(S.match.pl[(rec(id) || {}).auth]);
    const order = t => roster[t].slice().sort((x, y) => contrib(y) - contrib(x));   // người cuối = thủ môn
const box = PEN_BOX[mapKeyOf(S.mode)], geo0 = GEO[mapKeyOf(S.mode)] || GEO["2v2"];
const goalX = t => (t === 1 ? -1 : 1) * (box ? box.gx : geo0.W);   // đỏ chốt lưới bên trái, xanh bên phải
const closest = t => {
  let best = null, bd = Infinity;
  for (const id of roster[t]) {
    const d = room.getPlayerDiscProperties(id);
    if (!d) continue;
    const dist = Math.hypot(d.x - goalX(t), d.y);
    if (dist < bd) { bd = dist; best = id; }
  }
  return best;
};
const gkPick = [closest(1), closest(2)];
    clearBuffs();
S.pen = { roster, groups: [order(1), order(2)], gkPick, res: [[], []], phase: "init", tok: 0, timer: null, tries: 0,
          kicker: null, gk: null, side: 0, kicked: false, winner: 0,
          reg: { red: sc.red, blue: sc.blue, time: sc.time } };
    const P = S.pen;
    S.betsLocked = true; S.paused = false;
    say("🥅 ═══════════════════════════ 🥅", C.super, "bold", 2);
    say(`   HÒA ${sc.red}-${sc.blue} — VÀO LOẠT ĐÁ LUÂN LƯU!`, C.super, "bold");
    say(`   🔴 ĐỎ: ${P.groups[0].map(nameOf).join(", ") || "-"}   |   🔵 XANH: ${P.groups[1].map(nameOf).join(", ") || "-"}`, C.info);
say("   5 lượt mỗi đội, vẫn hòa thì sút tiếp (sudden death). Thủ môn = người đứng gần khung thành nhà nhất.", C.sys, "small");
    room.stopGame();                       // onGameStop bị chặn vì S.pen đã được đặt
    penLater(2500, penLoadMap);
    return true;
}
 
function penLoadMap() {
    const P = S.pen; if (!P) return;
    if (running()) { if (++P.tries > 8) return abortPen("không dừng được trận"); room.stopGame(); return penLater(300, penLoadMap); }
    P.tries = 0;
    try {
room.setCustomStadium(PENALTY_JSON);
        S.mapKey = "penalty";              // khác mọi mapKey thường → trận sau applyStadium sẽ tải lại map
        room.setScoreLimit(0); room.setTimeLimit(0);
    } catch (e) {
        logErr("penalty map", e); S.mapKey = null;
        return penDecide(Math.random() < 0.5 ? 1 : 2, "lỗi map luân lưu — tung đồng xu");
    }
    nextKick();
}
 
function nextKick() {
    const P = S.pen; if (!P) return;
    for (const g of [0, 1]) P.groups[g] = P.groups[g].filter(id => S.players.has(id) && room.getPlayer(id));
    if (!P.groups[0].length || !P.groups[1].length) {
        const w = P.groups[0].length ? 1 : P.groups[1].length ? 2 : 0;
        return w ? penDecide(w, "đội kia không còn người") : abortPen("không còn ai");
    }
    const w = penWinnerOf(P);
    if (w) return penDecide(w);
    if (Math.min(P.res[0].length, P.res[1].length) >= CONFIG.PEN_MAX_ROUNDS)
        return penDecide(Math.random() < 0.5 ? 1 : 2, "quá nhiều lượt — tung đồng xu");
 
    const side = P.res[0].length <= P.res[1].length ? 0 : 1;      // ĐỎ sút trước, rồi XANH
    const kg = P.groups[side], dg = P.groups[1 - side];
    P.side = side;
    P.kicker = kg[P.res[side].length % kg.length];
const pick = P.gkPick[1 - side];
P.gk = dg.includes(pick) ? pick : dg[dg.length - 1];   // nếu người đó đã rời thì lấy người cuối nhóm
    P.phase = "setup"; P.tries = 0;
    room.stopGame();
    penLater(400, penSetup);
}
 
function penSetup() {
    const P = S.pen; if (!P || P.phase !== "setup") return;
    if (running()) { if (++P.tries > 8) return abortPen("không dừng được trận"); room.stopGame(); return penLater(300, penSetup); }
    P.tries = 0;
    if (!S.players.has(P.kicker) || !S.players.has(P.gk)) { P.phase = "between"; return penLater(300, nextKick); }
// Người sút: ĐỎ (side 0) mặc áo đỏ, XANH (side 1) mặc áo xanh. Thủ môn luôn áo vàng.
room.setTeamColors(1, 60, 0xFFFFFF, [P.side === 0 ? PEN_KIT_RED : PEN_KIT_BLUE]);
room.setTeamColors(2, 60, 0x000000, [0xFFD700]);// thủ môn áo vàng
    for (const p of list()) if (p.team !== 0) room.setPlayerTeam(p.id, 0);
    room.setPlayerTeam(P.kicker, 1);
    room.setPlayerTeam(P.gk, 2);
    P.phase = "starting"; P.kicked = false;
    // watchdog đặt TRƯỚC startGame: nếu onGameStart đến (đồng bộ hay không) thì penOnGameStart sẽ ghi đè bằng timer sút
    penLater(3000, () => {
        if (P.phase !== "starting") return;
        if (++P.tries > 3) return abortPen("không bắt đầu được lượt sút");
        P.phase = "setup"; penSetup();
    });
    room.startGame();
}
 
function penOnGameStart() {                   // gọi từ đầu onGameStart khi S.pen
    const P = S.pen; if (!P || P.phase !== "starting") return;
    P.phase = "kick"; P.kicked = false;
    try { room.setPlayerDiscProperties(P.kicker, { invMass: 9999 }); } catch (e) {}   // (tùy chọn, giống De Paul) người sút không vô tình đẩy bóng
    const n = P.res[P.side].length + 1;
    tell(P.kicker, `⚽ Bạn có ${Math.round(CONFIG.PEN_KICK_TIMEOUT_MS / 1000)}s để sút!`, C.ok, "bold", 2);
    tell(P.gk, "🧤 Bạn là thủ môn — hãy cản phá!", C.info, "bold", 2);
    penLater(CONFIG.PEN_KICK_TIMEOUT_MS, () => penResolve(false, "không sút kịp giờ"));
}
 
function penKick(pl) {                        // onPlayerBallKick khi S.pen
    const P = S.pen;
    if (!P || P.phase !== "kick" || pl.id !== P.kicker || P.kicked) return;
    P.kicked = true;
    penLater(CONFIG.PEN_RESOLVE_MS, () => penResolve(false, "bóng không vào lưới"));
}
 
function penGoal(team) {                      // onTeamGoal khi S.pen: ĐỎ ghi = vào, XANH "ghi" = bóng ra phễu = trượt
    const P = S.pen;
    if (!P || P.phase !== "kick") return;
    penResolve(team === 1, team === 1 ? "" : "bị cản phá / sút ra ngoài");
}
 
function penResolve(scored, why) {
    const P = S.pen; if (!P || P.phase !== "kick") return;
    P.phase = "between";
    P.res[P.side].push(!!scored);
    say(scored ? `⚽ VÀOOO! ${nameOf(P.kicker)} đã lạnh lùng đặt bóng vào lưới!` : `❌ TRƯỢT! ${why}`, scored ? C.goal : C.err, "bold", 1);
    penBoard();
    penLater(CONFIG.PEN_BETWEEN_MS, nextKick);
}
 
// người sút / thủ môn rời phòng hoặc bị đưa ra Spec
function penPlayerGone(id) {
    const P = S.pen; if (!P) return;
    for (const g of [0, 1]) P.groups[g] = P.groups[g].filter(x => x !== id);
    for (const t of [1, 2]) P.roster[t] = P.roster[t].filter(x => x !== id);
    if (P.phase === "kick" && id === P.kicker) return penResolve(false, "người sút đã rời");
    if ((id === P.kicker || id === P.gk) && P.phase !== "between" && P.phase !== "done") {
        P.phase = "between";
        try { room.stopGame(); } catch (e) {}
        say("🧤 Người đang tham gia lượt này đã rời — làm lại lượt sút.", C.warn);
        penLater(1000, nextKick);
    }
}
 
function penTeamChange(ch) {                  // onPlayerTeamChange khi S.pen
    const P = S.pen; if (!P || P.phase !== "kick") return;
    if (ch.id === P.kicker || ch.id === P.gk) { if (ch.team === 0) penPlayerGone(ch.id); }
    else if (ch.team !== 0) room.setPlayerTeam(ch.id, 0);        // ai bị kéo vào sân giữa lượt → đẩy ra
}
 
function penDecide(winner, note) {
    const P = S.pen; if (!P) return;
    P.phase = "done"; P.winner = winner; P.tries = 0;
    say(`🏆 ${tname(winner)} THẮNG LUÂN LƯU ${cnt(P.res[0])}-${cnt(P.res[1])}${note ? " (" + note + ")" : ""}!`, C.super, "bold", 2);
    room.stopGame();
    penLater(2000, penFinish);
}
 
function penFinish() {
    const P = S.pen; if (!P) return;
    if (running()) { if (++P.tries > 8) return abortPen("không dừng được trận"); room.stopGame(); return penLater(300, penFinish); }
    penRestoreTeams(P);                       // trả mọi người về đội gốc để commitMatch/plan tính đúng
    penLater(800, penCommit);                 // chờ room áp dụng việc đổi đội
}
 
function penCommit() {
    const P = S.pen; if (!P) return;
    S.result = { winner: P.winner, red: P.reg.red, blue: P.reg.blue, time: P.reg.time, pen: [cnt(P.res[0]), cnt(P.res[1])] };
    S.pen = null;
    handleMatchEnd();                         // Elo, thống kê, cược, đội thắng ở lại… như trận thường
}
 
function abortPen(why) {
    const P = S.pen; if (!P) return;
    if (P.timer) clearTimeout(P.timer);
    S.pen = null; S.result = null;
    say(`⚠️ Hủy luân lưu (${why}) — trận không tính kết quả.`, C.warn, "bold");
    try { penRestoreTeams(P); } catch (e) { logErr("abortPen", e); }
    if (running()) room.stopGame(); else handleMatchEnd();   // result = null → coi như trận hủy: hoàn cược, không tính Elo
}
 
// Gọi trong tick() sau dòng `const geo = ...`
function checkOvertime(geo) {
    if (S.mode === "practice" || !S.match || S.result || S.pen || S.recovery || S.picking.active || S.paused) return;
    const sc = room.getScores();
    if (!sc || !(S.curTimeMin > 0) || sc.red !== sc.blue) return;
    const over = sc.time - S.curTimeMin * 60;          // dùng time limit tự lưu, không phụ thuộc đơn vị của sc.timeLimit
    if (over < 0) return;
    if (!S.otSaid) {
        S.otSaid = true;
        say(CONFIG.PEN_OT_SEC > 0
            ? `⏱️ HÒA ${sc.red}-${sc.blue}! Bù giờ vàng ${CONFIG.PEN_OT_SEC}s — ai ghi bàn trước thắng, hết giờ vẫn hòa sẽ ĐÁ LUÂN LƯU!`
            : `⏱️ HÒA ${sc.red}-${sc.blue}! Hai đội sẽ vào loạt ĐÁ LUÂN LƯU!`, C.super, "bold", 2);
    }
    if (over < CONFIG.PEN_OT_SEC) return;
    const b = room.getDiscProperties(0);
    const calm = b && Math.abs(b.x) < geo.W * 0.5;     // chỉ cắt khi bóng ở giữa sân (không cắt ngang pha dứt điểm)
if (calm || over > CONFIG.PEN_OT_SEC + 10) startShootout();
}
function scheduleBalance() {
    if (S.balTimer) return;
    S.balTimer = setTimeout(() => { S.balTimer = null; try { checkBalance(); } catch (e) { logErr("checkBalance", e); } }, 80);
}
// 4v4 → 5v5 ngay trong trận: không pause, không pick, tự thêm 1 người vào MỖI đội
// Kiểu De Paul: 5v5 đang chạy, hàng chờ có người → tự thêm ngay, không pause, không pick
function fillInPlace() {
    if (!running() || S.paused || S.recovery || S.pen || S.result || !S.match) return false;
    if (S.mode !== "5v5") return false;
    syncQueue(); pruneLineup();
    const max = limits("5v5").max, min = limits("5v5").min;
    const n = { 1: tm(1).filter(valid).length, 2: tm(2).filter(valid).length };
    const heads = S.queue.filter(id => valid(id) && room.getPlayer(id).team === 0);
    const placed = [];
    const add = t => {
        const id = heads.shift(); if (id == null) return;
        room.setPlayerTeam(id, t); n[t]++; placed.push([id, t]);
        S.lineup.order[t].push(id);
        const r = rec(id); if (r) { r.lastActive = Now(); r.afkWarned = false; }
    };
    while (heads.length) {
        if (n[1] === n[2]) {
            if (n[1] >= max || heads.length < 2) break;       // bằng nhau: chỉ thêm khi có ≥2 người (mỗi đội 1)
            add(1); add(2);
        } else {
            const t = n[1] < n[2] ? 1 : 2;
            if (n[t] >= max) break;
            add(t);
        }
    }
    S.queue = S.queue.filter(x => !placed.some(([id]) => id === x));
    if (placed.length) {
        say(`🔁 ${placed.map(([id, t]) => `${nameOf(id)} → ${t === 1 ? "🔴" : "🔵"}`).join(", ")} vào sân, trận tiếp tục!`, C.info, "bold", 1);
        recLog(`bù tại chỗ 5v5: ${placed.map(([id, t]) => nameOf(id) + "(" + tname(t) + ")").join(", ")}; đội ${n[1]}-${n[2]}`);
    }
    // Lệch mà hàng chờ trống → đưa người đóng góp ít nhất của đội thừa ra (như De Paul)
    if (n[1] !== n[2] && !heads.length && Math.min(n[1], n[2]) >= min) dropLeastContributor(n[1] > n[2] ? 1 : 2, "hàng chờ trống");
    return placed.length > 0;
}function checkBalance() {
    if (S.pen) return;                                                                                             if (S.picking.active) return;
    if (!running()) { if (!S.planTimer) schedulePlan(1200); return; }
    syncQueue();

    if (S.mode === "practice") {
        const n = eligible().length;
        const m = pickMode();
        if (n >= 2 && m && m !== "practice") return restartRound("⚔️ Đủ người — chuyển sang thi đấu!");
        for (const p of list()) if (p.team === 0 && !rec(p.id).afk) room.setPlayerTeam(p.id, 1);
        if (n === 0) room.stopGame();
        return;
    }
    // Đủ người cho chế độ lớn hơn (1v1→2v2→3v3→4v4→5v5): chuyển luôn, tự xếp người, đổi map
    if (!S.recovery && !S.paused && S.mode !== "practice") {
        if (S.mode === "5v5") fillInPlace(); 
        const m = pickMode(); 
        if (m && m !== "practice" && MODE_INFO[m].size > MODE_INFO[S.mode].size) {
            S.autoArrange = true;
return restartRound();        }
    }
    const r = teamIds(1).length, b = teamIds(2).length;
if (r === 0 || b === 0) return restartRound();
    if (r !== b) unrate("số người 2 đội không bằng nhau");
}  
/* ═══════ ĐIỂM ĐÓNG GÓP  ═══════ */
const PTS = { g: 5, a: 2, og: -3, pass: 1, sot: 0.5, ss: 2, err: -2, atog: 2, touch: 0.001 };
const GOAL_DIM = {
  "1v1": { x: 368, y: 50 }, "2v2": { x: 400, y: 70 },
  "3v3": { x: 665, y: 80 }, "5v5": { x: 773, y: 100 },
};
const matchPoints = x => { if (!x) return 0; let s = 0; for (const k in PTS) s += PTS[k] * (x[k] || 0); return s; };

function bump(p, key, n = 1) {
  const M = S.match; if (!M || !p) return;
  const x = M.pl[p.auth] || (M.pl[p.auth] = { name: p.name, g: 0, a: 0, og: 0 });
  x[key] = (x[key] || 0) + n;
}

S.kicks = [];
function trackKick(pl) {
  if (!S.match || S.mode === "practice" || S.pen || !pl.team) return;
  const gd = GOAL_DIM[mapKeyOf(S.mode)], b = room.getDiscProperties(0);
  if (!gd || !b) return;
  const me = rec(pl.id); if (!me) return;
  const goalX = pl.team === 1 ? gd.x : -gd.x;
  const shot = goalX * b.xspeed > 0 &&
    Math.abs(b.x + b.xspeed * 97.5) > gd.x &&
    Math.abs(b.y + b.yspeed * (goalX - b.x) / b.xspeed) < gd.y;
  const prev = S.kicks[0];
  S.kicks.unshift({ id: pl.id, team: pl.team, x: b.x, y: b.y, shot });
  if (S.kicks.length > 3) S.kicks.pop();
  if (!prev) return;
  const dist = Math.hypot(b.x - prev.x, b.y - prev.y), pp = rec(prev.id);
  if (prev.id !== pl.id) bump(me, "touch");
  if (shot) bump(me, "sot");
  if (prev.team !== pl.team) {
    if (prev.shot) { bump(me, "ss"); if (dist < 30) { bump(pp, "sot", -1); prev.shot = false; } }
  } else if (prev.id !== pl.id && dist > 12) bump(pp, "pass");
}

function motmOf(m, res) {
  const teamOf = {};
  for (const p of list()) if (p.team) teamOf[rec(p.id).auth] = p.team;
  let best = null;
  for (const [auth, x] of Object.entries(m.pl)) {
    const v = matchPoints(x) + (res && teamOf[auth] === res.winner ? 1 : 0);
    if (v > 0 && (!best || v > best[1])) best = [auth, v];
  }
  return best;
}

/* ═══════ NGƯỜI VÀO THAY ═══════ */
const SUB_RULE = { ENABLED: true, MIN_POINTS: 3 };
function isLowSub(m, auth) {
  if (!SUB_RULE.ENABLED || !m || !m.starters || m.starters[auth]) return false;
  return matchPoints(m.pl && m.pl[auth]) < SUB_RULE.MIN_POINTS;
}

/* ═══════ TÍNH KẾT QUẢ CHO NGƯỜI THOÁT / AFK ═══════ */
const ELO_LEAVE = { ENABLED: true, K: 24, DRAW_PENALTY: 6, WIN_PENALTY: 8, COMP: 0, GRACE_MS: 30e3, AFK_ADMIN: false };

function eloLeavePenalty(team, p, kind) {
  const Kc = ELO_LEAVE, m = S.match;
  if (!Kc.ENABLED || !team || !m || !m.rated) return;
  if (!running() || S.result || S.pen || S.mode === "practice") return;
  if (kind === "AFK_ADMIN" && !Kc.AFK_ADMIN) return;
  if (Now() - (m.t0 || 0) < Kc.GRACE_MS) return;
  m.quit = m.quit || {};
  if (m.quit[p.auth]) return;
  if (isLowSub(m, p.auth)) return;
  const sc = room.getScores();
  if (!sc || sc.time < CONFIG.MIN_RATED_SECONDS) return;

  const mine = sc[team === 1 ? "red" : "blue"], theirs = sc[team === 1 ? "blue" : "red"];
  const s = stat(p.auth, p.name), ec = econ(p.auth, p.name);
  const why = kind === "QUIT" ? "thoát" : "AFK";
  const before = s.rating;
  let delta = 0, line;

  if (mine < theirs) {
    const avg = ids => ids.length ? ids.reduce((a, id) => { const r = rec(id); return a + (r ? stat(r.auth, r.name).rating : 1000); }, 0) / ids.length : 1000;
    const mates = teamIds(team).filter(id => id !== p.id && valid(id));
    const foes = teamIds(3 - team).filter(valid);
    const eMine = 1 / (1 + Math.pow(10, (avg(foes) - avg([...mates, p.id])) / 400));
    delta = -Math.max(1, Math.round(Kc.K * eMine));
    if (ec.eloGuard) { delta = 0; ec.eloGuard = false; tell(p.id, "🛡️ Bảo hộ ELO đã dùng: không bị trừ điểm!", C.info, "bold"); }
    s.games++; s.losses++; s.streak = 0;
    line = `📉 ${p.name} ${why} khi đội đang thua ${mine}-${theirs} → tính THUA, ${delta} Elo`;
  } else {
    delta = -(mine === theirs ? Kc.DRAW_PENALTY : Kc.WIN_PENALTY);
    s.streak = 0;
    line = `📉 ${p.name} ${why} giữa trận (${mine}-${theirs}) → phạt ${delta} Elo`;
  }
  s.leaves = (s.leaves || 0) + 1;
  s.rating = Math.max(0, s.rating + delta);
  m.quit[p.auth] = true;
  say(`${line} (${s.rating}).`, C.warn, "bold");

  if (Kc.COMP > 0 && before > s.rating)
    for (const id of teamIds(team).filter(i => i !== p.id && valid(i))) {
      const r = rec(id); if (!r) continue;
      const ms = stat(r.auth, r.name); ms.rating += Kc.COMP;
      tell(id, `🤝 ${p.name} bỏ trận — bạn được bù +${Kc.COMP} Elo (${ms.rating}).`, C.ok, "bold");
    }
  Store.touch();
  if (S.players.has(p.id)) checkAndApplyEloRank(p, false);
}                                                                                           function moveAfk(p, reason, kind = "AFK_AUTO") {
    const pl0 = room.getPlayer(p.id), was = pl0 ? pl0.team : 0;      // lưu đội TRƯỚC khi đẩy ra
    p.afk = true; p.afkSince = Now(); p.afkWarned = false;
    removePlayerBuff(p.id);
    room.setPlayerTeam(p.id, 0);
    syncQueue();
    tell(p.id, `💤 ${reason}. Gõ !back để quay lại hàng chờ.`, C.warn, "bold", 2);
    say(`💤 ${p.name} bị chuyển sang AFK.`, C.sys, "small");
    repPenalty({ team: was }, p, "AFK giữa trận");
       eloLeavePenalty(was, p, kind); 
    if (S.picking.active) onPickChange(p.id);
    else if (was && S.mode !== "practice" && !S.pen && (running() || S.recovery))
        onTeamMemberLeftMidMatch({ id: p.id, team: was }, p);
    scheduleBalance();
}
/* ───────── Phục hồi khi người thoát giữa trận ───────── */

function recLog(m) { audit("[RECOVERY] " + m); }          // vào console + !log, không spam chat

function ensureRecovery() {
    if (!S.recovery) S.recovery = { t0: Now(), prevMode: S.mode, prevMap: S.mapKey, leavers: [], stopTries: 0, wd: null, resume: false, snap: null, maxWait: null };
    return S.recovery;
}

function finishRecovery(why) {
    const R = S.recovery;
    if (!R) return;
    if (R.wd) clearTimeout(R.wd);
    if (R.maxWait) clearTimeout(R.maxWait);
    recLog("kết thúc phục hồi: " + why);
    S.recovery = null;
}

function waitForPlayers() {
    const n = eligible().length, need = AUTO_PLAN.tiers[AUTO_PLAN.tiers.length - 1].min;
    if (!S.idleSaid) { S.idleSaid = true; say(`⏳ Cần tối thiểu ${need} người để thi đấu (hiện ${n}). Chờ thêm người vào phòng...`, C.warn); }
    finishRecovery(`chờ người (${n}/${need})`);
}

function mapFailed() {                                      // tải map lỗi: giữ trận dừng, thử lại sau mapRetryMs
    S.hold = true; S.holdUntil = Now() + AUTO_PLAN.mapRetryMs;
    schedulePlan(AUTO_PLAN.mapRetryMs + 50);
}

function refundBets() {
    let n = 0;
    for (const grp of [S.bets.winner, S.bets.scorer])
        for (const [auth, b] of Object.entries(grp)) { econ(auth, b.bettorName).coins += b.amount; n++; }
    S.bets = { winner: {}, scorer: {} }; S.betsLocked = false;
    if (n) { Store.touch(); say("💸 Trận bị hủy — đã hoàn lại tiền cược.", C.info); }
}

// Thông báo khi chế độ đổi vì số người (mỗi thay đổi chỉ báo một lần)
function announceModeChange(prev, key) {
    const set = S.modeSetting, n = eligible().length;
    let note = "", msg = "";
    if (set !== "auto" && MODE_INFO[set] && key !== set && MODE_INFO[key] && MODE_INFO[key].size > 0) {
        note = `${set}>${key}`;
        msg = `⬇️ Không đủ người cho ${set} (hiện ${n} người, cần ${needPlayers(set)}) — tự chuyển sang ${key}.`;
    } else if (prev && prev !== key && MODE_INFO[prev] && MODE_INFO[key] && MODE_INFO[prev].size > 0 && MODE_INFO[key].size > 0) {
        note = `${prev}>${key}`;
        msg = MODE_INFO[key].size < MODE_INFO[prev].size
            ? `⬇️ Không đủ người cho ${prev} (hiện ${n} người) — tự chuyển sang ${key}.`
            : `⬆️ Đủ người cho ${key} (hiện ${n} người) — chuyển từ ${prev}.`;
    }
    if (note === (S.modeNote || "")) return;
    S.modeNote = note;
if (msg) { recLog(`đổi chế độ ${prev}→${key} (${n} người)`); }
}

const fmtClock = t => String(Math.floor(t / 60)).padStart(2, "0") + ":" + String(Math.floor(t % 60)).padStart(2, "0");

// Chế độ mục tiêu khi phục hồi: theo bảng AUTO_PLAN nhưng KHÔNG nâng cấp so với chế độ lúc xảy ra sự cố
function recoveryTargetMode() {
    let key = pickMode();
    const R = S.recovery;
    if (key && R && !AUTO_PLAN.recoveryAllowUpgrade &&
        MODE_INFO[R.prevMode] && MODE_INFO[R.prevMode].size > 0 && MODE_INFO[key].size > MODE_INFO[R.prevMode].size) {
        recLog(`không nâng chế độ khi phục hồi: giữ ${R.prevMode} (bảng sẽ chọn ${key})`);
        key = R.prevMode;
    }
    return key;
}

/* ───────── Giữ trận cũ khi có người thoát: TẠM DỪNG → pick đủ người → TIẾP TỤC ─────────
   Giới hạn của HaxBall API (đã đối chiếu):
   - Không có API đặt tỉ số / đồng hồ; stopGame() rồi startGame() LUÔN ra trận mới 0–0.
   - Vì vậy bot KHÔNG gọi stopGame: dùng pauseGame(true) để engine giữ nguyên tỉ số, thời gian, vị trí bóng và cầu thủ,
     rồi pauseGame(false) sau khi đội hình đủ. Không có hàm đọc trạng thái pause nên dựa vào callback onGamePause/onGameUnpause.
   - Không đổi được map / giới hạn bàn thắng / thời gian khi trận đang chạy → nếu bảng chế độ đòi map KHÁC thì không thể giữ trận:
     bot dừng trận và dựng trận mới như cũ (có thông báo).
   - Người được pick vào giữa trận xuất hiện ở vị trí xuất phát của đội, không phải chỗ ai đó đứng lúc trước. */

function canResumeInPlace() {
    if (S.mode !== "5v5") return false;                                                                                                            if (!AUTO_PLAN.resumeMatch || !running() || !S.match || S.result || S.mode === "practice") return false;
    if (typeof room.pauseGame !== "function") return false;
    const key = recoveryTargetMode();
    return !key || mapKeyOf(key) === S.mapKey;     // key null = chưa đủ người: vẫn giữ trận tạm dừng để chờ
}

function startResumeRecovery(R) {
    R.resume = true;
    const sc = room.getScores();
    R.snap = sc ? { red: sc.red, blue: sc.blue, time: sc.time } : null;
    try { room.pauseGame(true); } catch (e) { logErr("pauseGame", e); }
    const who = R.leavers.map(l => `${l.name}(${tname(l.team)})`).join(", ");
    recLog(`TẠM DỪNG để giữ trận: ${who} thoát; tỉ số ${R.snap ? R.snap.red + "-" + R.snap.blue : "?"}, thời gian ${R.snap ? fmtClock(R.snap.time) : "?"}, chế độ ${S.mode}`);
    armResumeTimer(R);
    S.picking = newPicking(S.lineup.win || 1);
    S.picking.active = true;
    evaluate();
}

function armResumeTimer(R) {
    if (R.maxWait) clearTimeout(R.maxWait);
    R.maxWait = setTimeout(() => {
        try { if (S.recovery === R && R.resume) abortResume("chờ đủ người quá lâu"); } catch (e) { logErr("resumeTimer", e); }
    }, AUTO_PLAN.resumeMaxWaitMs);
}

// Số người đổi trong lúc giữ trận: chọn lại chế độ theo bảng. Cùng map thì đổi chế độ + đưa người dư về Spec; khác map thì bỏ trận.
function reconsiderResumeMode() {
    const key = recoveryTargetMode();
    if (key && mapKeyOf(key) !== S.mapKey) { abortResume(`cần đổi map ${S.mapKey}→${mapKeyOf(key)}`); return false; }
    if (key && key !== S.mode) {
        recLog(`chế độ ${S.mode}→${key} (cùng map ${S.mapKey})`);
        announceModeChange(S.mode, key);
        S.mode = key;
        trimToMax(limits(key).max, S.lineup.win || 1);
    }
    return true;
}

function resumeMatch() {
    const R = S.recovery;
    if (!R || !R.resume) return false;
    if (!running()) { abortResume("trận đã bị dừng"); return false; }
    if (inTransit() || !lineupValid(S.mode)) { watchTransit(); return false; }       // chưa xác nhận xong / chưa hợp lệ → chờ, evaluate sẽ gọi lại
    const sc = room.getScores(), s0 = R.snap;
    if (s0 && sc && (sc.red < s0.red || sc.blue < s0.blue || sc.time + 1 < s0.time)) {
        recLog(`LỖI: trạng thái trận không khớp (trước ${s0.red}-${s0.blue} ${fmtClock(s0.time)}, hiện ${sc.red}-${sc.blue} ${fmtClock(sc.time)})`);
        abortResume("trạng thái trận đã bị thay đổi");
        return false;
    }
    recLog(`TIẾP TỤC trận cũ: ${sc ? `tỉ số ${sc.red}-${sc.blue}, thời gian ${fmtClock(sc.time)}, ` : ""}đội ${tm(1).length}-${tm(2).length}, chế độ ${S.mode}, ${R.leavers.length} người đã thoát`);
    cancelPick();
    finishRecovery("tiếp tục trận cũ");               // xóa S.recovery TRƯỚC khi bỏ pause (để guardUnpause không chặn)
    try { room.pauseGame(false); } catch (e) { logErr("unpause", e); }
    return true;
}

function abortResume(why) {                            // không thể giữ trận → dừng và dựng trận mới (đường cũ)
    const R = S.recovery;
    if (!R || !R.resume) return;
    recLog(`không thể tiếp tục trận cũ: ${why} → dừng trận, dựng trận mới`);
    R.resume = false;
    if (R.maxWait) { clearTimeout(R.maxWait); R.maxWait = null; }
    cancelPick();
    if (running() && R.stopTries === 0) { R.stopTries = 1; try { room.stopGame(); } catch (e) { logErr("abortResume", e); } }
    if (!running() && !S.planTimer) schedulePlan(AUTO_PLAN.recoveryDelayMs);
    armRecoveryWatchdog();
}

// Gọi ở dòng ĐẦU onGameUnpause: không cho ai bỏ tạm dừng khi đội hình chưa đủ. true = đã chặn.
function guardUnpause() {
    if (S.recovery && S.recovery.resume && S.picking.active) { try { room.pauseGame(true); } catch (e) {} return true; }
    return false;
}

// Kiểu De Paul (1v1–4v4): người thoát giữa trận → bù người đầu hàng chờ NGAY, trận chạy tiếp, giữ tỉ số.
// Trả false nếu phải đổi chế độ/map hoặc không bù được → rơi về đường dừng trận & dựng lại.
function tryRefillInPlace(pl, p) {
    if (!running() || S.recovery || S.pen || S.result || !S.match) return false;
    if (S.mode === "practice") return false;
    const target = pickMode();
    if (!target || target !== S.mode) return false;                  // số người đòi đổi chế độ/map → dựng lại
    syncQueue(); pruneLineup();
    const lim = limits(S.mode);
    const cnt = t => tm(t).filter(id => id !== pl.id && valid(id)).length;
    const n = { 1: cnt(1), 2: cnt(2) };
    const heads = S.queue.filter(id => id !== pl.id && valid(id) && room.getPlayer(id).team === 0);
    const placed = [];

    while (heads.length && n[1] !== n[2]) {                          // bù cho đội thiếu
        const t = n[1] < n[2] ? 1 : 2;
        if (n[t] >= lim.max) break;
        const id = heads.shift();
        room.setPlayerTeam(id, t);
        placed.push([id, t]); n[t]++;
    }
    if (n[1] !== n[2]) {                                             // hết người chờ → bỏ bớt 1 người đội thừa (như De Paul)
        const big = n[1] > n[2] ? 1 : 2;
        if (Math.min(n[1], n[2]) >= lim.min && dropLeastContributor(big, "hàng chờ trống")) n[big]--;
    }
    if (n[1] !== n[2] || n[1] < lim.min) return false;

    for (const [id, t] of placed) {
        S.lineup.order[t].push(id);
        S.queue = S.queue.filter(x => x !== id);
        const r = rec(id); if (r) { r.lastActive = Now(); r.afkWarned = false; }
    }
    for (const t of [1, 2]) if (S.lineup.cap[t] == null) {           // đội trưởng bị thoát → người pick sớm nhất / người vào sớm nhất
        const c = S.lineup.order[t].find(valid) ?? firstJoined(tm(t).filter(id => id !== pl.id && valid(id)));
        if (c != null) { S.lineup.cap[t] = c; say(`👑 ${rec(c).name} là đội trưởng đội ${tname(t)}.`, C.warn, "bold"); }
    }
    const names = placed.map(([id]) => nameOf(id)).join(", ");
    recLog(`bù tại chỗ ${S.mode}: ${p.name} thoát, vào thay: ${names || "-"}; đội ${n[1]}-${n[2]}, spec ${S.queue.length}`);
    say(`🔁 ${p.name} (${tname(pl.team)}) thoát${names ? ` — ${names} vào thay` : ""}, trận tiếp tục!`, C.info, "bold");
    scheduleBalance();
    return true;
}
function onTeamMemberLeftMidMatch(pl, p) {
   syncQueue();
   const queued = S.queue.filter(id => id !== pl.id && valid(id) && room.getPlayer(id).team === 0).length;
   const wantPick = S.mode === "5v5" && AUTO_PLAN.resumeMatch && queued >= 2;   // 5v5 + hàng chờ từ 2 người → pause cho đội trưởng pick
   if (!wantPick && tryRefillInPlace(pl, p)) return;
   const R = ensureRecovery();
    R.leavers.push({ id: pl.id, name: p.name, team: pl.team });
    recLog(`${p.name} (#${pl.id}, ${tname(pl.team)}) thoát; chế độ ${S.mode}; còn đội ${tm(1).length}-${tm(2).length}, spec ${S.queue.length}`);
    if (running() && !R.resume && R.stopTries === 0) {
        if (canResumeInPlace()) { startResumeRecovery(R); return; }          // giữ trận: tạm dừng, không stopGame
        R.stopTries = 1; try { room.stopGame(); } catch (e) { logErr("recoveryStop", e); }
    }
    if (R.resume) return;
    if (!running() && !S.planTimer && !S.picking.active) schedulePlan(AUTO_PLAN.recoveryDelayMs);
    armRecoveryWatchdog();
}

function armRecoveryWatchdog() {
    const R = S.recovery;
    if (!R || R.wd || R.resume) return;
    R.wd = setTimeout(() => {
        try {
            if (S.recovery !== R || R.resume) return;
            R.wd = null;
            if (running()) {
                if (R.stopTries < 2) { R.stopTries = 2; recLog("trận chưa dừng — thử dừng lại"); try { room.stopGame(); } catch (e) {} armRecoveryWatchdog(); }
else { recLog("LỖI: không dừng được trận — hủy phục hồi"); S.recovery = null; }
            } else if (!S.planTimer && !S.picking.active) schedulePlan(0);
        } catch (e) { logErr("recoveryWatchdog", e); }
    }, AUTO_PLAN.watchdogMs);
}


/* ═══════════════════ 9. THỐNG KÊ ═══════════════════ */

function stat(auth, name) {
const s = Store.data.stats;
if (!s[auth]) s[auth] = { name, games: 0, wins: 0, losses: 0, goals: 0, assists: 0, og: 0, cs: 0, rating: 1000, leaves: 0, last: Now(), milestones: [] };
if (!s[auth].milestones) s[auth].milestones = [];
s[auth].name = name; s[auth].last = Now();
return s[auth];
}

function rankOf(auth) {
const all = Object.entries(Store.data.stats).filter(([, v]) => v.games > 0).sort((a, b) => b[1].rating - a[1].rating);
const i = all.findIndex(([k]) => k === auth);
return i < 0 ? null : i + 1;
}

function timeGreeting() {
const h = new Date().getHours();
if (h < 5) return "🌙 Khuya rồi vẫn còn chiến, máu quá!";
if (h < 11) return "☀ Chào buổi sáng, chúc một ngày đá bóng vui vẻ!";
if (h < 14) return "🍜 Giờ trưa rồi, tranh thủ đá vài trận cho sảng khoái!";
if (h < 18) return "🌤️ Chiều rồi, vào sân giải trí thôi nào!";
return "🌆 Buổi tối vui vẻ, chúc trận đấu hấp dẫn!";
}

const MATCH_SLOGANS = [
"🔥 Trận đấu đỉnh cao sắp bắt đầu!", "⚡ Sẵn sàng cho 1 trận cầu bùng nổ!",
"🏆 Ai sẽ là người hùng trận này?", "💪 Cống hiến hết mình vì màu áo!",
"🎯 Một cơ hội, một khoảnh khắc để toả sáng!", "🚀 Bùng nổ ngay từ tiếng còi khai cuộc!",
];


const RULE_REMINDERS = [
"📜 Nhắc nhẹ: cấm chửi tục / spam chat — vi phạm sẽ tự bị mute.",
"📜 Nhắc nhẹ: 1 IP chỉ được 1 slot — chống mở nhiều tab gây bug đội hình.",
"📜 Nhắc nhẹ: AFK quá lâu sẽ tự bị đẩy ra khán đài, gõ !back để quay lại.",
"📜 Nhắc nhẹ: gõ !help để xem toàn bộ lệnh, !luat để xem nội quy đầy đủ.",
];
const CMD_TIPS = [
  "💡 Gõ !help để xem toàn bộ lệnh  |  !luat để xem nội quy phòng.",
  "💡 !daily — điểm danh mỗi ngày nhận xu, chuỗi càng dài thưởng càng nhiều!",
  "💡 !shop xem cửa hàng màu chat/danh hiệu/rương.",
  "💡 !stats xem thống kê của bạn  |  !top xem bảng xếp hạng Elo  |  !ranks xem các mức rank.",
  "💡 !betwin <do|xanh> <xu> — làm khán giả cược đội thắng, thắng nhận xu!",
  "💡 !kho xem rương báu  |  !openbox mở rương  |  !nhapcode <mã> nhập giftcode.",
  "💡 Cần nghỉ? Gõ !afk — quay lại bằng !back để vào hàng chờ.",
  "💡 !queue xem hàng chờ & đội hình  |  !kyluc xem kỷ lục cá nhân.",
];
let tipIdx = 0;
const ROOM_MILESTONES = [50, 100, 250, 500, 1000, 2500, 5000];
const ROOM_MILESTONE_BONUS = 200;

function announceMatchRecap(m, res) {
    say(`🏆 ${res.red} - ${res.blue} — ${res.winner === 1 ? "ĐỎ" : "XANH"} thắng${res.pen ? " luân lưu " + res.pen.join("-") : ""}`, C.goal, "bold", 1);
    const scorers = Object.values(m.pl).filter(x => x.g > 0).sort((a, b) => b.g - a.g);
    const assisters = Object.values(m.pl).filter(x => x.a > 0).sort((a, b) => b.a - a.a);
    const owns = Object.values(m.pl).filter(x => x.og > 0);
    const gks = Object.values(m.pl).filter(x => (x.sv || 0) >= 3).sort((a, b) => b.sv - a.sv);
    const p1 = [];
    if (scorers.length) p1.push("⚽ " + scorers.map(x => `${x.name} x${x.g}`).join(", "));
    if (assisters.length) p1.push("🎯 " + assisters.map(x => `${x.name} x${x.a}`).join(", "));
    if (owns.length) p1.push("🤦 " + owns.map(x => `${x.name} x${x.og}`).join(", "));
    if (p1.length) wrap(p1.join("  |  ")).forEach(l => say(l, C.info));
    const p2 = [];
    if (scorers.length) p2.push(`🌟 MOTM: ${scorers[0].name} (${scorers[0].g} bàn)`);
    if (gks.length) p2.push(`🧤 GK: ${gks[0].name} (${gks[0].sv})`);
    if (p2.length) say(p2.join("  |  "), C.super, "bold");
}
// ═══════════════════ HỆ THỐNG RANK ELO ═══════════════════
const TOP_MIN_ELO = 1300; // phải đạt mức này mới được xét Top 1/2/3

const RANK_SYSTEM = [
  { weight: 1, reqElo: 0,    title: "🌱Amateur",      color: 0xA0A0A0 },
  { weight: 2, reqElo: 1050, title: "🥉Semi-Pro",     color: 0xFFFFFF },
  { weight: 3, reqElo: 1100, title: "🥈Professional", color: 0x87CEEB },
  { weight: 4, reqElo: 1150, title: "🥇World Class",  color: 0xF1C40F },
  { weight: 5, reqElo: 1225, title: "💎Legend",       color: 0x00FFFF },
  { weight: 6, reqElo: 1300, title: "🏆Champion",     color: 0xFF4500 },
  { weight: 7, top: 3,       title: "👑Untouchable",  color: 0xFF0000 },
  { weight: 8, top: 2,       title: "🌌Beyond Real",  color: 0xFF5050 },
{ weight: 9, top: 1,       title: "THE CONQUEROR",       color: 0xFF3030 },
];
const SEASON = 2;   // reset Elo sang mùa mới thì tăng số này để mọi người nhận quà lại

const RANK_REWARDS = {
  "🌱Amateur":      { coins: 1000 },
  "🥉Semi-Pro":     { coins: 3000,  boxes: { common: 1 } },
  "🥈Professional": { coins: 8000,  boxes: { rare: 1 } },
  "🥇World Class":  { coins: 20000, ruby: 1, fx: ["classic", "emoji", "rainbow"] },
  "💎Legend":       { coins: 40000, ruby: 2, boxes: { epic: 1 }, fx: ["ez", "pulse", "mexican", "giant"] },
  "🏆Champion":     { coins: 70000, ruby: 3, boxes: { legendary: 1 }, fx: ["sword", "dance", "spin", "freeze", "quake"] },
  "👑Untouchable":  { fx: ["ring"] },
  "🌌Beyond Real":  { fx: ["ring"] },
  "THE CONQUEROR":  { fx: ["ring"] },
};
// Nhận quà các mốc rank đã đạt mà chưa nhận trong mùa này (an toàn khi gọi nhiều lần)
function claimRankRewards(p, e, rank) {
  e.rankClaimed = e.rankClaimed || [];
  const w = getRankInfo(rank.title).weight;
  for (const r of RANK_SYSTEM) {
    const rw = RANK_REWARDS[r.title];
    const key = SEASON + ":" + r.title;
    if (!rw || r.weight > w || e.rankClaimed.includes(key)) continue;
    e.rankClaimed.push(key);
    e.coins += rw.coins || 0;
    e.ruby += rw.ruby || 0;
    for (const [bt, n] of Object.entries(rw.boxes || {})) e.boxes[bt] = (e.boxes[bt] || 0) + n;
    if (rw.title) grantTitle(e, rw.title);

    const parts = [];
    if (rw.coins) parts.push(rw.coins.toLocaleString() + " xu");
    if (rw.ruby) parts.push(rw.ruby + " ruby");
    for (const [bt, n] of Object.entries(rw.boxes || {})) parts.push(n + " rương " + bt);
    if (rw.title) parts.push("danh hiệu [" + rw.title + "]");
    if (rw.fx) parts.push("hiệu ứng ghi bàn " + rw.fx.join("/").toUpperCase());
    if (!parts.length) continue;
    tell(p.id, `🎁 Quà rank [${r.title}]: ${parts.join(", ")}`, C.super, "bold", 1);
  }
  Store.touch();
}

// Danh sách hiệu ứng ghi bàn người chơi được dùng (cộng dồn theo rank hiện tại)
function fxStylesFor(p) {
  const e = econ(p.auth, p.name), w = getRankInfo(e.eloRank).weight, out = [];
  for (const r of RANK_SYSTEM) {
    const rw = RANK_REWARDS[r.title];
    if (rw && rw.fx && r.weight <= w) out.push(...rw.fx);
  }
  return out;
}
function getRankInfo(title) { return RANK_SYSTEM.find(r => r.title === title) || { weight: 0, color: 0xA0A0A0 }; }

function computeRank(auth, ordered) {
  const s = Store.data.stats[auth];
  if (!s || s.games === 0) return null;
  ordered = ordered || Object.entries(Store.data.stats)
    .filter(([, v]) => v.games > 0).sort((a, b) => b[1].rating - a[1].rating);
  const idx = ordered.findIndex(([k]) => k === auth);
  if (idx >= 0 && idx < 3 && s.rating >= TOP_MIN_ELO) return RANK_SYSTEM.find(r => r.top === idx + 1);
  let cur = RANK_SYSTEM[0];
  for (const r of RANK_SYSTEM) if (r.reqElo !== undefined && s.rating >= r.reqElo) cur = r;
  return cur;
}

function checkAndApplyEloRank(p, silent = false, ordered) {
  const rank = computeRank(p.auth, ordered);
  if (!rank) return;
  const e = econ(p.auth, p.name);

  claimRankRewards(p, e, rank);          // nhận quà trước (idempotent)

  if (e.eloRank === rank.title) {
    if (e.eloColor !== rank.color) { e.eloColor = rank.color; Store.touch(); }
    return;
  }
  const oldW = getRankInfo(e.eloRank).weight, first = !e.eloRank;
  e.eloRank = rank.title;
  e.eloColor = rank.color;
  Store.touch();
  if (silent) return;
  if (first) tell(p.id, `🎖️ Rank của bạn: [${rank.title}]`, C.info, "bold");
  else if (rank.weight > oldW) say(`🎉 THĂNG HẠNG! ${p.name} vừa lên Rank: [${rank.title}]! 🔥`, C.super, "bold", 1);
  else tell(p.id, `⚠️ Rank của bạn đã thay đổi: [${rank.title}]`, C.warn);
}
function refreshAllRanks() {
  const ordered = Object.entries(Store.data.stats)
    .filter(([, v]) => v.games > 0).sort((a, b) => b[1].rating - a[1].rating);
  for (const p of S.players.values()) checkAndApplyEloRank(p, false, ordered);
}
const ELO_BONUS = {
  MOTM: 3,          // Elo thưởng cho MOTM (kể cả khi đội thua)
  STREAK_MIN: 2,    // từ chuỗi thắng này trở đi mới có thưởng
  STREAK_PER: 1,    // mỗi bậc chuỗi cộng thêm bấy nhiêu Elo
  STREAK_CAP: 5,    // thưởng chuỗi tối đa
  ANNOUNCE_AT: 3,   // từ chuỗi này trở đi mới thông báo cả phòng
};

function commitMatch() {
const m = S.match, res = S.result; S.match = null;
if (!m) return;
if (res) announceMatchRecap(m, res);
if (!m.rated) return;

for (const [auth, x] of Object.entries(m.pl)) {
const s = stat(auth, x.name); s.goals += x.g; s.assists += x.a; s.og += x.og;
}
Store.touch();

if (!res || res.time < CONFIG.MIN_RATED_SECONDS) return;

let motmAuth = null, motmGoals = 0;
{
  const mo = motmOf(m, res);
  if (mo) {
    motmAuth = mo[0]; motmGoals = m.pl[motmAuth].g || 0;
    say(`🌟 MOTM: ${m.pl[motmAuth].name} (${mo[1].toFixed(1)} điểm)`, C.super, "bold");
    const e = econ(motmAuth, m.pl[motmAuth].name);
    e.coins += 300; Store.touch();
    const mp = list().find(p => rec(p.id).auth === motmAuth);
    if (mp) tell(mp.id, "🌟 Bạn là MOTM! +300 xu thưởng.", C.super, "bold", 1);
  }
}
const done = p => m.quit && m.quit[rec(p.id).auth];
const team = { 1: list().filter(p => p.team === 1 && !done(p)), 2: list().filter(p => p.team === 2 && !done(p)) };
if (!team[1].length || !team[2].length) return;
const avg = t => team[t].reduce((a, p) => a + stat(rec(p.id).auth, p.name).rating, 0) / team[t].length;
const ra = avg(1), rb = avg(2), K = res.pen ? 12 : 24;
const e1 = 1 / (1 + Math.pow(10, (rb - ra) / 400));
const win = res.winner, eWin = win === 1 ? e1 : 1 - e1;
const gain = Math.max(1, Math.round(K * (1 - eWin)));

const streakUp = {}, streakEnd = {}, lines = [];
for (const t of [1, 2]) for (const p of team[t]) {
if (isLowSub(m, rec(p.id).auth)) { tell(p.id, "ℹ️ Bạn vào thay giữa trận và đóng góp ít — không cộng/trừ Elo trận này.", C.info); continue; }
const s = stat(rec(p.id).auth, p.name), won = t === win;
const pr = rec(p.id), ec = econ(pr.auth, pr.name);
let delta = won ? gain : -gain;
if (!won && ec.eloGuard) { delta = 0; ec.eloGuard = false; tell(p.id, "🛡️ Bảo hộ ELO đã dùng: không bị trừ điểm!", C.info, "bold"); }

// ── Chuỗi thắng + thưởng Elo ──
let bonus = 0, tags = "";
if (won) {
  s.streak = (s.streak || 0) + 1;
  if (s.streak > (s.bestStreak || 0)) s.bestStreak = s.streak;
  if (s.streak >= ELO_BONUS.STREAK_MIN) {
    bonus += Math.min((s.streak - ELO_BONUS.STREAK_MIN + 1) * ELO_BONUS.STREAK_PER, ELO_BONUS.STREAK_CAP);
    tags += `🔥${s.streak}`;
  }
  if (s.streak >= ELO_BONUS.ANNOUNCE_AT)
    (streakUp[s.streak] = streakUp[s.streak] || []).push(p.name);
} else {
  if ((s.streak || 0) >= ELO_BONUS.ANNOUNCE_AT)
    (streakEnd[s.streak] = streakEnd[s.streak] || []).push(p.name);
  s.streak = 0;
}
if (motmAuth && motmGoals > 0 && pr.auth === motmAuth) { bonus += ELO_BONUS.MOTM; tags += "🌟"; }
delta += bonus;

s.games++; won ? s.wins++ : s.losses++;
s.rating = Math.max(0, s.rating + delta);
const conceded = t === 1 ? res.blue : res.red;
if (won && conceded === 0) s.cs++;
lines.push(`${p.name}${tags ? " " + tags : ""} ${delta >= 0 ? "+" : ""}${delta} (${s.rating})`);
if (won) {
const mult = (ec.x2GamesLeft > 0 ? 2 : 1);
if (ec.x2GamesLeft > 0) ec.x2GamesLeft--;
const earn = Math.floor(ECO.WIN_COINS * mult);
ec.coins += earn;
tell(p.id, `🎁 Thắng trận: +${earn.toLocaleString()} xu`, C.ok);
}
}

say("📈 Elo: " + lines.join(" | "), C.info, "small");
for (const [n, names] of Object.entries(streakUp))
  say(`🔥 ${names.join(", ")} đang có chuỗi ${n} trận thắng liên tiếp!`, C.super, "bold", 1);
for (const [n, names] of Object.entries(streakEnd))
  say(`💔 Chuỗi ${n} trận thắng của ${names.join(", ")} đã bị chấm dứt!`, C.warn, "bold");
refreshAllRanks(); // cập nhật rank cho mọi người online sau khi Elo đã tính xong
resolveWinnerBets(win);

Store.data.totalMatches++;
for (const threshold of ROOM_MILESTONES) {
if (Store.data.totalMatches >= threshold && !Store.data.matchMilestones.includes(threshold)) {
Store.data.matchMilestones.push(threshold);
const online = [...S.players.values()];
for (const op of online) econ(op.auth, op.name).coins += ROOM_MILESTONE_BONUS;
say(`🎊 PHÒNG VỪA CÁN MỐC ${threshold} TRẬN ĐẤU! Mỗi người đang online nhận ${ROOM_MILESTONE_BONUS} xu chúc mừng! 🎉`, C.super, "bold", 2);
break;
}
}

Store.save();
}

/* ═══════════════════ 10. CHAT: LỌC & CHỐNG SPAM ═══════════════════ */

const BAD_EXACT = new Set(["dm", "dmm", "dcm", "dcmm", "vcl", "vkl", "clm", "cmm", "dmcs", "loz", "fuck", "fck", "shit", "bitch"]);
const BAD_RAW = new Set(["địt", "đụ", "lồn", "cặc", "đĩ", "đéo"]);
const BAD_SQUASH = ["ditme", "ditmemay", "concac", "thangcho", "nigger", "nigga", "faggot", "fuckyou"];
const RE_LINK = /(https?:\/\/|www\.|discord\.gg|\.(com|net|gg|io|vn|xyz|tk)\b)/i;

function filterChat(p, msg) {
const t = Now();
if (S.chatLocked && p.role < 1) return "Chat đang bị khoá bởi quản trị.";
const mute = getMute(p);
if (mute) return `Bạn đang bị mute${mute.until ? " (còn " + fmtDur(mute.until - t) + ")" : ""}: ${mute.reason}`;
if (p.role >= 1) return null;

if (S.slowMs && t - p.lastChat < S.slowMs) return `Chế độ chậm: chờ ${Math.ceil((S.slowMs - (t - p.lastChat)) / 1000)}s.`;
if (t - p.lastChat < CONFIG.CHAT_MIN_GAP_MS) return violation(p, "Gõ quá nhanh");

p.msgTimes = p.msgTimes.filter(x => t - x < CONFIG.CHAT_WINDOW_MS);
if (p.msgTimes.length >= CONFIG.CHAT_MAX_IN_WINDOW) return violation(p, "Spam tin nhắn");

const n = norm(msg);
if (msg.length > CONFIG.MAX_MSG_LEN) return violation(p, "Tin nhắn quá dài");
if (n === p.lastNorm && t - p.lastChat < CONFIG.DUP_WINDOW_MS) return violation(p, "Lặp lại tin nhắn");
if (/(.)\1{6,}/u.test(msg)) return violation(p, "Lặp ký tự quá nhiều");

const letters = msg.replace(/[^\p{L}]/gu, "");
if (letters.length >= 8 && letters.replace(/[^\p{Lu}]/gu, "").length / letters.length > 0.7) return violation(p, "Đừng VIẾT HOA toàn bộ");
if (RE_LINK.test(msg)) return violation(p, "Không được gửi link/quảng cáo");

const tokens = n.split(/[^a-z0-9]+/).filter(Boolean);
const rawTokens = msg.toLowerCase().split(/[^\p{L}\p{N}]+/u).filter(Boolean);
const squash = n.replace(/[^a-z0-9]/g, "");

if (tokens.some(w => BAD_EXACT.has(w)) || rawTokens.some(w => BAD_RAW.has(w)) || BAD_SQUASH.some(w => squash.includes(w)))
return violation(p, "Ngôn từ không phù hợp");

p.msgTimes.push(t); p.lastChat = t; p.lastNorm = n;
return null;
}

function violation(p, why) {
const t = Now();
if (t - p.lastStrike > 60e3) p.strikes = 0;
p.strikes++; p.lastStrike = t;
if (p.strikes >= CONFIG.STRIKES_TO_MUTE) {
p.strikes = 0; p.autoMutes++;
S.mutes.set(p.auth, { until: t + CONFIG.AUTO_MUTE_MS, reason: "Vi phạm chat liên tục", by: "AutoMod" });
say(`🔇 ${p.name} bị AutoMod mute ${fmtDur(CONFIG.AUTO_MUTE_MS)}.`, C.warn);
if (p.autoMutes >= CONFIG.AUTO_MUTES_TO_KICK) {
addBan(p, 10, "Spam chat lặp lại", "AutoMod");
}
return why + " — bạn đã bị mute.";
}
return `${why} (cảnh cáo ${p.strikes}/${CONFIG.STRIKES_TO_MUTE})`;
}

/* ═══════════════════ 10.5. KINH TẾ: XU · RUBY · SHOP · CƯỢC TRẬN · MINIGAME ═══════════════════ */

const ECO = {
START_COINS: 300, DAILY_BASE: 200, DAILY_STREAK_BONUS: 20, DAILY_STREAK_CAP: 300,
WIN_COINS: 500, BET_HOUSE_EDGE: 0.92, BET_MAX_MULT: 30,
RUBY_BUY_RATE: 15000, MAX_STAKE: 5_000_000,
};

let rubySellRate = 8000;

function isPosInt(n, max = ECO.MAX_STAKE) { return Number.isSafeInteger(n) && n > 0 && n <= max; }

function econ(auth, name) {
const e = Store.data.econ;
if (!e[auth]) e[auth] = {
name, coins: ECO.START_COINS, ruby: 0, chatColor: null, title: null, titles: [],boxes: { common: 0, rare: 0, epic: 0, legendary: 0 }, x2GamesLeft: 0,boldChat: false, eloGuard: false,lastDaily: "", dailyStreak: 0,
};
e[auth].name = name;
return e[auth];
}

function grantTitle(e, title) { if (!title) return; if (!e.titles.includes(title)) e.titles.push(title); e.title = title; }

function teamAvgRating(team) {
const ids = teamIds(team); if (!ids.length) return 1000;
let sum = 0; for (const id of ids) { const p = rec(id); sum += stat(p.auth, p.name).rating; }
return sum / ids.length;
}

function winProb(a, b) { return 1 / (1 + Math.pow(10, (b - a) / 400)); }
function betMultiplier(prob) { return Math.round(Math.min(ECO.BET_HOUSE_EDGE / Math.max(prob, 0.02), ECO.BET_MAX_MULT) * 10) / 10; }

const SHOP_ITEMS = {
101: { icon: "🟢", name: "Màu Xanh Lục Bảo", cost: 3000, type: "color", color: 0x2ECC71 },
102: { icon: "🔵", name: "Màu Xanh Dương VIP", cost: 5000, type: "color", color: 0x3498DB },
103: { icon: "🟣", name: "Màu Tím Huyền Bí", cost: 10000, type: "color", color: 0x9B59B6 },
104: { icon: "🟡", name: "Màu Hoàng Kim", cost: 25000, type: "color", color: 0xF1C40F },
105: { icon: "🟠", name: "Màu Cam Lửa", cost: 50000, type: "color", color: 0xE67E22 },
106: { icon: "🔴", name: "Màu Đỏ Quyền Lực", cost: 80000, type: "color", color: 0xE74C3C },
107: { icon: "🌸", name: "Màu Hồng Neon", cost: 120000, type: "color", color: 0xFF69B4 },
108: { icon: "🩵", name: "Màu Xanh Băng Tuyết", cost: 200000, type: "color", color: 0x00FFFF },
109: { icon: "🔥", name: "Màu Rồng Lửa Ultimate", cost: 350000, type: "color", color: 0xFF4500 },
110: { icon: "🌌", name: "Màu Vũ Trụ Thần Thoại", cost: 500000, type: "color", color: 0xDA70D6 },
// ── DANH HIỆU THƯỜNG (tối đa 500.000 xu) ──
216: { icon: "🍀", name: "Danh hiệu [🍀MAY MẮN]", cost: 2000, type: "title", value: "🍀MAY MẮN" },
219: { icon: "🍕", name: "Danh hiệu [🍕FOODIE]", cost: 5000, type: "title", value: "🍕FOODIE" },
212: { icon: "🎮", name: "Danh hiệu [🎮PRO GAMER]", cost: 30000, type: "title", value: "🎮PRO GAMER" },
217: { icon: "🦁", name: "Danh hiệu [🦁SƯ TỬ SÂN CỎ]", cost: 45000, type: "title", value: "🦁SƯ TỬ SÂN CỎ" },
218: { icon: "🎸", name: "Danh hiệu [🎸ROCKSTAR]", cost: 60000, type: "title", value: "🎸ROCKSTAR" },
213: { icon: "🥷", name: "Danh hiệu [🥷NINJA SÂN CỎ]", cost: 80000, type: "title", value: "🥷NINJA SÂN CỎ" },
214: { icon: "🎪", name: "Danh hiệu [🎪PHÙ THỦY BÓNG ĐÁ]", cost: 120000, type: "title", value: "🎪PHÙ THỦY BÓNG ĐÁ" },
220: { icon: "🌟", name: "Danh hiệu [🌟NGÔI SAO SÁNG]", cost: 160000, type: "title", value: "🌟NGÔI SAO SÁNG" },
215: { icon: "🚀", name: "Danh hiệu [🚀TÊN LỬA]", cost: 200000, type: "title", value: "🚀TÊN LỬA" },
221: { icon: "🧙", name: "Danh hiệu [🧙PHÁP SƯ]", cost: 240000, type: "title", value: "🧙PHÁP SƯ" },
203: { icon: "⚡", name: "Danh hiệu [⚡ISHOWSPEED]", cost: 280000, type: "title", value: "⚡ISHOWSPEED" },
204: { icon: "🎯", name: "Danh hiệu [🎯GOLDEN BOOT]", cost: 320000, type: "title", value: "🎯GOLDEN BOOT" },
205: { icon: "🔥", name: "Danh hiệu [🔥SÁT THỦ VÒNG CẤM]", cost: 360000, type: "title", value: "🔥SÁT THỦ VÒNG CẤM" },
222: { icon: "🎼", name: "Danh hiệu [🎼The Maestro]", cost: 400000, type: "title", value: "🎼The Maestro" },
223: { icon: "🎻", name: "Danh hiệu [🎻THE VIRTUOSO]", cost: 430000, type: "title", value: "🎻THE VIRTUOSO" },
225: { icon: "🔟", name: "Danh hiệu [🔟La Pulga]", cost: 460000, type: "title", value: "🔟La Pulga" },
226: { icon: "7️⃣", name: "Danh hiệu [7️⃣El Bicho]", cost: 460000, type: "title", value: "7️⃣El Bicho" },
224: { icon: "🕵️", name: "Danh hiệu [🕵️THE ENIGMA]", cost: 500000, type: "title", value: "🕵️THE ENIGMA" },

// ── DANH HIỆU "CHƠI CHO VUI" (giữ nguyên giá cũ) ──
202: { icon: "👟", name: "Danh hiệu [👟NEYMAR]", cost: 99999999999, type: "title", value: "👟NEYMAR" },
206: { icon: "🛡️", name: "Danh hiệu [🛡️BUFFON]", cost: 8000000000, type: "title", value: "🛡️BUFFON" },
207: { icon: "🏅", name: "Danh hiệu [🏅Ballon d'Or]", cost: 1200000000, type: "title", value: "🏅Ballon d'Or🏅🏅🏅" },
208: { icon: "👑", name: "Danh hiệu [👑HUYỀN THOẠI]", cost: 25000000000, type: "title", value: "👑HUYỀN_THOẠI" },
209: { icon: "💎", name: "Danh hiệu [💎CR7]", cost: 99999999999, type: "title", value: "💎CR7" },
210: { icon: "🐐", name: "Danh hiệu [CHAN ĐÊ-GOAT]", cost: 99999999999, type: "title", value: "🐐CHAN ĐÊ-GOAT" },
211: { icon: "🆘", name: "Danh hiệu [🆘❌🆘SA TỊ]", cost: 99999999999, type: "title", value: "🆘❌🆘SA TỊ" },501: { icon: "🎒", name: "KIT TÂN THỦ (màu+hiệu+rương)", cost: 5000, type: "kit", color: 0x2ECC71, title: "🎒TÂN THỦ", boxes: { common: 1 } },
502: { icon: "🛡", name: "KIT CHIẾN BINH (màu+hiệu+2 rương hiếm)", cost: 50000, type: "kit", color: 0x3498DB, title: "🛡️CHIẾN BINH", boxes: { rare: 2 } },
503: { icon: "💼", name: "KIT ĐẠI GIA (màu+hiệu+rương sử thi)", cost: 500000, type: "kit", color: 0xF1C40F, title: "💼ĐẠI GIA", boxes: { epic: 1 } },
504: { icon: "🐲", name: "KIT HUYỀN THOẠI (màu+hiệu+rương huyền thoại)", cost: 5000000, type: "kit", color: 0xFF4500, title: "🐲HUYỀN THOẠI", boxes: { legendary: 1 } },
505: { icon: "🌌", name: "KIT TỐI THƯỢNG (màu+hiệu+2 rương cao cấp)", cost: 50000000, type: "kit", color: 0xDA70D6, title: "🌌TỐI THƯỢNG", boxes: { epic: 1, legendary: 1 } },
302: { icon: "🛡️", name: "Bảo Hộ ELO Không Trừ Điểm", cost: 50000, type: "perk_elo_guard" },
303: { icon: "⚡", name: "X2 Xu Thưởng Thắng Trận (3 trận)", cost: 100000, type: "perk_x2_elo" },
304: { icon: "✨", name: "Chat Chữ Đậm Bold Cao Cấp", cost: 150000, type: "perk_bold_chat" },
405: { icon: "🎁", name: "Rương Báu Thường x3", cost: 10000, type: "box_pack", count: 3, boxType: "common" },
406: { icon: "💜", name: "Rương Báu Sử Thuyết x1", cost: 50000, type: "box_pack", count: 1, boxType: "epic" },
407: { icon: "💛", name: "Rương Báu Huyền Thoại x1", cost: 200000, type: "box_pack", count: 1, boxType: "legendary" },
};

const VIP_SHOP_ITEMS = {
401: { icon: "🎫", name: "Thẻ VIP 1 NGÀY", cost: 300000, type: "vip", tier: 1, tag: "🎫VIP_NGÀY", durationMs: 86400000 },
402: { icon: "⭐", name: "Thẻ VIP 1 TUẦN", cost: 1000000, type: "vip", tier: 2, tag: "⭐VIP_TUẦN", durationMs: 7 * 86400000 },
403: { icon: "💎", name: "Thẻ VIP 1 THÁNG", cost: 5000000, type: "vip", tier: 3, tag: "💎VIP_THÁNG", durationMs: 30 * 86400000 },
404: { icon: "👑", name: "Vé VIP VĨNH VIỄN TỐI CAO", cost: 20000000, type: "vip", tier: 4, tag: "👑SUPER_VIP", durationMs: 0 },
};

const RUBY_ITEMS = {
1: { icon: "🔴", name: "Màu chat Ruby Đỏ Máu", cost: 10, type: "color", color: 0x8B0000 },
2: { icon: "💎", name: "Danh hiệu [RUBY LORD]", cost: 20, type: "title", value: "💎RUBY LORD" },
3: { icon: "🔄", name: "Reset ELO âm về 1000", cost: 15, type: "elo_reset" },
4: { icon: "🖤", name: "Màu chat Đen Tuyền (Ruby)", cost: 8, type: "color", color: 0x000000 },
5: { icon: "🩷", name: "Màu chat Hồng Pastel (Ruby)", cost: 12, type: "color", color: 0xFFC0CB },
6: { icon: "🎖️", name: "Danh hiệu [🎖️CHIẾN THẦN]", cost: 25, type: "title", value: "🎖️CHIẾN THẦN" },
7: { icon: "🦅", name: "Danh hiệu [🦅ĐẠI BÀNG]", cost: 35, type: "title", value: "🦅ĐẠI BÀNG" },
};

function buyShopItem(p, id) {
const item = SHOP_ITEMS[id]; if (!item) return tell(p.id, "❌ Không có vật phẩm ID đó. Gõ !shop.", C.err);
const e = econ(p.auth, p.name);
if (e.coins < item.cost) return tell(p.id, `❌ Cần ${item.cost.toLocaleString()} xu.`, C.err);
e.coins -= item.cost;
switch (item.type) {
case "color": e.chatColor = item.color; break;
case "title": grantTitle(e, item.value); break;
case "perk_elo_guard": e.eloGuard = true; break;
case "perk_x2_elo": e.x2GamesLeft += 3; break;
case "perk_bold_chat": e.boldChat = true; break;
case "box_pack": e.boxes[item.boxType] = (e.boxes[item.boxType] || 0) + item.count; break;
case "kit":
e.chatColor = item.color; grantTitle(e, item.title);
for (const [bt, n] of Object.entries(item.boxes || {})) e.boxes[bt] = (e.boxes[bt] || 0) + n;
break;
}
Store.touch();
tell(p.id, `🎉 Mua thành công ${item.icon} ${item.name}!`, C.ok, "bold");
if (item.cost >= 1_000_000) say(`💸 ${p.name} vừa tậu ${item.icon} ${item.name} với giá ${item.cost.toLocaleString()} xu — quá chịu chơi! 🤑`, C.super, "bold", 1);
}

function buyVipItem(p, id) {
const item = VIP_SHOP_ITEMS[id]; if (!item) return tell(p.id, "❌ Không có gói VIP đó. Gõ !vipshop.", C.err);
const e = econ(p.auth, p.name);
if (e.coins < item.cost) return tell(p.id, `❌ Cần ${item.cost.toLocaleString()} xu.`, C.err);
e.coins -= item.cost; e.vipTier = item.tier;
e.vipExpire = item.durationMs > 0 ? Now() + item.durationMs : 0;
grantTitle(e, item.tag);
Store.touch();
say(`👑 ${p.name} vừa nâng cấp ${item.icon} ${item.name}!`, C.super, "bold", 1);
}

function buyRubyItem(p, id) {
const item = RUBY_ITEMS[id]; if (!item) return tell(p.id, "❌ Không có vật phẩm Ruby đó. Gõ !rubyshop.", C.err);
const e = econ(p.auth, p.name);
if (e.ruby < item.cost) return tell(p.id, `❌ Cần ${item.cost} 💎 ruby.`, C.err);
e.ruby -= item.cost;
switch (item.type) {
case "color": e.chatColor = item.color; break;
case "title": grantTitle(e, item.value); break;
case "elo_reset": { const s = stat(p.auth, p.name); if (s.rating < 1000) s.rating = 1000; break; }
}
Store.touch();
tell(p.id, `💎 Mua thành công ${item.icon} ${item.name} bằng Ruby!`, C.info, "bold");
}

const BOX_TABLE = {
common: [{ w: 60, type: "coins", min: 100, max: 400 }, { w: 30, type: "coins", min: 400, max: 900 }, { w: 10, type: "box", boxType: "rare", count: 1 }],
rare: [{ w: 50, type: "coins", min: 800, max: 2000 }, { w: 30, type: "box", boxType: "epic", count: 1 }, { w: 20, type: "ruby", amount: 1 }],
epic: [{ w: 50, type: "coins", min: 3000, max: 8000 }, { w: 30, type: "ruby", amount: 2 }, { w: 20, type: "box", boxType: "legendary", count: 1 }],
legendary: [{ w: 50, type: "coins", min: 15000, max: 40000 }, { w: 30, type: "ruby", amount: 5 }, { w: 20, type: "title", value: "👑HUYỀN THOẠI RƯƠNG" }],
};

function rollBox(type) {
const table = BOX_TABLE[type], tw = table.reduce((s, r) => s + r.w, 0);
let roll = Math.random() * tw;
for (const r of table) { if (roll < r.w) return r; roll -= r.w; }
return table[table.length - 1];
}

const CODE_POOL = [
  { label: "1.000 xu",  t: "coins", n: 1000 }, { label: "5.000 xu",  t: "coins", n: 5000 },
  { label: "20.000 xu", t: "coins", n: 20000 }, { label: "1 ruby", t: "ruby", n: 1 },
  { label: "5 ruby",    t: "ruby", n: 5 },     { label: "1x Rương Thường", t: "box", n: 1, b: "common" },
  { label: "1x Rương Sử Thi", t: "box", n: 1, b: "epic" },
];
function applyReward(e, r) {
  if (r.t === "coins") e.coins += r.n;
  else if (r.t === "ruby") e.ruby += r.n;
  else if (r.t === "box") e.boxes[r.b] = (e.boxes[r.b] || 0) + r.n;
}
function adminCreateCode(name) {
const picks = [...CODE_POOL].sort(() => Math.random() - 0.5).slice(0, 2);
Store.data.codes[name] = { rewards: picks, usedBy: {} };
Store.save();
return Store.data.codes[name];
}

function redeemCode(p, name) {
if (!name) return tell(p.id, "💡 !nhapcode <mã>", C.warn);
const code = Store.data.codes[name];
if (!code) return tell(p.id, "❌ Code không tồn tại hoặc đã hết hạn.", C.err);
if (code.usedBy[p.auth]) return tell(p.id, "❌ Bạn đã nhập code này rồi.", C.err);
code.usedBy[p.auth] = true;
const e = econ(p.auth, p.name);
let msg = "🎁 Bạn nhận: ";
code.rewards.forEach(r => { applyReward(e, r); msg += r.label + ", "; });
tell(p.id, msg.slice(0, -2), C.super, "bold");
Store.touch(); Store.save();
}
function resolveWinnerBets(winTeam) {
const winSide = winTeam === 1 ? "do" : "xanh";
const entries = Object.entries(S.bets.winner);
if (entries.length) {
for (const [auth, bet] of entries) {
const e = econ(auth, bet.bettorName);
if (bet.side === winSide) {
const payout = Math.floor(bet.amount * bet.mult);
e.coins += payout;
say(`🎉 ${bet.bettorName} thắng kèo! +${payout.toLocaleString()} xu`, C.ok, "bold", 1);
} else say(`💸 ${bet.bettorName} thua kèo, mất ${bet.amount.toLocaleString()} xu.`, C.err, "small");
}
Store.touch();
}
S.bets.winner = {}; S.betsLocked = false;
}

const PRACTICE_MILESTONES = [[60, "⏱️CẦN CÙ"], [300, "🏃BỀN BỈ"], [1000, "🧘KIÊN TRÌ VÔ SONG"]];
const LOGIN_STREAK_MILESTONES = [[3, "🔥CHĂM CHỈ"], [7, "⚡SIÊNG NĂNG"], [30, "👑TRUNG THÀNH"]];

/* ═══════════════════ 11. HỆ THỐNG LỆNH ═══════════════════ */

const CMDS = new Map(), CMD_LIST = [];
function def(names, level, usage, desc, fn) {
const c = { names, level, usage, desc, fn };
CMD_LIST.push(c); names.forEach(n => CMDS.set(n, c));
}

function matchKey(s) { return norm(s).replace(/[\s_]+/g, ""); }

function resolve(p, q, opt = {}) {
if (!q) { tell(p.id, "⚠ Thiếu người chơi. Dùng #id, @tên, hoặc chỉ cần gõ vài chữ trong tên.", C.warn); return null; }
if (q[0] === "@") q = q.slice(1);
let hits = [];
if (q[0] === "#") { const t = S.players.get(parseInt(q.slice(1), 10)); if (t) hits = [t]; }
else {
const all = [...S.players.values()], k = matchKey(q);
hits = all.filter(t => matchKey(t.name) === k);
if (!hits.length) hits = all.filter(t => matchKey(t.name).startsWith(k));
if (!hits.length) hits = all.filter(t => matchKey(t.name).includes(k));
}
if (!hits.length) { tell(p.id, `❌ Không tìm thấy "${q}".`, C.err); return null; }
if (hits.length > 1) { tell(p.id, "⚠ Nhiều kết quả: " + hits.slice(0, 5).map(h => `#${h.id} ${h.name}`).join(", ") + " — dùng #id.", C.warn); return null; }
const t = hits[0];
if (t.id === p.id && !opt.self) { tell(p.id, "⚠ Không thể áp dụng lên chính bạn.", C.warn); return null; }
if (opt.outrank && t.id !== p.id && !(p.role > t.role)) {
tell(p.id, `⛔ Bạn không có quyền với ${ROLE_NAME[t.role]} ${t.name}.`, C.err);
audit(`${p.name} (${ROLE_NAME[p.role]}) bị chặn khi tác động lên ${t.name} (${ROLE_NAME[t.role]})`);
if (t.role === ROLE.SUPER) alertSupers(`${p.name} đã thử tác động lên Super Admin ${t.name}!`);
return null;
}
return t;
}

function runCommand(p, text) {
const t = Now();
if (t - p.lastCmd < CONFIG.CMD_COOLDOWN_MS) return tell(p.id, "⏳ Bạn gõ lệnh quá nhanh.", C.warn);
p.lastCmd = t;
const parts = text.slice(1).trim().split(/\s+/);
const name = (parts.shift() || "").toLowerCase();
const c = CMDS.get(name);
if (!c) return tell(p.id, `❓ Không có lệnh !${name}. Gõ !help.`, C.warn);
if (p.role < c.level) {
if (name !== "login") audit(`${p.name} thử dùng !${name} nhưng thiếu quyền`);
return tell(p.id, `⛔ Lệnh này yêu cầu ${ROLE_NAME[c.level]}.`, C.err);
}                                                                                                              if (S.pen && PEN_BLOCKED.has(name)) return tell(p.id, "⛔ Đang đá luân lưu — lệnh này tạm khóa (!stop để hủy).", C.warn);
try { c.fn(p, parts, text); } catch (e) { logErr("lệnh !" + name, e); tell(p.id, "⚠ Lỗi khi chạy lệnh.", C.err); }
}

const HELP_GROUPS = {
  game: {
    title: "⚽ NHÓM LỆNH TRẬN ĐẤU / PHÒNG",
   names: ["afk", "back", "queue", "admins", "stats", "top", "ranks", "reset", "bb", "betwin", "kyluc"],
  },
  shop: {
    title: "🛍️ NHÓM XU / RUBY / SHOP",
    names: ["vi", "daily", "give", "shop", "buy", "rubyshop", "unequip", "buyruby", "sellruby", "doiruby", "tygia", "kho", "openbox", "nhapcode"],
  }
};

function printHelpGroup(p, key) {
const g = HELP_GROUPS[key];
tellLines(p.id, `╔═══ ${g.title} ═══╗`, C.super);
for (const name of g.names) {
const c = CMDS.get(name);
if (c && !c.hidden && c.level <= p.role) tell(p.id, `  ▸ ${c.usage} — ${c.desc}`, C.info);
}
tell(p.id, "╚" + "═".repeat(g.title.length + 6) + "╝", C.super, "small");
}

def(["help", "h", "lenh"], 0, "!help [game|shop|<lệnh>]", "Xem danh sách lệnh theo nhóm", (p, a) => {
const sub = (a[0] || "").toLowerCase();
if (HELP_GROUPS[sub]) return printHelpGroup(p, sub);
if (sub) {
const c = CMDS.get(sub.replace("!", ""));
if (!c || c.level > p.role) return tell(p.id, "❓ Không có lệnh đó.", C.warn);
return tell(p.id, `📖 ${c.usage} — ${c.desc} [${ROLE_NAME[c.level]}]`, C.info);
}
tell(p.id, "🏟️ ═══════ TRUNG TÂM TRỢ GIÚP ═══════ 🏟️", C.super, "bold");
tell(p.id, "  ⚽ !help game  — lệnh trận đấu, hàng chờ, cược,", C.info);
tell(p.id, "  🛍️ !help shop  — ví xu, ruby, shop, rương, giftcode", C.info);
tell(p.id, "  📜 !luat        — nội quy phòng", C.info);
if (p.role >= 1) tell(p.id, "  🛡️ !adminhelp  — danh sách lệnh quản trị", C.admin);
tell(p.id, "💡 !help <tên lệnh> để xem chi tiết 1 lệnh.", C.sys, "small");
});

def(["luat", "noiquy", "rules"], 0, "!luat", "Xem nội quy phòng", p => {
tell(p.id, "📜 ═══════ NỘI QUY PHÒNG ═══════ 📜", C.super, "bold");
tell(p.id, "  1️⃣ Cấm chửi tục, phân biệt vùng miền, spam chat — vi phạm sẽ bị auto-mute/ban.", C.info);
tell(p.id, "  2️⃣ Cấm phá team (cố tình đá phản, câu giờ, không thi đấu nghiêm túc).", C.info);
tell(p.id, "  3️⃣ Cấm mở nhiều tab/nhiều tài khoản cùng lúc (1 IP = 1 slot).", C.info);
tell(p.id, "  4️⃣ AFK quá lâu sẽ tự bị đẩy ra khán đài, gõ !back để quay lại hàng chờ.", C.info);
tell(p.id, "  5️⃣ Cược trận/minigame chỉ mang tính giải trí bằng xu ảo, không quy đổi thật.", C.info);
tell(p.id, "  6️⃣ Nghe theo hướng dẫn của 🛡️Admin/👑Super Admin đang online.", C.info);
tell(p.id, "🤝 Chơi vui vẻ, fair-play — cảm ơn bạn đã tôn trọng nội quy!", C.ok, "small");
});

def(["adminhelp"], 1, "!adminhelp", "Xem toàn bộ lệnh quản trị (Admin/Super)", p => {
tell(p.id, "🛡️ ═══════ LỆNH QUẢN TRỊ ═══════ 🛡️", C.admin, "bold");
const names = lvl => CMD_LIST.filter(c => c.level === lvl && !c.hidden).map(c => "!" + c.names[0]).join(" ");
tellLines(p.id, "🛡️ ADMIN: " + names(1), C.admin);
if (p.role >= 2) tellLines(p.id, "👑 SUPER: " + names(2), C.super);
tell(p.id, "💡 !help <tên lệnh> để xem cú pháp chi tiết từng lệnh.", C.sys, "small");
});

def(["login", "dangnhap"], 0, "!login <mật khẩu>", "Đăng nhập quyền quản trị", (p, a) => {
const pw = a.join(" ");
const t = Now();
const f = S.loginFails.get(p.auth) || { n: 0, lockUntil: 0 };
if (f.lockUntil > t) return tell(p.id, `🔒 Bị khoá đăng nhập ${fmtDur(f.lockUntil - t)}.`, C.err);
if (pw && pw === CONFIG.PASSWORDS.SUPER) {
S.loginFails.delete(p.auth); setRole(p, ROLE.SUPER, true);
audit(`${p.name} đăng nhập SUPER`); announceStaffWelcome(p);
} else if (pw && pw === CONFIG.PASSWORDS.ADMIN) {
S.loginFails.delete(p.auth); setRole(p, Math.max(p.role, ROLE.ADMIN), true);
audit(`${p.name} đăng nhập ADMIN`); announceStaffWelcome(p);
} else {
f.n++;
if (f.n >= CONFIG.LOGIN_MAX_FAILS) f.lockUntil = t + CONFIG.LOGIN_LOCK_MS;
S.loginFails.set(p.auth, f);
audit(`${p.name} đăng nhập SAI (${f.n})`);
alertSupers(`${p.name} nhập sai mật khẩu quản trị (${f.n} lần).`);
tell(p.id, `❌ Sai mật khẩu (${f.n}/${CONFIG.LOGIN_MAX_FAILS}).`, C.err);
if (f.n >= CONFIG.LOGIN_BAN_FAILS) addBan(p, CONFIG.LOGIN_BAN_MIN, "Dò mật khẩu quản trị", "System");
}
});
CMDS.get("login").hidden = true;

def(["logout", "dangxuat"], 0, "!logout", "Đăng xuất quyền quản trị", p => {
if (!p.role) return tell(p.id, "Bạn chưa đăng nhập.", C.warn);
setRole(p, ROLE.USER); tell(p.id, "🔓 Đã đăng xuất.", C.ok);
});

def(["afk"], 0, "!afk", "Tự chuyển sang AFK", p => { if (p.afk) return tell(p.id, "Bạn đã AFK rồi.", C.warn);moveAfk(p, "Bạn đã bật AFK", "AFK_SELF"); });
def(["back", "b"], 0, "!back", "Quay lại từ AFK", p => {
if (!p.afk) return tell(p.id, "Bạn không AFK.", C.warn);
p.afk = false; p.lastActive = Now(); syncQueue();
say(`✅ ${p.name} đã quay lại (hàng chờ #${S.queue.indexOf(p.id) + 1 || "-"}).`, C.ok, "small");
scheduleBalance();
});

def(["queue", "q", "hangcho"], 0, "!queue", "Xem hàng chờ & đội hình", p => {
syncQueue();
const nm = id => (rec(id) || {}).name;
const r = teamIds(1).map(nm).join(", ") || "-", b = teamIds(2).map(nm).join(", ") || "-";
tell(p.id, `⚽ [${MODE_INFO[S.mode].label}] 🔴 ${r}  vs  🔵 ${b}`, C.info);
tell(p.id, "⏳ Hàng chờ: " + (S.queue.map((id, i) => `${i + 1}.${nm(id)}`).join("  ") || "trống"), C.info);
});

def(["admins", "staff"], 0, "!admins", "Xem quản trị viên đang online", p => {
const s = [...S.players.values()].filter(x => x.role > 0).map(x => `${BADGE[x.role]}${x.name}`);
tell(p.id, "🛡 Quản trị online: " + (s.join(", ") || "không có"), C.info);
});

def(["stats", "me", "rank"], 0, "!stats [tên]", "Xem thống kê", (p, a) => {
let auth = p.auth, s;
if (a[0]) {
const q = matchKey(a.join(" "));
const online = [...S.players.values()].find(x => matchKey(x.name).includes(q));
if (online) auth = online.auth;
else { const hit = Object.entries(Store.data.stats).find(([, v]) => matchKey(v.name).includes(q)); if (!hit) return tell(p.id, "❌ Không có dữ liệu.", C.err); auth = hit[0]; }
}
s = Store.data.stats[auth];
if (!s) return tell(p.id, "📊 Chưa có dữ liệu trận đấu.", C.warn);
const wr = s.games ? Math.round(s.wins / s.games * 100) : 0, rk = rankOf(auth);
tell(p.id, `📊 ${s.name}: ${s.games} trận (${s.wins}T/${s.losses}B, ${wr}%) | ⚽${s.goals} 🅰${s.assists} 🤦${s.og} 🧤${s.cs} | ⭐${s.rating}${rk ? " | Hạng #" + rk : ""} | 🔥${s.streak || 0} (kỷ lục ${s.bestStreak || 0})`, C.info);
});
const BOARDS = {
  elo:     { title: "🏆 BẢNG XẾP HẠNG ELO", val: v => v.rating,  ok: v => v.games > 0, tie: v => -v.games },
  goals:   { title: "⚽ VUA PHÁ LƯỚI",       val: v => v.goals,   ok: v => v.goals > 0, tie: v => v.games },
  assists: { title: "🎯 VUA KIẾN TẠO",       val: v => v.assists, ok: v => v.assists > 0, tie: v => v.games },
  gk:      { title: "🧤 GOALKEEPER (SẠCH LƯỚI)", val: v => v.cs,  ok: v => v.cs > 0, tie: v => v.games },
streak:  { title: "🔥 CHUỖI THẮNG DÀI NHẤT", val: v => v.bestStreak || 0, ok: v => (v.bestStreak || 0) > 0, tie: v => v.games },
};

function boardRows(key) {
  const b = BOARDS[key];
  return Object.entries(Store.data.stats)
    .filter(([, v]) => b.ok(v))
    .sort((x, y) => b.val(y[1]) - b.val(x[1]) || b.tie(x[1]) - b.tie(y[1]));
}
function showBoard(p, key, limit = 10) {
  const b = BOARDS[key], rows = boardRows(key);
  tell(p.id, b.title, C.super, "bold");
  if (!rows.length) return tell(p.id, "  Chưa có dữ liệu.", C.warn, "small");
  const medal = ["🥇", "🥈", "🥉"];
  const line = rows.slice(0, limit)
    .map(([, v], i) => `${medal[i] || (i + 1) + "."}${v.name}(${b.val(v)})`)
    .join("  ");
  wrap(line, 125).forEach(l => tell(p.id, " " + l, C.info, "small"));
  const me = rows.findIndex(([auth]) => auth === p.auth);
  if (me >= limit) tell(p.id, ` 📌 Bạn: #${me + 1} (${b.val(rows[me][1])})`, C.ok, "small");
}
def(["top", "bxh"], 0, "!top", "Xem tất cả bảng xếp hạng", p => {
  for (const k of Object.keys(BOARDS)) showBoard(p, k, 10);
});
def(["ranks", "elo", "bangrank", "capbac"], 0, "!ranks", "Bảng các mức Elo và rank", p => {
const tiers = RANK_SYSTEM.filter(r => r.reqElo !== undefined).sort((a, b) => a.reqElo - b.reqElo);
const ordered = Object.values(Store.data.stats).filter(v => v.games > 0).sort((a, b) => b.rating - a.rating);
tell(p.id, "🎖️ ═══════ BẢNG RANK ELO ═══════ 🎖️", C.super, "bold");
tell(p.id, "  ELO              │  RANK", C.admin, "bold");
tiers.forEach((r, i) => {
const nx = tiers[i + 1], range = nx ? `${r.reqElo} – ${nx.reqElo - 1}` : `${r.reqElo}+`;
tell(p.id, `  ${range.padEnd(15)}│  ${r.title}`, C.info);
});
RANK_SYSTEM.filter(r => r.top).sort((a, b) => b.top - a.top).forEach(r => {
const h = ordered[r.top - 1], who = h && h.rating >= TOP_MIN_ELO ? `  —  ${h.name} (${h.rating})` : "";
tell(p.id, `  ${("Top " + r.top + " (≥" + TOP_MIN_ELO + ")").padEnd(15)}│  ${r.title}${who}`, C.info);
});
const s = Store.data.stats[p.auth], cur = computeRank(p.auth);
if (!s || !cur) return tell(p.id, "📌 Bạn chưa có rank — hãy chơi một trận thi đấu để được xếp hạng.", C.warn);
if (cur.top) return tell(p.id, `📌 Của bạn: ${cur.title} (Top ${cur.top}, ${s.rating} elo) — đang ở nhóm rank cao nhất!`, C.ok, "bold");
const next = RANK_SYSTEM.find(r => r.reqElo !== undefined && r.reqElo > s.rating);
tell(p.id, `📌 Của bạn: ${cur.title} (${s.rating} elo)` + (next ? ` — còn ${next.reqElo - s.rating} elo nữa lên ${next.title}` : " — hãy leo lên Top 3 bảng Elo để mở khóa các rank cao nhất!"), C.ok, "bold");
});

def(["reset", "practice"], 0, "!reset", "Đặt lại bóng về tâm", p => {
if (S.mode !== "practice" || !running()) return tell(p.id, "⚠ Chỉ dùng được ở chế độ luyện tập.", C.warn);
room.setDiscProperties(0, { x: 0, y: 0, xspeed: 0, yspeed: 0 }); applyBallBuff();
tell(p.id, "🎯 Đã đặt lại bóng.", C.ok);
});

def(["bb", "bye"], 0, "!bb", "Rời phòng", p => room.kickPlayer(p.id, "Hẹn gặp lại 👋", false));

def(["vi", "wallet", "taikhoan"], 0, "!vi", "Xem ví xu/ruby", p => {
const e = econ(p.auth, p.name);
tell(p.id, `💰 ${e.coins.toLocaleString()} xu | 💎 ${e.ruby} ruby | Danh hiệu: ${e.title || "chưa có"}`, C.info, "bold");
});
def(["unequip", "thao"], 0, "!unequip [title|color|all]", "Gỡ danh hiệu / màu chat đang đeo", (p, a) => {
  const v = (a[0] || "all").toLowerCase();
  const e = econ(p.auth, p.name);
  const doTitle = ["all", "title", "t"].includes(v);
  const doColor = ["all", "color", "c"].includes(v);
  if (!doTitle && !doColor) return tell(p.id, "💡 !unequip [title|color|all]", C.warn);
  const done = [];
  if (doTitle && e.title) { e.title = null; done.push("danh hiệu"); }
  if (doColor && e.chatColor != null) { e.chatColor = null; done.push("màu chat"); }
  if (!done.length) return tell(p.id, "ℹ Bạn không đang đeo gì để gỡ.", C.warn);
  Store.touch();
  tell(p.id, `🧹 Đã gỡ ${done.join(" và ")}.`, C.ok, "bold");
});
def(["daily", "diemdanh"], 0, "!daily", "Điểm danh nhận xu", p => {
const e = econ(p.auth, p.name);
const today = new Date().toDateString(), yesterday = new Date(Now() - 86400e3).toDateString();
if (e.lastDaily === today) return tell(p.id, "❌ Hôm nay điểm danh rồi!", C.err);
e.dailyStreak = e.lastDaily === yesterday ? e.dailyStreak + 1 : 1;
const reward = ECO.DAILY_BASE + Math.min((e.dailyStreak - 1) * ECO.DAILY_STREAK_BONUS, ECO.DAILY_STREAK_CAP - ECO.DAILY_BASE);
e.coins += reward; e.lastDaily = today; Store.touch();
tell(p.id, `🎁 Điểm danh +${reward.toLocaleString()} xu! Chuỗi ${e.dailyStreak} ngày.`, C.ok, "bold");
});

def(["give", "chuyenxu"], 0, "!give <tên|#id> <xu>", "Chuyển xu", (p, a) => {
const t = resolve(p, a[0]); if (!t) return;
const amount = parseInt(a[1], 10);
if (!isPosInt(amount)) return tell(p.id, "💡 !give <tên|#id> <xu>", C.warn);
const e = econ(p.auth, p.name), te = econ(t.auth, t.name);
if (e.coins < amount) return tell(p.id, "❌ Không đủ xu.", C.err);
e.coins -= amount; te.coins += amount; Store.touch();
say(`💸 ${p.name} đã chuyển ${amount.toLocaleString()} xu cho ${t.name}!`, C.ok, "bold");
});

function shopCategory(it) {
if (it.type === "color") return "🎨 MÀU CHAT";
if (it.type === "title") return "🏷️ DANH HIỆU";
if (it.type === "box_pack") return "🎁 RƯƠNG BÁU";
if (it.type === "kit") return "🎒 KIT TỔNG HỢP (giá trị hơn mua lẻ)";
return "✨ PERK ĐẶC BIỆT";
}

def(["shop"], 0, "!shop", "Xem shop xu", p => {
const groups = {};
for (const [id, it] of Object.entries(SHOP_ITEMS)) {
const cat = shopCategory(it);
(groups[cat] = groups[cat] || []).push(`   ${id}.  ${it.icon}  ${it.name}  —  ${it.cost.toLocaleString()} xu`);
}
tell(p.id, "🛍️ ═══════════ SHOP XU ═══════════ 🛍️", C.super, "bold");
for (const cat of ["🎒 KIT TỔNG HỢP (giá trị hơn mua lẻ)", "🎨 MÀU CHAT", "🏷️ DANH HIỆU", "✨ PERK ĐẶC BIỆT", "🎁 RƯƠNG BÁU"]) {
if (!groups[cat]) continue;
tell(p.id, cat + ":", C.admin, "bold");
groups[cat].forEach(l => tell(p.id, l, C.info));
}
tell(p.id, "👉 Gõ !buy <ID> để mua — ví dụ: !buy 401", C.sys, "small");
});

def(["buy"], 0, "!buy <ID>", "Mua vật phẩm shop xu", (p, a) => buyShopItem(p, parseInt(a[0], 10)));
def(["rubyshop"], 0, "!rubyshop", "Xem Ruby shop", p => {
tell(p.id, "💎 ═══════════ RUBY SHOP ═══════════ 💎", C.info, "bold");
for (const [id, it] of Object.entries(RUBY_ITEMS)) tell(p.id, `   ${id}.  ${it.icon}  ${it.name}  —  ${it.cost} 💎`, C.info);
tell(p.id, "👉 Gõ !buyruby <ID> để mua — ví dụ: !buyruby 1", C.sys, "small");
});

def(["buyruby"], 0, "!buyruby <ID>", "Mua vật phẩm Ruby", (p, a) => buyRubyItem(p, parseInt(a[0], 10)));

def(["sellruby"], 0, "!sellruby <số>", "Bán Ruby lấy xu", (p, a) => {
const amt = parseInt(a[0], 10);
if (!isPosInt(amt, 1e6)) return tell(p.id, "💡 !sellruby <số>", C.warn);
const e = econ(p.auth, p.name);
if (e.ruby < amt) return tell(p.id, "❌ Không đủ ruby.", C.err);
const gain = Math.floor(amt * rubySellRate);
e.ruby -= amt; e.coins += gain; Store.touch();
tell(p.id, `💱 Bán ${amt} ruby được ${gain.toLocaleString()} xu (tỷ giá ${rubySellRate.toLocaleString()}/ruby).`, C.ok);
});

def(["doiruby"], 0, "!doiruby <số>", "Đổi xu lấy Ruby", (p, a) => {
const amt = parseInt(a[0], 10);
if (!isPosInt(amt, 1e6)) return tell(p.id, "💡 !doiruby <số>", C.warn);
const cost = Math.ceil(amt * ECO.RUBY_BUY_RATE);
const e = econ(p.auth, p.name);
if (e.coins < cost) return tell(p.id, `❌ Cần ${cost.toLocaleString()} xu.`, C.err);
e.coins -= cost; e.ruby += amt; Store.touch();
tell(p.id, `💱 Đổi ${cost.toLocaleString()} xu ➔ ${amt} ruby.`, C.ok, "bold");
});

def(["tygia"], 0, "!tygia", "Xem tỷ giá Ruby", p => tell(p.id, `💱 Mua 1 ruby = ${ECO.RUBY_BUY_RATE.toLocaleString()} xu | Bán 1 ruby = ${rubySellRate.toLocaleString()} xu`, C.info));

def(["kho"], 0, "!kho", "Xem rương báu", p => {
const e = econ(p.auth, p.name);
tell(p.id, `🎒 Rương: Thường ${e.boxes.common} | Hiếm ${e.boxes.rare} | Sử Thi ${e.boxes.epic} | Huyền Thoại ${e.boxes.legendary}`, C.super);
tell(p.id, "👉 !openbox <common|rare|epic|legendary> <số>", C.sys, "small");
});

def(["openbox"], 0, "!openbox <loại> <số>", "Mở rương báu", (p, a) => {
const type = (a[0] || "").toLowerCase(), qty = parseInt(a[1], 10) || 1;
if (!BOX_TABLE[type]) return tell(p.id, "💡 !openbox <common|rare|epic|legendary> <số>", C.warn);
const e = econ(p.auth, p.name);
if ((e.boxes[type] || 0) < qty || qty <= 0) return tell(p.id, `❌ Không đủ rương ${type}.`, C.err);
e.boxes[type] -= qty;
tell(p.id, `📦 Mở ${qty}x rương ${type.toUpperCase()}:`, C.super, "bold");
for (let i = 0; i < qty; i++) {
const r = rollBox(type);
if (r.type === "coins") { const amt = Math.floor(Math.random() * (r.max - r.min + 1)) + r.min; e.coins += amt; tell(p.id, `  ➔ 💰 +${amt.toLocaleString()} xu`, C.ok); }
else if (r.type === "ruby") { e.ruby += r.amount; tell(p.id, `  ➔ 💎 +${r.amount} ruby`, C.ok); }
else if (r.type === "box") { e.boxes[r.boxType] = (e.boxes[r.boxType] || 0) + r.count; tell(p.id, `  ➔ 🎁 +${r.count}x rương ${r.boxType}`, C.ok); }
else if (r.type === "title") { grantTitle(e, r.value); tell(p.id, `  ➔ 🏷️ Danh hiệu [${r.value}]`, C.ok); }
}
Store.touch();
});

def(["nhapcode"], 0, "!nhapcode <mã>", "Nhập giftcode", (p, a) => redeemCode(p, a[0]));

def(["betwin", "cuoc"], 0, "!betwin <do|xanh> <xu>", "Cược đội thắng", (p, a) => {
if (S.mode === "practice") return tell(p.id, "⚠ Cược chỉ mở ở chế độ thi đấu (1v1/2v2).", C.warn);
if (!running()) return tell(p.id, "⚠ Chưa có trận đang diễn ra.", C.warn);
if (S.betsLocked) return tell(p.id, "⛔ Cược đã ĐÓNG (đã có bàn thắng).", C.err);
const pl = room.getPlayer(p.id);
if (!pl || pl.team !== 0) return tell(p.id, "⛔ Chỉ khán giả mới được cược.", C.err);
const side = (a[0] || "").toLowerCase(), amount = parseInt(a[1], 10);
if (!["do", "xanh"].includes(side) || !isPosInt(amount)) return tell(p.id, "💡 !betwin <do|xanh> <xu>", C.warn);
if (S.bets.winner[p.auth]) return tell(p.id, "⛔ Bạn đã cược trận này rồi.", C.err);
const e = econ(p.auth, p.name);
if (e.coins < amount) return tell(p.id, "❌ Không đủ xu.", C.err);
const redProb = winProb(teamAvgRating(1), teamAvgRating(2));
const prob = side === "do" ? redProb : 1 - redProb;
const mult = betMultiplier(prob);
e.coins -= amount;
S.bets.winner[p.auth] = { side, amount, mult, bettorName: p.name };
Store.touch();
say(`🎯 ${p.name} cược ${amount.toLocaleString()} xu cho đội ${side === "do" ? "🔴ĐỎ" : "🔵XANH"} (x${mult})!`, C.info, "bold", 1);
});
def(["kyluc", "records"], 0, "!kyluc", "Xem kỷ lục cá nhân", p => {
const e = econ(p.auth, p.name);
tell(p.id, `📌 Chuỗi ngày ghé phòng: ${e.loginStreak || 0} ngày | Phút luyện tập: ${e.practiceMinutes || 0} phút`, C.info, "bold");
});
def(["list", "ds"], 1, "!list", "Danh sách người chơi", p => {
tellLines(p.id, "👥 " + [...S.players.values()].map(x => `#${x.id} ${x.name}${x.role ? "(" + ROLE_NAME[x.role][0] + ")" : ""}${x.afk ? "💤" : ""}`).join(" | "), C.info);
});

def(["kick", "k"], 1, "!kick <tên|#id> [lý do]", "Đá người chơi", (p, a) => {
const t = resolve(p, a[0], { outrank: true }); if (!t || !guard(p)) return;
const reason = a.slice(1).join(" ") || "Vi phạm nội quy";
audit(`${p.name} KICK ${t.name}: ${reason}`);
say(`🥾 ${t.name} bị kick bởi ${p.name}: ${reason}`, C.warn);
room.kickPlayer(t.id, `Bị kick: ${reason}`, false);
});

def(["ban"], 1, "!ban <tên|#id> [phút|perm] [lý do]", "Ban người chơi", (p, a) => {
const t = resolve(p, a[0], { outrank: true }); if (!t || !guard(p)) return;
let mins = parseInt(a[1], 10), ri = 2;
if (/^perm$/i.test(a[1] || "")) mins = 0;
else if (isNaN(mins)) { mins = 60; ri = 1; }
if (mins === 0 && p.role < ROLE.SUPER) return tell(p.id, "⛔ Chỉ Super Admin được ban vĩnh viễn.", C.err);
if (p.role < ROLE.SUPER) mins = clamp(mins, 1, CONFIG.ADMIN_MAX_BAN_MIN);
const reason = a.slice(ri).join(" ") || "Vi phạm nội quy";
audit(`${p.name} BAN ${t.name} ${mins || "perm"}p: ${reason}`);
say(`🔨 ${t.name} bị ban ${mins ? mins + " phút" : "vĩnh viễn"} bởi ${p.name}: ${reason}`, C.err);
addBan(t, mins, reason, p.name);
});

def(["mute", "m"], 1, "!mute <tên|#id> [phút=10] [lý do]", "Cấm chat", (p, a) => {
const t = resolve(p, a[0], { outrank: true }); if (!t || !guard(p)) return;
let mins = parseInt(a[1], 10), ri = 2;
if (isNaN(mins)) { mins = 10; ri = 1; }
mins = clamp(mins, 1, p.role >= 2 ? 1440 : CONFIG.ADMIN_MAX_MUTE_MIN);
const reason = a.slice(ri).join(" ") || "Vi phạm chat";
S.mutes.set(t.auth, { until: Now() + mins * 60e3, reason, by: p.name });
audit(`${p.name} MUTE ${t.name} ${mins}p`);
say(`🔇 ${t.name} bị mute ${mins} phút bởi ${p.name}.`, C.warn);
});

def(["unmute", "um"], 1, "!unmute <tên|#id>", "Gỡ mute", (p, a) => {
const t = resolve(p, a[0], { self: true }); if (!t) return;
if (S.mutes.delete(t.auth)) { say(`🔊 ${t.name} được gỡ mute bởi ${p.name}.`, C.ok); audit(`${p.name} UNMUTE ${t.name}`); }
else tell(p.id, "ℹ Người này không bị mute.", C.warn);
});

def(["move", "mv"], 1, "!move <tên|#id> <red|blue|spec>", "Chuyển đội", (p, a) => {
const t = resolve(p, a[0], { self: true }); if (!t) return;
const map = { red: 1, r: 1, blue: 2, b: 2, spec: 0, s: 0 }, team = map[(a[1] || "").toLowerCase()];
if (team === undefined) return tell(p.id, "⚠ Chọn red / blue / spec.", C.warn);
if (team) { t.afk = false; t.lastActive = Now(); room.setPlayerTeam(t.id, team); S.queue = S.queue.filter(x => x !== t.id); }
else toSpec(t.id);
syncQueue(); scheduleBalance(); audit(`${p.name} MOVE ${t.name} → ${a[1]}`);
});

def(["setafk"], 1, "!setafk <tên|#id>", "Ép sang AFK", (p, a) => {
const t = resolve(p, a[0], { outrank: true }); if (!t) return;
moveAfk(t, `${p.name} chuyển bạn sang AFK`, "AFK_ADMIN");
});

def(["swap"], 1, "!swap", "Đổi màu 2 đội", p => {
const r = teamIds(1), b = teamIds(2);
r.forEach(id => room.setPlayerTeam(id, 2)); b.forEach(id => room.setPlayerTeam(id, 1));
swapLineup();
say(`🔄 ${p.name} đã đổi phe 2 đội.`, C.info);
});

def(["rr", "restart"], 1, "!rr", "Chơi lại trận", p => { audit(`${p.name} RR`); restartRound(`🔁 ${p.name} khởi động lại trận.`); });
def(["start"], 1, "!start", "Bắt đầu trận", p => {
S.hold = false; if (!running()) { schedulePlan(300); say(`▶️ ${p.name} bắt đầu trận.`, C.ok); }
});

def(["stop"], 1, "!stop", "Dừng trận", p => {
if (S.pen) abortPen("admin dừng trận");
cancelPick(); S.recovery = null;
S.hold = true; S.holdUntil = Now() + CONFIG.HOLD_MAX_MS; S.result = null; S.winnerTeam = 0;
if (running()) room.stopGame(); say(`⏹ ${p.name} dừng trận.`, C.warn);
});

def(["pause"], 1, "!pause", "Tạm dừng / tiếp tục", p => {
if (!running()) return tell(p.id, "⚠ Chưa có trận.", C.warn);
room.pauseGame(!S.paused);
});

def(["buff"], 1, "!buff <tên|#id> <loại> [giây]", "Buff người chơi", (p, a) => {
if (!a[0] || !a[1]) return tell(p.id, "🎁 Loại buff: " + Object.entries(BUFFS).filter(([, v]) => v.role <= p.role).map(([k, v]) => `${k}(${v.label})`).join(" "), C.info);
const t = resolve(p, a[0], { self: true }); if (!t) return;
const type = a[1].toLowerCase(), b = BUFFS[type];
if (!b) return tell(p.id, "❌ Không có buff đó.", C.err);
if (b.role > p.role) return tell(p.id, "⛔ Buff này chỉ dành cho Super Admin.", C.err);
if (t.role >= p.role && t.id !== p.id && p.role < 2) return tell(p.id, "⛔ Không thể buff quản trị ngang/cao hơn.", C.err);
let sec = parseInt(a[2], 10); if (isNaN(sec)) sec = 60;
sec = p.role >= 2 ? clamp(sec, 0, CONFIG.SUPER_MAX_BUFF_SEC) : clamp(sec, 5, CONFIG.ADMIN_MAX_BUFF_SEC);
S.buffs.set(t.id, { type, until: sec ? Now() + sec * 1e3 : 0, by: p.name });
applyPlayerBuff(t.id); unrate("có buff");
audit(`${p.name} BUFF ${t.name} ${type} ${sec}s`);
say(`✨ ${t.name} nhận buff ${b.label}${sec ? " (" + sec + "s)" : ""} từ ${p.name}!`, C.super, "bold", 1);
});

def(["unbuff"], 1, "!unbuff <tên|#id|all>", "Gỡ buff", (p, a) => {
if ((a[0] || "").toLowerCase() === "all") { [...S.buffs.keys()].forEach(id => removePlayerBuff(id)); return say(`🧹 ${p.name} gỡ toàn bộ buff.`, C.info); }
const t = resolve(p, a[0], { self: true }); if (!t) return;
removePlayerBuff(t.id); say(`🧹 Gỡ buff của ${t.name}.`, C.info);
});

def(["clear", "cls"], 1, "!clear", "Xoá chat", p => { for (let i = 0; i < 25; i++) room.sendAnnouncement(" ", null, 0xFFFFFF, "small", 0); say(`🧽 ${p.name} đã xoá chat.`, C.sys); });
def(["say", "announce"], 1, "!say <nội dung>", "Thông báo toàn phòng", (p, a) => { if (a.length) say("📢 " + a.join(" "), C.super, "bold", 2); });

def(["buffcoin"], 1, "!buffcoin <tên|#id> <±xu>", "Cộng/trừ xu", (p, a) => {
const t = resolve(p, a[0], { self: true }); if (!t || !guard(p)) return;
const amt = parseInt(a[1], 10); if (isNaN(amt)) return tell(p.id, "💡 !buffcoin <tên|#id> <±xu>", C.warn);
const e = econ(t.auth, t.name); e.coins = Math.max(0, e.coins + amt); Store.touch();
audit(`${p.name} BUFFCOIN ${t.name} ${amt}`);
tell(t.id, `🏦 Bạn được ${p.name} ${amt >= 0 ? "cộng" : "trừ"} ${Math.abs(amt).toLocaleString()} xu! Số dư: ${e.coins.toLocaleString()}`, C.super, "bold", 1);
tell(p.id, `✅ Đã chỉnh xu của ${t.name}: ${amt >= 0 ? "+" : ""}${amt}`, C.ok);
});

def(["buffruby"], 1, "!buffruby <tên|#id> <±ruby>", "Cộng/trừ ruby", (p, a) => {
const t = resolve(p, a[0], { self: true }); if (!t || !guard(p)) return;
const amt = parseInt(a[1], 10); if (isNaN(amt)) return tell(p.id, "💡 !buffruby <tên|#id> <±ruby>", C.warn);
const e = econ(t.auth, t.name); e.ruby = Math.max(0, e.ruby + amt); Store.touch();
audit(`${p.name} BUFFRUBY ${t.name} ${amt}`);
tell(t.id, `💎 Bạn được ${p.name} ${amt >= 0 ? "cộng" : "trừ"} ${Math.abs(amt)} ruby!`, C.super, "bold", 1);
tell(p.id, `✅ Đã chỉnh ruby của ${t.name}: ${amt >= 0 ? "+" : ""}${amt}`, C.ok);
});

def(["setadmin"], 2, "!setadmin <tên|#id>", "Cấp quyền Admin", (p, a) => {
const t = resolve(p, a[0]); if (!t) return;
if (t.role >= 1) return tell(p.id, "ℹ Người này đã là quản trị.", C.warn);
setRole(t, ROLE.ADMIN, true); audit(`${p.name} SETADMIN ${t.name}`);
say(`⚡ ${t.name} được ${p.name} cấp quyền Admin.`, C.admin, "bold", 1);
});

def(["removeadmin", "deladmin"], 2, "!removeadmin <tên|#id>", "Thu hồi quyền Admin", (p, a) => {
const t = resolve(p, a[0]); if (!t) return;
if (t.role === ROLE.SUPER) return tell(p.id, "⛔ Không thể hạ Super Admin.", C.err);
if (t.role === 0) return tell(p.id, "ℹ Người này không phải quản trị.", C.warn);
setRole(t, ROLE.USER); audit(`${p.name} REMOVEADMIN ${t.name}`);
say(`🔻 ${t.name} bị thu hồi quyền Admin.`, C.warn);
});

def(["banlist", "bans"], 2, "!banlist [trang]", "Danh sách ban", (p, a) => {
const bans = Store.data.bans, pg = Math.max(1, parseInt(a[0], 10) || 1), t = Now();
if (!bans.length) return tell(p.id, "📭 Không có ai bị ban.", C.info);
bans.slice((pg - 1) * 5, pg * 5).forEach((b, i) => tell(p.id,
`#${(pg - 1) * 5 + i + 1} ${b.name} — ${b.reason} — ${b.until ? "còn " + fmtDur(b.until - t) : "vĩnh viễn"} — bởi ${b.by}`, C.info));
tell(p.id, `Trang ${pg}/${Math.ceil(bans.length / 5)} — dùng !unban <số>`, C.sys, "small");
});

def(["unban"], 2, "!unban <số thứ tự|tên>", "Gỡ ban", (p, a) => {
const bans = Store.data.bans; let i = parseInt(a[0], 10) - 1;
if (isNaN(i)) i = bans.findIndex(b => matchKey(b.name).includes(matchKey(a.join(" "))));
if (i < 0 || i >= bans.length) return tell(p.id, "❌ Không tìm thấy mục ban.", C.err);
const [b] = bans.splice(i, 1); Store.save();
audit(`${p.name} UNBAN ${b.name}`); say(`✅ ${b.name} được gỡ ban bởi ${p.name}.`, C.ok);
});

def(["clearbans"], 2, "!clearbans", "Xoá toàn bộ ban", p => { Store.data.bans.length = 0; Store.save(); room.clearBans(); audit(`${p.name} CLEARBANS`); say("🧹 Đã xoá toàn bộ danh sách ban.", C.ok); });

def(["ballbuff"], 2, "!ballbuff <fast|slow|big|small|heavy|light|bouncy|off>", "Buff quả bóng", (p, a) => {
const v = (a[0] || "").toLowerCase();
if (v === "off") { S.ballBuff = null; if (running()) room.setDiscProperties(0, sanitizeProps(ORIGINAL_BALL[mapKeyOf(S.mode)] || BALL_ENGINE_DEFAULT)); return say("⚽ Bóng trở lại đúng kích thước gốc của map.", C.info); }
if (!BALL_BUFFS[v]) return tell(p.id, "⚠ Loại: " + Object.keys(BALL_BUFFS).join(", ") + ", off", C.warn);
S.ballBuff = v; applyBallBuff(); unrate("buff bóng"); say(`⚽ Bóng được buff: ${v} (bởi ${p.name})`, C.super, "bold", 1);
});

def(["scorelimit"], 2, "!scorelimit <số>", "Đặt giới hạn bàn thắng", (p, a) => {
const n = clamp(parseInt(a[0], 10) || 0, 0, 20); S.limits.score = n; tell(p.id, `✅ Score limit = ${n} (từ trận sau).`, C.ok);
});

def(["timelimit"], 2, "!timelimit <phút>", "Đặt giới hạn thời gian", (p, a) => {
const n = clamp(parseInt(a[0], 10) || 0, 0, 30); S.limits.time = n; tell(p.id, `✅ Time limit = ${n} phút (từ trận sau).`, C.ok);
});

def(["password", "pass"], 2, "!password <mật khẩu|off>", "Đặt/xoá mật khẩu phòng", (p, a) => {
if (!a[0] || a[0] === "off") { room.clearPassword(); return tell(p.id, "🔓 Đã bỏ mật khẩu phòng.", C.ok); }
room.setPassword(a.join(" ")); tell(p.id, "🔒 Đã đặt mật khẩu phòng.", C.ok);
});

def(["slow"], 2, "!slow <giây|0>", "Chế độ chat chậm", (p, a) => {
S.slowMs = clamp(parseInt(a[0], 10) || 0, 0, 60) * 1000;
say(S.slowMs ? `🐢 Chat chậm: ${S.slowMs / 1000}s/tin.` : "🐇 Tắt chat chậm.", C.info);
});

def(["chatlock", "lockchat"], 2, "!chatlock", "Khoá/mở chat", p => {
S.chatLocked = !S.chatLocked; say(S.chatLocked ? "🔒 Chat bị khoá — chỉ quản trị được nói." : "🔓 Đã mở chat.", C.info);
});

def(["resetstats"], 2, "!resetstats <tên|all confirm>", "Xoá thống kê", (p, a) => {
if (a[0] === "all") {
  if (a[1] !== "confirm") return tell(p.id, "⚠ Gõ: !resetstats all confirm", C.warn);
  Store.data.stats = {};
  for (const e of Object.values(Store.data.econ)) { delete e.eloRank; delete e.eloColor; }
  Store.save();
  return say("🧹 Đã xoá toàn bộ thống kê.", C.warn);
}
const t = resolve(p, a[0], { self: true }); if (!t) return;
delete Store.data.stats[t.auth]; Store.save(); tell(p.id, `✅ Đã xoá thống kê của ${t.name}.`, C.ok);
});

def(["log"], 2, "!log", "Xem nhật ký quản trị", p => S.audit.slice(-10).forEach(l => tell(p.id, l, C.sys, "small")));

def(["setcoin"], 2, "!setcoin <tên|#id> <xu>", "Đặt số xu chính xác", (p, a) => {
const t = resolve(p, a[0], { self: true }); if (!t) return;
const v = parseInt(a[1], 10); if (isNaN(v) || v < 0) return tell(p.id, "💡 !setcoin <tên|#id> <xu>", C.warn);
econ(t.auth, t.name).coins = v; Store.touch();
audit(`${p.name} SETCOIN ${t.name} = ${v}`);
tell(p.id, `✅ Đặt xu ${t.name} = ${v.toLocaleString()}`, C.ok);
});

def(["setruby"], 2, "!setruby <tên|#id> <ruby>", "Đặt số ruby chính xác", (p, a) => {
const t = resolve(p, a[0], { self: true }); if (!t) return;
const v = parseInt(a[1], 10); if (isNaN(v) || v < 0) return tell(p.id, "💡 !setruby <tên|#id> <ruby>", C.warn);
econ(t.auth, t.name).ruby = v; Store.touch();
audit(`${p.name} SETRUBY ${t.name} = ${v}`);
tell(p.id, `✅ Đặt ruby ${t.name} = ${v}`, C.ok);
});
def(["exclusive", "doccuyen"], 2, "!exclusive <tên|#id> <màu hex> <danh hiệu> | !exclusive <tên|#id> off", "Cấp/gỡ danh hiệu + màu chat độc quyền", (p, a) => {
  const t = resolve(p, a[0], { self: true }); if (!t) return;
  const e = econ(t.auth, t.name);
  if ((a[1] || "").toLowerCase() === "off") {
    delete e.exclusive; Store.save();
    return tell(p.id, `🧹 Đã gỡ độc quyền của ${t.name}.`, C.ok);
  }
  const hex = /^#?([0-9a-f]{6})$/i.exec(a[1] || ""), title = a.slice(2).join(" ");
  if (!hex || !title) return tell(p.id, "💡 !exclusive <tên|#id> 00FF9C 🍀LUCKY PASS", C.warn);
  if (!t.realAuth) tell(p.id, "⚠ Người này không có auth thật (đang dùng conn/IP) — độc quyền có thể không giữ được lâu dài.", C.warn);
  e.exclusive = { title, color: parseInt(hex[1], 16) };
  Store.save();
  audit(`${p.name} EXCLUSIVE ${t.name}: ${title}`);
  say(`🌟 ${t.name} được trao danh hiệu độc quyền [${title}]! 🌟`, e.exclusive.color, "bold", 2);
});
def(["taocode"], 2, "!taocode <tên>", "Tạo giftcode mới", (p, a) => {
const name = a[0]; if (!name) return tell(p.id, "💡 !taocode <tên>", C.warn);
adminCreateCode(name);
audit(`${p.name} TAOCODE ${name}`);
say(`🎁 [GIFTCODE MỚI] "${name}" đang chờ! Gõ !nhapcode ${name} để nhận quà!`, C.super, "bold", 1);
});
def(["kit", "ao"], 1, "!kit [số|random|reset]", "Xem/chọn áo đội", (p, a) => {
  const v = (a[0] || "").toLowerCase();
  if (!v) {
    tell(p.id, "👕 Danh sách áo: " + KIT_PAIRS.map((k, i) => `${i + 1}.${k.r} vs ${k.b}`).join("  "), C.info);
    return tell(p.id, "💡 !kit <số> | !kit random | !kit reset", C.sys, "small");
  }
  if (v === "random") return autoKits();
  if (v === "reset") { resetKits(); return say(`👕 ${p.name} đã đặt lại áo mặc định.`, C.info); }
  const i = parseInt(v, 10) - 1;
  if (!(i >= 0 && i < KIT_PAIRS.length)) return tell(p.id, "⚠ Số không hợp lệ. Gõ !kit để xem danh sách.", C.warn);
  autoKits(i);
});
def(["save"], 2, "!save", "Lưu dữ liệu ngay", p => { Store.save(); tell(p.id, "💾 Đã lưu.", C.ok); });
/* ═══════════════════ KIT ĐỘI — CẶP ÁO PHỐI SẴN ═══════════════════ */
// Mỗi cặp: r/b = tên đội ĐỎ/XANH (hiện trong chat để mọi người biết ai là ai).
// Kit = [góc, màu chữ số áo, [tối đa 3 màu áo]]. Góc 0 = sọc dọc, 90 = sọc ngang, 45 = chéo.
// Quy tắc phối: không dùng đen/navy làm màu chính (khó nhìn trên sân cỏ), hai đội luôn khác tông.
const KIT_PAIRS = [
  { r: "Real Madrid",     b: "Barcelona",
    red:  [0, 0x1B2A5A, [0xFFFFFF]],
    blue: [0, 0xFFD34D, [0xA50044, 0x004D98, 0xA50044]] },
  { r: "Man United",      b: "Brazil",
    red:  [0, 0xFFFFFF, [0xDA291C]],
    blue: [0, 0x0B7A3E, [0xFFDF00]] },
  { r: "Liverpool",       b: "Argentina",
    red:  [0, 0xFFE9A0, [0xC8102E]],
    blue: [0, 0x0B2A5C, [0x74ACDF, 0xFFFFFF, 0x74ACDF]] },
  { r: "Việt Nam",        b: "Nhật Bản",
    red:  [0, 0xFFDD00, [0xDA251D]],
    blue: [0, 0xFFFFFF, [0x1F5FD6]] },
  { r: "Bayern",          b: "Dortmund",
    red:  [0, 0xFFFFFF, [0xDC052D]],
    blue: [0, 0x1A1A1A, [0xFDE100]] },
  { r: "Arsenal",         b: "Man City",
    red:  [0, 0xFFFFFF, [0xEF0107]],
    blue: [0, 0x1C2C5B, [0x6CABDD]] },
  { r: "AC Milan",        b: "Napoli",
    red:  [0, 0xFFFFFF, [0xE3001B, 0x8E0A14, 0xE3001B]],
    blue: [0, 0xFFFFFF, [0x12A0D7]] },
  { r: "Juventus (hồng)", b: "Inter",
    red:  [0, 0x4A1230, [0xF4A6C8]],
    blue: [0, 0xFFFFFF, [0x1E6FE0, 0x0A2A66, 0x1E6FE0]] },
  { r: "Atletico",        b: "PSG",
    red:  [0, 0xFFFFFF, [0xCB3524, 0xFFFFFF, 0xCB3524]],
    blue: [0, 0xFFFFFF, [0x0A4DA2]] },
  { r: "Hà Lan",          b: "Pháp",
    red:  [0, 0x1A1A1A, [0xF36C21]],
    blue: [0, 0xFFFFFF, [0x1E4FA8]] },
  { r: "Tây Ban Nha",     b: "Đức",
    red:  [0, 0xFFD700, [0xC60B1E]],
    blue: [0, 0x1A1A1A, [0xFFFFFF]] },
  { r: "Bồ Đào Nha",      b: "Ý",
    red:  [0, 0xFFE066, [0xC8102E]],
    blue: [0, 0xFFFFFF, [0x1E6BD6]] },
  { r: "Celtic",          b: "Rangers",
    red:  [90, 0x111111, [0x16A34A, 0xFFFFFF, 0x16A34A]],
    blue: [0, 0xFFFFFF, [0x2563EB]] },
  { r: "Cam cháy",        b: "Tím neon",
    red:  [90, 0xFFFFFF, [0xFF7A00, 0xFFB347, 0xFF7A00]],
    blue: [90, 0xFFFFFF, [0x7B2FBE, 0xB07CE8, 0x7B2FBE]] },
  { r: "Kẹo ngọt",        b: "Biển xanh",
    red:  [232, 0x5A1A40, [0xFFCCFA, 0xFF99DD, 0xFF6176]],
    blue: [129, 0xFFFFFF, [0x4D39CC, 0x7A70FF, 0x7DB1FF]] },
  { r: "Cyberpunk hồng",  b: "Cyberpunk lam",
    red:  [45, 0xFFFFFF, [0xFF2E93, 0xFF7A00, 0xFF2E93]],
    blue: [45, 0x08323A, [0x00D4FF, 0x00FFA3, 0x00D4FF]] },
];
const DEFAULT_KIT = { 1: [60, 0xFFFFFF, [0xE84545]], 2: [60, 0xFFFFFF, [0x4F8EF7]] };

S.kitCur = S.kitCur || { 1: DEFAULT_KIT[1], 2: DEFAULT_KIT[2] };
S.kitPairIdx = -1;

function applyKits() {
  for (const t of [1, 2]) room.setTeamColors(t, ...S.kitCur[t]);
}

function autoKits(forceIdx) {
  let i = forceIdx;
  if (i == null) {
    do { i = Math.floor(Math.random() * KIT_PAIRS.length); }
    while (i === S.kitPairIdx && KIT_PAIRS.length > 1);   // không lặp lại cặp vừa dùng
  }
  S.kitPairIdx = i;
  const k = KIT_PAIRS[i];
  S.kitCur = { 1: k.red, 2: k.blue };
  applyKits();
  say(`👕 Áo đấu: 🔴 ĐỎ = ${k.r}  |  🔵 XANH = ${k.b}`, C.info, "bold");
}

function resetKits() {
  S.kitCur = { 1: DEFAULT_KIT[1], 2: DEFAULT_KIT[2] };
  S.kitPairIdx = -1;
  applyKits();
}

/* ═══════════════════ 12. SỰ KIỆN PHÒNG ═══════════════════ */

on("onRoomLink", url => console.log("[VN-HAX] Link phòng:", url));

on("onPlayerJoin", pl => {
const t = Now();
const auth = pl.auth || ("c:" + pl.conn);

const ban = findBan(pl.auth, pl.conn);
if (ban) return room.kickPlayer(pl.id, `Bạn bị ban: ${ban.reason} (${ban.until ? "còn " + fmtDur(ban.until - t) : "vĩnh viễn"})`, false);

if (/^[\s\u200B-\u200F\u2028-\u202F\u2060-\u206F\uFEFF]*$/.test(pl.name)) return room.kickPlayer(pl.id, "Tên không hợp lệ.", false);
const dup = [...S.players.values()].some(x => matchKey(x.name) === matchKey(pl.name));
if (dup) return room.kickPlayer(pl.id, "Trùng tên với người đang online — hãy đổi tên.", false);

const sameIp = [...S.players.values()].filter(x => x.conn === pl.conn).length;
if (sameIp >= CONFIG.MAX_PER_IP) return room.kickPlayer(pl.id, "Quá số kết nối cho phép trên một mạng.", false);

S.joinTimes = S.joinTimes.filter(x => t - x < CONFIG.JOIN_FLOOD_WINDOW_MS); S.joinTimes.push(t);
if (S.joinTimes.length >= CONFIG.JOIN_FLOOD_COUNT) {
try { room.setRequireRecaptcha(true); S.captchaUntil = t + 2 * 60e3; } catch (e) {}
return room.kickPlayer(pl.id, "Phòng đang bị flood, thử lại sau.", false);
}

const p = { id: pl.id, name: pl.name, auth, realAuth: !!pl.auth, conn: pl.conn, role: 0, afk: false, afkSince: 0, afkWarned: false,
joinedAt: t, lastActive: t, lastChat: 0, lastCmd: 0, msgTimes: [], lastNorm: "", strikes: 0, lastStrike: 0, autoMutes: 0, acts: [] };
S.players.set(pl.id, p);
checkAndApplyEloRank(p, true);

const saved = pl.auth && Store.data.staff[pl.auth];
if (saved) { setRole(p, saved.role, false); announceStaffWelcome(p); }

tell(p.id, "🏟️ ═══════════════════════ 🏟️", C.super, "bold");
tell(p.id,   `👋 Chào mừng ${p.name} đến với phòng! ${timeGreeting()}`, C.ok, "bold");
tell(p.id, "  ⚽ Tự động chia đội 1v1 / 2v2 / 5v5 — thắng trụ lại, thua ra hàng chờ.", C.info);
tell(p.id, "  📜 !help — trung tâm trợ giúp  |  !luat — nội quy phòng", C.info);
tell(p.id, "🏟️ ═══════════════════════ 🏟️", C.super, "small");
announceExclusiveWelcome(p);
tell(p.id, `🛡️ Uy tín của bạn: ${repOf(econ(p.auth, p.name))}/${REP_MAX} (AFK/thoát giữa trận: -1, hết uy tín bị ban ${REP_BAN_MIN} phút | đá xong 1 trận: +1)`, C.info, "bold");
{
const e = econ(p.auth, p.name);
const today = new Date().toDateString(), yesterday = new Date(t - 86400e3).toDateString();
if (e.lastLoginDate !== today) {
e.loginStreak = e.lastLoginDate === yesterday ? (e.loginStreak || 0) + 1 : 1;
e.lastLoginDate = today; Store.touch();
if (!e.loginMg) e.loginMg = [];
for (const [th, title] of LOGIN_STREAK_MILESTONES) {
if (e.loginStreak >= th && !e.loginMg.includes(th)) {
e.loginMg.push(th); grantTitle(e, title); Store.touch();
tell(p.id, `🔥 Bạn đã ghé phòng ${th} ngày liên tiếp — nhận danh hiệu [${title}]!`, C.super, "bold");
}
}
}
}

syncQueue();
if (S.picking.active) onPickChange(pl.id);     // người mới vào khi đang chọn → cập nhật số thứ tự hàng chờ
if (!running()) { if (!S.planTimer) schedulePlan(1000); } else scheduleBalance();
});
const REP_MAX = 3, REP_BAN_MIN = 10;
const repOf = e => { if (e.rep == null) e.rep = REP_MAX; return e.rep; };

// Trừ 1 uy tín khi đang thi đấu mà thoát / bị AFK. `pl.team` phải là đội TRƯỚC khi bị đẩy ra spec.
function repPenalty(pl, p, why = "thoát giữa trận") {
    if (!running() || !pl.team || S.mode === "practice" || !S.match || S.result || S.pen) return;
    const e = econ(p.auth, p.name);
    e.rep = repOf(e) - 1;
    Store.touch();
    if (e.rep <= 0) {
        e.rep = REP_MAX;                                   // hết hạn ban thì có lại đủ uy tín
        say(`🚫 ${p.name} ${why} và đã HẾT UY TÍN — bị ban ${REP_BAN_MIN} phút!`, C.err, "bold", 1);
        addBan(p, REP_BAN_MIN, `${why} (hết uy tín)`, "System");
    } else {
        say(`⚠️ ${p.name} ${why} — bị trừ 1 uy tín (còn ${e.rep}/${REP_MAX}).`, C.warn, "bold");
    }
}

// Hồi 1 uy tín cho mỗi người đã đá trọn một trận (không AFK, trận đủ dài, không phải practice)
function repRegen(res) {
    if (!res || S.mode === "practice" || res.time < CONFIG.MIN_RATED_SECONDS) return;
    for (const pl of list()) {
        if (pl.team === 0) continue;
        const p = rec(pl.id);
        if (!p || p.afk) continue;
        const e = econ(p.auth, p.name);
        if (repOf(e) >= REP_MAX) continue;
        e.rep++;
        Store.touch();
    }
}
on("onPlayerLeave", pl => {
const p = rec(pl.id);
if (!p) return;

if (pl.team !== 0 && !p.afk) eloLeavePenalty(pl.team, p, "QUIT");
S.players.delete(pl.id); S.buffs.delete(pl.id);
repPenalty(pl, p);
S.gone.set(pl.id, { rec: p, t: Now() });
S.queue = S.queue.filter(x => x !== pl.id);
if (S.pen) { if (S.players.size === 0) abortPen("phòng trống"); else penPlayerGone(pl.id); return; }

// [FIX 7] Phòng trống: xoá luôn phiên phục hồi đang dở
if (S.players.size === 0) { cancelPick(); S.recovery = null; if (running()) room.stopGame(); S.queue = []; return; }

// Đang chọn người: xử lý đội trưởng / thành viên thoát giữa chừng
if (S.picking.active) { onPickChange(pl.id); return; }
if (pl.team !== 0 && S.mode !== "practice" && (running() || S.recovery)) { onTeamMemberLeftMidMatch(pl, p); return; }
if (running()) scheduleBalance(); else if (!S.planTimer) schedulePlan(1000);
});

on("onPlayerChat", (pl, raw) => {
    const p = rec(pl.id); if (!p) return false;
    const msg = String(raw).trim(); if (!msg) return false;
    p.lastActive = Now();

    // ---- BẮT SỐ CHỌN NGƯỜI (ĐỘI TRƯỞNG, LUÂN PHIÊN) ----
    if (handlePickChat(pl.id, msg)) return false;
    // ----------------------------------------------------

    if (msg[0] === "!") { runCommand(p, msg); return false; }
    const bad = filterChat(p, msg);
    if (bad) { tell(p.id, "🚫 " + bad, C.err); return false; }

    if (p.role >= 1) { say(`${BADGE[p.role]}${p.name}: ${msg}`, p.role === 2 ? C.super : C.admin, "normal", 1); return false; }

const e = econ(p.auth, p.name);
const ex = e.exclusive;
const rankTag = e.eloRank ? `[${e.eloRank}] ` : "";
const titleTag = ex ? `[${ex.title}]  ` : (e.title ? `[${e.title}]  ` : "");
const color = ex ? ex.color : (e.chatColor || e.eloColor || C.sys);
say(`${rankTag}${titleTag}${p.name}: ${msg}`, color, e.boldChat || ex ? "bold" : "normal", 1);
    return false;
}, false);

on("onPlayerActivity", pl => { const p = rec(pl.id); if (p) { p.lastActive = Now(); p.afkWarned = false; } });

on("onPlayerBallKick", pl => {
if (S.pen) return penKick(pl);
trackKick(pl);                                                                       S.kickoffWaiting = false;
S.touches.push({ id: pl.id, team: pl.team, t: Now() });
if (S.touches.length > 5) S.touches.shift();
const p = rec(pl.id); if (p) p.lastActive = Now();
if (S.match && S.mode !== "practice" && pl.team) {
  const r = rec(pl.id);
  if (r && inOwnBox(pl.team, room.getDiscProperties(0))) {
    const x = S.match.pl[r.auth] || (S.match.pl[r.auth] = { name: r.name, g: 0, a: 0, og: 0 });
    x.sv = (x.sv || 0) + 1;
  }
}
});

on("onPlayerTeamChange", (ch, by) => {
const p = rec(ch.id); if (!p) return;
if (ch.team !== 0) { p.afk = false; p.lastActive = Now(); }
syncQueue();
if (S.pen) { penTeamChange(ch); return; }
onTeamChangeEvent(ch, by);if (by) scheduleBalance();
});

on("onPlayerAdminChange", (ch, by) => {
const p = rec(ch.id); if (!p) return;
const should = p.role >= 1;
if (ch.admin !== should) {
room.setPlayerAdmin(ch.id, should);
if (by) { audit(`${by.name} thay đổi admin gốc của ${ch.name} → bị hoàn tác`); const a = rec(by.id); if (a && a.role < 2) tell(a.id, "⛔ Hãy dùng !setadmin (Super Admin).", C.err); }
}
});

on("onPlayerKicked", (kicked, reason, ban, by) => {
if (!by) return;
const actor = rec(by.id);
const g = S.gone.get(kicked.id);
const victim = rec(kicked.id) || (g && g.rec);
if (!actor || !victim) return;

if (!(actor.role > victim.role)) {
if (ban) { try { room.clearBan(kicked.id); } catch (e) {} }
audit(`LẠM QUYỀN: ${actor.name} kick/ban ${victim.name} (${ROLE_NAME[victim.role]})`);
alertSupers(`${actor.name} đã kick/ban ${victim.name} trái quyền — đã hoàn tác.`);
if (actor.role < 2) { setRole(actor, ROLE.USER); say(`🛑 ${actor.name} bị tước quyền vì lạm dụng kick/ban.`, C.err); }
return;
}

if (actor.role === ROLE.ADMIN && !guard(actor)) { if (ban) { try { room.clearBan(kicked.id); } catch (e) {} } return; }

if (ban) {
try { room.clearBan(kicked.id); } catch (e) {}
addBan(victim, actor.role >= 2 ? 0 : CONFIG.ADMIN_MAX_BAN_MIN, reason || "Ban qua giao diện", actor.name, false);
}
audit(`${actor.name} ${ban ? "BAN" : "KICK"} (UI) ${victim.name}: ${reason || ""}`);
});

on("onStadiumChange", (name, by) => {
if (!by) return;
S.mapKey = null;
const a = rec(by.id); audit(`${a ? a.name : "?"} đổi map thủ công → khôi phục`);
say("🗺️ Bản đồ chỉ do hệ thống điều khiển — đang khôi phục.", C.warn);
restartRound();
});

on("onGameStart", () => {
if (S.pen) { penOnGameStart(); return; }
if (guardGameStart()) return;
S.otSaid = false;S.match = { t0: Now(), rated: S.mode !== "practice", pl: {}, warned30: false, bigGapAnnounced: false };
S.match.starters = {};
for (const pl of list()) if (pl.team !== 0) S.match.starters[rec(pl.id).auth] = true;
S.touches = [];S.kicks = []; S.kickoffWaiting = true; S.kickoffTeam = 1;
S.paused = false;
S.bets = { winner: {}, scorer: {} }; S.betsLocked = false;
const t = Now(); for (const p of S.players.values()) { p.lastActive = t; p.afkWarned = false; }
if (S.buffs.size || S.ballBuff) unrate("có buff");
reapplyBuffs();

say(`🎮 Bắt đầu — ${MODE_INFO[S.mode].label}${S.mode === "practice" ? " (không tính thống kê)" : ""}!`, C.ok, "bold", 1);
if (S.mode !== "practice") {
say(pickRandom(MATCH_SLOGANS), C.super, "bold");
}
});

on("onPositionsReset", () => {
if (S.pen) return;                                                                         S.kickoffWaiting = true; S.touches = [];S.kicks = [];
reapplyBuffs();
});

const GOAL_HYPE = [
"SIÊU PHẨM không thể cản phá! 🔥", "Thủ môn đứng hình nhìn bóng bay vào lưới! 😱",
"Đẳng cấp mãi mãi là đẳng cấp! 👑", "Cả sân như muốn vỡ oà! 👏",
"Một pha xử lý như trong mơ! 🌙", "Cú sút như đại bác, không có cửa cản! 💣",
"Kỹ thuật điêu luyện, khung thành rung chuyển! 🎯", "Nhanh như chớp, đối thủ chỉ biết đứng nhìn! ⚡",
"Xứng đáng lên top bàn thắng đẹp nhất tuần! 📸", "Một tuyệt phẩm sân cỏ! 🎨",
];

const OWN_GOAL_HYPE = [
"trong nỗi đau khôn nguôi! 😭", "khiến cả đội chết lặng! 💔",
"một khoảnh khắc đắng lòng không ai muốn nhớ! 🤦", "và khán đài vỗ tay... đầy mỉa mai! 😅",
];

function pickRandom(arr) { return arr[Math.floor(Math.random() * arr.length)]; }
/* ═══════════════════ HIỆU ỨNG GHI BÀN (từ World Class) ═══════════════════ */
const FX = { ENABLED: true, MIN_WEIGHT: 4, AVATAR_MS: 4000, FLASH_COUNT: 10, FLASH_MS: 200 };  // 4 = World Class
const FX_COL = { 1: 0xFF5C5C, 2: 0x5C9DFF };
const FW_COLORS = [0xFFD700, 0xFF4D4D, 0x4DA6FF, 0x7CFC98, 0xFF69B4, 0xFFFFFF, 0xFFA500];
const roleAvatar = p => (p.role === 2 ? "👑" : p.role === 1 ? "⚡" : null);
S.fx = { tok: 0, timers: [], undo: [] };

// Chỉ người có rank >= World Class mới có hiệu ứng
function fxEligible(p) {
  if (!p) return false;
  const e = econ(p.auth, p.name);
  return getRankInfo(e.eloRank).weight >= FX.MIN_WEIGHT;
}

function fxClear() {
  S.fx.tok++;
  S.fx.timers.forEach(clearTimeout); S.fx.timers = [];
  S.fx.undo.forEach(f => { try { f(); } catch (e) {} }); S.fx.undo = [];
}
function fxLater(ms, fn) {
  const tok = S.fx.tok;
  S.fx.timers.push(setTimeout(() => {
    try { if (tok === S.fx.tok) fn(); } catch (e) { logErr("fx", e); }
  }, ms));
}
function fxAvatar(id, emoji, ms) {
  if (!rec(id)) return;
  try { room.setPlayerAvatar(id, emoji); } catch (e) { return; }
  const undo = () => { const q = rec(id); if (q) try { room.setPlayerAvatar(id, roleAvatar(q)); } catch (e) {} };
  S.fx.undo.push(undo);
  fxLater(ms, () => { undo(); S.fx.undo = S.fx.undo.filter(f => f !== undo); });
}
function fxFlash(target, colors, count, gap) {
  const get = () => target === "ball" ? room.getDiscProperties(0) : room.getPlayerDiscProperties(target);
  const set = o => target === "ball" ? room.setDiscProperties(0, o) : room.setPlayerDiscProperties(target, o);
  let orig;
  try { const d = get(); if (!d) return; orig = d.color; } catch (e) { return; }
  const undo = () => { try { set({ color: orig }); } catch (e) {} };
  S.fx.undo.push(undo);
  for (let i = 0; i < count; i++)
    fxLater(i * gap, () => { if (running()) set({ color: colors[i % colors.length] }); });
  fxLater(count * gap, () => { undo(); S.fx.undo = S.fx.undo.filter(f => f !== undo); });
}

function fwPark(off, n) {
  const base = FW_BASE[S.mapKey]; if (base == null) return;
  for (let i = 0; i < n; i++)
    try { room.setDiscProperties(base + off + i, { x: 0, y: 3000, xspeed: 0, yspeed: 0, ygravity: 0, radius: 2 }); } catch (e) {}
}
function fwBurst(cx, cy, off, n, cols, speed) {
  const base = FW_BASE[S.mapKey];
  if (base == null || !running()) return;
  for (let i = 0; i < n; i++) {
    const ang = (i / n) * Math.PI * 2 + Math.random() * 0.3, v = speed * (0.5 + Math.random() * 0.7);
    try {
      room.setDiscProperties(base + off + i, {
        x: cx, y: cy, xspeed: Math.cos(ang) * v, yspeed: Math.sin(ang) * v,
        xgravity: 0, ygravity: 0.02, damping: 0.97, radius: 3.5, color: cols[i % cols.length],
      });
    } catch (e) {}
  }
  fxLater(450, () => { for (let i = 0; i < n; i++) try { room.setDiscProperties(base + off + i, { radius: 1.5 }); } catch (e) {} });
  fxLater(900, () => fwPark(off, n));
}
function fireworks(team) {
  if (S.mode === "practice" || FW_BASE[S.mapKey] == null) return;
  const geo = GEO[mapKeyOf(S.mode)] || GEO["2v2"];
  const b = room.getDiscProperties(0), sign = team === 1 ? 1 : -1;
  const cx = b ? clamp(b.x, -geo.W, geo.W) : sign * geo.W;
  const cy = b ? clamp(b.y, -geo.H, geo.H) : 0;
  const mine = team === 1 ? [0xFF4D4D, 0xFFD700, 0xFFFFFF] : [0x4DA6FF, 0xFFD700, 0xFFFFFF];
  S.fx.undo.push(() => fwPark(0, FW_N));
  fwBurst(cx, cy, 0, 20, mine.concat(FW_COLORS), 5);
  fxLater(400, () => fwBurst(cx * 0.6, -geo.H * 0.5, 20, 20, FW_COLORS, 4));
  fxLater(800, () => fwBurst(cx * 0.6,  geo.H * 0.5, 40, 20, FW_COLORS, 4));
}
const FX_EZ_OFF = 60, FX_EZ_N = 20;
const FX_STYLES = ["classic", "sword", "ez", "ring",
  "giant", "freeze", "quake", "rainbow", "pulse", "dance", "spin", "mexican", "emoji"];
FX.STYLE = "random";                           // random | classic | sword | ez
const EXPLODE_PUSH_BALL = false;               // true = nổ cũng hất cả bóng
 
// ── Chữ "EZ" dạng chấm (E: 11 chấm + Z: 9 chấm = 20 đĩa) ──
const EZ_PIXELS = (() => {
  const E = ["###", "#..", "###", "#..", "###"];
  const Z = ["###", "..#", ".#.", "#..", "###"];
  const out = [];
  [[E, 0], [Z, 4]].forEach(([g, c0]) =>
    g.forEach((row, r) => [...row].forEach((ch, c) => { if (ch === "#") out.push([c0 + c, r]); })));
  return out;
})();
 
// ── Thanh kiếm: [dx, dy, màu, bán kính], mũi kiếm ở (0,0), cán phía trên ──
// Viền (tối) vẽ trước, ruột (sáng) vẽ đè lên → có nét viền
const FX_SWORD_OFF = 0, FX_SWORD_N = 110;
const FX_SPARK_OFF = 110, FX_SPARK_N = 20;     // tia lửa lúc kiếm cắm xuống
const SWORD_SCALE = 2.2;                       // độ lớn thanh kiếm
const SWORD_STEP = 3;                          // khoảng cách chấm (đơn vị gốc) — nhỏ = liền nét hơn
const SWORD_ANGLE_DEG = 55;                    // góc cắm: 90 = thẳng đứng
const SWORD_FLY_DIST = 560;                    // quãng đường bay trước khi chạm sân
const SWORD_FLY_SPEED = 11;                    // tốc độ bay (px / tick)
const SWORD_STUCK_MS = 450;                    // kiếm rung bao lâu trước khi nổ

// Mũi kiếm ở (0,0), cán nằm phía y âm. r = bán kính chấm, step = khoảng cách chấm.
const SWORD_SHAPES = (() => {
  const hex = (cx, cy, r) => {
    const a = [];
    for (let i = 0; i <= 6; i++) { const t = Math.PI / 3 * i + Math.PI / 6; a.push([cx + r * Math.cos(t), cy + r * Math.sin(t)]); }
    return a;
  };
  const STEEL = 0xFFFFFF, ICE = 0x7FD4FF, GOLD = 0xFFD700, GRIP = 0x8D6E63;
  return [
    { color: STEEL, r: 3,   step: SWORD_STEP, pts: [[6, -64], [6, -12], [0, 0], [-6, -12], [-6, -64]] }, // viền lưỡi (hở đầu trên, chắn kiếm che)
    { color: ICE,   r: 2.2, step: SWORD_STEP, pts: [[0, -14], [0, -60]] },                               // rãnh giữa lưỡi
    { color: GOLD,  r: 4,   step: 4.5,        pts: [[-20, -68], [20, -68]] },                            // chắn kiếm
    { color: GRIP,  r: 3.5, step: 4,          pts: [[0, -73], [0, -95]] },                               // cán
    { color: GOLD,  r: 4,   step: 4,          pts: hex(0, -102, 6) },                                    // đuôi kiếm (vòng)
    { color: STEEL, r: 3,   step: 4,          pts: [[0, -102], [0, -102]] },                             // đuôi kiếm (tâm)
  ];
})();

function samplePoly(pts, step) {
  const out = [pts[0]];
  let carry = 0;
  for (let i = 1; i < pts.length; i++) {
    const [x0, y0] = pts[i - 1], [x1, y1] = pts[i], len = Math.hypot(x1 - x0, y1 - y0);
    if (len < 1e-6) continue;
    let d = step - carry;
    while (d <= len + 1e-6) { out.push([x0 + (x1 - x0) * d / len, y0 + (y1 - y0) * d / len]); d += step; }
    carry = len - (d - step);
  }
  const f = pts[0], l = pts[pts.length - 1];
  if (f[0] === l[0] && f[1] === l[1] && out.length > 1) {
    const e = out[out.length - 1];
    if (Math.hypot(e[0] - f[0], e[1] - f[1]) < step * 0.6) out.pop();
  }
  return out;
}

const SWORD_DOTS = (() => {
  const dots = [];
  for (const s of SWORD_SHAPES)
    for (const p of samplePoly(s.pts, s.step)) dots.push({ x: p[0], y: p[1], color: s.color, r: s.r });
  return dots.slice(0, FX_SWORD_N);
})();
// ── 🎆 Chữ EZ nhấp nháy ──
function ezText(cx, cy, team) {
  const base = FW_BASE[S.mapKey];
  if (base == null || !running()) return;
  const gap = 16, cols = [0xFFD700, 0xFFFFFF, FX_COL[team]];
  EZ_PIXELS.forEach(([c, r], i) => {
    try {
      room.setDiscProperties(base + FX_EZ_OFF + i, {
        x: cx + (c - 3) * gap, y: cy + (r - 2) * gap, xspeed: 0, yspeed: 0,
        xgravity: 0, ygravity: 0, damping: 1, radius: 5, color: 0xFFD700,
      });
    } catch (e) {}
  });
for (let k = 1; k <= 4; k++)
    fxLater(k * 450, () => {
      EZ_PIXELS.forEach((_, i) => {
        try { room.setDiscProperties(base + FX_EZ_OFF + i, { color: cols[(k + i) % cols.length] }); } catch (e) {}
      });
    });
  fxLater(2600, () => fwPark(FX_EZ_OFF, FX_EZ_N));
  S.fx.undo.push(() => fwPark(FX_EZ_OFF, FX_EZ_N));
}
 
function ezFx(team) {
  fireworks(team);                                 // pháo hoa sẵn có
  const geo = GEO[mapKeyOf(S.mode)] || GEO["2v2"];
  const b = room.getDiscProperties(0), sign = team === 1 ? 1 : -1;
  const cx = clamp((b ? b.x : sign * geo.W) - sign * 100, -geo.W * 0.8, geo.W * 0.8);
  const cy = b ? clamp(b.y, -geo.H * 0.4, geo.H * 0.4) : 0;
  ezText(cx, cy, team);
}
 
// ── Vụ nổ: 1 đợt hạt + hất người chơi khác ra xa (mạnh) ──
function explodeAt(cx, cy, excludeId, R) {
  fwBurst(cx, cy, 0, 20, [0xFFA500, 0xFF4500, 0xFFD700, 0xFFFFFF], 8);
  R = R * 1.3;                                      // vùng ảnh hưởng rộng hơn
  const push = (d, setFn) => {
    let dx = d.x - cx, dy = d.y - cy, dist = Math.hypot(dx, dy);
    if (dist >= R) return;
    if (dist < 1) { const a = Math.random() * Math.PI * 2; dx = Math.cos(a); dy = Math.sin(a); dist = 1; }
    const f = 8 + 14 * (1 - dist / R);              // trước là 3 + 7 → giờ 8 đến 22, càng gần tâm càng mạnh
    try { setFn({ xspeed: d.xspeed + dx / dist * f, yspeed: d.yspeed + dy / dist * f }); } catch (e) {}
  };
  for (const pl of list()) {
    if (pl.team === 0 || pl.id === excludeId) continue;
    const d = room.getPlayerDiscProperties(pl.id);
    if (d) push(d, o => room.setPlayerDiscProperties(pl.id, o));
  }
  if (EXPLODE_PUSH_BALL) { const b = room.getDiscProperties(0); if (b) push(b, o => room.setDiscProperties(0, o)); }
} 
// ── 🌊 Sóng xung kích: 2 vòng hạt + 1 lần nổ đẩy lùi (đĩa 0-47) ──
const RING_PUSH_MIN = 14;       // lực ở rìa vùng nổ
const RING_PUSH_MAX = 34;       // lực sát tâm
const RING_PUSH_SELF = true;    // true = đẩy cả người vừa ghi bàn

function shockPush(cx, cy, excludeId, R, fMin, fMax) {
  const push = (d, setFn) => {
    let dx = d.x - cx, dy = d.y - cy, dist = Math.hypot(dx, dy);
    if (dist >= R) return;
    if (dist < 1) { const a = Math.random() * Math.PI * 2; dx = Math.cos(a); dy = Math.sin(a); dist = 1; }
    const f = fMin + (fMax - fMin) * (1 - dist / R);
    try { setFn({ xspeed: d.xspeed + dx / dist * f, yspeed: d.yspeed + dy / dist * f }); } catch (e) {}
  };
  for (const pl of list()) {
    if (pl.team === 0 || pl.id === excludeId) continue;
    const d = room.getPlayerDiscProperties(pl.id);
    if (d) push(d, o => room.setPlayerDiscProperties(pl.id, o));
  }
  if (EXPLODE_PUSH_BALL) { const b = room.getDiscProperties(0); if (b) push(b, o => room.setDiscProperties(0, o)); }
}

function ringFx(team, shooter) {
  const base = FW_BASE[S.mapKey];
  if (base == null || !running()) return;
  const geo = GEO[mapKeyOf(S.mode)] || GEO["2v2"];
  const box = PEN_BOX[mapKeyOf(S.mode)], gx = box ? box.gx : geo.W;
  const cx = (team === 1 ? 1 : -1) * gx * 0.9, cy = 0;       // ĐỎ ghi → lưới bên phải
  const R = clamp(gx * 0.55, 200, 420);                       // vùng nổ co giãn theo cỡ sân
  const cols = [0xFFD700, FX_COL[team], 0xFFFFFF];
  const skip = (!RING_PUSH_SELF && shooter) ? shooter.id : -1;
  S.fx.undo.push(() => fwPark(0, 48));
  const ring = (off, speed) => {
    for (let i = 0; i < 24; i++) {
      const a = i / 24 * Math.PI * 2;
      try {
        room.setDiscProperties(base + off + i, {
          x: cx, y: cy, xspeed: Math.cos(a) * speed, yspeed: Math.sin(a) * speed,
          xgravity: 0, ygravity: 0, damping: 0.965, radius: 3.5, color: cols[i % cols.length],
        });
      } catch (e) {}
    }
  };
  say("💥 S Ó N G   X U N G   K Í C H !", 0x00E5FF, "bold", 2);
  ring(0, 9);
  shockPush(cx, cy, skip, R, RING_PUSH_MIN, RING_PUSH_MAX);   // 1 lần nổ duy nhất
  fxLater(250, () => ring(24, 7));                            // vòng 2 chỉ để làm đẹp
  fxLater(1400, () => fwPark(0, 48));
}

// ── ⚔️ Kiếm rơi xuống chỗ bóng vừa vào lưới rồi nổ ──
// ── ⚔️ Kiếm bay tới chỗ bóng vừa vào lưới, cắm xuống, rung, bắn tia lửa rồi nổ ──
function swordStrike(shooter, team) {
  const base = FW_BASE[S.mapKey];
  if (base == null || !running()) return;
  const geo = GEO[mapKeyOf(S.mode)] || GEO["2v2"];
  const box = PEN_BOX[mapKeyOf(S.mode)];
  const gx = box ? box.gx : geo.W;

  // Khung thành bị ghi: đội ĐỎ ghi → lưới bên phải (+), XANH ghi → lưới bên trái (-)
  const side = team === 1 ? 1 : -1;
  const tx = side * gx, ty = 0;

  // Kiếm bay từ phía ngoài sân vào khung thành theo đường chéo
  const rad = SWORD_ANGLE_DEG * Math.PI / 180;
  const dx = -side * Math.cos(rad) * -1, dy = Math.sin(rad);
  const cs = dy, sn = -dx, sc = SWORD_SCALE;
  const at = (px, py, d) => [px + (d.x * cs - d.y * sn) * sc, py + (d.x * sn + d.y * cs) * sc];

  const sx = tx - dx * SWORD_FLY_DIST, sy = ty - dy * SWORD_FLY_DIST;
  const ms = Math.round(SWORD_FLY_DIST / SWORD_FLY_SPEED * 1000 / 60);
  const first = base + FX_SWORD_OFF;

  SWORD_DOTS.forEach((d, i) => {
    const [x, y] = at(sx, sy, d);
    try {
      room.setDiscProperties(first + i, {
        x, y, xspeed: dx * SWORD_FLY_SPEED, yspeed: dy * SWORD_FLY_SPEED,
        xgravity: 0, ygravity: 0, damping: 1, radius: d.r, color: d.color,
      });
    } catch (e) {}
  });
  S.fx.undo.push(() => { fwPark(FX_SWORD_OFF, FX_SWORD_N); fwPark(FX_SPARK_OFF, FX_SPARK_N); });

  // Chạm khung thành → nổ ngay, không cắm, không rung
  fxLater(ms, () => {
    fwPark(FX_SWORD_OFF, FX_SWORD_N);
    say("💥 ⚔️ BÙMMM!!!", 0xFF8C00, "bold", 2);
    explodeAt(tx, ty, shooter ? shooter.id : -1, clamp(geo.W * 0.55, 220, 380));
  });
}
/* ═══════════ HIỆU ỨNG NHẸ (không dùng đĩa hạt, chạy được cả practice) ═══════════ */
const RAINBOW = [0xFF0000, 0xFF7F00, 0xFFFF00, 0x00FF00, 0x00BFFF, 0x4B00FF, 0x9400D3];
const ballOrig = () => ORIGINAL_BALL[mapKeyOf(S.mode)] || BALL_ENGINE_DEFAULT;

// 🗿 Khổng lồ + bóng nhấp nhô
function giantFx(shooter) {
  const id = shooter.id, orig = ballOrig(), ball = S.mode !== "practice";
  const undo = () => {
    try { room.setPlayerDiscProperties(id, PLAYER_RESET); } catch (e) {}
    if (ball) try { room.setDiscProperties(0, { radius: orig.radius }); } catch (e) {}
    reapplyBuffs();
  };
  try { room.setPlayerDiscProperties(id, { radius: 22 }); } catch (e) {}
  S.fx.undo.push(undo);
  if (ball) for (let i = 0; i < 8; i++)
    fxLater(i * 200, () => { try { room.setDiscProperties(0, { radius: i % 2 ? orig.radius : orig.radius * 2 }); } catch (e) {} });
  fxLater(2000, undo);
}

// 🥶 Đóng băng đội bị thủng lưới
function freezeFx(team) {
  const ids = teamIds(3 - team);
  const undo = () => {
    for (const id of ids) try { room.setPlayerDiscProperties(id, { damping: PLAYER_RESET.damping }); } catch (e) {}
    reapplyBuffs();
  };
  for (const id of ids) {
    try { room.setPlayerDiscProperties(id, { damping: 0.5 }); } catch (e) {}
    fxAvatar(id, "🥶", 2000);
  }
  S.fx.undo.push(undo);
  fxLater(2000, undo);
}

// 📳 Rung sân
function quakeFx() {
  say("📳 RUNG CHUYỂN SÂN CỎ!", 0xFFA500, "bold", 2);
  for (let i = 0; i < 6; i++)
    fxLater(i * 150, () => {
      for (const pl of list()) {
        if (pl.team === 0) continue;
        const d = room.getPlayerDiscProperties(pl.id); if (!d) continue;
        const a = Math.random() * Math.PI * 2;
        try { room.setPlayerDiscProperties(pl.id, { xspeed: d.xspeed + Math.cos(a) * 2, yspeed: d.yspeed + Math.sin(a) * 2 }); } catch (e) {}
      }
    });
}

// 🌈 Cầu vồng
function rainbowFx(shooter) {
  fxFlash(shooter.id, RAINBOW, 24, 100);
  if (S.mode !== "practice") fxFlash("ball", RAINBOW, 24, 100);
  say(`🌈 ${shooter.name} rực rỡ sắc màu!`, 0xFF69B4, "bold", 1);
}

// ❤️ Nhịp tim
function pulseFx(shooter) {
  const id = shooter.id;
  const undo = () => { try { room.setPlayerDiscProperties(id, { radius: PLAYER_RESET.radius }); } catch (e) {} reapplyBuffs(); };
  S.fx.undo.push(undo);
  fxAvatar(id, "❤️", 2400);
  for (let i = 0; i < 8; i++)
    fxLater(i * 300, () => { if (running()) try { room.setPlayerDiscProperties(id, { radius: i % 2 ? 15 : 24 }); } catch (e) {} });
  fxLater(2400, undo);
}

// 🕺 Nhảy múa
function danceFx(shooter) {
  const id = shooter.id;
  fxAvatar(id, "🕺", 2400);
  for (let i = 0; i < 12; i++)
    fxLater(i * 200, () => {
      if (!running()) return;
      try { room.setPlayerDiscProperties(id, { xspeed: (i % 4 < 2 ? 1 : -1) * 2.5, yspeed: (i % 2 ? 1 : -1) * 3.5 }); } catch (e) {}
    });
}

// 🌀 Xoay vòng
function spinFx(shooter) {
  const id = shooter.id;
  fxAvatar(id, "🌀", 2400);
  for (let i = 0; i < 24; i++)
    fxLater(i * 100, () => {
      if (!running()) return;
      const a = i * Math.PI / 6;
      try { room.setPlayerDiscProperties(id, { xspeed: Math.cos(a) * 4, yspeed: Math.sin(a) * 4 }); } catch (e) {}
    });
}

// 🙌 Sóng người (chạy từ trái sang phải sân)
function mexicanFx() {
  const all = list().filter(p => p.team !== 0)
    .map(p => ({ id: p.id, x: (room.getPlayerDiscProperties(p.id) || { x: 0 }).x }))
    .sort((a, b) => a.x - b.x);
  all.forEach((o, i) => fxLater(i * 180, () => fxAvatar(o.id, "🙌", 700)));
}

// 🎉 Avatar đổi liên tục
function emojiFx(shooter) {
  const id = shooter.id, seq = ["⚽", "🔥", "💥", "👑", "🎉", "🏆"];
  const undo = () => { const q = rec(id); if (q) try { room.setPlayerAvatar(id, roleAvatar(q)); } catch (e) {} };
  S.fx.undo.push(undo);
  seq.concat(seq).forEach((e, i) => fxLater(i * 300, () => { try { room.setPlayerAvatar(id, e); } catch (x) {} }));
  fxLater(seq.length * 2 * 300, undo);
}

const LIGHT_FX = {   // style không cần đĩa hạt → chạy được cả practice
  giant:   (s, t) => giantFx(s),
  freeze:  (s, t) => freezeFx(t),
  quake:   ()     => quakeFx(),
  rainbow: s      => rainbowFx(s),
  pulse:   s      => pulseFx(s),
  dance:   s      => danceFx(s),
  spin:    s      => spinFx(s),
  mexican: ()     => mexicanFx(),
  emoji:   s      => emojiFx(s),
};
function goalFx(shooter, team, assist, goals, force, styleOv) {
  if (!FX.ENABLED || (!force && !fxEligible(shooter))) return;   // force = bỏ qua điều kiện rank
  fxClear();
  const col = FX_COL[team], ic = team === 1 ? "🔴" : "🔵";
  [`${ic} G O O O ${ic}`, `${ic} G O O O O O A A A ${ic}`].forEach((f, i) => fxLater(i * 350, () => say(f, col, "bold", 0)));
  fxLater(700, () => say(`⚽🔥 GOOOOOAAAL!!! ${shooter.name} 🔥⚽`, 0xFFD700, "bold", 2));
  if (goals >= 3) fxLater(1200, () => say(`🎩🔥 HAT-TRICK ${shooter.name.toUpperCase()}! 🔥🎩`, 0xFF4500, "bold", 2));
  fxAvatar(shooter.id, "⚽", FX.AVATAR_MS);
  if (assist) fxAvatar(assist.id, "🎯", FX.AVATAR_MS);
  fxFlash(shooter.id, [0xFFD700, col], FX.FLASH_COUNT, FX.FLASH_MS);
  // tính style trước, rồi mới nháy
  const pool = fxStylesFor(shooter);
  const style = styleOv || (FX.STYLE === "random"
    ? pickRandom(pool.length ? pool : ["classic"])
    : (pool.includes(FX.STYLE) ? FX.STYLE : "classic"));

  if (style !== "rainbow") {
    if (S.mode !== "practice") fxFlash("ball", [0xFFD700, col, 0xFFFFFF, col], FX.FLASH_COUNT, FX.FLASH_MS);
    fxFlash(shooter.id, [0xFFD700, col], FX.FLASH_COUNT, FX.FLASH_MS);
  }

  if (LIGHT_FX[style]) return LIGHT_FX[style](shooter, team);
  if (S.mode === "practice") return;               // các style dưới cần đĩa hạt
  if (style === "sword") swordStrike(shooter, team);
  else if (style === "ez") ezFx(team);
  else if (style === "ring") ringFx(team, shooter);
  else fireworks(team);
}

function ownGoalFx(shooter) {
  if (!FX.ENABLED || !fxEligible(shooter)) return;
  fxClear();
  fxLater(0, () => say("😭 P H Ả N   L Ư Ớ I   N H À ...", 0x999999, "bold", 0));
  fxLater(500, () => say(`💀 ${shooter.name} vừa tặng bàn thắng cho đối thủ! 💀`, 0xFFB347, "bold", 2));
  fxAvatar(shooter.id, "💀", FX.AVATAR_MS);
  fxFlash(shooter.id, [0x555555, 0x999999], 6, 250);
}
def(["fx"], 1, "!fx", "Bật/tắt hiệu ứng ghi bàn", p => {
  FX.ENABLED = !FX.ENABLED;
  say("✨ Hiệu ứng ghi bàn: " + (FX.ENABLED ? "BẬT" : "TẮT"), C.info);
});
def(["fxstyle"], 1, "!fxstyle [random|tên style]", "Chọn kiểu hiệu ứng ghi bàn", (p, a) => {
  const v = (a[0] || "").toLowerCase();
  if (!v) return tell(p.id, `✨ Hiện tại: ${FX.STYLE} | Có: random, ${FX_STYLES.join(", ")}`, C.info);
  if (v !== "random" && !FX_STYLES.includes(v))
    return tell(p.id, `⚠ Chọn: random, ${FX_STYLES.join(", ")}`, C.warn);
  FX.STYLE = v;
  say(`✨ ${p.name} đặt hiệu ứng ghi bàn: ${v.toUpperCase()}`, C.info);
});
on("onTeamGoal", team => {
if (S.pen) return penGoal(team);                                                                 const t = Now(), last = S.touches[S.touches.length - 1];
S.kickoffWaiting = true; S.kickoffTeam = 3 - team;
const sc = room.getScores(), time = sc ? Math.floor(sc.time) : 0;
const mm = String(Math.floor(time / 60)).padStart(2, "0") + ":" + String(time % 60).padStart(2, "0");

if (last && t - last.t < 12e3 && rec(last.id)) {
const shooter = rec(last.id), own = last.team !== team;
const M = S.match, add = (p, k) => bump(p, k);
if (own) {
add(shooter, "og");
const pv = S.touches[S.touches.length - 2];
if (pv && pv.team === team && rec(pv.id) && last.t - pv.t < 6e3) add(rec(pv.id), "atog");
say(`🤦 ${mm} — ${shooter.name} phản lưới nhà ${pickRandom(OWN_GOAL_HYPE)}`, C.warn, "bold", 1);
ownGoalFx(shooter);
} else {
let assist = null;
for (let i = S.touches.length - 2; i >= 0; i--) {
const a = S.touches[i];
if (a.team === team && a.id !== last.id && last.t - a.t < 6e3 && rec(a.id)) { assist = rec(a.id); break; }
if (a.team !== team) break;
}
add(shooter, "g"); if (assist) add(assist, "a");
const pv = S.touches[S.touches.length - 2];
if (pv && pv.team !== team && pv.id !== last.id && rec(pv.id) && last.t - pv.t < 6e3) add(rec(pv.id), "err");
say(`⚽ ${mm} — ${shooter.name} ghi bàn! ${pickRandom(GOAL_HYPE)}${assist ? " (kiến tạo: " + assist.name + ")" : ""}`, C.goal, "bold", 1);

// 🎙️ BLV BÌNH LUẬN BÀN THẮNG
const gCount = M && M.pl[shooter.auth] ? M.pl[shooter.auth].g : 0;
goalFx(shooter, team, assist, gCount);
if (S.mode !== "practice" && gCount === 3) {
const e = econ(shooter.auth, shooter.name); e.coins += 500; Store.touch();
say(`🎩 HAT-TRICK! ${shooter.name} vừa lập cú hat-trick trong trận — tặng nóng 500 xu! 🎉`, C.super, "bold", 2);
} else if (S.mode !== "practice" && gCount === 5) {
const e = econ(shooter.auth, shooter.name); e.coins += 1000; Store.touch();
say(`🔥 5 BÀN THẮNG! ${shooter.name} ghi liền 5 bàn trong trận — quá huyền thoại! +1000 xu! 👑`, C.super, "bold", 2);
}
}

if (S.mode !== "practice") S.betsLocked = true;
} else {
say(`⚽ ${mm} — ${team === 1 ? "ĐỎ" : "XANH"} ghi bàn!`, C.goal, "bold", 1);
if (S.mode !== "practice") S.betsLocked = true;
}                                   // ← THÊM DÒNG NÀY

if (S.mode !== "practice" && sc) {
if (S.curTimeMin > 0) {
const remain = S.curTimeMin * 60 - sc.time;
if (remain >= 0 && remain <= 15) say("🚨 BÀN THẮNG PHÚT BÙ GIỜ! Khoảnh khắc quyết định số phận trận đấu! 😱", C.warn, "bold", 1);
}
}
S.touches = [];
});

on("onTeamVictory", sc => {
if (S.pen) return;                                                                                                   if (S.mode === "practice") return;
const winner = sc.red > sc.blue ? 1 : 2;
S.result = { winner, red: sc.red, blue: sc.blue, time: sc.time };
setTimeout(() => { try { if (running() && S.result) room.stopGame(); } catch (e) {} }, 3500);
});

on("onGameStop", () => { fxClear(); if (S.pen) return; handleMatchEnd(); });on("onGamePause", () => { S.paused = true; S.pausedAt = Now(); });
on("onGameUnpause", () => {
    if (guardUnpause()) return;          // đang chờ pick đủ người → ép pause lại, giữ S.paused = true
    S.paused = false; const t = Now();
    for (const p of S.players.values()) p.lastActive = t;
});
/* ═══════════════════ 13. VÒNG GIÁM SÁT THỜI GIAN VÀ AFK ═══════════════════ */

function tick() {
const t = Now();

for (const [id, g] of S.gone) if (t - g.t > 15e3) S.gone.delete(id);
if (S.captchaUntil && t > S.captchaUntil) { try { room.setRequireRecaptcha(false); } catch (e) {} S.captchaUntil = 0; }
if (t - S.lastSave > 60e3 && (!running() || S.paused)) { Store.flush(); S.lastSave = t; }

for (const [id, b] of [...S.buffs]) if (b.until && t > b.until) {
removePlayerBuff(id); const p = rec(id); if (p) say(`⌛ Buff của ${p.name} đã hết.`, C.sys, "small");
}

if (CONFIG.AFK_FLAG_KICK_MS) for (const p of S.players.values())
if (p.afk && p.role === 0 && t - p.afkSince > CONFIG.AFK_FLAG_KICK_MS) room.kickPlayer(p.id, "AFK quá lâu.", false);
if (S.pen) return;
if (!running()) {
if (!S.planTimer && !S.picking.active && eligible().length > 0 && (!S.hold || t > S.holdUntil)) schedulePlan(1500);
return;
}

if (S.paused) {
    // đang tạm dừng để chờ pick bù người: KHÔNG tự tiếp tục (armResumeTimer sẽ tự hủy sau resumeMaxWaitMs)
    if (S.recovery && S.recovery.resume) return;
if (t - S.pausedAt > CONFIG.PAUSE_MAX_MS) { room.pauseGame(false); }
    return;
}
const geo = GEO[mapKeyOf(S.mode)] || GEO["2v2"];
checkOvertime(geo);
if (S.match && !S.match.warned30 && S.mode !== "practice") {
const sc = room.getScores();
if (sc && S.curTimeMin > 0) {
const remain = S.curTimeMin * 60 - sc.time;
if (remain <= 30 && remain > 0) { S.match.warned30 = true; say("⏰ CÒN 30 GIÂY! Dồn hết sức cho pha bóng cuối cùng! 🔥", C.warn, "bold", 1); }
}
}

for (const pl of list()) {
if (pl.team === 0) continue;
const p = rec(pl.id);
if (S.kickoffWaiting && S.mode !== "practice" && pl.team !== S.kickoffTeam) { p.lastActive = t; continue; }
const idle = t - p.lastActive;
if (idle >= CONFIG.AFK_MOVE_MS) moveAfk(p, "Bạn bị chuyển AFK vì không hoạt động");
else if (idle >= CONFIG.AFK_WARN_MS && !p.afkWarned) { p.afkWarned = true; tell(p.id, "⚠️ Bạn có vẻ đang AFK! Hãy di chuyển, nếu không sẽ bị chuyển sang khán đài.", C.warn, "bold", 2); }
}

// [FIX 5] Practice: disc0 là đĩa giả nằm ngoài sân → không kiểm tra/kéo vào sân
const b = room.getDiscProperties(0);
if (b && S.mode !== "practice") {
if (!isFinite(b.x) || !isFinite(b.y) || !isFinite(b.xspeed) || !isFinite(b.yspeed)) {
room.setDiscProperties(0, { x: 0, y: 0, xspeed: 0, yspeed: 0 }); applyBallBuff();
audit("Bóng lỗi toạ độ (NaN) → đặt lại tâm");
} else if (Math.abs(b.x) > geo.outerW + 20 || Math.abs(b.y) > geo.outerH + 20) {
room.setDiscProperties(0, {
x: clamp(b.x, -geo.outerW, geo.outerW),
y: clamp(b.y, -geo.outerH, geo.outerH),
});
} else if (S.kickoffWaiting && Math.hypot(b.xspeed, b.yspeed) > 0.3) {
S.kickoffWaiting = false;
}
}

for (const pl of list()) {
if (pl.team === 0) continue;
const d = room.getPlayerDiscProperties(pl.id);
if (d && (!isFinite(d.x) || !isFinite(d.y) || Math.abs(d.x) > geo.outerW + 10 || Math.abs(d.y) > geo.outerH + 10))
room.setPlayerDiscProperties(pl.id, { x: (pl.team === 1 ? -1 : 1) * geo.sx, y: 0, xspeed: 0, yspeed: 0 });
}
}

/* ═══════════════════ 14. MASTER TICKER (VÒNG LẶP DÙNG CHUNG - 3S/LẦN) ═══════════════════ */

let lastMinuteCheck = Now();
let last5MinCheck = Now();
let last20MinCheck = Now();
let last30MinCheck = Now();
let lastTipCheck = Now();
setInterval(() => {
try {
const t = Now();

// 1. Tick giám sát AFK & Vật lý (3s/lần)
tick();

// 3. Vòng lặp phút (Chỉ còn nhắc nhở phòng vắng & Tính phút luyện tập)
if (t - lastMinuteCheck >= 60e3) {
lastMinuteCheck = t;
for (const p of S.players.values()) {
if (p.afk || t - p.lastActive > 3 * 60e3) continue;
const e = econ(p.auth, p.name);

if (S.mode === "practice") {
const pl = room.getPlayer(p.id);
if (pl && pl.team !== 0) {
e.practiceMinutes = (e.practiceMinutes || 0) + 1;
if (!e.practiceMg) e.practiceMg = [];
for (const [th, title] of PRACTICE_MILESTONES) {
if (e.practiceMinutes >= th && !e.practiceMg.includes(th)) {
e.practiceMg.push(th); grantTitle(e, title); Store.touch();
say(`⏱️ ${p.name} đã luyện tập tổng cộng ${th} phút — nhận danh hiệu [${title}]!`, C.super, "bold");
}
}
}
}
}
if (!running() || S.paused) Store.flush();
}

// 4. Vòng lặp 5 phút (Chỉ còn cập nhật tỷ giá bán Ruby)
if (t - last5MinCheck >= 300e3) {
last5MinCheck = t;
rubySellRate = 6000 + Math.floor(Math.random() * 4000);
}
// Nhắc lệnh định kỳ (mỗi 7 phút, xoay vòng lần lượt)
if (t - lastTipCheck >= 7 * 60e3) {
  lastTipCheck = t;
  if (S.players.size > 0) {
    say(CMD_TIPS[tipIdx % CMD_TIPS.length], C.info, "small");
    tipIdx++;
  }
}
// 5. Vòng lặp 20 phút (Nhắc nội quy)
if (t - last20MinCheck >= 1200e3) {
last20MinCheck = t;
if (S.players.size > 0) say(pickRandom(RULE_REMINDERS), C.sys, "small");
}

// 6. Vòng lặp 30 phút (BXH Elo)
if (t - last30MinCheck >= 1800e3) {
last30MinCheck = t;
if (S.players.size > 0) {
const top = Object.values(Store.data.stats).filter(v => v.games > 0).sort((a, b) => b.rating - a.rating).slice(0, 3);
if (top.length) {
say("🏆 ═══════ BẢNG VÀNG HIỆN TẠI ═══════ 🏆", C.super, "bold", 1);
const medal = ["🥇", "🥈", "🥉"];
top.forEach((v, i) => say(`   ${medal[i]}  ${v.name}  —  ${v.rating} elo`, C.info, "bold"));
}
}
}

} catch (e) {
logErr("MasterTicker", e);
}
}, 3000);

/* ═══════════════════ 15. KHỞI ĐỘNG CƠ BẢN ═══════════════════ */

try {
resetKits();
room.setTeamsLock(true);
room.setKickRateLimit(4, 0, 0);
applyStadium("practice"); applyLimits("practice");
} catch (e) { logErr("khởi động", e); }
let _lt = Now();
setInterval(() => {
  const t = Now(), drift = t - _lt - 1000; _lt = t;
  if (drift > 300) console.warn(`[LAG] vòng lặp trễ ${drift}ms (tab bị throttle hoặc máy host quá tải)`);
}, 1000);
