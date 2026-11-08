"""
H-HUB AI CLOUD - BLOX FRUIT MASTER COMPANION & ULTIMATE EDITION
Python standard library only. Deploy as a Render Web Service.
Start command: python main.py
"""

import os
import json
import re
import time
import difflib
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.parse import urlparse

HOST = "0.0.0.0"
PORT = int(os.environ.get("PORT", "10000"))
MAX_BODY = 256_000

# Trạng thái lưu trữ bộ nhớ trong (Session memory)
OBSERVATIONS = {}
HISTORY = {}

# ==========================================
# 1. CƠ SỞ DỮ LIỆU NHIỆM VỤ & ITEM ĐẶC BIỆT
# ==========================================
ADVANCED_QUESTS = {
    "saber": {
        "type": "item_unlock",
        "description": "Lấy kiếm Saber từ Expert Assassin ở Jungle (Sea 1)",
        "steps": [
            {"action": "move_to", "args": {"position": {"x": -1611, "y": 36, "z": 153}}, "reason": "Đến công tắc 1 tại Jungle"},
            {"action": "interact_prompt", "args": {"name": "Button1"}, "reason": "Nhấn công tắc mở cửa bí mật"}
        ]
    },
    "soul guitar": {
        "type": "puzzle_quest",
        "description": "Lấy súng Soul Guitar tại Haunted Castle (Sea 3, Đêm full moon)",
        "steps": [
            {"action": "move_to", "args": {"target_name": "Haunted Castle"}, "reason": "Di chuyển đến lâu đài ma"},
            {"action": "interact_prompt", "args": {"name": "Gravestone"}, "reason": "Cầu nguyện tại mộ lúc đêm"}
        ]
    },
    "tushita": {
        "type": "sword_quest",
        "description": "Lấy kiếm Tushita (Sea 3 - Đốt đuốc nhanh / Đánh Longma)",
        "steps": [
            {"action": "find_target", "args": {"category": "mob", "name": "Longma"}, "reason": "Tìm và tiêu diệt Longma"}
        ]
    },
    "dark dagger": {
        "type": "boss_quest",
        "description": "Săn Indra / Rip_Indra lấy Dark Dagger",
        "steps": [
            {"action": "find_target", "args": {"category": "boss", "name": "Rip_Indra"}, "reason": "Săn Boss Indra"}
        ]
    },
    "dough king": {
        "type": "raid_boss",
        "description": "Spawn Dough King (Sea 3 - Sweet Chateau)",
        "steps": [
            {"action": "accept_quest", "args": {"target": "Dough King"}, "reason": "Tham gia đánh Dough King"}
        ]
    }
}

# Cơ sở dữ liệu tư vấn & gợi ý hướng đi cho người chơi theo cấp độ/Sea
GAME_GUIDE_DATA = {
    "sea_1": "Ở Sea 1, bạn nên tập trung làm nhiệm vụ farm từ Starter Island -> Jungle -> Pirate Village -> Desert -> Snow Island -> Marine Fortress -> Prison -> Colosseum. Nhớ lấy kiếm Saber khi đạt Level 200 nhé!",
    "sea_2": "Sang Sea 2 (từ Level 700), hãy ưu tiên làm quest tại Kingdom of Rose, Green Zone, Snow Mountain, Hot and Cold, và săn Boss Don Swan hoặc làm nhiệm vụ Raid.",
    "sea_3": "Sea 3 (từ Level 1500) là nơi bạn chinh phục các vũ khí mạnh nhất như Tushita, Yama, Cursed Dual Katana, và Soul Guitar tại Haunted Castle hoặc Floating Turtle.",
    "money": "Để kiếm nhiều tiền (Beli) và Fragments nhanh, hãy chăm chỉ làm Sea Events (Săn Sea Beast), chạy Factory, hoặc tham gia Raid cùng bạn bè.",
    "fruit": "Trái ác quỷ tốt nhất cho việc cày cấp (Grind) thường là Buddha (Phật Tổ) hoặc Magma, Light ở Sea đầu."
}

