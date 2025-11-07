# kovaze_mushroom_hunter_onefile.py
# ONE-FILE VERSION - JUST SAVE & RUN!
# Copy everything below into a file named kovaze_mushroom_hunter.py
# Then: pip install aiohttp beautifulsoup4 && python kovaze_mushroom_hunter.py

import asyncio
import aiohttp
from bs4 import BeautifulSoup
import webbrowser
import time
import threading
import tkinter as tk
from tkinter import messagebox

# ==================== CONFIG ====================
START_ID = 1
END_ID = 25000
CHECK_INTERVAL = 45  # seconds between full scans
# ===============================================

def notify(title, msg):
    root = tk.Tk()
    root.withdraw()
    messagebox.showwarning(title, msg)
    root.destroy()
    print("\a" * 15)  # LOUD ALERT
    print(f"🍄🍄🍄 {title}: {msg}")

async def fetch(session, url):
    try:
        async with session.get(url, timeout=12) as resp:
            text = await resp.text()
            return text, str(resp.url)
    except:
        return None, None

async def check_blog(session, blog_id):
    url = f"https://kovaze.com/blog/{blog_id}"
    html, final_url = await fetch(session, url)
    if not html:
        return None

    # SKIP DELETED BLOGS
    if 'kovaze.com/blogs' in final_url or 'LATEST BLOGS' in html.upper():
        return None

    soup = BeautifulSoup(html, 'html.parser')

    # REAL CLICKABLE MUSHROOM ONLY
    for a in soup.find_all('a', href='/events/mushrooms'):
        if a.get_text(strip=True) == '🍄':
            # Not in Games section
            parent = a.find_parent(string=lambda t: t and 'Games' in t)
            if parent:
                continue
            return url
    return None

async def hunt():
    print("🍄 KOVAZE MUSHROOM HUNTER - ONE FILE EDITION")
    print(f"   Scanning blogs {START_ID}-{END_ID} every {CHECK_INTERVAL}s")
    print("   → Skips deleted blogs")
    print("   → GUI popup + sound + auto-open")
    print("   → ONLY real clickable 🍄\n")
    
    seen = set()

    while True:
        print(f"[{time.strftime('%H:%M:%S')}] Starting scan...")
        start = time.time()
        
        async with aiohttp.ClientSession() as session:
            tasks = []
            for i in range(START_ID, END_ID + 1):
                tasks.append(check_blog(session, i))
                if len(tasks) >= 600:  # batch for speed
                    results = await asyncio.gather(*tasks)
                    for url in results:
                        if url and url not in seen:
                            seen.add(url)
                            threading.Thread(target=notify, args=("MUSHROOM FOUND!", f"CLICK NOW → {url}")).start()
                            webbrowser.open(url)
                    tasks = []
            # final batch
            if tasks:
                results = await asyncio.gather(*tasks)
                for url in results:
                    if url and url not in seen:
                        seen.add(url)
                        threading.Thread(target=notify, args=("MUSHROOM FOUND!", f"CLICK NOW → {url}")).start()
                        webbrowser.open(url)

        print(f"Scan finished in {time.time()-start:.1f}s. Next in {CHECK_INTERVAL}s...\n")
        await asyncio.sleep(CHECK_INTERVAL)

if __name__ == "__main__":
    try:
        asyncio.run(hunt())
    except KeyboardInterrupt:
        print("\n🍄 Hunter stopped. Go get that Pink Mycena! 🩷")