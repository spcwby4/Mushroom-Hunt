# mushroom_hunter_noauto.py
# ONE FILE - NO AUTO-OPEN, NO AUTO-COLLECT
# Just prints clickable HTML links in a mini web server
# Open http://localhost:8000 in your browser and refresh!

import asyncio
import aiohttp
from bs4 import BeautifulSoup
import time
from http.server import HTTPServer, BaseHTTPRequestHandler
import threading

# CONFIG
START_ID = 1
END_ID = 25000
CHECK_INTERVAL = 45
PORT = 8000

found_links = []

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-type', 'text/html')
        self.end_headers()
        
        html = """
        <html><head><title>Kovaze Mushroom Hunt - Live Links</title>
        <meta http-equiv="refresh" content="30">
        <style>body{font-family:Arial;background:#111;color:#0f0;padding:20px;}
        a{color:#0f0;font-size:18px;margin:10px;display:block;}</style></head>
        <body><h1>🍄 REAL MUSHROOMS FOUND (click to collect)</h1>
        """
        if found_links:
            for link in found_links:
                html += f'<a href="{link}" target="_blank">{link} 🍄</a><br>'
        else:
            html += "<p>No mushrooms yet - scanning...</p>"
        html += f"<p>Last scan: {time.strftime('%H:%M:%S')}</p></body></html>"
        self.wfile.write(html.encode())

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
    if 'kovaze.com/blogs' in final_url or 'LATEST BLOGS' in html.upper():
        return None

    soup = BeautifulSoup(html, 'html.parser')
    for a in soup.find_all('a', href='/events/mushrooms'):
        if a.get_text(strip=True) == '🍄':
            if a.find_parent(string=lambda t: t and 'Games' in t):
                continue
            return url
    return None

async def hunt():
    global found_links
    print("🍄 MUSHROOM HUNTER - LINKS ONLY MODE")
    print(f"   Open http://localhost:{PORT} in your browser")
    print("   Page auto-refreshes every 30s")
    print("   Click any green link to open blog & collect manually\n")
    
    seen = set()

    while True:
        print(f"[{time.strftime('%H:%M:%S')}] Scanning {END_ID:,} blogs...")
        start = time.time()
        new_found = []
        
        async with aiohttp.ClientSession() as session:
            tasks = []
            for i in range(START_ID, END_ID + 1):
                tasks.append(check_blog(session, i))
                if len(tasks) >= 600:
                    results = await asyncio.gather(*tasks)
                    for url in results:
                        if url and url not in seen:
                            seen.add(url)
                            new_found.append(url)
                    tasks = []
            if tasks:
                results = await asyncio.gather(*tasks)
                for url in results:
                    if url and url not in seen:
                        seen.add(url)
                        new_found.append(url)

        if new_found:
            found_links = new_found + found_links[:50]  # keep latest
            print(f"🍄 {len(new_found)} NEW MUSHROOMS! Total active: {len(found_links)}")
            print("\a" * 10)  # loud beep
        else:
            print("   No new mushrooms this scan.")

        print(f"Scan done in {time.time()-start:.1f}s. Next in {CHECK_INTERVAL}s...\n")
        await asyncio.sleep(CHECK_INTERVAL)

if __name__ == "__main__":
    # Start web server
    server = HTTPServer(('localhost', PORT), Handler)
    threading.Thread(target=server.serve_forever, daemon=True).start()
    
    try:
        asyncio.run(hunt())
    except KeyboardInterrupt:
        print("\n🍄 Stopped. Refresh http://localhost:8000 to see links!")