# Danh sách mob thông thường theo Sea để phục vụ việc nhận diện auto quest
BLOXY_QUEST_DATABASE = {
    "sea_1": {"Bandit": 1, "Monkey": 10, "Gorilla": 20, "Pirate": 30, "Brute": 40, "Desert Bandit": 60, "Snow Bandit": 90},
    "sea_2": {"Raider": 700, "Mercenary": 725, "Swan Pirate": 775, "Factory Staff": 800, "Marine Lieutenant": 875, "Zombie": 950},
    "sea_3": {"Pirate Millionaire": 1500, "Pistol Billionaire": 1525, "Forest Pirate": 1575, "Mythological Pirate": 1600, "Reborn Skeleton": 1700}
}

ALLOWED_ACTIONS = {
    "inspect_status", "find_target", "move_to", "accept_quest",
    "collect_item", "interact_prompt", "wait", "stop", "attack_target", "solve_puzzle", "chat_message"
}

def norm(value):
    return re.sub(r"[^a-z0-9]+", " ", str(value).lower()).strip()

def match_name(query, candidates):
    if not candidates:
        return None
    q = norm(query)
    pairs = {norm(name): name for name in candidates if isinstance(name, str)}
    if q in pairs:
        return pairs[q]
    hits = difflib.get_close_matches(q, list(pairs), n=1, cutoff=0.45)
    if hits:
        return pairs[hits[0]]
    for key, original in pairs.items():
        if q and (q in key or key in q):
            return original
    return None

def get_all_known_mobs():
    mobs = []
    for sea_data in BLOXY_QUEST_DATABASE.values():
        mobs.extend(list(sea_data.keys()))
    return list(set(mobs))


