#!/usr/bin/env python3
"""
Ambicam NETSDK v2 – Batch RTMP Enable (FINAL FIXED)
SUCCESS = statusCode == 0
"""

import csv
import time
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By

PORTAL_URL = "http://cfg.ambicam.com:5913"
AUTH_HEADER = "Basic YWRtaW46"   # admin:
INPUT_CSV = "input.csv"
OUTPUT_CSV = "output.csv"
HEADLESS = True


class AmbicamBatchRTMP:
    def __init__(self, headless=True):
        self.headless = headless
        self.driver = None

    def start_browser(self):
        opts = Options()
        if self.headless:
            opts.add_argument("--headless=new")

        opts.add_argument("--no-sandbox")
        opts.add_argument("--disable-dev-shm-usage")
        opts.add_argument("--window-size=1920,1080")

        self.driver = webdriver.Chrome(options=opts)
        self.driver.implicitly_wait(10)

        self.driver.get(PORTAL_URL)
        time.sleep(8)

    def open_camera(self, device_id):
        print(f"\n🔍 Opening device: {device_id}")

        # filter device
        for inp in self.driver.find_elements(By.TAG_NAME, "input"):
            if "filter" in (inp.get_attribute("placeholder") or "").lower():
                inp.clear()
                inp.send_keys(device_id)
                time.sleep(2)
                break

        original_windows = set(self.driver.window_handles)

        for row in self.driver.find_elements(By.TAG_NAME, "tr"):
            if device_id in row.text:
                row.find_elements(By.TAG_NAME, "i")[-1].click()
                break
        else:
            return False, "Device not found"

        time.sleep(4)

        # accept popup
        self.driver.execute_script("""
            document.querySelectorAll('button').forEach(b=>{
                if(b.innerText.includes('OK')) b.click();
            });
        """)

        time.sleep(4)

        new_window = set(self.driver.window_handles) - original_windows
        if not new_window:
            return False, "Camera window not opened"

        self.driver.switch_to.window(new_window.pop())
        time.sleep(3)

        # login
        self.driver.execute_script("document.getElementById('submit')?.click();")
        time.sleep(3)

        return True, "Camera connected"

    def set_rtmp(self, rtmp_url):
        payload = {
            "Rtmp": {
                "RtmpUrl": rtmp_url,
                "Enabled": True,
                "Stream": 1
            }
        }

        js = f"""
            return fetch('/netsdk/v2/network', {{
                method: 'PUT',
                headers: {{
                    'Authorization': '{AUTH_HEADER}',
                    'Content-Type': 'application/json'
                }},
                body: JSON.stringify(arguments[0])
            }})
            .then(r => r.text())
            .then(t => {{
                try {{ return JSON.parse(t); }}
                catch(e) {{ return {{ raw: t }}; }}
            }})
            .catch(e => ({{ error: e.message }}));
        """

        return self.driver.execute_script(js, payload)

    def close_camera(self):
        if len(self.driver.window_handles) > 1:
            self.driver.close()
            self.driver.switch_to.window(self.driver.window_handles[0])

    def shutdown(self):
        if self.driver:
            self.driver.quit()


def main():
    bot = AmbicamBatchRTMP(HEADLESS)
    bot.start_browser()

    results = []

    with open(INPUT_CSV, newline="") as f:
        rows = list(csv.DictReader(f))

    for idx, row in enumerate(rows, 1):
        device_id = row["device_id"].strip()
        rtmp_url = row["rtmp_url"].strip()

        print(f"\n[{idx}/{len(rows)}] Processing {device_id}")

        ok, msg = bot.open_camera(device_id)
        if not ok:
            print(f"❌ {msg}")
            results.append([device_id, rtmp_url, "FAIL", msg])
            continue

        res = bot.set_rtmp(rtmp_url)
        print("RTMP RESPONSE:", res)

        # ✅ FINAL SUCCESS CHECK (NETSDK v2)
        if isinstance(res, dict) and res.get("statusCode") == 0:
            status = "SUCCESS"
            message = "RTMP SET SUCCESS"
            print("✅ RTMP SET SUCCESS")
        else:
            status = "FAIL"
            message = str(res)
            print("❌ RTMP FAILED")

        results.append([device_id, rtmp_url, status, message])
        bot.close_camera()

    bot.shutdown()

    with open(OUTPUT_CSV, "w", newline="") as f:
        writer = csv.writer(f)
        writer.writerow(["device_id", "rtmp_url", "status", "message"])
        writer.writerows(results)

    print("\n📄 Output saved to output.csv")
    print("✅ Batch RTMP process completed")


if __name__ == "__main__":
    main()
