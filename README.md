# kovaze_mushroom_hunter_v2.py
import asyncio
import aiohttp
from bs4 import BeautifulSoup
import webbrowser
import time
import os

# CONFIG
START_ID = 1
END_ID = 20000
CHECK_INTERVAL = 60  # seconds between full scans

async def fetch(session, url):
    try:
        async with session.get(url, timeout=10) as resp:
            if resp.status == 200:
                return await resp.text(), resp.url
            return None, None
    except:
        return None, None

async def check_blog(session, blog_id):
    url = f"https://kovaze.com/blog/{blog_id}"
    html, final_url = await fetch(session, url)
    if not html:
        return None
    
    # Skip if redirected to main /blogs page (deleted blog)
    if 'kovaze.com/blogs' in str(final_url) or 'LATEST BLOGS' in html:
        return None
    
    soup = BeautifulSoup(html, 'html.parser')
    
    # Find real clickable mushroom
    mushroom_link = soup.find('a', href='/events/mushrooms')
    if mushroom_link and '🍄' in mushroom_link.text.strip():
        parent_text = mushroom_link.parent.get_text() if mushroom_link.parent else ''
        if 'Games' not in parent_text and 'enrol' not in parent_text.lower():
            return url
    return None

async def hunt():
    print(f"🍄 Kovaze Mushroom Hunter v2 STARTED")
    print(f"   Scanning {START_ID}-{END_ID} every {CHECK_INTERVAL}s")
    print("   → Automatically SKIPS deleted blogs (/blogs redirect)")
    print("   → Only alerts on REAL clickable 🍄 in content")
    seen = set()
    
    while True:
        print(f"\n[{time.strftime('%H:%M:%S')}] Scanning {END_ID - START_ID + 1} blogs...")
        start_time = time.time()
        async with aiohttp.ClientSession() as session:
            tasks = [check_blog(session, i) for i in range(START_ID, END_ID + 1)]
            results = await asyncio.gather(*tasks)
        
        found = [url for url in results if url]
        
        if found:
            print(f"\n🍄🍄🍄 REAL MUSHROOM FOUND! ({len(found)} new)")
            for url in found:
                if url not in seen:
                    print(f"   → {url}  <--- OPEN THIS NOW & CLICK THE 🍄")
                    webbrowser.open(url)
                    print("\a" * 10)  # LOUD BEEPS
                    seen.add(url)
        else:
            print("   No real mushrooms this scan (deleted ones skipped).")
        
        elapsed = time.time() - start_time
        print(f"Scan done in {elapsed:.1f}s. Next scan in {CHECK_INTERVAL}s...")
        await asyncio.sleep(CHECK_INTERVAL)

if __name__ == "__main__":
    try:
        asyncio.run(hunt())
    except KeyboardInterrupt:
        print("\n🍄 Stopped. Good luck hunting!")