# ==========================================
# 2. BỘ NÃO TỰ SUY NGHĨ & TRÒ CHUYỆN (BRAIN)
# ==========================================
def brain_process(command, env):
    raw = command.strip()
    low = raw.lower()

    player = env.get("player", {})
    level = player.get("level", 1)
    sea = env.get("world", {}).get("sea", "sea_1" if level < 700 else ("sea_2" if level < 1500 else "sea_3"))

    # Lệnh dừng khẩn cấp
    if low in {"dừng", "stop", "hủy", "dừng ai"}:
        return "🤖 H-HUB AI: Đã dừng toàn bộ tác vụ theo yêu cầu của bạn.", [
            {"action": "stop", "args": {}, "reason": "Người dùng yêu cầu dừng"}
        ]

    # Xử lý hội thoại: Xin gợi ý, tư vấn hướng đi
    if any(q in low for q in ("gợi ý", "làm gì", "tiếp theo", "tư vấn", "hướng dẫn", "nên đi đâu", "cày ở đâu")):
        guide_text = GAME_GUIDE_DATA.get(sea, "Hãy tiếp tục hoàn thành nhiệm vụ phù hợp với cấp độ.")
        reply = (
            f"🤖 H-HUB AI: Chào bạn! Dựa vào trạng thái hiện tại (Level {level}, {sea}), "
            f"gợi ý cho bạn là: {guide_text} Bạn có muốn tôi tự động làm nhiệm vụ farm không?"
        )
        return reply, [
            {"action": "inspect_status", "args": {}, "reason": "Kiểm tra chi tiết trạng thái"},
            {"action": "chat_message", "args": {"message": "AI đã tư vấn hướng đi."}, "reason": "Phản hồi trò chuyện"}
        ]

    # Xử lý hội thoại: Chào hỏi, làm quen
    if any(c in low for c in ("chào", "hi ", "hello", "ai ơi", "giúp mình", "bạn là ai")):
        return (
            f"🤖 H-HUB AI: Chào bạn! Tôi là trợ lý AI quản lý hành trình Blox Fruit. "
            f"Tôi có thể tư vấn chiến thuật, nhận diện mọi nhiệm vụ đặc biệt (Saber, Soul Guitar, Tushita,...) "
            f"hoặc tự động điều khiển nhân vật. Bạn cần giúp gì?",
            [{"action": "chat_message", "args": {"message": "Gửi lời chào người chơi."}, "reason": "Gửi lời chào"}]
        )

    # Nhận diện các nhiệm vụ đặc biệt / lấy item / mở khóa
    for quest_key, quest_info in ADVANCED_QUESTS.items():
        if quest_key in low or quest_key.replace(" ", "") in low.replace(" ", ""):
            reply = f"🤖 H-HUB AI: Đã nhận diện nhiệm vụ chuyên biệt: {quest_info['description']}."
            return reply, quest_info["steps"]

    # Xử lý nhiệm vụ Raid / Săn Boss thế giới
    if "raid" in low or "đánh boss" in low:
        boss_name = "Boss"
        if "dough" in low: boss_name = "Dough King"
        elif "indra" in low: boss_name = "Rip_Indra"
        elif "beast" in low: boss_name = "Sea Beast"
        
        return f"🤖 H-HUB AI: Đã lên kế hoạch săn Boss/Raid: {boss_name}.", [
            {"action": "find_target", "args": {"category": "boss", "name": boss_name}, "reason": "Tìm vị trí Boss"},
            {"action": "move_to", "args": {"target_name": boss_name}, "reason": "Di chuyển đến mục tiêu"},
            {"action": "attack_target", "args": {"name": boss_name}, "reason": "Tiêu diệt Boss nhận phần thưởng"}
        ]

    # Xử lý nhiệm vụ giải đố chung
    if "giải đố" in low or "mở cổng" in low or "puzzle" in low:
        return "🤖 H-HUB AI: Đang tiến hành phân tích môi trường để giải đố cơ chế nhiệm vụ.", [
            {"action": "inspect_status", "args": {}, "reason": "Quét đối tượng tương tác xung quanh"},
            {"action": "solve_puzzle", "args": {"context": low}, "reason": "Tự động tương tác giải đố"}
        ]

    # Xử lý Tự động nhiệm vụ / Farm cấp độ
    if any(x in low for x in ("tự động nhiệm vụ", "auto quest", "làm nhiệm vụ", "auto làm nhiệm vụ")):
        sea_map = BLOXY_QUEST_DATABASE.get(sea, BLOXY_QUEST_DATABASE["sea_1"])
        target_mob = list(sea_map.keys())[0]
        for mob, req_lvl in sea_map.items():
            if level >= req_lvl:
                target_mob = mob

        reply = f"🤖 H-HUB AI: Đã nhận diện Level {level} ({sea}). Tự động kích hoạt nhiệm vụ tiêu diệt: {target_mob}."
        plan = [
            {"action": "inspect_status", "args": {}, "reason": "Kiểm tra trạng thái nhân vật"},
            {"action": "accept_quest", "args": {"target": target_mob}, "reason": "Nhận quest phù hợp cấp độ"},
            {"action": "find_target", "args": {"category": "mob", "name": target_mob, "count": 1}, "reason": f"Tìm {target_mob}"},
            {"action": "move_to", "args": {"target_name": target_mob}, "reason": "Di chuyển tới vị trí quái"},
            {"action": "attack_target", "args": {"name": target_mob}, "reason": "Tiêu diệt mục tiêu nhiệm vụ"}
        ]
        return reply, plan

    # Xử lý lệnh đánh quái chỉ định thủ công
    if any(word in low for word in ("đánh", "giết", "farm")):
        all_mobs = get_all_known_mobs()
        requested = re.sub(r"\b(đánh|giết|farm|quái|con|gần nhất|rồi nhặt trái)\b", " ", low)
        requested = re.sub(r"\d+", " ", requested).strip(" ,")
        target = "nearest" if "gần nhất" in low or not requested else match_name(requested, all_mobs)
        
        count_match = re.search(r"(?:đánh|giết)\s+(\d+)", low)
        count = min(int(count_match.group(1)), 100) if count_match else 1

        if not target:
            target = "Bandit"

        plan = [
            {"action": "find_target", "args": {"category": "mob", "name": target, "count": count}, "reason": "Tìm quái vật"},
            {"action": "move_to", "args": {"target_name": target}, "reason": "Tiếp cận mục tiêu"},
            {"action": "attack_target", "args": {"name": target, "count": count}, "reason": "Tiêu diệt quái"}
        ]
        return f"🤖 H-HUB AI: Đã lên kế hoạch đánh {target} (Số lượng: {count}).", plan

    # Phản hồi mặc định thông minh (Chủ động trò chuyện + Đưa ra phương án)
    return (
        f"🤖 H-HUB AI: Tôi đã hiểu yêu cầu '{command}'. "
        f"Nhân vật đang ở Level {level}. Bạn có thể gõ 'gợi ý cho tôi', 'auto nhiệm vụ', 'lấy saber', hoặc 'đánh quái' bất cứ lúc nào!",
        [{"action": "chat_message", "args": {"message": f"Phản hồi lệnh: {command}"}, "reason": "Ghi nhận lệnh tương tác"}]
    )


