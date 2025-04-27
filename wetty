import aiohttp
import asyncio
import re
from datetime import datetime, timedelta
from collections import OrderedDict

# 텔레그램 설정값
TELEGRAM_BOT_TOKEN = "1111111:2222222222222222222222222"
TELEGRAM_CHAT_ID = "333333333333333"
TELEGRAM_API_URL = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"

# 위치 API
IPINFO_API_URL = "http://ipinfo.io/{}/json"
OPENCAGE_API_URL = "https://api.opencagedata.com/geocode/v1/json"
OPENCAGE_API_KEY = "356e9f44f71d4563b706951a8dd0f6aa"

notified_ips = OrderedDict()
failed_ips = set()
IP_CACHE_DURATION = timedelta(hours=1)
MAX_CACHED_IPS = 500

client_timeout = aiohttp.ClientTimeout(total=5)

async def send_notification(ip, timestamp, location, session):
    try:
        message = (
            "⚠️ *외부 접속 감지 (Wetty)*\n"
            f"*IP*: `{ip}`\n"
            f"*위치*: {location}\n"
            f"*시간*: `{timestamp}`"
        )
        async with session.post(
            TELEGRAM_API_URL,
            data={
                "chat_id": TELEGRAM_CHAT_ID,
                "text": message,
                "parse_mode": "Markdown"
            }
        ) as response:
            print(f"[INFO] 텔레그램 전송 완료: {response.status}")
    except Exception as e:
        print(f"[ERROR] 텔레그램 전송 실패: {e}")

async def get_ip_location(ip, session):
    if ip in failed_ips:
        return "위치 정보 불가 (이전 실패)"
    try:
        async with session.get(IPINFO_API_URL.format(ip)) as response:
            if response.status == 200:
                data = await response.json()
                loc = data.get("loc")
                if loc:
                    latitude, longitude = loc.split(',')
                    return await get_address_from_coordinates(latitude, longitude, session)
                else:
                    return f"{data.get('city', '알 수 없음')}, {data.get('region', '')}, {data.get('country', '')}"
    except Exception as e:
        print(f"[ERROR] IP 위치 정보 조회 실패: {e}")

    failed_ips.add(ip)
    return "위치 정보 불가"

async def get_address_from_coordinates(lat, lon, session):
    try:
        async with session.get(OPENCAGE_API_URL, params={
            'q': f"{lat},{lon}",
            'key': OPENCAGE_API_KEY
        }) as response:
            if response.status == 200:
                data = await response.json()
                if data['results']:
                    return data['results'][0].get('formatted', '주소 정보 불가')
    except Exception as e:
        print(f"[ERROR] 주소 변환 실패: {e}")
    return "주소 정보 불가"

async def watch_logs(session):
    current_time = datetime.now().strftime("%Y-%m-%dT%H:%M:%S")

    process = await asyncio.create_subprocess_exec(
        "docker", "logs", "-f", "wetty", "--since", current_time,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.STDOUT
    )

    last_ip = None

    while True:
        line = await process.stdout.readline()
        if not line:
            await asyncio.sleep(0.3)
            continue

        line = line.decode("utf-8", errors="ignore").strip()
        try:
            ip_match = re.search(r'"x-forwarded-for":"([\d.]+)"', line)
            if ip_match:
                last_ip = ip_match.group(1)

            if "Process Started on behalf of user" in line and last_ip:
                timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

                if last_ip not in notified_ips:
                    if len(notified_ips) >= MAX_CACHED_IPS:
                        oldest_ip, _ = notified_ips.popitem(last=False)
                        print(f"[INFO] 캐시 한도 초과로 가장 오래된 IP 제거: {oldest_ip}")
                    notified_ips[last_ip] = datetime.now()
                    print(f"[INFO] Wetty 접속 감지 from {last_ip} @ {timestamp}")
                else:
                    print(f"[INFO] 기존 접속된 IP: {last_ip} @ {timestamp} (재접속 감지)")
                    notified_ips.move_to_end(last_ip)
                    notified_ips[last_ip] = datetime.now()

                location = await get_ip_location(last_ip, session)
                await send_notification(last_ip, timestamp, location, session)

        except Exception as e:
            print(f"[ERROR] 로그 처리 중 오류 발생: {e}")
            await asyncio.sleep(1)

async def cleanup_old_ips():
    while True:
        now = datetime.now()
        expired = [ip for ip, ts in notified_ips.items() if now - ts > IP_CACHE_DURATION]
        for ip in expired:
            print(f"[INFO] 오래된 IP 캐시 제거: {ip}")
            del notified_ips[ip]
        await asyncio.sleep(600)

async def main():
    async with aiohttp.ClientSession(timeout=client_timeout) as session:
        await asyncio.gather(
            watch_logs(session),
            cleanup_old_ips()
        )

if __name__ == "__main__":
    asyncio.run(main())

