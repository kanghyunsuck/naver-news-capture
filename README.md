# naver-news-capture
import time
import requests
from playwright.sync_api import sync_playwright
from PIL import Image

# 로컬 테스트용 토큰 및 아이디
TELEGRAM_TOKEN = os.environ.get("TELEGRAM_TOKEN")
CHAT_ID = os.environ.get("CHAT_ID")

def capture_and_make_pdf():
    photo_paths = []
    
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        page = browser.new_page(viewport={"width": 1280, "height": 1080})
        
        print("네이버 뉴스스탠드 접속 중...")
        page.goto('https://newsstand.naver.com/')
        page.wait_for_timeout(5000) 
        
        box = page.evaluate("""() => {
            let el = document.elementFromPoint(window.innerWidth / 2, window.innerHeight / 2);
            while (el && el.tagName !== 'BODY') {
                let rect = el.getBoundingClientRect();
                if (rect.width > 850 && rect.width < 1100) {
                    return { x: rect.x, y: rect.y, width: rect.width, height: rect.height };
                }
                el = el.parentElement;
            }
            return null;
        }""")
        
        if not box:
            box = {"x": 175, "y": 155, "width": 930, "height": 590}

        for i in range(1, 53):
            screenshot_path = f'newsstand_{i}.png'
            print(f"[{i}/52] 번째 뉴스판 캡처 중...")
            
            page.screenshot(path=screenshot_path, clip=box)
            photo_paths.append(screenshot_path)
            
            try:
                click_x = box['x'] + box['width'] + 40
                click_y = box['y'] + (box['height'] / 2)
                page.mouse.click(click_x, click_y)
            except Exception as e:
                pass
                
            page.wait_for_timeout(1500)
            
        browser.close()
        
    # 52장의 사진을 하나의 PDF 파일로 묶기
    print("사진들을 하나의 PDF 문서로 병합하는 중...")
    images = [Image.open(f).convert('RGB') for f in photo_paths]
    pdf_path = 'today_newsstand.pdf'
    # 첫 번째 이미지에 나머지 51장의 이미지를 페이지별로 이어 붙여 저장합니다.
    images[0].save(pdf_path, save_all=True, append_images=images[1:])
    
    return pdf_path

def send_document(pdf_path):
    print("텔레그램으로 PDF 문서 전송 시작...")
    url = f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendDocument"
    
    with open(pdf_path, 'rb') as doc:
        payload = {"chat_id": CHAT_ID}
        files = {"document": doc}
        response = requests.post(url, data=payload, files=files)
        
        if response.status_code == 200:
            print("PDF 전송 완료! 텔레그램을 확인해보세요.")
        else:
            print(f"전송 실패: {response.text}")

if __name__ == "__main__":
    final_pdf = capture_and_make_pdf()
    send_document(final_pdf)