# ==========================================
# 3. HTTP SERVER CHO RENDER DEPLOYMENT
# ==========================================
def send_json(handler, status, payload):
    body = json.dumps(payload, ensure_ascii=False).encode("utf-8")
    handler.send_response(status)
    handler.send_header("Content-Type", "application/json; charset=utf-8")
    handler.send_header("Content-Length", str(len(body)))
    handler.send_header("Cache-Control", "no-store")
    handler.end_headers()
    handler.wfile.write(body)

class Handler(BaseHTTPRequestHandler):
    server_version = "HHubAI-MasterUltimate/3.0"

    def log_message(self, fmt, *args):
        print("[HTTP]", fmt % args)

    def do_GET(self):
        path = urlparse(self.path).path
        if path in {"/health", "/"}:
            return send_json(self, 200, {
                "ok": True, 
                "service": "H-HUB AI Cloud - Blox Fruit Master Ultimate Edition", 
                "version": "3.0"
            })
        return send_json(self, 404, {"error": "Not found"})

    def do_POST(self):
        path = urlparse(self.path).path
        try:
            length = int(self.headers.get("Content-Length", "0"))
            if length <= 0 or length > MAX_BODY:
                raise ValueError("Body trống hoặc vượt giới hạn")
            data = json.loads(self.rfile.read(length).decode("utf-8"))
        except Exception as exc:
            return send_json(self, 400, {"error": str(exc)})

        if path == "/v1/observe":
            session = str(data.get("session_id", ""))[:100]
            if not session:
                return send_json(self, 400, {"error": "Thiếu session_id"})
            data["_received_at"] = time.time()
            OBSERVATIONS[session] = data
            return send_json(self, 200, {"ok": True, "session_id": session})

        if path == "/v1/think":
            session = str(data.get("session_id", ""))
            command = str(data.get("command", "")).strip()
            env = OBSERVATIONS.get(session, {})
            reply, plan = brain_process(command, env)
            
            # Lưu lịch sử chat/lệnh ngắn hạn
            HISTORY.setdefault(session, []).append({
                "command": command, "reply": reply, "plan": plan, "at": time.time()
            })
            HISTORY[session] = HISTORY[session][-30:]
            
            return send_json(self, 200, {"reply": reply, "plan": plan})

        return send_json(self, 404, {"error": "Unknown endpoint"})

if __name__ == "__main__":
    print(f"H-HUB AI Cloud Master Ultimate Edition listening on {HOST}:{PORT}")
    ThreadingHTTPServer((HOST, PORT), Handler).serve_forever()
